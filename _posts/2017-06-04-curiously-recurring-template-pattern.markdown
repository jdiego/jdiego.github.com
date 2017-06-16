---
layout: post
title:  "The Curiously Recurring Template Pattern"
date:   2017-06-04 10:00
categories: c++ programming template
---

The Curiously Recurring Template Pattern (CRTP) is a C++ idiom whose name was coined by James Coplien in 1995, 
in early C++ template code.

The "C" in CRTP made it travel the years in the C++ community by being this: a Curiosity. We often find definitions 
of what CRTP is, and it is indeed an intriguing construct.
But what's even more interesting is what CRTP means in code, as in what you can express and achieve by using it, 
and this is the point of this series.

# The Curiously Recurring Template Pattern in C++

C++ provides pretty good support for polymorphism by means of virtual functions. This is dynamic polymorphism 
(or runtime polymorphism), since the actual function to be called is resolved at runtime. It's usually 
implemented by adding a hidden pointer in every object of a class with virtual functions. The pointer will point for 
any given object at the actual functions to call for it, so even when the compiler only knows this object through a pointer 
to a base class, it can generate correct code.

The problem with dynamic polymorphism is its runtime cost. This usually consists of the following components [1]:

* Extra indirection (pointer dereference) for each call to a virtual method.
* Virtual methods usually can't be inlined, which may be a significant cost hit for some small methods.
* Additional pointer per object. On 64-bit systems which are prevalent these days, this is 8 bytes per object. 
For small objects that carry little data this may be a serious overhead.

Although in general dynamic polymorphism is a great tool, due to the aforementioned costs some applications prefer not to 
use it, at least for some performance-critical classes. So what is the alterantive?

It turns out that using templates, C++ provides an alternative way to implement polymorphism without the extra costs. 
There's a catch, of course - the types of objects have to be resolvable by the compiler at compile-time. This is called 
static polymorphism (or "simulated dynamic binding").

This technique has a name - it's called Curiously Recurring Template Pattern (CRTP from now on).

>The CRTP consists in inheriting from a template class and use the derived class itself as a template parameter of the base class.

This is what it looks like in code:

```c++
#include <iostream>
using namespace std;

template <typename Child>
struct Base
{
    void interface()
    {
        Child& derived = static_cast<Child&>(*this);
        derived.implementation();
    }
};

struct Derived : Base<Derived>
{
    void implementation()
    {
        cerr << "Derived implementation\n";
    }
};

int main()
{
    Derived d;
    d.interface();  // Prints "Derived implementation"
}
```
The purpose of doing this is using the derived class in the base class. From the perspective of the base object, 
the derived object is itself, but downcasted. Therefore the base class can access the derived class by static casting 
itself into the derived class.


```c++
 void interface()
 {
    T& derived = static_cast<T&>(*this);
    derived.implementation();
}
```

The key to the technique is the strange template trickery that's being used: note that `Derived` inherits from `Base<Derived>`. 
What gives? The idea is to "inject" the real type of the derived class into the base, at compile time, allowing the `static_cast` 
of this in the interface to produce the desired result.

>Note that contrary to typical casts to derived class, we don't use `dynamic_cast` here. A `dynamic_cast` is used when you 
want to make sure at run-time that the derived class you are casting into is the correct one. But here we don’t need this 
guarantee: the Base class is designed to be inherited from by its template parameter, and by nothing else. Therefore it 
takes this as an assumption, and a `static_cast` is enough.




## What could go wrong

If two classes happen to derive from the same CRTP base class we likely get to undefined behaviour when the 
CRTP will try to use the wrong class:

```c++
class Derived1 : public Base<Derived1>
{
    ...
};
 
class Derived2 : public Base<Derived1> // bug in this line of code
{
    ...
};
```

There is a solution to prevent this. It consists in adding a private constructor in the base class, and making 
the base class friend with the templated class:


