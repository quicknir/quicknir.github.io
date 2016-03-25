---
layout: post
title: "Writing Good C++ By Default, in the STL"
---

If you haven't seen Herb Sutter's recent <a href="https://www.youtube.com/watch?v=hEx5DNLWGgA" target="_blank">talk</a> at CppCon 2015 "Writing Good C++... by Default", you should. It's a great talk that largely discusses static analysis and various types of safety. In particular, he discusses lifetime safety: the risk that objects, particularly non-owning views such as references, pointers, and iterators, become invalidated by other operations. One of his slides got me thinking back to the topic of rvalue overloads and their return types, a topic I've written about in the <a href="https://turingtester.wordpress.com/2015/07/05/object-theft-etiquette-in-c-methods-with-a-side-of/" target="_blank">past</a>.

I'd like to apply some of the lessons of that talk to the STL itself, and in particular look at how some of the STL treatment of return types and value categories makes it easier than it should be to end up with dangling references and undefined behavior. I'll also talk about how you can avoid these situations in your own code.

## Hand it over, unique_ptr
My thoughts on this topic got kicked off at <a href="https://youtu.be/hEx5DNLWGgA?t=45m51s" target="_blank">this</a> point in the talk, where Herb shows the following code:

{% highlight cpp linenos %}
unique_ptr<A> myFun()
{
  unique_ptr<A> pa(new A());
  return pa;
}

const A& rA = *myFun();
{% endhighlight %}

This is from an actual StackOverflow <a href="http://stackoverflow.com/questions/30858850/dereferencing-a-temporary-unique-ptr" target="_blank">question</a>; the code above results in rA dangling. You can see why this would be surprising to someone; const references are typically safe because of lifetime extension of temporaries. They're also my default choice when I need const-only access to a non-primitive type, as they (almost) always work, and avoid unnecessary copying when possible. In this case, lifetime extension doesn't kick in because rA binds to the A that the unique_ptr owns, not the unique_ptr itself. The unique_ptr doesn't get its lifetime extended, gets destroyed, destroying its A in the process and causing rA to dangle.

Herb goes on to talk about how static analysis can catch this (and even gives a demo!). This is great stuff, as clearly it's easy to write code that has the demonstrated gotchas for the users. However, it's also worth asking: is it possible to write the code in such a way that this problem can't occur in the first place? After all, good interfaces are easy to use correctly and hard to use incorrectly. Here's how operator* for unique_ptr looks in the standard library that ships with gcc 4.9:

{% highlight cpp linenos %}
typename add_lvalue_reference<element_type>::type
operator*() const
{
_GLIBCXX_DEBUG_ASSERT(get() != pointer());
return *get();
}
{% endhighlight %}

This is returning a reference to something owned by the unique_ptr. Normally this is fine, but if the unique_ptr is a temporary, then it's passing a reference that's about to become invalidated. The best case scenario is that the user binds the return of * to a value rather than a reference. This will trigger the construction of a new object, and there will be no reference to dangle. But even this is sub-optimal, as it will be a copy construction, not a move construction. Both of these issues go away if we agree that when calling a method of a temporary object, it should not return a reference to an owned member of the temporary. In this case, we could change `operator*` as follows:

{% highlight cpp linenos %}
typename add_lvalue_reference<element_type>::type
operator*() const &;
{
_GLIBCXX_DEBUG_ASSERT(get() != pointer());
return *get();
}

element_type
operator*() const &&;
{
_GLIBCXX_DEBUG_ASSERT(get() != pointer());
return std::move(*get());
}
{% endhighlight %}

Now, unique_ptr's that are about to be destroyed return their contents by value instead. This would have prevented the original poster's problem, and would cause move construction rather than copy construction if they had declared a value type instead of a reference type. However, it introduces a new problem (can you see what it is?). I'll loop back to unique_ptr at the end, meanwhile I'll look at several simpler examples.

## Pointer, where'd you come from?
Later on, Herb <a href="https://youtu.be/hEx5DNLWGgA?t=1h16m56s" target="_blank">continues</a> by talking about functions that return pointers. Herb points out that such pointers are surprisingly constrained. They are raw pointers, and therefore they do not own anything, so they must point to an object owned by someone else. The contents of the function, including any arguments passed by value but excluding static variables, are destroyed on exit. So other than static (and global) variables, the only possibilities are pointer arguments, and arguments passed by reference, i.e. the owner is in the calling scope, which makes sense. He also points out that arguments passed by rvalue reference are not considered, because such arguments attract temporaries. With that in mind, let's consider the following function:

{% highlight cpp linenos %}
template< std::size_t I, class... Types >
constexpr std::tuple_element_t<I, tuple<Types...> >&&
    get( tuple<Types...>&& t );
{% endhighlight %}

This function is in the C++14 standard; it extracts individual values from a tuple. References, be they lvalue or rvalue, are ultimately non-owning views of other objects, just like pointers. So the same rules of lifetime safety apply. By the very rules discussed by Herb Sutter, there is no safe owner for the output of this function, and the static analyzer would complain.

