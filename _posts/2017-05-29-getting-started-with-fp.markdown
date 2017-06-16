---
layout: post
title:  "Getting started with functional programming"
date:   2017-05-29 19:42
categories: c++ programming
---

> The main feature of all functional programming languages is that functions can be treated like ordinary values.
They can be saved into variables, put into collections and structures, passed to other functions as arguments, 
and also returned from other functions as results.

Functions that take other functions as arguments or that return new functions are called higher-order functions.
They are probably the most important concept in functional programming.

Lets illustrate this with an example. Imagine that we have a group of people, and that we need to write out 
the names of all females in that group.

## Filtering
The first higher-level construct that we could use here, is collection filtering. Generally speaking, filtering 
is a simple algorithm that checks whether an item in the original collection satisfies a condition, and if it does, 
it gets put into the result. The filtering algorithm can not know in advance the different predicates users will 
use to filter their collections on. So this, construct needs to provide a way for the user to specify it.
Since filtering allows us to pass a predicate function to it it is, by definition, a higher-order function.

> The `filter` construct takes a collection and a predicate function (T -> bool) as arguments, and it returns
a collection of filtered items. We can write its types like this: 
`filter: (collection<T>, (T->bool)) -> collection<T>`

## Transforming

After the filtering is finished, we are left with the task of getting the names. We need a construct that 
takes a group of people (a collection), and returns their names. Similar to filtering, this construct can 
not know in advance what information we want to collect from the original items. Again, the construct needs 
to allow the user to specify the function that takes an item from the collection, does something with it, 
and returns a value which will be put into the resulting collection. It should be noted that the output 
collection does not need to contain items of the same type as the input collection (unlike filtering).

> This construction is called `map` or `transform`, and its type is: 
`transform: (collection<In>, (In -> Out)) -> collection<Out>`

When we compose these two constructs, by passing the result of one as input for the other, we
get the solution to our original problem - We get all name of females in the given group of people.

###  Examples from the STL, algorithm library
Imagine we have a list of scores that the website visitors have given to a particular film, 
and we want to calculate the average. The imperative way of implementing this is to loop over 
all the scores, sum them one by one, and divide that sum with the total number of scores.

The standard library provides us with a higher-order function that can sum all the items in a collection
-- `the std::accumulate` algorithm. It takes the collection (as a pair of iterators) and the 
initial value for summing, and it returns us the sum of the initial value and all items in the collection. 
We just need to divide it by the total number of scores.

This implementation does not specify how the summing should be performed, it states only what should
be done. 

```c++

double average_score(const std::vector<int>& scores)
{
    auto sum = std::accmulate(std::begin(scores), std::end(scores), 0);
    return sum / static_cast<double>(scores.size());
}
```

By default, the std::accumulate algorithm sums the items, but it allows us to
provide it with a custom function if we want to change that behaviour. If we needed to calculate 
the product of all the scores for some reason, it could be done by simply passing std::multiplies 
as the last argument of std::accumulate and setting the initial value to 1.

```c++

double scores_product(const std::vector<int>& scores)
{
    return std::accmulate(
        std::begin(scores), std::end(scores), 
        1, 
        std::multiplies<int>()
    );
}
```


## Folding

We often need to process a collection of items, one item at a time, in order to calculate something. 
The result might be as simple as a sum or a product of all items in the collection, or a number of 
all items that have a specific value, or that satisfy a predefined predicate. The result might also 
be something more complex like a new collection that contains only a part of the original one 
(filtering) or a new collection with the original items reordered (for sorting or partitioning).
The `std::accumulate` algorithm is an implementation of a general concept called `folding or reduction`.

> Fold is a higher-order function that that abstracts the process of iterating over recursive structures 
such as vectors, lists, trees, etc. that allows us to gradually build the result we need. 

In our previous example, we used folding to sum a collection of movie ratings. 
The `std::accumulate` algorithm first sums the initial value passed to it and the first item in the collection. 
That result is then summed with the next item in the collection, and so on. This is repeated until we reach 
the end of the collection. 

In general, folding does not require that the arguments and the result of the binary function passed to 
it have the same type. 

> Folding takes a collection that contains items of type T, an initial value of 
type R (which does not need to be the same as T) and a binary function f:(R,T) -> R. It calls the 
specified function on the initial value and the first item in the collection. The result is then passed
to the function f along with the next item in the collection. This is repeated until we process all items
from the collection. The algorithm returns the value of type R -- the value that the last invocation of
the function f returned.

In our previous example, the initial value was 0 (or 1 in the case of multiplication), and the binary 
function `f` was addition (or multiplication). As a first example of when a function `f` might return a 
different type to the type of the items in the collection that we are folding over, we will implement a 
simple function that counts the number of lines in a string by counting the number of occurrences of the 
newline character in it by using `std::accumulate`.

From the requirements of the problem, we can deduce the type of the binary function `f` that we will pass 
to the `std::accumulate` algorithm. The `std::string` is a collection of characters, so the type `T` will be a 
`char`, and since the resulting number of newline characters is an integer, the type `R` will be an `int`. 
This means that the type of the binary function `f` will be `(int, char) -> int`.