```c++
template <typename T>
class Base
{
public:
    // ...
private:
    Base(){};
    friend T;
};
```

Indeed, the constructors of the derived class have to call the constructor of the base class (even if you don’t write it 
explicitly in the code, the compiler will do his best to do it for you). Since the constructor in the base class is private, 
no one can access it except the friend classes. And the only friend class is the template class! So if the derived class is 
different from the template class, the code doesn't compile. Neat, right? So, here we go

```c++

#include <iostream>
using namespace std;

template <typename Child>
struct Base
{

    void interface()
    {
        Child& derived = static_cast<Child&>(*this);
        derived.implementation();
    }
    private:
        Base(){}
        friend Child;
};

struct Derived1 : Base<Derived1>
{
    void implementation()
    {
        cerr << "Derived 1 implementation\n";
    }
};
struct Derived2 : Base<Derived1>
{
    void implementation()
    {
        cerr << "Derived 2 implementation\n";
    }
};


int main()
{
    {
        Derived1 d;     // OK
        d.interface();  // Prints "Derived implementation"
    }
    {
        /*
            Clang error report:

            Derived2 d; error: call to implicitly-deleted default constructor of 'Derived2'
                 ^
            struct Derived2 : Base<Derived1>
            note: default constructor of 'Derived2' is implicitly deleted because base
            class 'Base<Derived1>' has an inaccessible default constructor
        */
        Derived2 d;     // ERROR

        d.interface();  // Prints "Derived implementation"
    }
}
```

Another risk with CRTP is that methods in the derived class will hide methods from the base class with the same name. 
As explained in Effective C++ Item 33, the reason for that is that these methods are not virtual. Therefore you want 
to be careful not to have identical names in the base and derived classes:

```c++
class Derived : public Base<Derived>
{
public:
    void doSomething(); // oops this hides the doSomething methods from the base class !
}
```

The first time I was shown CRTP my initial reaction was: "wait, I didn't get it". Then I saw it a couple 
of other times and I got it. So if you didn't get how it works, just re-read the post a couple of times.

## What the Curiously Recurring Template Pattern can bring to our C++ code

I don’t know about you, but the first few times I figured how the CRTP worked I ended up forgetting soon after, 
and in the end could never remember what the CRTP exactly was. This happened because a lot of definitions of 
CRTP stop there, and don’t show you what value the CRTP can bring to your code.

Synthetic examples are prone to not being exciting, and this one is no exception. Why not just implement interface in 
Derived, if its type is known at compile-time anyway, you may ask. This is a good question, which is why I plan to 
provide more examples to show how CRTP is useful.

Here I am presenting the one that I see most in code, Adding Functionality, and another one that is interesting 
but that I don’t encounter as often: creating Static Interfaces.

In order to make the code examples shorter, I have omitted the private-constructor-and-template-friend trick. 
But in practice you would find it useful to prevent the wrong class from being passed to the CRTP template.

## Adding functionality

Some classes provide generic functionality, that can be re-used by many other classes.

To illustrate this, let’s take the example of a class representing a sensitivity. A sensitivity is a 
measure that quantifies how much a given output would be impacted if a given input to compute it were to 
vary by a certain amount. This notion is related to derivatives. Anyway if you’re not (or no longer) 
familiar with maths, fear not: the following doesn't depend on mathematical aspects, the only thing that 
matters for the example is that a sensitivity has a value.

```c++
class Sensitivity
{
public:
    double get_value(void) const;
    void set_value(double value);
    // ...
};
```

Now we want to add helper operations for this sensitivity, like scaling it (multiplying it by a constant value), 
and say squaring it or setting it to the opposite value (unary minus). We can add the corresponding methods in 
the interface. I realize that in this case it would be good practice to implement these functionalities as 
non-member non-friend functions, but bear with me just a moment and let’s implement them as methods, in order 
to illustrate the point that is coming afterwards. We will come back to this.

