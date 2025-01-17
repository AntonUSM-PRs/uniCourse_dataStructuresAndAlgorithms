
## Domain data models

Domain data model is a struct that represents some thing from the real world.
In the following simple example, the struct `Car` represents a car object.

```cpp
struct Car
{
    std::string companyName;
    std::string modelName;
    std::string numberplateText;
    int producedYear;
    int numberOfDoors;
};
```

One of the key ideas is that you can have multiple variables of this type 
to represent data about different cars.

```cpp
int main()
{
    Car myCar{
        "Audi", // companyName
        "A4", // modelName
        "MD1515", // numberplateText
        2000, // producedYear
        4, // numberOfDoors
    };

    Car neighborsCar{
        "Toyota",
        "Camry",
        "JK1516",
        2002,
        6,
    };

    return 0;
}
```

## Validation

What you very often want to do, 
is you want to validate if the data in your object is valid.
For example, we may want to make sure the `companyName` has been specified,
the number of doors is not negative,
and the year produced is no more than the current year.

### Bool

There are multiple ways in C++ that you can represent errors.
The very basic but nonetheless useful way is to output `true` if it's valid,
and `false` otherwise.

The problem with this approach is that it does not give you enough information
to diagnose the problem when it occurs.

> [`time_t`](https://en.cppreference.com/w/cpp/chrono/c/time_t)

> [See example](validation/bool_example.cpp)

### String error

You could mitigate this problem by logging the error that happened,
or returning a string to be logged.
This is a valid solution, unless you want the code to be
used in other places to determine and deal with errors programmatically:
you will have the ability to help the user using the program to diagnose the problem
by looking at the logs, but your other functions won't be able to
easily determine what went wrong, because they'd have to be able to
analyze the string. 

> [See example](validation/string_example.cpp)

This approach also allows you to record multiple errors, outputting each,
separated by a newline character, for example.

> [See example](validation/string_multiline_example.cpp)


### `enum`

This can be useful when you want to record just the specific kind of error.
The idea is to give each error a unique number, and return that number.

> [See example](validation/enum_example.cpp)

### Flags

Enums can also be used to return multiple errors.
The idea is that you would use bits to represent the presence of each kind of error.
For example, we could say that the first bit represents if the company name is missing,
the second bit represents if the number of doors is negative, etc.
This way we can fit multiple errors into a single number.
These individual bits that represent the presence of something are called *flags*.

Example:
```
000 -- no error
001 -- company name is missing 
010 -- number of doors is negative
100 -- year produced is in the future
101 -- company name is missing and year produced is in the future
111 -- all errors
```

The key operator to help you define these flags is `<<`, which can be used to shift
a bit into the right position.
For example, doing `1 << 2` will give you `100` in binary, 
aka a number where the 2nd bit is set.
In general, `1 << n` will give you a number where the `n`th bit is set (from the right).

Next, we can use the `|` (bitwise or) operator to combine these flags.
The result of `a | b` will have `1` at all positions where they are `1` in either `a` or `b`.


Some examples:
```
000 | 000 = 000
001 | 000 = 001
001 | 001 = 001
100 | 001 = 101
111 | 000 = 111
```

Given a number, in order to check if a specific set of bits is set, 
we can use the `&` (bitwise and) operator.

`a & b` sets to `0` all the bits in the result that are `0` in `b`.
For all other positions, it copies the bit from `a`.

For example, `1010 & 0110`. First we remove all bits from `a` that are `0` in `b`.
So we know the result will be like `0__0`, because these `0` bits were `0` in `b`.
Now, we copy the other bits from `a`: `_01_`.
We end up with `0__0` and `_01_`, which we join to get `0010`.

> A set of flags, aka a number, where multiple bits corresponding 
> to the individual flags are set, which is typically used for checking if
> certain bits are set with the `&` operator, is called a *mask*.
> *Masking out bits* refers to the result of applying the `&` operator,
> where certain bits are bound to end up as `0` in the result.

```
111 & 100 = 100  -- highest bit copied over, the rest unset
000 & 100 = 000  -- same
101 & 001 = 001  -- lowest bit copied, others zeroed
110 & 111 = 110  -- all bits copied
```

Notice that if none of the bits that are set in `b` are also set in `a`,
the result will be all zeros.
Consequently, if at least one of the bits from `b` is also set in `a`,
the result will not be all zeros.
This means that *to check for presence of some flags, you can use `&` and compare to zero*.

I put it all together in the [example](./validation/enum_flags_example.cpp).


### Tagged union

This is a very common idea, especially in C.
The idea is to return the error enum value, with some additional data.
We can store some computed data in the error, for example, 
the line and column numbers if the error occurs while reading a file,
or we could store the actual value that caused the error and the expected value.

Since at any time only one kind of error is possible,
we can store the additional payload for all different types of errors in the same memory.
We can do that by using `union`.

When using unions, however, you have to be super careful to not read from the wrong field.
Reading from a field of the wrong type is considered undefined behavior!

> [See example](validation/tagged_union_example.cpp)

You can modify the validation function to accept an `std::vector` of errors,
add all of the errors recorded in there, and then loop through it on the
receiving end to handle all of the errors.
So this approach does generalize to multiple errors.

Also note that `union` cannot be used with non-trivial types, like `std::string`.

Tagged unions are a very flexible, 
but they are easy to mess up in their raw form.
You can easily crash your program by accessing a wrong field.
It is generally advisable to wrap the union in a class,
access the payload for the correct kind of error through a get method,
where you'd `assert` the correct kind of error is present.
This is to detect the read from a wrong field immediately,
and not later in the program, when it's no longer obvious 
what the source of the unexpected behavior was.

A simplified example would be like this:

```cpp
enum class ResultKind
{
    Valid,
    ErrorA,
    ErrorB,
};

struct PayloadErrorB
{
    size_t b1;
    size_t b2;
};

class Result
{
    ResultKind _kind;
    union {
        int _payloadErrorA;
        PayloadErrorB _payloadErrorB;
    };

public:
    static Result createValid() 
    { 
        return { ResultKind::Valid }; 
    }

    static Result createErrorA(int payload) 
    { 
        Result result;
        result._kind = ResultKind::ErrorA;
        result._payloadErrorA = payload;
        return result;
    }

    static Result createErrorB(PayloadErrorB payload) 
    { 
        Result result;
        result._kind = ResultKind::ErrorB;
        result._payloadErrorB = payload;
        return result;
    }

    ResultKind kind() const 
    { 
        return _kind; 
    }

    int& payloadErrorA()
    { 
        assert(_kind == ResultKind::ErrorA);
        return _payloadErrorA; 
    }

    PayloadErrorB& payloadErrorB()
    { 
        assert(_kind == ResultKind::ErrorB);
        return _payloadErrorB; 
    }
};
```

There are also libraries that offer more general tagged union implementations,
that can work for any payload types, see for example 
[`boost::variant`](https://www.boost.org/doc/libs/1_78_0/doc/html/variant.html), 
or [`std::variant`](https://en.cppreference.com/w/cpp/utility/variant).

### Exceptions

This is a very complicated system, initially designed to be able to
handle errors that occur in constructors, where you can't use the other approaches,
because constructors are void functions.
I'm not going to go deep into how they work, I'm only going to mention
how to use APIs that signal errors through exceptions.

> See a [very long and detailed video](https://www.youtube.com/watch?v=HXJmrMnnDYQ)
> on exception use cases.

You can think of a function that uses exceptions to return errors
as forcing you to handle the error, or your program will crash.
For contrast, in the previous approaches the error is treated as a regular return value,
and may be completely ignored without any repercussions 
(well, not checking it is likely a logical error anyway),
but this is for the most part a language limitation.

For example, let's look at a function `std::stoi` that converts a string to an integer.
It *throws* the exception `std::invalid_argument` if the string is not a valid integer,
and `std::out_of_range` if the integer is too big to fit into the 32 bits of an `int`.
See the example code [here](validation/exception_example.cpp).

You can try removing one of the `catch` blocks and triggering that exception.
It will make your program crash.


### My recommendation

Generally, never use exceptions in your own code, unless you are on the receiving end
of it, aka it's fine to catch it, but don't throw it in your code.
Consider using one of the appropriate other techniques.

Don't bother with tagged unions, unless you actually need the payload.
If you do, tagged unions are the go-to solution for me.


### To sum up:

- `bool` -- I don't care what the actual error is, just tell me if there is one;

- String -- I want to log the error, but I don't care about it programmatically;

- `enum` -- I want to know what kind of error it is, but I don't need any additional data; 

- Flags -- I want to be able to encode multiple errors at once;

- Tagged union -- I want to know the kind of error and also store some payload;

- Exceptions -- I'm lazy and I just don't care **OR** 
  I have carefully reviewed and understood how it's implemented 
  (not just the syntax, the actual mechanics),
  watched the video that explained the use cases,
  and I deliberately want to use exceptions because it fits my problem.


## Working with streams

Honestly, streams are a very nice abstraction to have around.


### What's a stream?

A *input stream* is an object is an object that you can read data from.
An *output stream* is an object that you can write data to.

For example `std::cout` is an output stream.
When you write data to it, it gets copies into the standard out (the console).

```cpp
// Pump data into the output stream
std::cout << "Hello";
```

Analogously, `std::cin` is an input stream.

```cpp
std::string x;
std::cin >> x;
```

`std::cout` and `std::cin` are some global variables, available in every program.
The type of these variables is `std::ostream` and `std::istream` 
respectively (o for output, i for input).


### File streams

You can also create streams that read from and write to files.
This can be achieved by using `std::fstream` (file stream, both input and output),
`std::istream` (input stream, read only), or `std::ostream` (output stream, write only).
They use RAII to close the handle at the end of the scope of the variable.

```cpp
#include <fstream>

int main()
{
    std::ifstream inputFile;
    std::ofstream outputFile;
    inputFile.open("input.txt");
    outputFile.open("output.txt");

    // Copy the contents of the file.
    // `rdbuf` makes a buffered reader.
    outputFile << inputFile.rdbuf();

    // And then some more stuff.
    outputFile << std::endl;
    outputFile << "Hello";

    return 0;
}
```

There are more methods to skip until a certain position (`seekg`),
read one byte at a time (`get` / `peek`), copy into a local buffer (`read`), etc.
There are similar methods for the writer as well.

I'm not going to describe these, try to play around and google what you need.


### Validation used in streams

Spot the bug in the following code.

```cpp
int a;
std::cin >> a;
```

The bug is that it doesn't check if the conversion failed or not.
Streams use the bool approach for this, but rather than returning `true` or `false`,
they set a flag on the object. 
You can check these flags by calling one of the 
[appropritate methods](https://cplusplus.com/reference/ios/ios/good/)

```cpp
int a;
std::cin >> a;

// It's a set of flags, but you can use the 
// helper methods to check for specific flags.
// std::ios_base::iostate state = std::cin.rdstate();

// Checks rdstate() for the goodbit flag.
if (std::cin.good())
{
    // Conversion succeeded.
}
else
{
    // Conversion failed.
}

```

The flags also allow contain information on whether the end of file has been reached.

```cpp
// Checks the eofbit flag (End Of File).
if (std::cin.eof())
{
    // End of file reached.
}
```


## Serialization

*Serialization* means converting an object in memory to a representation that can be stored
in a text file, or in a binary file.
*Deserialization* means converting from this file back into an object in memory.

For example, if we had an object like this:

```cpp
struct Car
{
    std::string companyName;
    std::string modelName;
    std::string numberplateText;
    int producedYear;
    int numberOfDoors;
};

// ...

Car myCar{
    "Audi", // companyName
    "A4", // modelName
    "MD1515", // numberplateText
    2000, // producedYear
    4, // numberOfDoors
};
```

We could represent it in a text file like this:

```
Audi
A4
MD1515
2000
4
```

Or like this:

```
companyName: Audi
modelName: A4
numberplateText: MD1515
producedYear: 2000
numberOfDoors: 4
```

Or like this:

```
Audi|A4|MD1515|2000|4
```

The idea is that *you* choose how you're going to represent it in the file.
The rules of how you store that in the text file are called the *serialization format*.
The most popular ones for flat data are CSV, 
and for structured data (nested objects) JSON and XML.

In the deserialization phase you'd read this string, and convert it into your object.
So in goes the string (or the stream), out comes the object.

```cpp
#include <iostream>
#include <string>

// std::optional<Car> is basically { Car value; bool isValid; }
std::optional<Car> deserializeCar(std::istream& stream)
{
    Car car;

    // read the first line into car.companyName
    std::getline(stream, car.companyName);
    if (!stream.good())
        return std::nullopt;
    
    // ...

    return car;
}
```