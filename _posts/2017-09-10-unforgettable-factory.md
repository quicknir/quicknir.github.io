---
layout: post
title: "Unforgettable Factory Registration"
---

Using a factory is a common pattern to use when we are working with polymorphic
objects. It exists to solve a very basic issue in C++: in order to construct
something you must name its type. But the entire point of runtime polymoprhism
is often that we cannot name the type, because we won't know it until runtime.

The most basic factory would just consist of some `if` statements coupled
together:

{% highlight cpp linenos %}
unique_ptr<Animal> makeAnimal(const std::string& type, int number)
{
  if (type == "Dog") return make_unique<Dog>(number);
  if (type == "Cat") return make_unique<Cat>(number);
  throw runtime_error("Invalid type!")
}
{% endhighlight %}

This isn't a great solution. In particular, we keep having to update this
central piece of code every time a new derived class gets written. Under the
best of circumstances, this is merely irritating and code smell. However, in
some cases it can be much worse. It's common for a library to provide an
interface and a few concrete classes deriving from it. If the library uses a
factory somewhere that's written like this, it will be impossible for the user
to inject their own classes into the library.

Aside from very simple use cases, better factories tend to predicated on
allowing classes to themselves for construction by the factory; this allows
library code to construct user classes that were written afterwards, and avoids
the issue of a central function that needs to change with each new class. This
is usually by done changing the logic of the factory from code---multiple
`if`s---into data, in particular an associative map. It usually looks something
like this:

{% highlight cpp linenos %}
unordered_map<string, unique_ptr<Animal>(*)(int)> factory_map;
factory_map["Dog"] = Dog::make;
factory_map["Cat"] = Cat::make;
unique_ptr<Animal> makeAnimal(const std::string& type, int number)
{
  return factory_map.at(type)(int);
}
{% endhighlight %}

## Registration

Using an associative map for this sort of thing is the way to go, but there's
still the whole issue of registration.  It's easier to forget to do, or do
incorrectly. Overwhelmingly, I've seen macros used for this purpose; something
along the lines of:

{% highlight cpp linenos %}
class Dog : Animal {
};
REGISTER_CLASS(factory_map, Dog);
{% endhighlight %}

It has to be done manually for each derived class and it's easy to forget. We're
going to look at how to automate this process, removing boilerplate and
eliminating the possibility of mistakes.

## Solution Sketch