```c++
class Sensitivity
{
public:
    double get_value(void) const;
    void set_value(double value);
 
    void scale(double multiplicator)
    {
        set_value(get_value() * multiplicator);
    }
    void square(void)
    {
        set_value(get_value() * get_value());
    }
    void set_to_opposite(void)
    {
        scale(-1);
    };
    // ...
};
```

So far so good. But imagine now that we have another class, that also has a value, and that also needs the 3 
numerical capabilities above. Should we copy and paste the 3 implementations over to the new class?
By now I can almost hear some of you scream to use template non-member functions, that would accept any class 
and be done with it. Please bear with me just another moment, we will get there I promise.

This is where the CRTP comes into play. Here we can factor out the 3 numerical functions into a separate class:

```c++
template <typename T>
struct NumericalFunctions
{
    void scale(double multiplicator);
    void square();
    void set_to_opposite();
};
```

and use the CRTP to allow Sensitivity to use it:

```c++
class Sensitivity : public NumericalFunctions<Sensitivity>
{
public:
    double get_value() const;
    void set_value(double value);
};
```
For this to work, the implementation of the 3 numerical methods need to access the 
`get_value` and `set_value` methods from the `Sensitivity` class:


```c++
template <typename T>
struct NumericalFunctions
{
    void scale(double multiplicator)
    {
        T& underlying = static_cast<T&>(*this);
        underlying.set_value(underlying.get_value() * multiplicator);
    }
    void square()
    {
        T& underlying = static_cast<T&>(*this);
        underlying.set_value(underlying.get_value() * underlying.get_value());
    }
    void set_to_opposite()
    {
        scale(-1);
    };
};
```
This way we effectively added functionality to the initial `Sensitivity` class by using the CRTP. 
And this class can be inherited from by other classes, by using the same technique.

## Why not non-member template functions?
Ah, there we are.
Why not use template non-member functions that could operate on any class, including `Sensitivity` 
and other candidates for numerical operations? They could look like the following:

```c++
template <typename T>
void scale(T& object, double multiplicator)
{
    object.set_value(object.get_value() * multiplicator);
}
 
template <typename T>
void square(T& object)
{
    object.set_value(object.get_value() * object.get_value());
}
 
template <typename T>
void set_to_opposite(T& object)
{
    object.scale(object, -1);
}
```

What’s all the fuss with the CRTP?
There is at least one argument for using the CRTP over non-member template functions: the CRTP shows in the interface.

With the CRTP, you can see that `Sensitivity` offers the interface of `NumericalFunctions`:


```c++
class Sensitivity : public NumericalFunctions<Sensitivity>
{
public:
    double get_value() const;
    void set_value(double value);
};
```

And with the template non-member functions you don’t. They would be hidden behind a 
include directive somewhere.

And even if you knew the existence of these 3 non-member functions, you wouldn't 
have the guarantee that they would be compatible with a particular class (maybe they call `get()` or 
`get_data()` instead of `get_value()` ?). Whereas with the CRTP the code binding `Sensitivity` has 
already been compiled, so you know they have a compatible interface.


## More realistic example

The following example is much longer - although it is also a simplification. It presents a generic base class for 
visiting binary trees in various orders. This base class can be inherited to specify special handling of some 
types of nodes. Here is the tree node definition and the base class:

