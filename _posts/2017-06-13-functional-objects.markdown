---
layout: post
title:  "Functional Objects"
date:   2017-06-13 00:02
categories: c++ programming functional
---
# Functional Objects

In this posts, we will see all the things that we can use as functions in C++. This is a bit dry topic, but it is necessary for the deeper understanding of functions in C++ and for achieving that sweet-spot of using higher level abstractions without incurring performance penalties. 

If we use duck-typing (if it walks like a duck, and quacks like a duck, it is a duck), we can say that anything that can be called like a function, is a function object. Namely, if we can write a name of an entity, followed by arguments in parenthesis like f(arg1, arg2, ...) that entity is a function object. 


## Functions and functional objects

A function is a named group of statements that can be invoked from other parts of the program, or from the function itself in the case of recursive functions. C++ provides several slightly different ways to define a function:
```C++

// The 'old-school' C-like syntax
int max(int arg1, int arg2) 
{ 
    ... 
}

// With a trailing return type, available since C++11 
auto max(int arg1, int arg2) -> int 
{ 
    ... 
}

//With the automatic return type deduction, available since C++ 14
auto max(int arg1, int arg2){
    double x;
    ....
    return x; // The return type will be a double
}
```
While some people prefer to write functions with the trailing return type even in the cases when it is not necessary, it is not a common practice. This syntax is mainly useful when writing template functions where the return type depends on the types of the arguments. 

Since C++14, it is also allowed to omit the return type and to rely on the compiler to deduce it automatically from the expression in thereturn statement. The type deduction in this case follows the rules of template argument deduction.

In the case of functions with multiple return statements, all of them need to return of the same type. If the types differ, the compiler will be report an error, even if one of them implicitly cast to the other. Once the return type is deduced, it can be used in the rest of the function. This allows writing recursive functions with automatic return type deduction.

```c++
auto factorial(int n)
{
    if (n == 0)
        return 1;   // Deducing the return type to be int
    else
        return factorial(n - 1) * n; // It's already known that factorial returns 
                                     // an int, and multiplying ints returns an 
                                     // int, so we are good. 
}
```

Swapping the if and else branches would result in an error because the compiler would arrive at the recursive call to factorial before the return type of the function was deduced. This means that when writing recursive functions, it is necessary either to specify the return type, or to first write the return statements that do not recurse.

When we writing generic functions that just forward the result of another function without modifying it, we do not know in advance what function will be passed to us, and we can not know whether we should pass its result back to the caller as a value, or as a reference. If we pass it as a reference, it might return a reference to a temporary value which will produce undefined behaviour. And if we pass it as a value, it might make an unnecessary copy of the result. Copying will have performance penalties, and will sometimes even be semantically wrong -- the caller might expect a reference to an existing object.

If we want to perfectly-forward the result, that is, to return the result returned to us without any modifications, we can use the `decltype(auto)` as the specification 
of the return type.
```c++
template <typename Object, typename Function>
decltype(auto) call_on_object(Object &&object, Function function)
{
    return function(std::forward<Object>(object));
}
```
In the above code snippet, we have a simple template function that takes an object, and a function that should be invoked on that object. 
We are perfectly forwarding the object argument to the function, and by using the `decltype(auto)` as the return type, we are perfectly 
forwarding the result of the function back to our caller.


## Perfect Forwarding for Arguments

We sometimes need to write a function that wraps in another function. In that case, we have the problem of how to pass the arguments
from the wrapper to the function we need to call. If we have the same setup as in the last snippet, the user is passing us a function 
we know nothing about. We do not know how it expects us to pass it the argument.

The first option we have is to make our `call_on_object` function accept the `object` argument by-value, and pass it on to the wrapped
function.  This will lead to problems if the wrapped function accepts a reference to the object because it needs to change it. 
The change will not be visible outside of the `call_on_object` function since it will be performed on the local copy of the object 
that exists only in the `call_on_object` function.

```c++
template <typename Object, typename Function>
decltype(auto) call_on_object(Object object, Function function) // argument by-value
{
    // Changes made by function aren't visible outside of the call_on_object function.
    return function(object);
}
```

