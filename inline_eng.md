# The *inline* keyword as ODR guard dismisser

I am sure that the `inline` keyword in C++ is known to the majority of programmers that works with this language for a while. From my experience, however, I can say that some sources for learning C++ from basics show only one, yet less important meaning of it. This fact led me to write this text.

At the beginning I have one tricky question for you. Think about whether removing the `inline` keyword from anywhere in a C++ source code can lead, in the broad sense, to a build error? I left this question without answer for now. I like to make examples as references, so let's start with one.

## Example of simple code

Let's see the example. A program that calculate the value of the *π* number, as well as *e* number (to be fair, we will just return those values, but this doesn't make any difference our the example).

To the *π* number calculation we can write two following source files:

```c++
//pi.hpp

#ifndef CALCULATING_PI
#define CALCULATING_PI
float calculate_pi();
#endif //CALCULATING_PI
```

```c++
//pi.cpp

#include "pi.hpp"
#include <iostream>

inline void log(const char* cstring)
{
    std::cout << "[ PI ]: " << cstring << std::endl;
}

float calculate_pi()
{
    log("starting calculations");
    //...
    log("calculations ready");
    return 3.1415;
}
```

As in typical code – in the header file we have declaration of function that we expose as our "library" API, as well as implementation of it in the source file. Additionally, we have helper function `log`, that we made `inline` – it is very short function.

Let's go further - we need a similar code, which will calculate the value of the *e* number.

```c++
//e.hpp

#ifndef CALCULATING_E
#define CALCULATING_E
float calculate_e();
#endif //CALCULATING_E
```

```c++
//e.cpp

#include "e.hpp"
#include <iostream>

inline void log(const char* cstring)
{
    std::cout << "[ e ]: " << cstring << std::endl;
}

float calculate_e()
{
    log("starting calculations");
    //...
    log("calculations ready");
    return 2.7183;
}
```