```c++
struct TreeNode
{
    enum Kind {RED, BLUE};

    TreeNode(Kind kind_, TreeNode* left_ = NULL, TreeNode* right_ = NULL)
        : kind(kind_), left(left_), right(right_)
    {}

    Kind kind;
    TreeNode* left;
    TreeNode* right;
};
template <typename Derived>
class GenericVisitor
{
public:
    void visit_preorder(TreeNode* node)
    {
        if (node) {
            dispatch_node(node);
            visit_preorder(node->left);
            visit_preorder(node->right);
        }
    }

    void visit_inorder(TreeNode* node)
    {
        if (node) {
            visit_inorder(node->left);
            dispatch_node(node);
            visit_inorder(node->right);
        }
    }

    void visit_postorder(TreeNode* node)
    {
        if (node) {
            visit_postorder(node->left);
            visit_postorder(node->right);
            dispatch_node(node);
        }
    }
    void handle_RED(TreeNode* node)
    {
        std::cerr << "Generic handle RED\n";
    }

    void handle_BLUE(TreeNode* node)
    {
        std::cerr << "Generic handle BLUE\n";
    }
private:
    // Convenience method for CRTP
    Derived& derived()
    {
        return *static_cast<Derived*>(this);
    }
    void dispatch_node(TreeNode* node)
    {
        switch (node->kind) {
            case TreeNode::RED:
                derived().handle_RED(node);
                break;
            case TreeNode::BLUE:
                derived().handle_BLUE(node);
                break;
            default:
                assert(0);
        }
    }
};
//
class SpecialVisitor : public GenericVisitor<SpecialVisitor>
{
public:
    void handle_RED(TreeNode* node)
    {
        std::cerr << "handle to RED type is special handle_RED\n";
    }
};

    
```

Now you can easily implement special handling of various kinds of nodes in subclasses, 
and use visiting services provided by the base class.

To reiterate - this is a simplified example, as there are only two kinds of nodes, but in reality 
there can be many more. Such code would be quite useful inside compilers, where the source is usually 
parsed into a tree with many different kinds of nodes. Multiple passes in the compiler then process 
the trees by implementing their own visitors. As a matter of fact, the Clang compiler frontend has 
such a class, named RecursiveASTVisitor, which implements a much more complete version of the visitor 
displayed above.

Without CRTP, there's no way to implement such functionality except resorting to dynamic polymorphism 
and virtual functions.


## Static interfaces

The second usage of the CRTP is to create static interfaces. In this case, the base class does represent the interface and the 
derived one does represent the implementation, as usual with polymorphism. But the difference with 
traditional polymorphism is that there is no virtual involved and all calls are resolved during 
compilation.

```c++
// Code from StackOverflow
template <class Derived>
struct base 
{
    void foo() 
    {
        // Here we are specifying the interface statically for the structure of types:
        static_cast<Derived *>(this)->foo();
    };
};

struct my_type : base<my_type> 
{
  void foo(); // required to compile.
};

struct your_type : base<your_type> 
{
  void foo(); // required to compile.
};
```

Let’s take an CRTP base class modeling an amount, with one method,

```c++
template <typename T>
class Amount
{
public:
    double get_value() const
    {
        return static_cast<T const&>(*this).get_value();
    }
};
```
Say we have two implementations for this interface: one that always returns a constant, 
and one whose value can be set. These two implementations inherit from the 
CRTP Amount base class:

```c++
class Constant42 : public Amount<Constant42>
{
public:
    double get_value() const {return 42;}
};
 
class Variable : public Amount<Variable>
{
public:
    explicit Variable(int value) : value_(value) {}
    double get_value() const {return value_;}
private:
    int value_;
};
```

Finally, let’s build a client for the interface, that takes an amount and that prints it to the console:

```c++
template<typename T>
void print(Amount<T> const& amount)
{
    std::cout << amount.get_value() << '\n';
}
```
The function can be called with either one of the two implementations:

```c++
Constant42 c42;
print(c42);         // Prints 42

Variable v(43);
print(v);           // Prints 43
```

The most important thing to note is that, although the `Amount` class is used polymorphically, there isn't 
any virtual in the code. This means that the polymorphic call has been resolved at compile-time, thus 
avoiding the run-time cost of virtual functions. 
From a design point of view, we were able to avoid the virtual calls here because the information of which 

>In a postmodern C++ World, this technique is not the best one for static interfaces, and nowhere as good 
as what concepts are expected to bring. Indeed, the CRTP forces to inherit from the interface, while 
concepts also specify requirements on types, but without coupling them with a specific interface. This 
allows independent libraries to work together.

