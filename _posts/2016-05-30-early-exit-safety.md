---
layout: post
title: "Early Exit Safety"
---

Many moons ago, the recommendation when writing functions in various languages
was often to only have one exit point. Fortunately since then, things have
changed. Very few would find the following piece of code controversial (at
least, for that reason):

{% highlight cpp linenos %}
bool is_present(double const* array, int size, double target) {
  for (int i = 0; i != size; ++i) {
    if (array[i] == target) return true;
  }
  return false;
}
{% endhighlight %}

At this point, you might get a raised eyebrow if you wrote it like this: 

{% highlight cpp linenos %}
bool is_present(double const* array, int size, double target) {
  bool output = false;
  for (int i = 0; i != size; ++i) {
    if (array[i] == target) {
      output = true;
      break;
    }
  }
  return output;
}
{% endhighlight %}

You'd even be specifically violating a well known style guide, that of llvm:

> Instead of this sort of loop, we strongly prefer to use a predicate function
> (which may be static) that uses early exits to compute the predicate.

What makes the first version preferable? There are a few ways to look at it. The
first version has one less bit of mutable state compared to the second version
(the variable `output`), and for some people that's enough. Another way to look
at it is this: when you're looking at a function, there are typically branches
in the code. The moment you see a return statement, you know you've fully
understand that branch. In some sense, early return is branch pruning for your
brain, and reduces the complexity involved in understanding a function. Indeed,
maybe once again the llvm style guide puts it best:

> When reading code, keep in mind how much state and how many previous decisions
> have to be remembered by the reader to understand a block of code... One great
> way to do this is by making use of early exits...

So what's the problem?

## Leaving the Party Early

Let's consider a function that receives a string, and has to determine whether
all of the parentheses and braces in it are correctly matched. This
means they must be balanced, and strictly nested.

{% highlight cpp linenos %}
bool are_matched(const char * text, int size) {
  char * stack = malloc(size);
  int si = 0;
  for (int i = 0; i != size; ++i) {
    char c = text[i];
    if (c == '(' || c == '{') {
      stack[si++] = c
    }
    else if (c == ')' && stack[si-1] == '(') {
      --si;
    }
    else if (c == '}' && stack[si-1] == '{') {
      --si;
    }
  }
  free(stack);
  return si == 0;
}
{% endhighlight %}

This simply uses a stack to keep track of the sequence of openers, and
annihilates them with closers (`)` or `}`)as it encounters them. Unfortunately, this
implementation has several bugs, all related to how closers are handled. One
problem is that `stack_index` can become negative. Another is that this doesn't
behave correctly for mismatched closers, for instance feeding the function `(})`
will yield a result of `true`. The simple solution is that the moment that a
closer is encountered with either an empty stack, or with the wrong opener, one
can immediately return `false`:

{% highlight cpp linenos %}
bool are_matched(const char * text, int size) {
  char * stack = malloc(size);
  int si = 0;
  for (int i = 0; i != size; ++i) {
    char c = text[i];
    if (c == '(' || c == '{') {
      stack[si++] = c;
    }
    if (si == 0 && ( c==')' || c=='}' ) ) {
      return false;
    }
    if (c == ')') {
      if (stack[si-1] != '(') {
        return false;
      }
      else {
        --si;
      }
    }
    if (c == ')') {
      if (stack[si-1] != '(') {
        return false;
      }
      else {
        --si;
      }
    }
  }
  free(stack);
  return si == 0;
}
{% endhighlight %}

This could be written without early exit points, but it would be significantly
harder to read. However, we've introduced a new problem: this solution leaks
memory in many cases. In other words: the way in which we are writing code is
not early exit safe, so introducing early exits causes problems. Let's look at
some of the solutions in turn.

#### Track all Exits

One solution is simply to add a call to `free` just prior to each of the
`return false` lines. However, this scales poorly *both* in terms of the
complexity of the function, and in terms of the number of things we have to do
on exit. Even in our very simple function, we are now repeating the same code 4
times. If a new resource is acquired, we'd have to release it in four places. If
a new exit point is added, we need to remember to call `free` there as well.
Clearly, this is quite bug prone.

#### Converge all Exits

Another way of thinking is this: instead of putting cleanup code at every exit,
why don't I simply bring every simple function, we are now repeating the same code 4
times. If a new resource is acquired, we'd have to release it in four places. If
a new exit point is added, we need to remember to call `free` there as well.
Clearly, this is quite bug prone.

#### Converge all Exits

Another way of thinking is this: instead of putting cleanup code at every exit,
why don't I simply bring every exit to a common location *just prior* to
exiting, and do my cleanup there? This is a common approach in C, where `goto`
is used for this. There are a couple of different ways to do this: one is with
multiple labels, and the other is with a single label, I'll show the latter here.

