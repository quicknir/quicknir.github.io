---
layout: post
title: C++'s Rule of Zero
---

Some of you probably already know about the Rule of Zero. More of you have probably heard of the Rule of 3 (pre C++ 11) or the rule of 5. Let's start by reviewing what the Rule of Zero is.

## What is the Rule of Zero?

The idea behind it is as follows: classes should not define any of the special functions (copy/move constructors/assignment, and the destructor) unless they are classes dedicated to resource management. It's described in <a href="http://flamingdangerzone.com/cxx11/rule-of-zero/">this</a> blog post. There are a couple of good reasons for this, one more conceptual, and one more practical. The conceptual reason stems from the single responsibility principle, the idea that a class should be responsible for one thing. If you make a class responsible for more than one thing, then you have rather tightly coupled the implementation and interface of two separate things. Your class' job is to orchestrate the state of its member variables to provide some combined state, and interface to that state. When you write special member functions, you are basically picking up the pieces left behind by members that don't manage their resource in quite the way you want. The right thing to do is to make sure you are using the right members. This leads us directly to the practical reason: in c++, a class can't choose for which members it will redefine the special functions. What do I mean? Well, if you don't write the special functions, the compiler generates them for you by trying to apply the expected operation on a member by member basis. So if you use the default move constructor, the compiler will generate a move constructor that simply attempts to move construct all data members (and base classes). As soon as you write a custom move constructor that does some special handling for one data member, you now have to write code to deal with every other member, even if you just need to change the one. This may not seem like a big deal, after all, what's one line of code per variable, per custom special function? The real problem isn't so much having to write it, as it is that the compiler won't enforce it and it's easy to forget. Consider this code:

{% highlight cpp linenos %}
class Example {
 public:
  Example(const Example & other) : m_double(other.m_double) { }

 private:
  double m_double = 0.0;
  bool m_bool = false;
};
{% endhighlight %}

You won't get a warning in your IDE or in any sanitizer for this, because m_bool will deterministically get set to false by virtue of its inline initialization, and the fact that it does not get mentioned in the constructor initialization list. But this is almost certainly a bug, in that whenever you copy an Example object, the new copy will have m_bool = false regardless of the source of the copy. Something like this is very easy to introduce, and could be very hard to dig up afterwards.

## Why people break the rule
Of course, people won't generally write code like the example above, because doubles and booleans are simple, and you are nearly guaranteed to be happy with their default resource semantics. So let's take a very common real case: pointers. It's quite common for a class to have pointers. Of course, we're savvy modern c++11 programmers, so we'll use a unique_ptr, preventing any possible leaks. The problem is that while unique_ptr provides the correct destructor, unless we want the class owning the unique_ptr to be uncopyable, it doesn't do what we want for the copy constructor/assignment. Let's suppose for a moment that Example wants to own a non-polymorphic class Pointee by pointer, we end up with code that looks like this:

{% highlight cpp linenos %}
class Example {
 public:
  Example(const Example & other) 
      : m_pointer(make_unique<Pointee>(*other.m_pointer))
      , m_bool(other.m_bool) 
  { }

  // Won't be default generated unless you add this
  Example(Example &&) = default;
  
  // similar code for copy/move assignment

 private:
  unique_ptr<Pointee> m_pointer;
  bool m_bool = false;
};
{% endhighlight %}

Code along these lines is very common in the wild, and it's not optimal. In a larger class, every modification that adds or removes variables will be vulnerable to introducing a bug into the copy/move semantics. Can we do better?

## How to follow the rule
The conclusion from the above code is that unique_ptr doesn't actually represent the correct resource management for our class. So rather than making this the issue of the entire Example class, including all its other members, we should fix the problem at the root. Let's write a class that has the proper resource semantics.

{% highlight cpp linenos %}
template<class T>
class DeepCopyPointer {
 public:
  DeepCopyPointer(const DeepCopyPointer & other) 
      : m_pointer(make_unique<T>(*other.m_pointer))
  { }

  DeepCopyPointer(DeepCopyPointer &&) = default;
  
  // similar code for copy/move assignment

 private:
  unique_ptr<T> m_pointer;
};
{% endhighlight %}

This looks better, right? Well, there's a bit of a problem: this class doesn't actually provide any of the interface that a unique_ptr does. We could make Example a friend and have it access the unique_ptr directly, but this is ugly. Another problem is that this code is pretty specialized, it's just for unique_ptr, it's only for non-polymorphic objects: if we are storing a base class pointer and the object's type is different, this copy won't work properly. We'd probably want to copy by calling a clone method or similar.

