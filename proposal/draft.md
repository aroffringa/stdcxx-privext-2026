Private Extension Member Functions
==========================================

* Document Number: -
* Date: 2026-04-29
* Programming Language C++
* Reply-to: André Offringa <offringa@gmail.com>
* Audience: WG21

Introduction
=============================

This proposal adds a new mechanism for declaring non-virtual 
private class member functions and private static class member functions outside of the
class definition, with internal linkage.

It is based on an earlier proposal from 2013 by Matthew Fioravante
\[[N3863](#N3863)\] \[[Proposal2013](#Proposal2013)\].
Large parts of his text were reused, following the MIT license. See
the [git repository](https://github.com/aroffringa/stdcxx-privext-2026)
for the license.

Impact on the standard
=============================

This proposal is a core language extension. It does not require
any new keywords. It proposes one additional syntax by reusing an existing
keyword in a manner congruent with the keyword's original purpose.
The new feature does not break any legacy code. 
Some expressions which used to be a compilation error are now valid C++.

Motivation: Problem Definition
================

Good class design follows the principle of encapsulation.
The game of encapsulation is the game of hiding as many details
as possible, exposing only the minimum of details required for the users
to use the interface. By minimizing the exposed information, we reduce
the complexity of our interface, making it easier to understand and use.
We also give ourselves much more freedom and flexibility with
implementation. Any class details not present in the interface can
be changed without affecting the users of the interface.

Analysis of the quality of encapsulation enabled by the C++ class model
--------------------------------

In C++, a system interface can take the form of a class definition.
This class definition is normally written in a header
file, in order to allow its use in multiple translation units.
In order to maximize encapsulation, we must minimize the number
of class details in the interface to the bare minimum required by
the users of the class.

With this idea, the public and protected aspects must be a part
of the interface because they are directly exposed to the class user or
child classes.
Private virtual member functions are also part of the interface to the child class
because the child class may choose to override them.

Private data members are not technically part of the interface as
they cannot be accessed by users or child classes.
However, the compiler needs to know
the size of the entire object in order to create instances of it.
The size cannot be computed without seeing all of the data members,
regardless of access control.

Friend declarations also are not technically part of the interface.
However, allowing friend declarations to be declared anywhere would 
make it very easy for users to abuse friends in order to break access
control. This proposal does not address any aspects of the
friend feature.

Non-virtual private member functions and private static member functions are not part
of the interface, because the direct users and child classes cannot
call private member functions, and so they do not need to see their signatures,
much less know of their existence.
The compiler also does not need to be aware of the private
member functions signatures until they are called. On all major platforms, changing
the non-virtual private member functions (which are not called by inline functions)
does not affect the ABI of the class itself
\[[KDEABI](#KDEABI)\].

We conclude that non-virtual private member function and private static member function
declarations are neither a part of the class interface nor influence the ABI of the
class, and thus should not be required in the class definition.

Practical Concerns
---------------------

High-level discussions about encapsulation aside, there are some very real
practical problems that arise from requiring private member function
declarations in the class definition.

* Whenever the class developer adds, removes, or modifies a private member function
    signature, all users of the class *must* recompile. Large C++ applications
    already have prohibitively long compilation times. Waiting for compilation
    wastes a lot of programmer time and reduces productivity.
    Books such as \[[Lakos01](#Lakos01)\] have large sections dedicated to
    techniques for reducing compile times.
    Long compilation time in C++ is a big problem that needs to be addressed.
    This proposal is one major step in the reduction of programmer
    time wasted waiting for compilation during development.
* Unnecessary dependencies are introduced into the class header file. 
    All of the symbols used in private member function signatures must be exposed
    to the clients of the class. This matters for shared libraries
    where we wish to minimize symbol dependencies to optimize library load time and
    avoid symbol name clashes. While some platform dependent techniques
    such as visibility control can be used to mitigate the symbol
    pollution, this requires extra work from the programmer which he
    would not need to do at all if the private member function signature along
    with all of its symbol dependencies were
    safely encapsulated within the project's internal source files.
* Symbols with internal linkage (i.e keyword `static` and anonymous namespaces)
    cannot be
    used in private member function signatures, even if those private member functions
    are only called within one translation unit. This artificially 
    restricts the programmer from passing and returning internally linked
    symbols to and from their private member functions. Moreover, the private
    member functions themselves are often good candidates for internal linkage,
    but this is currently not possible. 
    Internal linkage is
    another good technique for reducing the set of symbols exported by
    a binary and can also enable compiler optimizations.
* Some classes may have multiple implementations which can be swapped
    out at compile time. One example is a cross platform file IO
    library such as iostream. It is not unreasonable to expect 
    each platform to require a different set of private member function
    signatures to implement its behavior.
    Because these signatures are required to be in the
    definition, we are then forced to also include all of the
    complex conditional compilation details into the header files.

Problem Solution
------------

We propose that private non-virtual
member functions and private static member functions should be able to be declared outside
of the class definition, with internal linkage.
Not only does this change not break encapsulation, it actually improves
encapsulation because unessessary implementation details are being removed from the interface.
Unlike PIMPL or other related encapsulation techniques, this new feature has
no run time overhead.

As a side note, some programming languages provide a mixin feature
which allows programmers to reopen a class and extend its interface
arbitrarily.
We are not proposing or even endorsing any form
of mixin here. Adding, removing, or changing private member functions does
not change the interface of the class.

High-Level Description
====================

The basic idea behind this proposal is simple: Just allow the
programmer to declare additional class non-virtual private member functions
and private static member functions
which are not present in the class definition. We call these additional
class member functions *private extension member functions (PEMF)*  and
*private static extension member functions (PSEMF)*.

PEMF and PSEMFs have internal linkage.

Declaring private extension member functions
----------------------------

In order to declare a private extension member functions, we declare
a new class member function outside of the class definition 
and prefix it with the `private` keyword:

Header file:

    class Foo {
      public:
        int PublicFunction();
      private:
        int i;

        int PrivateFunction();
    };

Implementation:

    // Private extension member function
    private void Foo::PEMF() {
      i++;
    }

    int Foo::PublicFunction() {
      PEMF();
      return PrivateFunction();
    }

    int Foo::PrivateFunction() {
       return i * 2;
    }

Private extension constructors and assignment
--------------

We can also declare additional private constructors:

    class Foo {
      public:
        Foo();
    };

    // PEMF constructor
    private Foo::Foo(int i, int j) : i_(i), j_(j) {}

    // Public constructor delegates to the private extension constructor
    Foo::Foo() : Foo(0, 0) {}

We do not allow private extension default constructors, copy constructors, copy assignment,
move constructor, move assignment, or destructors. We do not allow these, because allowing
these would produce complicated situations. The compiler will for example generate
a default constructor when no user-defined constructor is declared inside the class. If a
private extension constructor is later defined, a complicated situation arises, with
somewhat of a trivial way to violate the one-definition rule.

In summary, all of the following are errors:

    class Foo {};

    private Foo::Foo(); //Error: Cannot declare a PEMF default constructor
    private Foo::Foo(const Foo&); //Error: Cannot declare a PEMF copy constructor
    private Foo::Foo(Foo&&); //Error: Cannot declare a PEMF move constructor
    private Foo& Foo::operator=(const Foo&); //Error: Cannot declare a PEMF copy assignment operator
    private Foo& Foo::operator=(Foo&&); //Error: Cannot declare a PEMF move assignment operator
    private Foo::~Foo(); //Error: Cannot declare a PEMF destructor

Class definition visibility and the private keyword
----------------------

Any class member function which has been declared but not found in the class definition 
requires the `private` keyword, otherwise a compiler error will ensue.
Likewise any member function definition which has been previously
declared in the class definition must not be prefixed by the `private` keyword.
For this reason, all class member function declarations will require the class definition
to have been previously seen.

The following are compilation errors:

    private void Foo::p1(); //<-Error: class definition not visible here!

    class Foo {
      void p2();
    };

    void Foo::p3(); //<-Error: PEMFs must use the private keyword!
    private void Foo::p2() {} //<-Error: member function definitions cannot use the private keyword!

Static Member Functions
-------------------------------

As was discussed earlier, private static member functions are also 
implementation details and do not belong in the class definition.
We can define private static extension member functions by using the `static`
keyword together with the `private` keyword:

    class Foo {
      public:
        static int sf();
    };

    private void Foo::a(); //<-PEMF

    private static int Foo::b() { return 42; } //<-PSEMF
    int Foo::sf() { return b(); }

The `static` keyword may appear before or after `private`.

Forward declaring private extension member functions
--------------------------

Private extension member functions can be forward declared. The keyword
`private` is used both in the forward declaration and in the definition:

Header file:

    struct Foo {
      PublicFunction();
    };

Implementation:
    
    private void Foo::PrivateFunction();
    
    void Foo::PublicFunction() {
      PrivateFunction();
    }
    
    private void Foo::PrivateFunction() {
      // Implementation
    }

This allows for a more flexible ordering of the functions. This copies
the behaviour of static free functions, which can also be forward declared.
In that scenario, the `static` keyword is also used in both the declaration
and the definition.

Internal Linkage
--------------------------

Private extension member functions (static and non-static) will have internal
linkage by default, i.e., they are local to one translation unit. This is
because almost all private extension member functions are likely to be used
only within one translation unit (TU). 
Symbols used within only one TU can be given internal linkage. This reduces
the number of symbols in the entire application and allows more aggressive
optimizations. For example, the compiler can inline all calls and completely
remove the function body from the compiled binary.

We allow for a future extension to support private extension member functions
with external linkage, described under "Alternatives and additions".

Class Templates and PEMFs
--------------------

Class templates are supported in the natural way and do not require any special rules or caveats.

    template <typename T>
    class X {};

    template <typename T>
    private void X<T>::f1(); //<-PEMF for class template X

    template <>
    private void X<int>::f2(); //<-Specialization of PEMF for X<int>

Explicit instantiation of a class template will instantiate all of its member functions.
This will recursively instantiate only the PEMFs (and additional PEMFs called by these,
in a recursive manner) called by those member functions.
This procedure follows the same rules as explicit instantiation
of a class template which calls free function templates.

    template <typename T>
    class X {
      public:
        void pubf(); 
    };

    template <typename T>
    private void X<T>::a() { //<-PEMF for class template
      /* stuff */
    }

    template <typename T>
    private void X<T>::b() { //<-PEMF for class template
      /* stuff */
    }

    // templated f() calls both a() and b()
    template <typename T>
    private void X<T>::pubf() {
      a();
      b();
    }

    // Some specializations of pubf()
    template <> void X<char>::pubf() { b(); }
    template <> void X<int>::pubf() { a(); }
    template <> void X<float>::pubf() { }
    template <> void X<double>::pubf() { a(); }
 
    // A specialization of a()
    template <>
    private void X<double>::a() { b(); }

    // Explicit instantiations
    template class X<char>;  /* Will instantiate the following:
                              * X<char>::pubf();
                              * X<char>::b();
                              */
    template class X<short>; /* Will instantiate the following:
                              * X<short>::pubf();
                              * X<short>::a();
                              * X<short>::b();
                              */
    template class X<int>;   /* Will instantiate the following:
                              * X<int>::pubf();
                              * X<int>::a();
                              */
    template class X<float>; /* Will instantiate the following:
                              * X<float>::f();
                              */
    template class X<double>;/* Will instantiate the following:
                              * X<double>::pubf();
                              * X<double>::b();
                              * X<double>::a();
                              */


Other properties of PEMFs
--------------------

For completeness, in addition to what is already discussed, private
extension member functions will have the following properties:

- After defining a PEMF or PSEMF, they participate in member look-up as if
  they were declared as a private member function inside the class definition.
- Private extension member functions support the same qualifiers as normal member functions,
  including `const`/`volatile` qualifiers, ref-qualification, exception
  specification, attributes and trailing return type.
- The parameter list may include an explicit `this` parameter to deduce `this`.
- Overloading member functions by their parameter list is allowed.
- Operator overloading using private extension member functions is allowed,
  with the exception of the assignment operator (see section about constructor
  and assignment operator).
- Conversion operators of the form `private S::operator T() const` are also
  allowed.
- Declaring a PEMF with the same name as a base class member function causes the
  base class member function name to be hidden, from that point on, in the same way as
  when the private member function was declared inside the class.
- Adding `inline` to a private extension member function is, as usual, a directive
  for the compiler to expand the function inline.

Technical Summary
=====================
    
In summary, this proposal makes the following changes to the standard:

* Allow the programmer to declare additional class member functions outside of the
  class definition. These so called *private extension member function*
  declarations must be prefixed by the `private` keyword.
  These member functions will have private access control and internal linkage.
* The programmer may combine the `static` keyword with the `private` keyword
  to declare a private static extension member function.

Counter Arguments
==================

We will now address some arguments against this proposal.

Violating Access Control
------------------------

One immediate and common objection to this proposal is that it may break encapsulation by allowing
the programmer to subvert access control. By definition, a PEMF has private access control and thus
they cannot be called outside of the class scope. There is however, one exploit
discovered by Richard Smith where merely the existence of an additional class member function
which is never called allows a violation of access control.

    class A {
      int n;
    public:
      A() : n(42) {}
    };
    
    template<typename T> struct X {
      static decltype(T()()) t;
    };
    template<typename T> decltype(T()()) X<T>::t = T()();
    
    int A::*p;
    private int A::expose_private_member() { // note, not called anywhere
      struct DoIt {
        int operator()() {
          p = &A::n;
          return 0;
        }
      };
      return X<DoIt>::t; // odr-use of X<DoIt>::t triggers instantiation
    }
    
    int main() {
      A a;
      return a.*p; // read private member
    }
    
This example would be a newly introduced standards conforming way to violate access
control if this proposal were to be accepted.

This is not actually a problem. This example is artificially contrived to exploit
access control. It is not a common error that would be made by novice programmers nor
is it an obvious or easy to use tool to abuse access control and write poor interfaces.
Indeed, many member functions of violating access control already exist
within the current language \[[GotW076](GotW076)\]. We agree with the author
of that article.

> The issue here is of protecting against Murphy vs. protecting against Machiavelli... that is, protecting against accidental misuse (which the language does very well) vs. protecting against deliberate abuse (which is effectively impossible). In the end, if a programmer wants badly enough to subvert the system, he'll find a way, \[[GotW076](GotW076)\].

There are already current workarounds
--------------------

Likely the most compelling argument against the proposal is that there are
already a set of current workarounds:

- Nested classes can be used to implement a partial variant of PEMF in the current language.
- Friend functions can be used to implement functions that access the private data of the class.
  These can have internal storage. The symbols do however clutter the interface of the class,
  and it adds dependencies types used in the signature of the member function.
- A friend class declaration can be used to avoid listing the functions in the class interface,
  and avoids a dependency on types used in the function signatures. These can however not have
  internal storage.
- The PIMPL idiom can also be used to hide implementation details. This adds some runtime cost.

For hiding implementation details, nested classes are superior to friends because they can be further extended with additional
nested sub classes where as all friends have to be declared in the original class definition.
An example is provided.

Public header file:

    class X {
      public:
        void doWork();

      private:
        int i_;

        struct XHelper;
    };

Private implementation:

    struct X::XHelper {
      static void doWorkHelper(X& x) { //<-PEMF
        x.i_ = 42;
      }

      struct XHelper2;
    };

    struct X::XHelper::XHelper2 {
      static void doMoreWorkHelper(X& x) { //<-PEMF
        x.i_++;
      }
    };

    void X::doWork() {
      XHelper::doWorkHelper(*this);
      XHelper::XHelper2::doMoreWorkHelper(*this);
    }

Practically, this achieves most of the benefits of PEMF, but it has some drawbacks:

* We still leak the `XHelper` symbol (implementation) to the header file (interface).
* All of the PEMFs implemented by the helper cannot access the data members of X
    directly. They must use C style syntax `x.i_ = 42` instead of the more natural `i_ = 42` provided by the implicit `this` pointer.
* The set of PEMFs are restricted to the class definition of `XHelper`. We cannot arbitrarily introduce new PEMFS without modifying `XHelper` or creating yet another subclass of `XHelper`.
* `XHelper` and all of its member functions cannot have internal linkage.
* `XHelper` static member function signatures cannot use symbols with internal linkage unless `XHelper` itself resides in only one translation unit.
* This technique is obscure and not well known, a first class language feature would be more accessible to new users.
* The fact that this technique was discovered shows a need for PEMFs in the community.

ODR pitfalls
--------------------

Because private class member functions are proposed to be no longer all declared
in the class scope, this proposal adds new ways of violating the ODR. On
possible way is by the instantiation of a template class with and without
a private extension member function defined:

Header:

    template<typename T>
    class Foo {
    public:
      Foo() {
        A(T()); // dependent lookup, overload resolved at instantiation
      }

    private:
      int A(double) {
        return 1;
      }
      // int A(int); -> not declared, extended later
    }

    struct X {
      Foo<int> f; // If Foo instantiated here → A(int) does not participate
    };
    
Implementation:

    template<typename T>
    private int Foo<T>::A(double) {}

    struct Y {
      Foo<int> f; // Foo instantiated here → A(int) does participate
    };


Alternatives and Additions
===================

The current approach uses the private keyword to declare a PEMF or PSEMF.
What are the consequences of this syntax?

Pros:

 - Completely backwards compatible
 - Does not invent new keywords or unusual syntax
 - Private keyword has a natural meaning here

Cons:

 - Novice programmers may ask why they cannot declare public and protected extension member functions.

Might there be a better approach? We will discuss some alternatives.

As an interesting side note: the authors of the original 2013 proposal and
the 2026 proposal came independently with the exact same syntax and solution
for private extension member functions.

Use a different keyword
-------------------------

We could use a keyword other than private. 
Here are some possible candidates, we believe them all to be inferior to private.

* explicit: The explicit keyword could be reused here and it even
    seems somewhat natural as you are "explicitly" defining
    a new member function. All of the current uses of this keyword
    are related to type conversions and it would be better to keep
    it this way for consistency and ease of understanding.
    Do we really want another overloaded keyword like
    static in the language?
* [[attribute]]: This is almost like inventing a new keyword, except
    you're cheating by instead using a generalized attribute.
    Attributes should not be used for back door keywords 
    to implement new language features.
* Invent a new keyword: This probably doesn't even need to be said.
    New keywords are something that should only be introduced if
    there is a really good reason. This is a small proposal that
    does not merit a new keyword.



Extend a class without `private` keyword
------------------------------

Another approach is to avoid the use of a keyword entirely.

    class Foo {};

    void Foo::f1(); //<-PEMF
    static void Foo::f2(); //<-PSEMF

Pros:

 - Completely backwards compatible
 - Avoids the confusion private vs public/protected extension member functions.

Cons:

 - Previously if the user made a typo when defining a class member function, he would get a compiler error. Now he will silently declare a new private extension member function. This error is likely to be caught during link time however.


Reopening the class scope
-------------------------

One possible addition to this proposal would be to allow reopening the class private scope
to add not only new member functions but also typedefs and nested types. This is somewhat
reminiscent of the style of the `extern "C"` feature.
Credit goes to Péter Radics for the initial idea for the idea of reopening the class scope
and Vicente J. Botet Escriba for the handy syntax using private.

    class Foo {};

    private Foo { //<-Reopen Foo's private scope
      
      void f(); //<-PEMF
      static void g(); //<-PSEMF

      int x_; //<-Error: cannot add data members to Foo.
      
      static int y_; //<-Add a static data member.

      using real = float; //<-An alias, which is private to Foo

      class Bar; //<-Forward declaration of a nested private class Foo::Bar

      class Baz {}; //<-Define a nested private class Foo::Baz

      friend class Gaz; //<-Error: Cannot declare extended friends: this breaks encapsulation.
    };

    private static Foo { //<-Reopen Foo's private scope again, this time with internal linkage
      /* stuff */
    };

    namespace {
       private Foo { //<-Reopen yet again with internal linkage using anonymous namespace.
         /* stuff */
       };
    };

A downside of this is that the use of an anonymous namespace for what often
will be the common pattern is verbose. It is also inconsistent to declare the
class in two different namespaces.

Extending other types of private fields of a class
----------------------

The following class fields are, like private member functions, also not part
of the class interface and do not change the ABI of the class:

- Private static data members
- Private aliases
- Private nested classes

One may wonder why we would then allow extension of a class with
private member functions but not these other types of class fields.
The reason is that private extension member functions solve a
genuine problem with practical consequences (discussed in the motivation),
whereas there are simple alternatives for private aliases, private static
member data and private nested classes. Both of these can be moved more
local to the intended place of use with minimal effort and without exposing
the dependencies of these into the header file. In general, such extensibility
would have much less use.

As a consequence, being able to extend a class with private aliases, private
static data or private nested classes would mostly be to have
consistency in the language: all class features that are not part of the
interface and do not influence the ABI are extensible, instead of only one
category of features. While consistency can be a valid argument, allowing
the extension of all these fields would have a highly specific and detailed
effect, and it complicates the language more than when we allow just the
addition of private static members function.

We propose therefore to only allow extension of private static data members.

Explicitly enabling private extension member functions
----------------------

The issue of introducing an access loophole can be mitigated by requiring the
extended class to be marked in some way. A possible syntax for this could be similar
to the idea of reopening the class, where `public` in front of the class starts
the definition of a class that allows extending, and `private` is used to
extend it. Declaring a class without `public` or `private` means the class is
not extendable. For example:

    public class Foo {
      int a;
    }

    private class Foo {
      void SetA(); //<-PEMF
    }

Whereas this would be an error:

    class Foo {
      int a;
    }

    private class Foo { //<-Error: reopening a class not marked as extendable
      void SetA();
    }

This would solve the newly created loophole, where access can be gained to the
private data of a class that is not meant to be accessed. As mentioned before,
the private access is not for hiding secrets but is to protect against
accidentally breaking the invariant of the class. For that, explicit marking
seems to overdo it.

Another argument against this approach is that requiring a class to be marked
to allow extension is the wrong default setting: protecting a class against
extension seems hardly ever relevant.

Allowing external linkage
----------------------

In this proposal, we focus only on PEMFs and PSEMF with internal linkage,
because this covers most of the use-cases. A future extension could allow
external linkage, which would allow declaring extensions that are used by
different translation units, to support the case in which the implementation
of a class is spread over multiple translation units.

We could use the `extern` keyword to denote external linkage. 

The syntax for this idea might look like the following:

    class Foo {};

    // PEMF with internal linkage
    private void Foo::f1(); 
    // PEMF with external linkage
    extern private void Foo::f2();
    // PSEMF with internal linkage
    private static void Foo::f3(); 
    static private void Foo::f3(); 
    // PSEMF with external linkage
    extern private static void Foo::f4();
    extern static private void Foo::f4();


Use `static` for internal linkage
----------------------

The 2013 version of this proposal made PEMF and PSEMF have external linkage,
and used the keyword `static` at a specific place (before `private`) to
specify internal linkage.

The downside of this member function is that the word static has two different
functions within the same line of code, and its meaning depends on its
position:

    class Foo {
    };

    private void Foo::f1(); //<-PEMF
    private static void Foo::f2(); //<-PEMF with internal linkage
    static private void Foo::f3(); //<-PSEMF
    static private static void Foo::f4(); //<-PSEMF with internal linkage

In the discussions on the mailing list around 2013 and in the discussions
in 2026, there was a preference to use internal linkage by default.


Changes compared to the 2013 proposal
====================

The high-level changes between this proposal and the 2013 proposal
\[[N3863](#N3863)\] are:

- This proposal uses internal linkage by default for private extension
  member functions. The 2013 proposal offered this as an alternative, but
  not as default. Making internal linkage the default simplified the use
  of the `static` keyword and is considered a default that is more often
  the desired default.
- References to modules were removed from the proposal. In 2013, a counter
  argument was that modules may solve this issue, which now no longer holds.
  The at that time upcoming modules proposal might have been a reason why
  the proposal didn't progress (No explicit reasons for the lack of
  progression are known.)
- Some new arguments for and against that were mentioned on the std proposal
  mailing list have been incorporated, and with the help of discussions on
  the mailing list, many sections were added or extended with details.

Acknowledgements
====================

Thank you to everyone on the std proposals forum for feedback and suggestions.

References
==================
* <a name="Proposal2013"></a>[Proposal2013] stdcxx-privext Git repository of M. Fioravante, <https://github.com/mateofio/stdcxx-privext>.
* <a name="N3863"></a>[N3863] M. Fioravante, Private Extension Methods, open-std.org <https://open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3863.html>.
* <a name="Lakos01"></a>[Lakos01] Lakos, John. *Large-Scale C++ Software Design*, Addison-Wesley, July 1996, ISBN 0201633620.
* <a name="KDEABI"></a>[KDEABI] *Policies/Binary Compatibility Issues With C++ - KDE TechBase*, <http://techbase.kde.org/Policies/Binary_Compatibility_Issues_With_C++>.
* <a name="GotW076"></a>[Gotw076] Sutter, Herb. *GotW #76: Uses and Abuses of Access Rights*, <http://www.gotw.ca/gotw/076.htm>.