The second attempt is to pass the object by-reference. This would make the changes to the `object` visible to the caller of 
our function. But we will have a problem if function does not actually change the object, but accepts the argument as a 
`const-reference`. The caller  will not be able to invoke `call_on_object` on a constant object, nor on a temporary. So, the idea 
to accept the object as an ordinary reference still is not right. And we can not make that reference `const`, since the function 
might want to change the original object.

Some pre-C++11 libraries approached this problem by creating overloads for both `const` and `non-const` references. This is not 
practical because the number of overloads grows exponentially with the number of arguments that need to be forwarded.

This is solved in C++11 with the `forwarding references` (formerly known as `universal references`, as named by Scott Meyers). A forwarding reference is written as a `double reference` on a `templated type`. In the following code, the `fwd` argument is a forwarding reference to type `T`, while value is not (it is a normal `rvalue` reference).

```c++
template <typename T>
void foo(T&& fwd, int&& value) // fwd is a forwarding reference
                            // value is a normal rvalue reference
{ ... }
```

When foo is called on an `lvalue` of type `A`, then `T` resolves to `A&` and hence, by the reference collapsing rules, the argument
type effectively becomes `A&`. But, when foo is called on an `rvalue` of type `A`, then `T` resolves to `A`, and  hence the argument
type becomes `A&&`. 

The forwarding reference allows us to accept both const and non-const objects, and temporaries. Now, we only need to pass that argument 
on, and pass it on in the same value-category as we have received it, which is exactly what `std::forward` does.

> Without the use of `std::forward`, everything would work quite nicely, except that `object` would always see as its argument something that has a name for `call_on_object`, and such a thing is an `lvalue`. Another way of putting this is to say that `std::forward`'s purpose is to forward the information whether at the call site, the wrapper saw an `lvalue` or an `rvalue`.

## Function pointers

A function pointer is a variable that stores the address of a function that can later be called through that pointer. They are a 
low-level construct C++ inherited from the C programming language that allows writing polymorphic code. The (runtime) polymorphism 
is achieved simply by changing which function the pointer points to, thus changing the behaviour when that function pointer is called.

Function pointers (and references) are also function objects, since they can be called like ordinary functions. Additionally, all types 
that can be implicitly converted to a function pointer are also function objects, as can be seen in the following listing, but this 
should be avoided. We should prefer proper function objects (which we will see in a few moments) since they are more powerful and 
easier to deal with. Objects that can be converted to function pointers can be useful when we need to interface with a C library 
that takes a function pointer and we need to pass it something more complex than a simple function.
```c++
#include <iostream>

int ask(void) { return 42;}

typedef decltype(ask) * function_ptr;

class convertible_to_function_ptr 
{ 
    public:
        // The casting operator can only return a pointer to function
        // While it can return different functions depending on some
        // condition, it can not pass any data to them without resorting
        // to dirty tricks.
        operator function_ptr () const
        {
             return ask;
        }
};
int main(int argc, char *argv[])
{
    // CASE 1: A pointer to function
    auto ask_ptr = &ask;
    std::cout << ask_ptr() << std::endl;
    
    // CASE 2: A reference to function
    auto& ask_ref = ask;
    std::cout << ask_ref() << std::endl;

    // CASE 3: An object that can be implicitly converted to a function pointer
    convertible_to_function_ptr ask_wrapper;
    std::cout << ask_wrapper() << std::endl;


    return 0;
}
```

## Call operator overloading

Instead of creating types that can be implicitly converted to function pointers, C++ allows a much nicer way to create 
new types that behave like functions -- by creating classes and overloading their `call operators`. Unlike other operators, 
the call operator can have an arbitrary number of arguments, and the arguments can have any type, so we can create a 
function object of any signature we like. The syntax for overriding the call operator is as simple as defining a member 
function -- just with a special name of `operator()` -- we specify the return type, and all the arguments that the 
function needs.

```c++
// C++ Style
class function_object 
{ 
    public:
        return_type operator() (arguments)
        {
            ...
        } 
};
```
These C++ function objects have one advantage compared to the ordinary functions — each instance can have their own state, 
be it mutable or not. The state is used to customize the behaviour of the function, without the caller having to specify 
it.

