---
title: nodiscard("should have a reason")
layout: page
date: October 24th, 2019
author:
  - JeanHeyd Meneide \<<phdofthehouse@gmail.com>\>
  - Isabella Muerte \<<https://twitter.com/slurpsmadrips/>\>
redirect_from:
  - /vendor/future_cxx/papers/source/n2430.html
  - /vendor/future_cxx/papers/source/n2448.html
hide: true
---

<style>
pre {
  margin-top: 0px;
  margin-bottom: 0px;
}
.ins, ins, ins *, span.ins, span.ins * {
  background-color: rgb(200, 250, 200);
  color: rgb(0, 136, 0);
  text-decoration: underline;
}
.del, del, del *, span.del, span.del * {
  background-color: rgb(250, 200, 200);
  color: rgb(255, 0, 0);
  text-decoration: line-through;
  text-decoration-color: rgb(255, 0, 0);
}
math, span.math {
  font-family: serif;
  font-style: italic;
}
ul {
  list-style-type: "— ";
}
blockquote {
  counter-reset: paragraph;
}
div.numbered, div.newnumbered {
  margin-left: 2em;
  margin-top: 1em;
  margin-bottom: 1em;
}
div.numbered:before, div.newnumbered:before {
  position: absolute;
  margin-left: -2em;
  display-style: block;
}
div.numbered:before {
  content: counter(paragraph);
  counter-increment: paragraph;
}
div.newnumbered:before {
  content: "�";
}
div.numbered ul, div.newnumbered ul {
  counter-reset: list_item;
}
div.numbered li, div.newnumbered li {
  margin-left: 3em;
}
div.numbered li:before, div.newnumbered li:before {
  position: absolute;
  margin-left: -4.8em;
  display-style: block;
}
div.numbered li:before {
  content: "(" counter(paragraph) "." counter(list_item) ")";
  counter-increment: list_item;
}
div.newnumbered li:before {
  content: "(�." counter(list_item) ")";
  counter-increment: list_item;
}
</style>