We also know that the first time `f` is called, its first argument will be zero since we are just starting 
to count, and when it is invoked the last time it should return the final count as the result. Moreover, since 
`f` does not know whether it is currently processing the first, the last, or any other random character in the string, 
it can not use the character position in its implementation. It only knows the value of the current character, 
and the result that the previous call to `f` returned. This basically forces the meaning of the number of newlines 
in the previously processed part of the string to its first argument. From this point on, the implementation 
is straightforward: if the current character is a newline, return the same value it was passed to us as the previous
count, otherwise increment that value and return it. 
```c++
int f(int previous_count, char c)
{
    // Increasing the count if the current character is a newline.
    return (c != '\n') ? previous_count: previous_count + 1;
}
int count_lines(const std::string& s)
{
    return std::accumulate(
        // Folding the whole string
        s.cbegin(s), s.cend(s), 
        // Starting the count with 0
        0, 
        f
    );
}
```

## Filtering and transformation

Lets try to implement the problem from the beginning of this chapter using STL algorithms. 
Just a short reminder, the task is to get the names of all females in a given group of people.

If we are allowed to change the original collection, we can use the `std::remove_if` 
algorithm with the erase-remove idiom to remove all persons that are not females.

```c++
// Filtering by erase-remove idiom
people.erase(
    // Marking the items for removal
    std::remove_if(people.begin(), people.end(), is_not_female),
    // Removing the marked items
    people.end()
);
```

### Erase-remove idiom

If we want to use the STL algorithms to remove elements that satisfy a predicate, or all elements 
that have a specific value from a collection, we can do so with the `std::remove_if` and `std::remove algorithms`.
Unfortunately, since these algorithms operate on a range of elements defined by a pair of iterators, 
and do not know what the underlying collection is, they can not actually remove the elements from it. 
These two algorithms only move all the elements that do not satisfy the criteria for removal to the 
beginning of the collection.

Other elements are left in an unspecified state, and the algorithm returns the iterator to the first of them
(or to the end of the collection if there are no element that should be removed). We need to pass this iterator 
to the `erase` member function of the collection which performs the actual removal.


Alternatively, if we do not want to change the original collection, which should be the right way to go since we 
aim to be as pure as possible, we can use the `std::copy_if` algorithm to copy all the items that satisfy the 
predicate we are filtering on to the new collection.

The algorithm requires us to pass it a pair of iterators that define the input collection, one iterator that points 
to the destination collection we want to copy the items to, and a predicate that returns whether a specific item 
should or should not be copied. Since we do not know the number of females in advance, we need to create an empty
vector `females` and use the `std::back_inserter(females)` as the destination iterator.

```c++
// Filtering by copying

// Creating a new collection to store the filtered items in.
std::vector<person_t> females;

// Copying the items that satisfy the condition to the
// new collection.
std::copy_if(
    people.cbegin(), people.cend(), 
    std::back_inserter(females), 
    is_female
);
```

The next step is to get the names of the people in the filtered collection. We can use std::transform for this. 
We need to pass it the input collection as a pair of iterators, the transformation function and where the results 
should be stored. In this case, we know that the number of names is the same as the number of females, so we can 
immediately create the names vector to be of the needed size to store the result and use `names.begin()` 
as the destination iterator instead of relying on `std::back_inserter`. This way, we will remove the potential 
memory reallocations needed for vector resizing.

```c++
std::string name(const person_t &person)
{
    return person.name();
}
// Reserving space
std::vector<std::string> name(females.size());
// Transforming to get the names
std::transform(
    // What we are transforming
    females.cbegin(), females.cend(), 
    // Where to store the results
    names.begin(), 
    // the transform function
    name
);
// Printing
std::copy(name.begin(), name.end(), std::ostream_iterator<std::string>(cout, "\n"));
```


### Composability problems of STL algorithms

This solution is valid and will work correctly for any type of the input collection that can be iterated on 
-- from vectors and lists, to sets, hash maps and trees. It also shows the exact intent of the program  
-- to copy all females from the input collection, and then get the names for all of them.

Unfortunately, it is not as efficient nor as simple as a hand-written loop. 

The STL-based implementation makes unnecessary copies of people (which might be an expensive operation, or 
it could even be disabled if the copy constructor is deleted or private) and it creates an additional vector 
which is not actually needed. We could try to compensate for these problems by trying to use references or 
pointers instead of copies, or by creating smart iterators that will skip all persons that are not females, etc. 
but the need to do this extra work is a clear indication that the STL has lost in this case  -- the hand-written 
loop is just better and requires less effort.

So, what went wrong? If you recall the signatures for transform and filter that we saw earlier, you can see that 
they are designed to be composable, that is, to call one on the result of the other.

> filter : (collection<T>, (T -> bool)) -> collection<T> <br>
> transform : (collection<T>, (T -> T2)) -> collection<T2> <br>
> transform(filter(people, is_female), name)