## A better solution by breaking another rule
Let's do something more generic. Let's write a class that mimics another class, but allows you to change its copy behavior. To mimic that other class, we're going to inherit from it. Now, we are interested in mimicking unique_ptr, isn't it bad to inherit from it? People often say that you shouldn't inherit from types like unique_ptr or vector, because they don't have virtual functions, and especially destructors. The answer is that it's not bad to inherit, <em>as long as you never use it polymorphically</em>. With that in mind:

{% highlight cpp linenos %}
template <class T, class F>
class copyer : public T {
 public:
  copyer(T && t)
      : T(std::forward(t)) { };

  copyer(copyer && other) = default;
  copyer & operator=(copyer && other) = default;

  copyer(const copyer & other)
      : T(F()(other)) { }

  copyer & operator=(const copyer & other) {
    std::swap(*this, copyer(other));
    return *this;
  }
};
{% endhighlight %}

T is the class to be mimicked, and F is a functor that will help us redefine the copying behavior. Note that we are going with the defaults for move construction/assignment, we assume that we like T's defaults for move construction/assignment. I chose to do this because it's quite a bit trickier to write this code generically if T is not moveable (or not moveable in a way we like). To get the deep copy unique_ptr we want, we just do this:

{% highlight cpp linenos %}
template <class T>
struct F {
   std::unique_ptr<T> operator()(const std::unique_ptr<T> & other) {
    return std::make_unique<T>(*other);
  }
};

template <class T>
using copying_ptr = copyer<std::unique_ptr<T>, F<T>>;
{% endhighlight %}

We can now write Example like this:

{% highlight cpp linenos %}
class Example {
 
 private:
  copying_ptr<Pointee> m_pointer;
  bool m_bool = false;
};
{% endhighlight %}

That's it, we get all the special functions for free now. And we don't have to worry about adding bugs to Example each time we add/remove variables. We can also easily reuse copyer, if Example wanted to alter copy semantics in a different way, it could just define a private nested struct and use that in templating copyer instead of F.

## In summary
Don't start writing out big special functions for your complex classes, just because a couple of members don't have the right default behavior. Use techniques like this one to instead have members that do have the default behavior; your code will be cleaner and less bug prone. 

<strong>Edit</strong>: Here's a brief follow up on exception safety, inspired by some excellent comments by Ross Smith (CaptainCrowbar) on [Reddit](https://www.reddit.com/r/cpp/comments/3bccp5/c_rule_of_zero_why_it_matters_and_a_nifty).

## Strong Exception Guarantee
There is one place where the default generated special functions fall short. The strong exception guarantee says that if a function fails by throwing, then the state of the program is the same as before the function was called. For constructors, this is automatically upheld provided each member has a proper destructor; an exception thrown in the constructor body means that the partially constructed object will be destroyed. Move assignments are moves, and it's extremely common and desirable for types to have noexcept move operations. So copy assignment is the odd man out. The default copy assignment will try to copy assign each member in sequence. For example:

{% highlight cpp linenos %}
class Example {
 
 private:
  std::vector<int> m_vec1;
  std::vector<int> m_vec2;
};

Example e1;
Example e2;
....
e2 = e1;
{% endhighlight %}

What could happen here is that `e1.m_vec1` will get copied into `e2.m_vec1`, then the copy of `e1.m_vec2` into `e2.m_vec2` throws. Now, `e2` is in a half copied state, and the strong exception guarantee is broken. So here's how we fix it:

{% highlight cpp linenos %}
class Example {
 public:
  Example(const Example &) = default;
  Example(Example &&) = default;
  Example & operator=(Example &&) = default;
  
  Example & operator=(const Example & other ) { 
    std::swap(*this, Example(other)); 
    return *this;
  }
 
 private:
  std::vector<int> m_vec1;
  std::vector<int> m_vec2;
};
{% endhighlight %}

I simply use `std::swap` instead of the using namespace std technique, because I know I won't write a custom swap; in the vast majority of cases it's not necessary. Now, if an exception is thrown, it will throw while the temporary is being constructed in Example(other), before the swap is executed, and `e2` will be unchanged. Note that `std::swap` calls the move constructor and move assignment, so the three defaulted special functions are all being used to produce a copy assignment that gives the strong guarantee. So, is this compatible with the spirit of the Rule of Zero? It may seem not, as we explicitly wrote a special function. But I would argue that it is. The main thing I emphasized is the undesirability of letting your class deal with resource management details of its members. Here, this is not happening. Notice that none of the special function code will need to be changed as you add or remove members, provided they are ones that provide the correct semantics. Maybe the take away is this: the real Zero in the rule of Zero is that your special functions should reference exactly zero of your member variables. Whether it stems from not being written, explicitly defaulted, or copied and swapped, all of these approaches are fine.
