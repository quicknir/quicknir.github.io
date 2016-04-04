---
layout: post
title: Joint Allocations in C++
---

This post was inspired by my recent view of <a href="https://youtu.be/TH9VCN6UkyQ?t=1h5m52s" target="_blank">this</a> YouTube. The video is by a gentleman by the name of Jon Blow, who's talking about a new language for game development. I'm not going to talk much about the video except to use some of the goals and code presented as a motivation for this post. In particular, let's consider the following piece of code from the talk (identical modulo whitespace):

{% highlight cpp linenos %}
// C++11, more-real version of Mesh.
// Very different from the slower/simpler version!

struct Mesh {
  void * memory_block = NULL;
  
  Vector3 * positions = NULL;
  int * indices = NULL;
  Vector2 *uvs = NULL;

  int num_indices = 0;
  int num_vertices = 0;
}

int positions_size = num_vertices * sizeof(positions[0]);
int indices_size = num_indices * sizeof(indices[0]);
int uvs_size = num_uvs * sizeof(indices[0]);
mesh->memory_block = new char[positions_size + indices_size + uvs_size];
mesh->positions = (Vector3 *)mesh->memory_block;
mesh->indices = (int *)(mesh->memory_block + positions_size);
mesh->uvs = (Vector2 *)(mesh->memory_block + positions_size + 
                                             indices_size);
};
{% endhighlight %}

The goal of all this is to make sure the three array members get their memory in a contiguous block resulting from a single allocation. I'm not a game developer, but approaching this as a general software engineer this code has several issues, the biggest one being the extremely bug prone code at the bottom. Aside from being bug prone, it's also not reusable. Let's see what we can do.

## Picking the Right Abstractions

Let's start with the members of the class. One of the first things you'll probably notice is that all of the data members are either raw pointers or integers, very low level types. Of course, as you dig down, eventually all C++ classes are made up of such primitives. The question is whether the gun is being jumped; should we be using these low level types in this particular class, or use classes that better represent what we need and leave the low level types to them? Conceptually, the Mesh class needs two types of members:
<ol>
	<li>Non-owning arrays</li>
	<li>A memory owner</li>
</ol>
In the original code, both these concepts are present. But the non-owning arrays are essentially split between two members each: a pointer to the start of the array, and an integer telling you how many elements there are. There's also a memory owner, memory_block. The choice of a raw void pointer for something that is supposed to express ownership is a bit of a surprising choice in C++11 code.

Let me briefly mention that rather than tackling this with one owner and multiple views, we could also take an allocator based route: multiple owning arrays, each with an allocator linked to the same memory pool. Each approach has its pros and cons. The allocator approach gives you greater flexibility than what is really needed here, and also involves some space overhead (each array will need a stateful allocator instance with a pointer back to the memory pool). One of my goals in approaching this is to keep it relatively close in spirit to the original solution, so I'll be sticking with owner-viewer solution.

## Non-owning Arrays
The bad news about these is that the standard library doesn't give you anything here (that I'm aware of). The good news is that writing a non-owning array is quite easy. We'll either need two pointers or a pointer and an integer; I'll be using two pointers but either approach is quite reasonable.

{% highlight cpp linenos %}
template <class T>
class ArrayView {
  static_assert(std::is_trivial<T>::value,
                "Error, T must be trivial type");
 public:

  ArrayView(char * begin, char * end)
      : m_begin(reinterpret_cast<T *>(begin))
      , m_end(reinterpret_cast<T *>(end))
  { }

  T & operator[](int64_t index) {
#ifdef ARRAY_VIEW_BOUNDS_CHECKING
    assert(m_begin != nullptr && index >= 0 &&
           index < (m_end-m_begin));
#endif
    return m_begin[index];
  }

  int64_t size() const { return m_end - m_begin; }
  int64_t memory_size() const {
    return sizeof(T) * size();
  }
  // ... More methods, const operator[], begin, end, etc
 private:
  T * m_begin = nullptr;
  T * m_end = nullptr;
};
{% endhighlight %}

There are a few things worth noting here.

First, this is a very thin wrapper, as you can see. It takes a block of memory, and provides a typed array interface to it. It doesn't zero elements without being asked. It doesn't call any constructors of its elements; in fact I disallow non-trivial types to make sure it doesn't do something dangerous. 

Second, we are able to implement optional bounds checking here. I feel like this is worth mentioning because it was discussed in the original talk as something desirable that wasn't in the C++ solution. With raw pointers, you can't get bounds checking because a) operator[] is built in and you can't add the check, and b) a pointer serving as an array does not know its size. We solve both issues by using this thin wrapper.

Third, this solution will make Mesh slightly larger than it was before, both because pointers are larger than int on most architectures today, and because we are storing the size of each array independently whereas one of the integers in Mesh does double duty. This assumed to not be a big deal, after all even the two ints stored by Mesh are redundant (they can be calculated by pointer subtraction).

## A Memory Owner

This part is easier: to express ownership of heap memory, we'll use a `unique_ptr`. And since we want to own some arbitrary number of bytes, we'll use a `unique_ptr` to `char[]`. This still isn't a trivial decision; notice that `unique_ptr` cannot be copied. We might prefer a variant of `unique_ptr` that would copy the underlying memory. However, because the members of Mesh are intertwined, we will anyway need to write out copy members for it if we want them.