The `std::copy_if` and `std::transform` algorithms do not fit into these types at all. 
They do require the same basic elements, but in a significantly different manner. 
They take the input collection through a pair of input iterators, which makes it impossible 
to invoke them on a result of a function that returns a collection, without first saving that 
result to a variable. And they do not return the transformed collection as the result of the algorithm call, 
but they also require the iterator to the output collection to be passed in as the algorithm argument. 
Instead, the result of the algorithm is an output iterator to the element in the destination range, one past 
the last element stored. This effectively kills the ability to compose these two algorithms without creating intermediary variables 
like we saw earlier.

While this might seem like a design problem that should not have happened, there are practical reasons why these problems exist. 
First, passing a collection as a pair of iterators allows us to invoke the algorithm on parts of a collection instead of on the whole. 
Passing the output iterator as an argument instead of the algorithm returning the collection allows us to have different collection 
structures for input and output (for example, if we wanted to collect all the different female names, we might passed the iterator 
to a `std::set` as the output iterator). 

Even having the result of the algorithm to be an iterator to the element in the 
destination range, one past the last element stored has its merits. If we wanted to create an array of all people, but order them
in a way to have females first, and then others (like the `std::stable_partition` algorithm, but to create a new collection instead 
of changing the old one), it would be as simple as calling `std::copy_if` twice, once to copy all the females and once to get 
everyone else. The first call will return us the iterator to the place where we should start putting non-females. 

```c++
std::vector<person_t> separated(people.size());

// Returns the location after the last person copied.
const auto last = std::copy_if(people.cbegin(), people.cend(), separated.begin(), is_female);
// Storing the rest after all females. 
std::copy_if(people.cbegin(), people.cend(), last, is_not_female);
```

So, while the standard algorithms do provide us a way to write our code in a more functional manner than doing everything 
with hand-written loops and branches, they are not designed to be composed in a way common to other functional programming 
libraries and languages.
We will see a more composable approach that is taken in the ranges library in the next posts.

> Ranges to the rescue!!!!!!!!! <br>
The issue of unnecessary memory allocations arises from the composability problems of STL algorithms. 
This problem with STL algorithms has been known for some time, and there are a few libraries created to fix this.
We will cover a concept called ranges in detail in next post. 
Ranges are planned to be included in the standard library starting from C++17 as a Technical Specification, but are 
also available in the form of a third-party library for earlier versions of C++.

## Recursion and tail call optimization

In pure functional programming languages, where the loops do not exist, functions that iterate over collections are 
usually implemented using recursion. For example, we can process a non-empty vector recursively by first processing
its head - the first element - and them by processing its tail - all other elements - which also be seen as a vector.
If the head satisfies the predicate, we will include it in the result, if we get an empty vector, there is nothing
to process, so we are also returning an empty vector.

The probably the most important issue with this approach is that we can get into trouble if we call our function with 
a large collection of items. Every recursive call will take some memory on the stack, and at some point the stack will 
overflow and our program will crash. Also, even if the collection is not big enough to induce the stack to overflow, 
function calls are not free  — simple jumps that a for loop gets compiled to are significantly faster.

Here we need to rely on our compiler to be able to transform the recursion into loops. And in order for the compiler to 
be able to do so, we need to implement a special form of recursion called tail recursion. 

>Tail recursion is a kind of recursion where the recursive call is the last thing in the function, that is, the 
function must not do anything at all after recursing.



Recursion is a powerful mechanism that allows implementing iterative algorithms in the languages that do not have loops. 
But recursion is still a really low-level construct. We can achieve true inner purity with it, but in many cases it just 
makes no sense. We said that we want to be pure so that we can make less mistakes while writing the code, since we do not 
need to think about the mutable state. But, by writing the tail-recursive functions like the above one, we are simply 
simulating the mutable state and loops in another way. And while it is a nice exercise for the gray cells, it is not 
much more than that.

Recursion, like hand-written loops, has its place in the world, but in most cases in C++, it should raise the red 
flag during the code review. It needs to be checked for correctness, and that it covers all the use-cases without ever 
overflowing the stack.

Since recursion is a low level construct, implementing it manually is often avoided even in pure functional programming languages. 
Some might say that the higher level constructs are especially popular in functional programming exactly because recursion is this convoluted.
We have seen what folding is, but we still do not know its roots. We saw that it takes one element at a time, and applies the 
specified function to the previously accumulated value and the currently processed element, which produces the new accumulated 
value. So, as you can you notice that this is exactly what our tail-recursive 
implementation of `names_for_helper` function did. In essence, folding is nothing more than a nicer way to write tail-recursive 
functions that iterate over a collection  — the common parts are abstracted out, and you need to specify only the collection, 
the starting value, and the essence of the accumulation process without the need to write recursive functions.

```c++
std::vector<std::string> append_name_if( std::vector<std::string> previously_collected, const person_t &person)
{
    if (filter(person)) 
        previously_collected.push_back(name(person));
    else
        return previously_collected; 
}
// Implementation using fold
std::vector<std::string> names_for(const std::vector<std::string>& people)
{
    return std::accumulate( people.cbegin(), people.cend(), std::vector<std::string>{}, append_name_if);
}
```