Say we have a list of people, each person has a name, age and a few things that we do not care about at the moment. Now, 
we want to allow the user to count the number of people that are older than some specified age. If the age limit was fixed, 
we could create an ordinary function that checks whether the person is older than that predefined value.

This solution is not scalable because we would need to create separate functions for all different age limits that we need, 
or to employ some error-prone hack like having the age limit we are currently testing against saved in a global variable.
Instead, it would be much wiser to create a proper function object that keeps the age limit as its inner state. 
This will allow us to write the implementation of the predicate only once, and instantiate it several times for different 
age limits.

```c++
#include <iostream>
#include <algorithm>
#include <vector>
#include <string>

class older_than
{
public:
    older_than(int limit): m_limit(limit) {}
    bool operator()(const person_t& person) const
    {
        return person.age > m_limit;
    }
private:
    int m_limit;
};
int main(int , char *[])
{
    std::vector<person_t> persons = {
        {"Bob", 42},
        {"Alice", 33},
        {"Donald", 29},
        {"Mario", 62},
    };
    older_than older_than18(18);
    std::cout << std::count_if(std::begin(persons), std::end(persons), older_than_42) << '\n';
    std::cout << std::count_if(std::begin(persons), std::end(persons), older_than(20)) << '\n';
    std::cout << std::count_if(std::begin(persons), std::end(persons), older_than18) << '\n';
    return 0;
}
```
The `std::count_if` algorithm does not care what we have passed to it as a predicate function, as long as it can 
be invoked like a normal function - and by overloading the call operator we did just that. 

We are here relying on the fact that template functions like `std::count_if` do not require arguments of specific types 
to be passed to them. They require only that the argument has all the features needed by the template function implementation. 
In this case, `std::count_if` requires the first two arguments to behave like forward iterators, and the third argument to behave 
like a function. Some people call this a weakly-typed part of C++, but that is a misnomer --this is still strongly typed, it 
is just that templates are using duck-typing. There are no runtime checks whether the type is a function object or not, 
all the needed checks are performed at compile-time.

> Function objects terminology: Classes with the overloaded call operator are often called `functors` in various places. 
This term is highly problematic because it is already used for something quite different in the category theory. While 
we could ignore this fact, the category theory is important for functional programming (and programming in general), so 
we will keep calling them function objects, just for avoid troubles.

## Creating generic function objects

In the previous example, we have created a function object that checks whether a person is older than some predefined age limit. 
This solved the problem of not needing to create different functions for different age limits, bit it is still overly restricting 
-- it accepts only persons as input. There are quite a few types that could have the age information in them -- from concrete 
things like cars and pets, to more abstract ones like software projects.

Again, instead of writing a different function object for each type, we want to be able to write the function object once, 
and to be able to use it for any type that has the age information.

We could solve this in an object-oriented way by creating a super-class with a virtual `age()` member function, but that would 
have runtime performance penalties, and it would force all the classes that want to support our `older_than` function object 
to inherit from that super-class. This ruins encapsulation, so we are not even going to bother ourselves with this approach.

We could have just made the call operator a templated member function. This way, we will not need to specify the type when we
instantiate the older_than function object, and the compiler will automatically deduce the type of the argument when invoking 
the call operator.

```c++
class older_than
{
public:
    older_than(int limit): m_limit(limit) {}
    template<typename T>
    bool operator()(T&& object) const
    {
        return std::forward<T>(object).age() > m_limit;
    }
private:
    int m_limit;
};
```

So far, we have seen how to write different types of function objects in C++. Function objects like these are perfect for 
passing them to the algorithms from the standard library, or to our own higher-order functions. The only problem is that 
their syntax is a bit verbose and that we need to define them outside of the scope we are using them in.