Let's start by a sketch of some of the functionality that we want. Because this
is all going to be automated, I'm going to opt to simply inject the factory
interface directly into the base class using the Curiously Recurring Template
Pattern
[CRTP](https://eli.thegreenplace.net/2011/05/17/the-curiously-recurring-template-pattern-in-c),
instead of having it as a separate entity.

{% highlight cpp linenos %}
template <class Base, class... Args>
class Factory {
public:
  static std::unique_ptr<Base> make(const std::string &s, Args... args) {
    return data().at(s)(args...);
  }

  friend Base;

private:
  using FuncType = std::unique_ptr<Base> (*)(Args...);
  Factory() = default;

  static auto &data() {
    static std::unordered_map<std::string, FuncType> s;
    return s;
  }
};
{% endhighlight %}

The factory is templated on the base class that it is both injecting interface
into, and that it produces unique pointers to. The `Args` template parameter
represent the arguments required to produce a derived instance from the factory.
Note that we use the common CRTP trick of making the constructor `private`, and
the template a `friend` to make it more difficult to misuse (`class Foo :
Factory<Bar, int>` will not compile).

So far, so good. But we haven't dealt with registration at all. So let's take a
stab at it. The two techniques to use are:

 1. CRTP to inject functionality into derived classes automatically.
 2. Use initialization of a static member to force code to be executed before
    main.

Let's give this a shot then; we'll declare a nested class inside `Factory`:

{% highlight cpp linenos %}
template <class Base, class... Args>
class Factory {
...
  template <class T>
  struct Registrar : Base {
    friend T;

    static bool registerT() {
      const auto name = T::name;
      Factory::data()[name] = [](Args... args) -> std::unique_ptr<Base> {
        return std::make_unique<T>(args...);
      };
      return true;
    }
    static bool registered;

  private:
    Registrar() = default;
  };
};

// The really fun part
template <class Base, class... Args>
template <class T>
bool Factory<Base, Args...>::Registrar<T>::registered =
    Factory<Base, Args...>::Registrar<T>::registerT();
{% endhighlight %}

The intention here is that classes wishing to implement the `Base` interface
will inherit from `Registrar` instead of from `Base` directly. This will
instantiate the template, including the `registered` member, causing it to get
initialized and causing the derived class to get registered. We assume (for now)
that the derived class provides a static member `name` to indicate how it would
like to be named in the factory.

There's just one little problem: 1) and 2) are not compatible with one another!
The CRTP pattern involves templates, and unused members of class templates are
not instantiated. This seems annoying but it's quite useful in other contexts:
it allows us to write class templates that may have only part of their members
usable for certain parameters; for example a `std::vector` will not be copyable
if its contained type is not copyable, which works because the copy constructor
is not instantiated unless it's used.

So, we need to make sure that `registered` gets used. But of course, we have to
use it from some part of the class that is itself guaranteed to be used. What's
guaranteed to be used? Well, `Registrar`'s constructor will have to be
instantiated, so let's use that:

{% highlight cpp linenos %}
  ...
  private:
    Registrar() { (void)registered; }
{% endhighlight %}

Pretty strange looking, but this code will work. Before we get a feel for using
it, we're going to make some improvements.

## Automatic name

One nice change would be to obviate the necessity of the derived member having a
static `name` member. As it turns out, we can do this, at least if you're not
disabling RTTI. Based on the discussion
[here](https://stackoverflow.com/questions/281818/unmangling-the-result-of-stdtype-infoname),
we can write:

{% highlight cpp linenos %}
std::string demangle(const char *name) {

  int status = -4; // some arbitrary value to eliminate the compiler warning

  std::unique_ptr<char, void (*)(void *)> res{
      abi::__cxa_demangle(name, NULL, NULL, &status), std::free};

  return (status == 0) ? res.get() : name;
}
{% endhighlight %}

We can then change `registerT`

{% highlight cpp linenos %}
    static bool registerT() {
      const auto name = demangle(typeid(T).name());
      Factory::data()[name] = [](Args... args) -> std::unique_ptr<Base> {
        return std::make_unique<T>(args...);
      };
      return true;
    }
{% endhighlight %}

As I've implemented it here, the name will include the namespace. Obviously
that's simply enough to remove if you don't want it to include that.

## Better safety

We've already employed the CRTP trick of making the constructor private and make
the intended derived a friend, so that it's impossible to accidentally template
on something else when inheriting from the CRTP class. However, so far nothing
is stopping users from inheriting directly from the base class, in which case no
registration would occur. Preventing this with friendship is a bit tricky due to
the various nested template classes, but we use a slightly different approach
called the
[Passkey](https://stackoverflow.com/questions/3217390/clean-c-granular-friend-equivalent-answer-attorney-client-idiom/3218920#3218920)
idiom.

We declare another nested class inside `Factory`, and slightly change the
constructor of Registrar:

{% highlight cpp linenos %}
template <class T, class ... Args>
class Factory {
  ...
  template <class T>
  struct Registrar : Base {
  ...
  private:
    Registrar() : Base(Key{}) { (void)registered; }
  };
  ...
private:
  class Key {
    Key(){};
    template <class T> friend struct Registrar;
  };
};
{% endhighlight %}

The user base class will need to declare the constructor to take a `Key`, which
in turn can only be created by `Registrar`. Thus, no class can derive without
going through `Registrar`.

## A small bonus

If we're using a factory, it means we are interested in polymorphic creation and
ownership of classes. Along with this, usually comes polymorphic copying. Since
we're already using the CRTP pattern, we can add a `clone` method to
`Registrar` and `Factory`:

{% highlight cpp linenos %}
template <class Base, class... Args>
class Factory {
public:
  virtual std::unique_ptr<Base> clone() const = 0;
  ...
  template <class T>
  struct Registrar {
  ...
    std::unique_ptr<Base> clone() const override {
      return std::make_unique<T>(static_cast<const T &>(*this));
    }
  ...
{% endhighlight %}

## Usage

Finally, after building up that whole structure, we can see what user code looks
like. The first piece of user code is the base class. That code could look like this:

{% highlight cpp linenos %}
struct Animal : Factory<Animal, int> {
  Animal(Key) {}
  virtual void makeNoise() = 0;
};
{% endhighlight %}

Every line of code here is actually expressing something relevant to their
design, with the exception of line 2, which is a small price to pay to make sure
that users of this base class don't accidentally inherit directly, and then try
to understand why their class isn't registered.

Now let's look at the next piece of user code, a derived class:

{% highlight cpp linenos %}
class Dog : public Animal::Registrar<Dog> {
public:
  Dog(int x) : m_x(x) {}

  void makeNoise() { std::cerr << "Dog: " << m_x << "\n"; }

private:
  int m_x;
};
{% endhighlight %}

Every single line of code here implements some kind of functional for the
derived; boilerplate here is at an absolute minimum. Beyond automatic
registration, we also protect at compile time against a number of errors:

 - It's not possible to have `Dog` inherit from `Animal` directly
 - It's not possible to have `Dog` inherit from `Registrar<Cat>`.
 - It's not possible to provide a constructor with an incompatible signature.

Finally, let's take a look at a trivial bit of code that makes use of all this.
Let's assume that we have a `Cat` class as well defined by `:s/Dog/Cat` on our
previous bit of code. Then we can write:

{% highlight cpp linenos %}
int main() {
  auto x = Animal::make("Dog", 3);
  auto y = Animal::make("Cat", 2);
  x->makeNoise();
  y->makeNoise();
  x = y->clone();
  x->makeNoise();
}
{% endhighlight %}

Which prints out:

```
Dog: 3
Cat: 2
Cat: 2
```

as you would expect. See a full copy of the working code
[here](http://coliru.stacked-crooked.com/a/0410251ac74729ad).

We've managed here to separate out much of the boilerplate that often goes with
writing polymorphic code in C++, with a relatively small and simple amount of
code. As we can see, the user code that defines the interface and various
implementations can focus on the required logic, and get many pieces of
desirable interface for free.