## Getting rid of static_cast

Writing repeated static_casts in CRTP base classes quickly becomes cumbersome, 
as it does not add much meaning to the code. It would be nice to factor out these `static_casts`. 
This can be achieved by forwarding the underlying type to a higher hierarchy level:

```c++
template <typename T>
struct crtp
{
    T& underlying() { return static_cast<T&>(*this); }
    // Plus it deals with the case where the underlying object is 
    // const, which we hadn’t mentioned yet. But the real life is always
    // worse.....
    T const& underlying() const { return static_cast<T const&>(*this); }
};
```
As Bruce Buffer usually say: It's time .....

```c++
template <typename T>
struct NumericalFunctions : crtp<T>
{
    void scale(double multiplicator)
    {
        this->underlying().set_value(this->underlying().get_value() * multiplicator);
    }
    ...
};
```

Note that the `static_cast` is gone and a `this->` appeared. Without it the code would not compile. 
Indeed, the compiler is not sure where underlying is declared. Even if it is declared in the template 
class `crtp`, in theory nothing guarantees that this template class won’t be specialized and rewritten 
on a particular type, that would not expose an underlying method. For that reason, names in template 
base classes are ignored in C++.

Using `this->` is a way to include them back in the scope of functions considered to resolve the call. 
There are other ways to do it, although they are arguably not as adapted to this situation. In any case, 
you can read all about this topic in Effective C++ Item 43.

Anyway, the above code relieves you from writing the `static_casts`, which become really cumbersome when they are several of them.
All this works if you class only add one functionality via CRTP, but it stops working if there are more.

## Adding several functionalities with CRTP

For the sake of the example let’s split our CRTP classes into two: one that scales values and one 
that squares them:

```c++
template <typename T>
struct Scale : crtp<T>
{
    void scale(double multiplicator)
    {
        this->underlying().set_value(this->underlying().get_value() * multiplicator);
    }
};
 
template <typename T>
struct Square : crtp<T>
{
    void square()
    {
        this->underlying().set_value(this->underlying().get_value() * this->underlying().get_value());
    }
};
class Sensitivity : public Scale<Sensitivity>, public Square<Sensitivity>
{
public:
    double get_value() const { return value_; }
    void set_value(double value) { value_ = value; }
 
private:
    double value_;
};

```
This looks ok at first glance but does not compile as soon as we call a method of either one of the base class!
`error: 'crtp<Sensitivity>'` is an ambiguous base of `Sensitivity` The reason is that we have a diamond inheritance here.
I tried to solve this with virtual inheritance at first, but quickly gave this up because I didn't find how to do 
it simply and without impacting the clients of the `crtp` class. If you have a suggestion, please, voice it!

Another approach is to steer away from the diamond inheritance (which sounds like a good idea), by having every 
functionality (scale, square) inherit from its own `crtp` class. And this can be achieved by CRTP!
Indeed, we can add a template parameter to the `crtp` class, corresponding to the base class.
Note the addition of the `CrtpType` template parameter. 

> The private-constructor-and-friend-with-derived technique should also be applied on this template template parameter here.

```c++
template <typename T, template<typename> class CrtpType>
struct crtp
{
    T& underlying() { return static_cast<T&>(*this); }
    T const& underlying() const { return static_cast<T const&>(*this); }
private:
    crtp(){}
    friend CrtpType<T>;
};
```
Note that the template parameter is not just a `typename`, but rather a `template<typename>` class. 
This simply means that the parameter is not just a type, but rather a template itself, templated over 
a type whose name is omitted. For example `CrtpType` can be `Scale`.

This parameter is here to differentiate types only, and is not used in the implementation of `crtp` 
(except for the technical check in the friend declaration). Such an unused template parameter is called 
a "phantom type" (or to be more accurate here we could call it a phantom template).