## Lambdas and closures

 It is sometimes tedious having to write a proper function, or even a whole class in order just to be able to call an algorithm 
 from the standard library, or some other higher-order function. It can also be seen as bad software design, since we are 
 forced to create and name a function that might not be useful to anyone else, and is used only in one place in the 
 program - as an argument to some algorithm call.

 Fortunately, C++11 introduced lambdas, which are a syntactic sugar for creating anonymous (unnamed) function objects. 
 Lambdas allow us to create function objects inline -- at the place where we want to use them instead of having to do it 
 outside of the function we are currently writing. Lets see this on an example-- assume that we have a group of people, 
 and we want to collect only females from that group.

 We have seen how we can use the `std::copy_if` to filter a collection of people based on their gender. We relied on 
 the fact that a free function that accepts a person and returns `true` if that person was female exists.

 Instead, in C++, we can just use a lambda, and achieve the same effect while keeping the code localized and 
 not polluting the program name space.

 ```c++
// Getting all females from a group of people. We have been using a predicate which 
// was a free-standing function for this. Now, we are going to do the same with 
// a lambda.
std::copy_if(
    people.cbegin(), people.cend(), std::back_inserter(females), 
    // Here, we told the compiler to create an anonymous function object 
    // that is used only here, a function that takes a constant reference 
    // to a person, and returns whether that person is a female or not, 
    // and to pass that function object to the std::copy_if algorithm
    [] (person_t const& person) 
    {
        return person.gender() == person_t::female;
    }
 );
```
Syntactically, lambda expressions in C++ have three main parts: a head (`[]`), an argument list, and the body.
The head of the lambda specifies which variables from the surrounding scope will be visible inside of 
the lambda body (We will see how the C++14 expands on this in the section Lambdas in C++14).
The variables can be captured as values (default mode) or by references (specified by prefixing the variable name
with an ampersand). 

If they are passed in as values, their copies will be stored inside the lambda object itself, while if they 
are passed in as references, only a reference to the original variable will be stored. It is also allowed 
not to specify all the variables that need to be captured, but to tell the compiler to capture all variables 
from the outer scope that are used in the body. If you want to capture all variables by - value, you can 
define the lambda head to be `[=]`, and if you want to catch them by - reference, you could write `[&]`. 
These can even be combined. Lets see a few examples of different lambda heads:

* `[a, &b]` - `a` is captured by-value, `b` is by-reference
* `[]`  - a lambda that does not use any variable from the surrounding scope.These lambdas do not have 
    any internal state and can be implicitly cast to ordinary function pointers;
* `[=]` - captures all variables that are used in the lambda body by-value
* `[&]` - captures all variables that are used in the lambda body by-reference 
* `[this]` — captures `this` pointer by-value; Note that, conceptually speaking, capturing `this` 
        by reference doesn't make a whole lot of sense, since you can't change the value of `this`,
        once that it's a constant pointer, and you can only use it as a pointer to access members 
        of the class or to get the address of the class instance.
* `[&, a]` — captures all variables by-reference, except `a` which is captured by-value;
* `[=, &a]` — captures all variables by-value, except `a` which is captured by-reference;

Lambdas in C++ are a nicer syntax to create and instantiate a new function object and anything else.

Lets see this on another example. We are still going to work with a list of people, but this time they 
will be employees in a company. The company is separated into different teams where each team has a name. 
We are going to represent a company using a simple `company_t` class. This class will have a member function 
to get the name of the team each employee belongs to. Our task is to implement a member function that will 
accept the team name, and return how many employees it contains. We can do it by checking which team each 
employee belongs to, and counting only those whose team matches the function arguments. Here is the setup:

```c++
class company_t
{
    public:
        std::string team_name_for(const person_t &) const;
        int count_team_members(const std::string& team_name) const;

    private:
        std::vector<person_t> m_employees;

};

std::string company_t::team_name_for(const person_t& person) const
{
    // Just for testing
    if (person.gender() == person_t::gender_t::female)
        return "Team1";
    else
        return "Team2";
}

int company_t::count_team_members(const std::string &team_name) const
{
    return std::count_if(
        m_employees.begin(), m_employees.end(),
        // We need to capture the this pointer because we are using 
        // a non-static member function: team_name_for, and we have 
        // captured team_name because we need to use it for comparison
        [this, &team_name](const person_t& employee)
        {
            return team_name_for(employee) == team_name;
        }
    );
}
```