The reason (I speculate) that code like this gets written is that while in principle references may just be dressed up pointers, that dressing up is significant to practical usage. In particular, references when assigned to non-reference types quietly get copied or moved, avoiding dangling. Even if type inference via auto is used, the non-reference type is deduced, still avoiding dangling. These practical considerations mean that it's easier to get away with returning a dangerous reference than a dangerous pointer. But it's not fundamentally different.

Is there an easy solution here? Well, as it turns out, there is: just make the rvalue reference overload of get return by value. All ownership onerousness is obviated when output is its own owner (say that ten times fast):

{% highlight cpp linenos %}
MyClass f(OtherClass&&);
// Safe, lifetime extension
const auto & var1 = f(OtherClass{});
// Performs move construction, same as if ref were returned
auto var2 = f(OtherClass{});
{% endhighlight %}

One potential concern is performance; is it possible that `var2` would be cheaper to create if f returned an rvalue reference? In this particular case, it's unlikely to make a difference. Because of return value optimization (RVO), it's likely that creating `var2` will involve precisely one move construction regardless. However, it is true that when multiple consecutive function calls are involved, returning rvalue references in some cases can avoid move constructors entirely in favor of reference passing. Consider [this](http://coliru.stacked-crooked.com/a/ccfc69ed21d6b7fa) example, which shows both single and nested function calls. This doesn't seem like a performance hit that's likely to typically be relevant very often, though I'd be curious to see some real life counter-examples. In any case it doesn't seem like a worthwhile trade-off for safety, but I'm open to seeing common use cases where it does make a big difference.

## Safe Iteration Optional
Let's consider another example: Boost optional. This is not in the standard yet, but is in some proposal state (possibly accepted?). You may be able to use it without even bringing in Boost by looking in namespace `std::experimental`. If you aren't familiar with it, I'll briefly summarize: it optionally contains a single object. You can check whether an object is contained, or nothing. You can also call a method that will return the object if available, and throw otherwise; this method is called value. Its various signatures look like this (the `std::experimental` version):

{% highlight cpp linenos %}
constexpr T& value() &;
constexpr const T& value() const &;
constexpr T&& value() &&;
constexpr const T&& value() const &&;
{% endhighlight %}

T is the template type parameter of the class, i.e. the type of the optionally contained object.

If we go back to Herb Sutter's reasoning, we ask: how can we be returning these references, these non-owning views? In this case, our function doesn't explicitly take any arguments at all. Since they're instance methods, they always receive a pointer to an instance of the object. But the latter two overloads only get called when the instance itself they are being called on is an rvalue, that is it's about to be destructed. When the object is destroyed, it will destroy the internally held T object, and the returned reference will dangle:

{% highlight cpp linenos %}
optional<MyClass> func();
const auto & MyClass = func().value(); // dangling reference!
{% endhighlight %}

This is definitely code that I could see myself writing. If that's not persuasive, how about this?

{% highlight cpp linenos %}
optional<std::vector<double>> func2();
// I just want to iterate over func2's result if it exists, throw
// otherwise. How would I do that? Probably like this:
for (auto i : func2().value()) {
  // use i
}
{% endhighlight %}

The returned vector (assuming it's present in the optional) will be destroyed before iteration starts. That's because ranged for loop evaluates what's to the right of the ':' and assigns it to a variable that's auto &&. This is a very reasonable approach in generic programming: it binds to everything and avoids copying. But here, it breaks.

Here's a demonstration of these issues with optional (I use the Boost version): http://coliru.stacked-crooked.com/a/bcb85b71080ff389. In the first example, you can see that A's destructor gets called twice before hitting the next line; the first destruction is from returning from inside the function (and is ok), the second is from the returned object outside the function (and is not). In the second case, our iteration over the vector produces different output from the vector we returned; it's actually undefined behavior so it could have ordered us a pizza.

This can be fixed just like we can fix tuple: by making the rvalue overload methods return by value.

## Back to unique_ptr
Did you find the problem with our solution for unique_ptr yet? It's the joy of polymorphic objects: slicing. If you hold an object by pointer (or reference), it's simply not safe to manipulate it by value. So what now, do we throw in the towel?

No! Give me safety, or give me death. If returning a reference is unsafe, and return a value is unsafe, then the method is unsafe. We can solve this problem in another way (standardese omitted):

{% highlight cpp linenos %}
T& operator*() const & { ... }
T& operator*() const && = delete;
{% endhighlight %}

This simply prevents any code that calls `operator*` on an rvalue unique_ptr from compiling in the first place. Code that does not compile cannot cause dangling references!

Interesting fact: it is actually necessary to = delete the rvalue reference version; not defining it is insufficient. Why? Well, the const and value category qualifiers on methods follow similar rules to function binding. Namely, the const lvalue reference method can be called by an rvalue object.

## Lesson Learned
It may not be possible to fix things at this point in the STL, but you can absorb the lesson in your own coding. Any time you write a method for a class that returns internal references, think about what you want to happen when the class is an rvalue. If you're not sure, just = delete the rvalue overloads (delete both the & and const && overloads). This will force people to first assign the object, keeping it alive in order to call the method in question. A narrow interface is generally a better starting point; widening an interface can always be done later on and is not a breaking change. Alternately, if you're confident it yields correct behavior, you can return by value. Keep in mind that in order to do this efficiently, you'll likely need to call std::move as you return.

Happy coding, and may your lifetimes be exactly as long as they should be.
