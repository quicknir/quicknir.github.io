---
layout: post
title: "Object Theft Etiquette in C++: methods with a side of &&"
---

Sometimes in C++, an object has something that we want. We don't want to be honest citizens and put in the hard work of copying. Instead we want to steal the something away from the object, and let the object deal. The mechanism we use for this is move semantics. Let's look at a motivating example:

{% highlight cpp linenos %}
using namespace std;

class SetStringer {
 public:
  void insert(double x) { m_set.insert(x); m_cached = false; }
  
  const string & str() {
    if (m_cached) { return m_string; }
    
    m_string.clear();
    
    for (const auto & x : m_set) {
      m_string += to_string(x) + &quot;\n&quot;;
    }
    m_cached = true;
    return m_string;
  }  

 private:
  set<double> m_set;
  string m_string;
  bool m_cached = false;
};
{% endhighlight %}

This class seems reasonably written, right? Arguably the str method should be marked const, but then we'd need to mark m_cached and m_string as mutable. Since this isn't relevant to our discussion, I'm going to leave this alone. Also, those of you that are attentive and maybe even read my last <a href="https://turingtester.wordpress.com/2015/06/27/cs-rule-of-zero/">post</a> about resource management and special functions may notice something that could go wrong in the copy/move semantics of this class. I'll leave this as an exercise to the reader and post the solution next time.

{% highlight cpp linenos %}
SetStringer foo;
// ... add stuff to foo
string s = move(foo.str()); // Stealing fails!
{% endhighlight %}

We tried to steal the string to do something useful with it later, but weren't successful. The reason is that str() returns a const reference; even after the move casts it to a const rvalue reference, it still won't bind to string's move constructor. So we ended up paying for a full construction of string, instead of a move. For our second attempt, we'll just remove the const from the signature of str.

{% highlight cpp linenos %}
SetStringer foo;
// ... add stuff to foo 
string s = move(foo.str()); // Stealing succeeds
cerr << foo.str(); // this probably prints empty
foo.insert(5.0);
 // err what
cerr << foo.str();
{% endhighlight %}

The problem now is that foo did not behave in a sensible way.  The content of `m_string` was stolen, which is fine, but the `m_cached` bool remained with a value of true, which isn't. This means that the first `cerr <<` printed nothing, but mysteriously after we insert one element, a whole bunch are printed. More broadly, it's not safe to allow the outside world to directly mutate a member variable that is a part of the class' invariants; in this case the invariant is that when m_cached is true, `m_string`'s value follows directly from m_set&#039;s. By stealing the string, we put foo in an invalid state, and it&#039;s generally a bad idea to allow objects to get into invalid states.

What do we do?

## A Civilized Solution

Consider this code:

{% highlight cpp linenos %}
class SetStringer { // as before except where noted
  
  const string & str() & { // implemented as before }
  
  string && str() && {
    this->str();
    m_cached = false;
    m_set.clear();
    return move(m_string);
  }
};
{% endhighlight %}

The first str method is unchanged except for one little &; this means that this is an overload for when the object is an lvalue. The newly implemented str function on the other hand, has the && after the parentheses which indicates it's an overload for when the object is an rvalue.

Notice the implementation of str: As you might guess, the call to this->str() is not recursive, but rather calls the lvalue implementation, even though the object is an rvalue. Well, even though the object is an rvalue, the this pointer we are using is still an lvalue. As an aside, this just goes to show how much weirder rvalue and lvalue semantics are than const and non-const: when you are implementing a non-const function in terms of a const one (as shown <a href="http://stackoverflow.com/questions/856542/elegant-solution-to-duplicate-const-and-non-const-getters">here</a>) you have to explicitly cast things. That's because within a const method <code>this</code> is a pointer to const, whereas within an rvalue method, <code>this</code> is still an lvalue. If this confuses and infuriates you, you may want to consider pausing to read <a href="https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers">this</a> excellent article by Scott Meyers.

Anyhow, the key point is that str() && sets m_cached to false, clears the set, and returns string after casting to an rvalue reference. Why clear the set? Well, the first version of this post did not clear the set. While this is technically still correct, it was confusing to people who (reasonably, given my poor choice of example) thought I was making the point that stealing from foo did not change its observable state. That's not the case; all we want to do is make sure that foo is in a valid state, any valid state.

{% highlight cpp linenos %}
SetStringer foo;
// ... add stuff to foo 
string s = move(foo).str(); // Stealing succeeds!
cerr << foo.str() // prints empty, fine
foo.insert(5.0);
cerr << foo.str(); // prints just the 5.0!
{% endhighlight %}

Notice how the syntax changed, we're now calling move on the object itself, instead of the return of its method. This makes sense: the object is in charge of its state, and if you want to steal something from the object, you should politely let the object itself know, rather then trying to steal it like a thief in the night (though naively, stealing like a thief would seem to be a good thing). And that's the etiquette we'd like to use for stealing.

All this is applicable to (and probably more useful with) std::forward as well; you can optionally steal/copy resources from object members inside a function, depending on whether that function received an rvalue or lvalue copy of the object. Also note that if you do have a class that you feel should be returning a non-const reference, in many situations you could just make the data member public. In that case, you can do the stealing based on either syntax; that is call move either on the member (analogous to calling move on the method, as we did at the start) or call move on the object and immediately access the member.

## The STL Jungle
There are extremely common objects that return non-const references: containers. And as we discussed, a method that returns a non-const reference is a situation where you can steal the rude way. In fact, because of the way most STL implementations currently are (and the fact that the standard doesn't mandate otherwise), that is the <em>only</em> way to do it. Notably, the STL does not provide rvalue and lvalue overloads for e.g. vector::operator[]. The best <a href="http://stackoverflow.com/questions/29311209/why-isnt-operator-overloaded-for-lvalues-and-rvalues">explanation</a> I've been able to find for this omission is that it would break backwards compatibility in some edge cases for older code. So in this specific case, you don't have much choice except to cast the return of the method, rather than the object itself. I'll dig more into the STL's relationship with rvalues in a future post, but for now you should be aware of this caveat.

Happy stealing!

Edit: because explicitly using move in the way I have above is relatively uncommon, I decided to provide another code snippet.

{% highlight cpp linenos %}
template <class T>
void func(T && x) {
  string z = forward<T>(x).str();
  // ...
}
{% endhighlight %}

Notice how in general, we can quite easily move construct from an argument in a function like this by calling forward on it. However, what if we want to move construct from a member that normally returns a const reference, like str? That's where str && comes in; it lets us take advantage of x being an rvalue, but still not break its invariants.