{% highlight cpp linenos %}
int func() {
  struct1* s1 = NULL;
  struct2* s2 = NULL;

  int output = 0;
  
  if (!s1 = get_resource_1()) {
    output = -1;
    goto cleanup;
  }
  if (!s2 = get_resource_2()) {
    output = -1;
    goto cleanup;
  }
  
  /* do work; set output and goto cleanup instead of returning */

cleanup:
  if (s1) {
    cleanup_1(s1);
  }
  if (s2) {
    cleanup_2(s2);
  }
  return output;
}
{% endhighlight %}

We're assuming here the very exit to a common location *just prior* to
exiting, and do my cleanup there? This is a common approach in C, where `goto`
is used for this. There are a couple of different ways to do this: one is with
multiple labels, and the other is with a single label, I'll show the latter here.

{% highlight cpp linenos %}
int func() {
  struct1* s1 = NULL;
  struct2* s2 = NULL;

  int output = 0;
  
  if (!s1 = get_resource_1()) {
    output = -1;
    goto cleanup;
  }
  if (!s2 = get_resource_2()) {
    output = -1;
    goto cleanup;
  }
  
  /* do work; set output and goto cleanup instead of returning */

cleanup:
  if (s1) {
    cleanup_1(s1);
  }
  if (s2) {
    cleanup_2(s2);
  }
  return output;
}
{% endhighlight %}

We're assuming here the v very simple function, we are now repeating the same code 4
times. If a new resource is acquired, we'd have to release it in four places. If
a new exit point is added, we need to remember to call `free` there as well.
Clearly, this is quite bug prone.

#### Converge all Exits

Another way of thinking is this: instead of putting cleanup code at every exit,
why don't I simply bring every exit to a common location *just prior* to
exiting, and do my cleanup there? This is a common approach in C, where `goto`
is used for this. There are a couple of different ways to do this: one is with
multiple labels, and the other is with a single label, I'll show the latter here.

{% highlight cpp linenos %}
int func() {
  struct1* s1 = NULL;
  struct2* s2 = NULL;

  int output = 0;
  
  if (!s1 = get_resource_1()) {
    output = -1;
    goto cleanup;
  }
  if (!s2 = get_resource_2()) {
    output = -1;
    goto cleanup;
  }
  
  /* do work; set output and goto cleanup instead of returning */

cleanup:
  if (s1) {
    cleanup_1(s1);
  }
  if (s2) {
    cleanup_2(s2);
  }
  return output;
}
{% endhighlight %}

We're assuming here the v very simple function, we are now repeating the same code 4
times. If a new resource is acquired, we'd have to release it in four places. If
a new exit point is added, we need to remember to call `free` there as well.
Clearly, this is quite bug prone.

#### Converge all Exits

Another way of thinking is this: instead of putting cleanup code at every exit,
why don't I simply bring every exit to a common location *just prior* to
exiting, and do my cleanup there? This is a common approach in C, where `goto`
is used for this. There are a couple of different ways to do this: one is with
multiple labels, and the other is with a single label, I'll show the latter here.

{% highlight cpp linenos %}
int func() {
  struct1* s1 = NULL;
  struct2* s2 = NULL;

  int output = 0;
  
  if (!s1 = get_resource_1()) {
    output = -1;
    goto cleanup;
  }
  if (!s2 = get_resource_2()) {
    output = -1;
    goto cleanup;
  }
  
  /* do work; set output and goto cleanup instead of returning */

cleanup:
  if (s1) {
    cleanup_1(s1);
  }
  if (s2) {
    cleanup_2(s2);
  }
  return output;
}
{% endhighlight %}

We're assuming here the very common convention in C that a function will return
a null pointer when it's unable to acquire the resource we request from it.
Although this approach is the best so far and would be quite sufficient for the
simple function that we wrote, it still has some undesirable properties.

For one, we have to add code in three separate places to support each resource.
It has to be declared, then the resource has to be acquired, and then it has to
be cleaned up. These cannot be three consecutive lines of code, so adding or
removing resources means making changes in three, far apart locations, which
isn't great.

There's another disadvantage as well: all of the variables representing our
resources have to be declared at the beginning. This is necessary because
`gotos` should not skip over variable declarations if they're going to reference
them afterwards. This means that even if we only need a resource inside an if
statement, or inside a loop, or only for the second half of the function, the
variable representing it is still in scope throughout the entire function from
beginning to end.

In our attempt to try to support the more modern thinking about early exits, we
end up with a variable declaration scheme similar to K&R!

#### Clean on exit

#### Clean on exit (improved)

{% highlight cpp linenos %}
int func() {
  struct1* s1 = get_resource_1();
  const auto s1_sg = makeScopeGuard([&] () { if (s1) cleanup_1(s1)});
  if (!s1) return -1;
  
  // same for s2
  
  // do work and return
}
{% endhighlight %}

#### Clean up after yourself