So, what actually happens in a C++ compiler when we write something like this?
It will create a new class that has two members - a pointer to the `company_t`
object and a reference to a `std::string` - as you can notice, one for each 
captured variable. And it will implement the call operator on that class with
the same arguments and the same body of the lambda. It will create a class
equivalent to the following:

```c++
class lambda_implementation
{
public:
    lambda_implementation(const company_t* _this, const std::string& team_name)
        : m_this(_this), m_team_name(team_name)
    {}
    // Note: The call operator is const by default
    bool operator()(person_t const& employee) const
    {
        return m->this->team_name_form(employee)
    }
private:
    const company_t* m_this;
    const std::string& m_team_name;
};
```
One thing worth noting is that the call operator of lambdas is constant by default.
If you want to change the value of the captured variables, you will need to declare the lambda as mutable.
In the following example, we are going to use the `std::for_each` algorithm to write all words beginning 
with an uppercase letter, and use the count variable to count how many words we wrote. 
Obviously, there are better ways to do this, but the point here is to demonstrate how mutable lambdas work.

```c++
int count = 0;
std::vector<std::string> words { "An", "ancient", "pond" };
std::for_each(
    words.cbegin(), words.cend(), 
    // we are capturing count by reference so that any changes 
    // we make to it are visible outside of the lambda
    // mutable comes after the argument list, and tells the compiler 
    // that the call operator on this lambda should not be const
    [&count](const std::string &word) mutable
    {
        if (isupper(word[0])) { 
            std::cout << word << " "; 
            ++count;
        }
    }
);
```

## Lambdas in C++14

While lambdas in C++11 are quite handy in a lot of cases, they do have limitations compared to the hand-written 
classes with the call operator. By writing our own classes, we were able to create as many member variables as 
we want, without needing to tie them to a variable in the surrounding scope. We could initialize them to some 
fixed value, to initialize them to a result of some function and similar. While this might sound like it is not 
a big deal, one could always declare a local variable with the desired value before creating the lambda, in 
some cases it is essential. Lambdas in C++11 can not capture objects by value that are are movable, but not 
copyable, i.e, classes that define the move-constructor, but that do not have the copy-constructor.

The most obvious example of this issue is when you want to give the ownership of a `std::unique_ptr` to a 
lambda. For example, assume that we want to create a network request, and we have the session data stored
int a unique pointer. We are creating a network request, and scheduling a lambda to be executed when that 
request is completed. We still want to be able to access the session data in the completion handler, 
so we need to capture it in the lambda.

```c++
// A C++11 code
std::unique_ptr<session_t> session = create_session();
auto request = server.request("GET /", session->id());
// We are scheduling a lambda to be executed when that request is completed.
request.on_completed(
    [session] // Error: There isn't no copy-constructor for std::unique_ptr<session_t>
    (response_t response){ ...}
);
```
C++14 introduces generalized lambda captures which not only solve this problem, but also allow you to 
define arbitrary new members in the lambda object. The syntax is expanded to allow specifying the member 
name separately from its initial value.

```c++
// A C++14 code
std::unique_ptr<session_t> session = create_session();
auto request = server.request("GET /", session->id());
// We are scheduling a lambda to be executed when that request is completed.
request.on_completed(
    [ 
        session = std::move(session),   // Capturing the session by move
        time = current_time()           // Creating a time lambda member and
                                        // setting its value to the current time  
    ] 
    (response_t response)
    {

    }
);
```
Another thing that C++14 introduced are *generic lambdas*. If you recall, we were able to create a function object 
that has its call operator templated so that we can invoke it with different types. Since lambdas are just a syntactic 
sugar for writing function objects, the requirement that we need to specify the argument types in C++11 lambdas was mostly artificial. 
C++14 allows us to create generic lambdas simply by specifying the argument type to be `auto`. Earlier, we wrote a generic 
function object which allowed us to count how many items in a given collection were older older than some predefined age limit, 
without caring about the type of those items. We can achieve the same effect using C++14 lambdas like this:
