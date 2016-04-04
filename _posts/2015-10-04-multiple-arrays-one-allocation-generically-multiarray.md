---
layout: post
title: "Multiple Arrays, One Allocation, Generically: MultiArray"
---

In last week's <a href="https://turingtester.wordpress.com/2015/09/13/joint-allocations-in-c/">post</a>, I discussed how one could simplify the task of allocating memory for several arrays simultaneously; a joint allocation. This function was nice in that it automatically performed a nice amount of pointer arithmetic. What was not so nice was the fact that the result was not very neatly wrapped up: calling the function resulted in an owning unique_ptr to the memory, and several ArrayViews which provided type safe (and optionally bounds checked) access and iteration to each array. These objects are tightly coupled, and it is impossible to write copy constructors for each of them individually that give us the correct behavior for them collectively. This means that a user would need to deal with all this themselves.

I realized as I was writing the last post that the only way to solve this problem and make the user side truly painless was to wrap up the data members into a single class, and handle all the construction and move techniques there so the user wouldn't have to. This would also potentially enable certain space optimizations. In this post, I will sketch out the implementation of this single class, MultiArray.

## Design Considerations
We have a pretty good sense of what interface we want for MultiArray: we'd like to be able to construct it by specifying a list of types and a list of sizes of the respective arrays. We'd like to be able to copy it and move it. And we'd like to be able to access the data to do the sort of things that we normally do with arrays. We'll also restrict ourselves to arrays of trivial types, as before.

Since this started as a discussion of how to encapsulate the unique_ptr + ArrayViews that constituted our previous implementation, this is a reasonable starting point for thinking about what our data members should be. There's a bit of potentially wasted space here: modulo alignment issues, the end pointer of one ArrayView should be the same as the beginning of the next. We could replace our N ArrayViews (where N is the number of distinct arrays in the MultiArray) with N pointers or N integer offsets. I chose N integer offsets, mostly just for variety. The disadvantage is that it adds a bit of extra work to extracting a pointer. The advantages are that it simplifies the implementation, and it enables easy space optimizations for MultiArray.

There's a number of things I won't address due to brevity. One of them is to properly deal with alignment issues; informing the user in some way that their arrays will be misaligned. Another is that MultiArray should properly be templated on an allocator like STL data structures; this would allow you to use whatever combination of joint data structures and custom allocators (suggested by some as an alternate solution to the original problem) you'd like.

## The Basics
Let's start by defining the class, and some of the things we'll need to write functions:

{% highlight cpp linenos %}
template <class ... Args>
class MultiArray {

 private:

  static constexpr int s_num_arrays = sizeof...(Args);
  std::array<int32_t, s_num_arrays> m_offsets;
  std::unique_ptr<char[]> m_memory;
};

{% endhighlight %}

We can see that MultiArray is variadically templated due to the use of ... inside the template declaration. This means it can accept a variable number of template parameters, which will allow it to be generic over both the number and type of arrays. Inside our class, we use the handy sizeof... operator. This extracts the number of template arguments in a template parameter pack; for us this is the number of arrays.

I happen to pick int32_t as the integer to the store the offsets. This is to demonstrate the point of optimizing space, though naturally it limits the size of the array. You could use a different integer, or just template it instead.

## Construction and Copying
Let's start with the constructor. We could template the constructor on a varying number of integers and ensure that we have the correct number, but this is rather complicated. Instead, we'll simply accept an array of sizes; because arrays can be constructed implicitly from initializer lists the syntax on the client side will still be nice.

{% highlight cpp linenos %}
  MultiArray(const std::array<int32_t, s_num_arrays> & sizes)
  {
    constexpr std::array<int32_t, s_num_arrays> type_sizes =
        {sizeof(Args)...};

    int32_t partial_sum = 0;
    for (std::size_t i = 0; i != s_num_arrays; ++i) {
      partial_sum += sizes[i] * type_sizes[i];
      m_offsets[i] = partial_sum;
    }

    m_memory.reset(new char[totalMemory()]);
  }
{% endhighlight %}