_**Document**_: n2448  
_**Previous Revisions**_: n2430  
_**Audience**_: WG14  
_**Proposal Category**_: New Features  
_**Target Audience**_: General Developers, Compiler/Tooling Developers  
_**Latest Revision**_: [https://thephd.github.io/vendor/future_cxx/papers/source/n2448.html](https://thephd.github.io/vendor/future_cxx/papers/source/n2448.html)

<p style="text-align: center">
<span style="font-style: italic; font-weight: bold">Abstract:</span>
<p>Many functions return a value, however, not all function return values are of equal importance to the caller. The recent `[[nodiscard]]` attribute allows compilers to issue a diagnostics, but only hands the user a generic error message. This proposal enhances the `[[nodiscard]]` attribute in the same manner as the `[[deprecated]]` attribute, giving developers the same power to guide their users to better APIs with the aid of the compiler by providing a `string literal` attribute argument clause.</p>
</p>



# Introduction

[Document N2267](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2267.pdf) introduced a new attribute `[[nodiscard]]` in the C2x working paper. This has provided significant improvements in reminding programmers of the safety issues of discarding the return value of a function. The `[[nodiscard]]` attribute has helped prevent a serious class of software bugs, but sometimes it is hard to communicate exactly **why** a function is marked as `[[nodiscard]]` and perhaps what actions should be taken to rectify the issue.

This paper supplies an addendum to allow a developer to add a string attribute token to let someone provide a small reasoning or reminder for why a function has been marked `[[nodiscard("potential memory leak")]]`.



# Design Considerations

This paper is an enhancement of a preexisting feature to help programmers provide clarity with their code. Anything that makes the implementation warn or error should also provide some reasoning or perhaps point users to a knowledge base or similar to have any questions they have about the reason for the nodiscard attribute answered.


Consider the following code example, before and after the change:

```c++
#define FOO_BASE 0xBA51CF00

#define FOO_LINK_TYPE 1

struct foo { /* ... */ };
[[nodiscard]] int foo_get_value(struct foo*);
```

### Status Quo:

```c++
[[nodiscard]] 
foo* foo_create(int, struct foo*);
[[nodiscard]] 
int foo_compare(struct foo*, struct foo*);

// Always > 0
const int kHandles = ...;

int main (int, char*[]) {

  foo* foo_handles[kHandles + 1] = { };
  foo_handles[0] = foo_create(BASE_FOO, NULL);
  for (int i = 1; i < kHandles; ++i) {
    foo_handles[i] = foo_create(FOO_LINK_TYPE, foo_handles[0])
  }
  
  /* sometime later */

  for (int i = 0; i < kHandles, 
    foo_compare(foo_handles[0], foo_handles[i]), foo_get_value(foo_handles[i]) > 0; 
    // ^ warning: function return value marked nodiscard was discarded
    ++i) {
      /* process... */
  }

  return 0;
}
```

⚠️ - warning, but it is a generic warning; what exactly went wrong here?

### With Proposal:

```c++
[[nodiscard("memory leaked")]] 
struct foo* foo_create(int, struct foo*);
[[nodiscard("value of foo comparison unused")]] 
int foo_compare(struct foo*, struct foo*);

// Always > 0
const int kHandles = ...;

int main (int, char*[]) {

  struct foo* foo_handles[kHandles + 1] = { };
  foo_handles[0] = foo_create(BASE_FOO, NULL);
  for (int i = 1; i < kHandles; ++i) {
    foo_handles[i] = foo_create(FOO_LINK_TYPE, foo_handles[0])
  }
  
  /* sometime later */

  for (int i = 0; i < kHandles, 
    foo_compare(foo_handles[0], foo_handles[i]), foo_get_value(foo_handles[i]) > 0; 
    // ^ warning: function return marked nodiscard was discarded - value of foo comparison unused
    ++i) {
      /* process... */
  }

  return 0;
}
```

✔️ - warning much more clearly makes it obvious that a comma was used with the return value of `foo_compare`, and not `&&`.


The design is very simple and follows the lead of the deprecated attribute. We propose allowing a string literal to be passed as an attribute argument clause, allowing for `[[nodiscard("use the returned token with lib_foobar")]]`. The key here is that there are some nodiscard attributes that have different kinds of "severity" versus others.

Adding a reason to nodiscard allows implementers of the standard library, library developers, and application writers to benefit from a more clear and concise error beyond `error:<line>: value marked [[nodiscard]] was discarded`. This makes it easier for developers to understand the intent for return values for the used libraries (and understand from which individual expression errors originate in complex expressions).



# Implementation Experience

This is in the official C++ Standard, and has been [merged into Clang already](http://lists.llvm.org/pipermail/cfe-commits/Week-of-Mon-20190715/280158.html) as well as [merged into GCC](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=91368). It would be good to maintain parity with C++ to allow headers that work in both languages to continue to use the same syntax, since this is going to be an increasingly useful existing practice.



# Proposed Wording

This proposed wording is currently relative to Working Paper N2385. The intent of this wording is to allow for the `[[nodiscard]]` attribute to be able to take a string literal.


## Changes

Rewrite §6.7.11.2 "The nodiscard attribute"'s **Constraint** subsection as follows:

> <ins>The nodiscard attribute shall be applied to the identifier in a function declarator or to the definition of a structure, union, or enumeration type. It shall appear at most once in each attribute list. If an attribute argument clause is present, it shall have the form:</ins>
> 
> <ins>( *string-literal* )</ins>

Add a clause just beneath the first clause in the **Recommended Practice** subsection as follows:

> <ins>The diagnostic message may include text provided by the string literal within the attribute argument clause of any nodiscard attribute applied to the name or entity.</ins>

Add a third example after the first two in the **Recommended Practice** subsection as follows:

> > ```c++
> > [[nodiscard("must check armed state")]] 
> > bool arm_detonator(int);
> > 
> > void call(void) {
> >   arm_detonator(3);
> >   detonate();
> > }
> > ```
> > <ins>A diagnostic for the call to `arm_detonator` using the *string literal* `"must check armed state"` from the *attribute argument clause* is encouraged.</ins>