## Generic Joint Allocation

We started by deciding what our types should be so that we would know how to write our generic joint allocator. These types will be part of the signature of our function, so understanding what they would be was a necessary first step. In order to do this generically, we'll need to use a variadic function, which generally means using recursion. Let's start by thinking about how we'd want to use it:

{% highlight cpp linenos %}
ArrayView<int> iview;
ArrayView<double> dview;
// Allocates space for 3 ints, 2 doubles, sets views accordingly
unique_ptr<char[]> memory_block = make_contiguous(iview, 3, dview, 2);
{% endhighlight %}

The function will take in the ArrayViews by reference and modify them appropriately; this simultaneously gives our function knowledge of the types. The only other information needed is how large each array needs to be. It returns the unique_ptr that owns the memory.

The meat of the work will consist of computing offsets and recursively passing them down to the next step. Note that the amount of memory needed will not be known until the base case. After the memory block is received from the recursive call, the ArrayView can be modified. So we have:

{% highlight cpp linenos %}
template <class T, class ... Ts>
std::unique_ptr<char[]> make_contiguous_helper(int64_t offset,
                                               ArrayView<T> & vv,
                                               int64_t size,
                                               Ts && ... args)
{
  auto align_req = std::alignment_of<T>::value;
  int64_t aligned_offset =
      ((offset+align_req-1) / align_req) * align_req;

  int64_t end_offset = aligned_offset + size * sizeof(T);
  auto memory = make_contiguous_helper(end_offset, args...);
  vv = ArrayView<T>(memory.get() + aligned_offset,
                    memory.get() + end_offset);
  return memory;
}
{% endhighlight %}

Note that the first two lines just handle alignment, which was not handled in the original code. All that's left now is to write the base case and the interface function:

{% highlight cpp linenos %}
std::unique_ptr<char[]> make_contiguous_helper(int64_t total_size) {
  return std::unique_ptr<char[]>(new char[total_size]);
}

template <class ... Ts>
std::unique_ptr<char[]> make_contiguous(Ts && ... args) {
  return make_contiguous_helper(0, args...);
}
{% endhighlight %}

The result is barely longer than the original code and completely generic.

## The New Mesh

Here's what Mesh looks like now:

{% highlight cpp linenos %}
struct Mesh {  
  ArrayView<Vector3> positions;
  ArrayView<int> indices;
  ArrayView<Vector2> uvs;

  std::unique_ptr<char[]> memory_block = nullptr;


  Mesh(int num_vertices, int num_indices)
      : memory_block(make_contiguous(positions, num_vertices,
                                     indices, num_indices,
                                     uvs, num_vertices))
  { }

};
{% endhighlight %}

Notice a minor subtlety: because of how our function is structured, we need the ArrayViews to be initialized before we can initialize the memory_block. So I switched the order of the member variables. Of course, you could also just let the memory_block be default initialized and do the work in the body instead.

It's quite possible you can call it a day here, but there is one more possible concern: what about copying and moving? As it turns out, moving just works out of the box. Unfortunately, if you want copying, you need to write it. This is unavoidable, because of how Mesh is structured, it is a resource handling class that cannot fully delegate the work to its members. So, we write:

{% highlight cpp linenos %}
struct Mesh {  
 ... // as before

  Mesh(Mesh &&) = default;
  Mesh & operator=(Mesh &&) = default;

  Mesh(const Mesh & other)
      : Mesh(other.positions.size(), other.indices.size())
  {
    ::memcpy(memory_block.get(), other.memory_block.get(),
             other.memory_size());
  }

  Mesh & operator=(const Mesh & other) {
    if (this == &other) return *this;

    if (other.positions.size() != positions.size() ||
        other.indices.size() != indices.size()) {
      memory_block = 
          make_contiguous(positions, other.positions.size(),
                          indices, other.indices.size(),
                          uvs, other.uvs.size());
    }
    ::memcpy(memory_block.get(), other.memory_block.get(),
             other.memory_size());
    return *this;
  }

  int64_t memory_size() const {
  return positions.memory_size() + indices.memory_size() +
      uvs.memory_size();
  }
};
{% endhighlight %}

## Closing (for now) Thoughts

Lately I've given some thought to the approach that languages take with regards to making certain things "first class". In other words, language features that cannot be implemented by the user; things that are part of the core language. This is a good example: the solution suggested in the video is to have the new language provide a keyword "@joint". Is this desirable?

On the one hand, it makes life easier by providing specialized syntax, doing the right thing easily and likely efficiently as well. On the other hand, it limits your options: if you need something similar to this but with a slight tweak, your odds of being able to do it are much better if you could in principle implement the original feature on your own. It also makes it possible that the language could become a mess of keywords addressing specific needs instead of a framework in which solutions can be constructed. Obviously, care is needed in expanding your core language.

What I've shown so far is a pretty simple and generic solution to joint allocation. However, this doesn't really solve the problem of a generic joint allocated data structure in a satisfactory way. Too much boilerplate, especially the copy methods, would have to be rewritten in order to have another class similar to Mesh but with different arrays that needed to be kept contiguous. This is what we'll tackle next time, and hopefully see that joint allocation is a problem that can be quite reasonably solved in C++ -- even without specialized keywords.