The above code is a calk of previously written files. As an addition, we can write the `main` to test our functions, as well as a simple `Makefile` just to make building simpler (for simplicity, I've hard-coded the g++ compiler).

```c++
//main.cpp

#include "pi.hpp"
#include "e.hpp"
#include <iostream>

int main()
{
    float pi = policz_pi();
    float e = policz_e();
    std::cout << "PI number: " << pi << "\ne number: " << e << std::endl;
    return 0;
}
```

```makefile
#Makefile

default: app.exe

pi.o: pi.cpp pi.hpp
	g++ -c pi.cpp -o pi.o

e.o: e.cpp e.hpp
	g++ -c e.cpp -o e.o

app.exe: pi.o e.o main.cpp
	g++ pi.o e.o main.cpp -o app.exe
```

Next step is to compile and run the program:

```
[ PI ]: starting calculations
[ PI ]: calculations ready
[ PI ]: starting calculations
[ PI ]: calculations ready
PI number: 3.1415
e number: 2.7183
```

### What happened here?

As we can see, something bad happened with the logging function from the `e.cpp` file. Instead of `[ e ]`, we can see the `[ PI ]` prefix. To be specific, the function from `pi.cpp` has been called here. Why? Better question would be – *"Why not?"*

Let's make deeper analysis. We have two functions with exactly the same name. Unwisely, both are placed in the global namespace and both are marked `inline`. Does it matter? Let's check – what will happen when we remove `inline` from both functions? The output from the `make` command would be following:

```
g++ -c pi.cpp -o pi.o
g++ -c e.cpp -o e.o
g++ pi.o e.o main.cpp -o app.exe
/opt/binutils-2.32/bin/ld: e.o: in function `log(char const*)':
e.cpp:(.text+0x0): multiple definition of `log(char const*)'; pi.o:pi.cpp:(.text+0x0): first defined here
collect2: error: ld returned 1 exit status
Makefile:10: recipe for target 'app.exe' failed
make: *** [app.exe] Error 1
```

Now we have an answer for the question from the beggining of this text. The compilation passed, but linking process failed. We can now see what is the clue of the `inline` keyword.

## What *inline* actually is?

We can mention two purposes of this keyword:

* `inline` informs the linker, that a function (or almost every symbol since C++17) with specified signature **and definition** can appear in more than one translation units. In such case linker is allowed to choose any of the definition, and ignore the rest. It shouldn't matter which one will be the chosen one, as they have to be identical. There is one thing we should be aware of – if there are two definitions of a function with the same signature in **the same compilation unit**, we don't have anything to worry about – compiler is obligated to emit an error.
* `inline` **suggest** compiler to insert the logic from the function in place that the function call appears (instead of real function call).

Technically those two meanings can be treated as one, as they are strictly connected – *how can linker claim that the function is defined twice when the function should not be a real function after compilation anymore?* I have separated them intentionally though, just to show which functionality of `inline` keyword is really important in modern compilers.

For the sake of accuracy, it should be mentioned that defining two `inline` functions with the same signature but not identical body, is Undefined Behavior. The compiler, or rather the linker, can do "what it wants" with it. In practice, as seen in our example, it simply rejected one of the definitions. More formal information on the possibility of defining a symbol once or more is specified in the One Definition Rule (ODR). I recommend the [Wikipedia article](https://en.wikipedia.org/wiki/One_Definition_Rule) on the subject for deeper information about ODR.

In addition, it is worth remembering that the implicit *inline* occurs in two situations:

* functions, member functions and static class fields marked with `constexpr`.
* functions defined in the body of a class (members and friend functions).

## When to use it?

After reviewing the example given at the beginning you might think: *Using `inline` is dangerous and it is better not to use it. It would be better if the linker informed us that we have a name collision.* Answering - this is because the example was rather a demonstration of how this tool should **not** be used. In fact, we were cheating on the linker here, because at some point we promised that if it encountered two functions with the same signature, they would have identical bodies too. The linker did not check it (it is not so simple), just "trusted" us. So why would anyone make that "promise"? Let's consider several cases that shows how useful this tool could be:

#### Short functions defined in headers

Using a real-life example. Let's say we have a header file with the following enumeration type:

```c++
enum class EReadMethod : bool
{
	UNBUFFERED,
	BUFFERED
};
```

We want also have an ability to print which value was used. Assuming that we don't have C++23 yet so we can't use *static reflection* feature, we need to write the to_string function. We can create a special source file or include the entire definition in the header file:

```c++
inline const char* to_string(EReadMethod val)
{
    switch(val)
    {
    case EReadMethod::UNBUFFERED:
        return  "UNBUFFERED";
    case EReadMethod::BUFFERED:
        return "BUFFERED";
    }
}
```

Defining such function in the header without making it `inline` might be considered as a mistake. It doesn't violate any C++ rule, but it is error-prone. If more than one compilation unit contained such header, it would introduce linker error about multiple definitions. Of course, on the other hand, there is a possibility that at some point somebody working on the same project will define different function with the same signature. But how likely is that? Every programmer should ask themselves this question every time they define such a function. Namespaces are also our friends which can help us to avoid such situations.

#### Header-only libraries

This is nearly identical use-case as the previous one. Only the goal differs – using header-only libraries is basically faster and easier.

#### Template classes

This is the area where – in my opinion – `inline` is most useful. Paradoxically, as most often we don't see explicitly written `inline` keyword there. This is because of the rule that states that every function defined inside the class body is implicitly `inline`. "Everyone" means also those inside non-template classes and non-member friend functions.

As we know, instantiation of a template requires its full definition to be visible. That fact indicates that we can't write only a declaration of a template in one compilation unit, and full definition in another *(to be strict, there are exceptions to this rule, but we are talking about general case)*. If `inline` wasn't there, whenever compiler instantiated the same template in different compilation units (for example `std::vector<int>`), linker would complain. If you use STL (or any other template-based library) a lot, such situation is very common.

#### Global variables and constants (C++17)

Starting from the ISO standard from 2017 we are also allowed to define constants and variables marked `inline`:

```c++
inline constexpr auto THREAD_POOL_SIZE{5};
inline ApplicationConfig globalAppConfig{};
```

It is a nice simplification introduced in the new standard – if you could do it with a function, why not with any kind of symbol? In older standards, we could define some constants like an enum member:

```c++
enum
{
	THREAD_POOL_SIZE = 5
};
```

This was rather a hacky workaround.

#### Static class members (C++17)

This is another nice simplification. Defining static members before C++17 was just ugly. The reason of it was understandable – the declaration of non-static member field doesn't cause any memory reservation for it. We need that memory only in case of making object of the class. This is not the case for static fields – we need that memory even without any object of the class. This was explicitly done by defining static field in source file, for example like that:

```c++
int ClassName::fieldName = 0;
```

This seems arguably inelegant. In C++17 we can replace that with only one statement in class body:

```c++
static inline int fieldName = 0;
```

If it happened that the linker found several definitions of this static field, it would "normalize" them to one. Seems nice!

## What about the second meaning of inline?

The second meaning is a suggestion for the compiler. It does not change the way the code will be executed. It will not change the meaning of our code in an algorithmic sense. All it can do is change the optimization paths.

I haven't recently looked at compilers source codes, but it's likely that modern compilers ignore this `inline` meaning. And this is because optimization is a specialty of compilers, and the programmer should not interfere with it too much. If you think you will optimize something better than a modern compiler, use assembler. Half jokingly, half seriously speaking.

## How to avoid the real multiple definition issue?
Situation from the provided example is unfortunate. On the other hand, we can see now that using `inline` makes sense in some contexts. What can we do to make `inline` work as we expected? There are some tips:

* Never use `inline` in source files (`*.cpp`) – use it only in headers.
* Use *internal linkage* for functions that are not part of your public API (by just placing them inside anonymous namespace or making them `static`).
* Do not use `inline` keyword just to tell compiler to place its body to the place it was called unless you really have very good reason to do this. Compiler just knows better in 99% of cases.

Additionally, I must admit that the double-definition case from the example doesn't exist in my personal statistics. The only case when I encountered such situation was during writing example for this article. But maybe I just have too little experience? I would like to know if any of the readers came across such a case!