Notice how we neatly avoid needing to use any recursion at all. The sizes of the types are computed in a one liner with some judicious use of the unpack operator (and at compile time). After that, we can just do simple loop iteration. In this calculation, each integer will be the offset in bytes of the end of the corresponding array from the start of the memory block. For this reason, even if I was implementing MultiArray with pointers rather than integers, I would use an array of char* pointers rather than a tuple of typed pointers. One downside of this solution you should be aware of is that the user can specify fewer sizes then there are types; sadly such an initializer list will silently be promoted to the larger array and cannot be detected in this function (although at least the remaining sizes are zero'ed).

{% highlight cpp linenos %}
  MultiArray(const MultiArray & other)
    : m_offsets(other.m_offsets)
    , m_memory(new char[other.totalMemory()])
  {
    std::memcpy(m_memory.get(), other.m_memory.get(), totalMemory());
  }

  MultiArray& operator=(const MultiArray & other) {
    if (this == &other) {
      return *this;
    }
    if (totalMemory() != other.totalMemory()) {
      m_memory.reset(new char[other.totalMemory()]);
    }
    m_offsets = other.m_offsets;
    std::memcpy(m_memory.get(), other.m_memory.get(), totalMemory());
    return *this;
  }

  int32_t totalMemory() const {
    return m_offsets.back();
  }
{% endhighlight %}

The nice thing about using integer offsets, seen above, is that they're independent of where the data is actually located. That means that when we copy, we can just copy the offsets themselves, making life quite simple. We allocate some memory for the pointer, and do the memcpy. The totalMemory utility function returns the integer offset for the end of the last array, which is the same as the total size of the memory block. Note that moving will just work (we just need to =default). Edit: I added a check for self assignment in the copy assignment operator. Without that check, self assignment results in UB at the memcpy because the ranges overlap. Thanks to Stephan T. Lavavej for pointing this out. 

## Access
This is going to be the trickiest part. There's a fair bit of dancing around language issues here, so I'll break it up into two parts: the interface, and the implementation. On the interface side, we'll want to pass an integer to specify which of the arrays we want an ArrayView to. Because the return type will actually be different based on which integer is passed, the integer will need to be passed as a non-type template parameter, similar to std::get for std::tuple. We'll make it a non-member function for consistency with get and tuple (a bit more on this later). Let's also define a helper typedef to get the correct return type. So far then, we have:

{% highlight cpp linenos %}
template <int I, class ... T>
using MultiArrayViewType = ArrayView<
    std::tuple_element_t<I, std::tuple<T...>>>;

template <int I, class ... Ts>
MultiArrayViewType<I, Ts...> getView(MultiArray<Ts...> & ja)
{
  // ...
}
{% endhighlight %}

Now, we run into another issue. To extract the ith ArrayView, we'll need the beginning and end offsets. In general, the end offset is the ith entry in m_offsets, and the beginning offset is simply the end of the previous array, so the i-1 entry. However, this clearly doesn't work for the 0th array. This would naturally be represented by partially specializing getView, but that is not allowed. So, we defer the implementation to a private functor:

{% highlight cpp linenos %}
template &<class ... Args&>
class MultiArray {
  // ..
  template &<int I, class ... Ts&>
  friend MultiArrayViewType&<I, Ts...&> getView(MultiArray&<Ts...&> && ja);

 private:

  template &<int I, int dummy = 0&>
  struct getViewHelper {
    auto operator() (MultiArray&<Args...&> ja) {
      return MultiArrayViewType&<I, Args...&>(ja.m_memory.get() + ja.m_offsets[I-1],
                                            ja.m_memory.get() + ja.m_offsets[I]);
    }
  };

  template &<int dummy&>
  struct getViewHelper&<0, dummy&> {
    auto operator() (MultiArray&<Args...&> ja) {
      return MultiArrayViewType&<0, Args...&>(ja.m_memory.get(),
                                            ja.m_memory.get() + ja.m_offsets[0]);
    }
  };
}
{% endhighlight %}

This code isn't beautiful to say the least, but it's not hard to understand either. There is one interesting caveat: nested class templates of class templates cannot be totally specialized while the outer class is unspecialized. So I added the dummy template parameter to prevent it from getting totally specialized. This is ugly, but acceptable since it isn't user facing. The ugly nail in the ugly coffin is finishing getView itself:

{% highlight cpp linenos %}
template &<int I, class ... Ts&>
MultiArrayViewType&<I, Ts...&> getView(MultiArray&<Ts...&> && ja)
{
  using HelperType = typename MultiArray&<Ts...&>::
      template getViewHelper&<I&>;
  return HelperType()(ja);
}
{% endhighlight %}

If you've never seen template used that way before, don't feel bad. It can occur in situations where there are nested templates of templates. Actually, avoiding that syntax is the motivation for making std::get for std::tuple a non-member. For more information about this usage of keyword template, and how it affected the design of tuple, see <a href="http://stackoverflow.com/questions/610245/where-and-why-do-i-have-to-put-the-template-and-typename-keywords">here</a> and <a href="http://stackoverflow.com/questions/3313479/stdtuple-get-member-function">here</a> respectively.

Edit: there is actually a simpler implementation here that I missed, thanks to Stephan T. Lavavej  for pointing it out:

{% highlight cpp linenos %}
template &<int I, class ... Ts&>
MultiArrayViewType&<I, Ts...&> getView(MultiArray&<Ts...&> && ja)
{
return MultiArrayViewType&<I, Ts...&>(ja.m_memory.get() + (I == 0 ? 0 : ja.m_offsets[I-1]),     
                                    ja.m_memory.get() + ja.m_offsets[I]);
}
{% endhighlight %}

## Back to the User
That got fairly intense, but the benefits of all this are quite substantial. Here's how we would write the Mesh class from before:

{% highlight cpp linenos %}
class Mesh {
  MultiArray<Vector3, int, Vector2> data;
 public:

  Mesh(int num_vertices, int num_indices)
      : data({num_vertices, num_indices, num_vertices})
  { }

  ArrayView&<Vector3> positions() {
    return getView<0>(data);
  }
  ArrayView&<int> indices() {
    return getView<1>(data);
  }
  ArrayView&<Vector2> uvs() {
    return getView<2>(data);
  }
};

// Using Mesh is fun
Mesh m(3, 2);

for (const auto x : m.positions()) {
  std::cerr << x;
}
{% endhighlight %}

The user code is now concerned almost entirely with meaningful things, and not with boilerplate. The code above consists entirely of specifying the types, sizes, and names of the multiple arrays in the Mesh. Adding or removing arrays from the Mesh is now trivial. Writing a new class that uses jointly allocated arrays is trivial. Go back to previous versions of the user code, with pointer arithmetic to calculate offsets, or long copy constructors. Now imagine modifying all that code every time you want to change Mesh. Or imagine rewriting it every time you want a new class with jointly allocated arrays.

## Summing Up
We've jumped through some variadic hoops in the name of making this code highly generic, and keeping the user code (that is, the code that deals with a specific instance to be solved) as clean as possible. Is it worth this effort to make code general? The answer obviously depends on the details of your situation: how many times you plan to solve the problem, the expected longevity of your codebase, the experience of your team, etc. In general though I lean towards yes. There's a passage from Alexander Stepanov (in <a href="http://www.cs.rpi.edu/~musser/gsd/notes-on-programming-2006-10-13.pdf">this</a> document) that I really like:
<blockquote>The significant thing was that making interfaces general – even if I did not quite know what it meant – I made them much more robust. The changes in the surrounding code or changes in the grammar of the input language did not affect the general functions: 95% of the code was impervious to change. In other words: decomposing an application into a collection of general purpose algorithms and data structures makes it robust.</blockquote>
I hope you've enjoyed reading this post. If you are interested in a fully fleshed out version of MultiArray for your project, please let me know (you can comment below or email me at quicknir@gmail.com); if there's enough interest I'll work out the details and post it to GitHub under a liberal license.
