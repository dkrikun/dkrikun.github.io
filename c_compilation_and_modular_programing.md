# Basics of compilation and modular programming in C

### Intro

From the title it may look like 101 C programming CS course material, but I'm
actually writing this to explain a few idiosyncrasies arising in everyday
software development. I've been lately answering some questions about
"advanced" C++, .dll's, boost-python etc.  that would be best understood by
going "back to basics".

So without further ado, let's ask ourselves a simple question:

### How to make a function public in C?

Hey, what do you mean 'public'? C aint Java or C# or even C++ we don't have
public or private here. In fact, we don't even have classes or objects, it's
not OOP language after all!

Right there is no 'private' keyword in C, but there are means to enforce
information hiding.  Let's talk about the compilation process.

### Translation units in C

For instance, we have two .c files here: main.c and fibo.c, as follows:

```fibo.c
int fibo(int n)
{
    if (n < 1) return 0;
    if (n == 1) return 1;
    return fibo(n-1) + fibo(n-2);
}
```

```main.c
int main(int argc, char** argv)
{
    return 0;
}
```

Each C file, after it has been processed with a preprocessor is called a
*translation unit*.

The term stems from the fact that each file is translated
(i.e. compiled) completely independent from each other.

In short, the preprocessor pastes coded from the #include'd files, cuts out
lines in the #ifdef/#if conditionals, and replaces #define'ed constants.
The resulting text might be quite big and unreadable, but conceptually it is
quite simple procedure.

We don't have any preprocessor statements for now, so those are practically our
translation units. Let's go and compile them:

```
>> gcc -c fibo.c
>> gcc -c main.c
>> ls
fibo.o main.o
```

Now we've got 2 object files, which contain the binary, machine-specific
assembly code. To create an executable application we still need to *link* the
object files together (we'll talk about it later in detail):

```
>> gcc fibo.o main.o -o fibo_app
>> ls
fibo.o main.o fibo_app
```

This is not very interesting yet. Now, we would like the `main()` to be able to
call `fibo()` now, like this:

```main.c
// how to access that fibo() from main?? hmm..

int main(int argc, char** argv)
{
    return fibo(42);
}
```

So, now the question is, how does one makes `fibo()` available in `main()`?

Because we've just established that each translation unit is compiled
independently, apparently `gcc -c main.c` has no knowledge of `fibo()`.

At this point, somebody would suggest that in order to make `fibo()` "public"
in their understanding of the C world, one should add a header file `fibo.h`
containing the *declaration* of `fibo()`:

```fibo.h
int fibo(int);
```
and include it in `fibo.c`:

```main.c
#include "fibo.h"

int main(int argc, char** argv)
// ...
```
Now main.c includes fibo.h which is the "public" api of fibo.c, so it can call
fibo(), right?

This is actually wrong, and it is confusing due to visual similarity with e.g.
C# where you can import a module and by that mean get access to its public
parts.

In C, the original form (without the header file) is already enough to make
`fibo()` "public" in fibo.c translation unit. To confirm it, we can build the
original code:
```
>> gcc -c fibo.c
>> gcc -c main.c
warning: implicit declaration of function 'fibo' ...
>> gcc fibo.o main.o -o fibo_app
```

It spits a warning, but builds and runs just fine. But who cares about a
little warning anyway? Joking, we'll get to it in a second, it is in fact very
important, but for now let's also inspect `fibo.o` with the `nm` utility:

```
>> nm fibo.o
00000000000000000 T fibo
```

'T' means the symbol fibo is in the text section and is global.
To actually, make it "private", we use `static` keyword:

```fibo.c
static int fibo(int n)
// ...
```
```
>> gcc -c fibo.c
>> nm fibo.o
00000000000000000 t fibo
```

Note the lowercase 't', which designates a local symbol.
Trying to compile fibo_app gives:
```
gcc fibo.o main.o -o fibo_app
>> /usr/bin/ld: main.o: in function `main':
>> main.c:(.text+0x1a): undefined reference to `fibo'
>> collect2: error: ld returned 1 exit status
```

Thanks to the `static` keyword, `fibo()` is now private to fibo.c, and no header
file could ever change this!

How does on earth is it possible to compile main.c without any knowledge about
`fibo()` defined in fibo.c?

It turns out, when compiling `main.c`, the compiler actually leaves out a few
bits of information, for the linker to fill it later, at the *linkage stage*.

This can be again observed with nm ('U' means undefined):
```
>> nm main.o
                 U fibo
                 U _GLOBAL_OFFSET_TABLE_
0000000000000000 T main
```

So, the `fibo()` call site is not connected to an actual function code yet,
and, unless it is defined as static, in `fibo.o`, the linker will find it and
connect the call site to the function code.


### That little warning ..

It is important to understand that the function signature is not a part of
symbol in an object file and even if it were, the compiler is still lacking
this information while compiling main.c.

However, it must understand how to call `fibo()` while compiling main.c, because
it needs to insert proper machine instructions to push and pop arguments from
stack and otherwise serve the *calling convention* properly.

To make it work we need to give the compiler a *prototype* of the function,
which includes exactly that information:

```main.c
int fibo(int);

int main(int argc, char** argv)
// ...
```

Now the compiler does know to insert the correct machine instructions to pass
exactly 1 integer to fibo, and return exactly 1 back from it.

This is what about that warning was about: *implicit declaration* of function
fibo. We provided no protype thus the compiler is whining, and justified so --
it does not know in fact how to call fibo correctly.

I am not in the mood right now to discuss why exactly this is only a warning,
while it should apparently be a fatal error, and how exactly the compiler
figures out how to call `fibo()`.

Suffice it to say 'for historical reasons', and for an extra sanity, always
provide function prototypes (or defintions) for the compiler.

### Side note: exporting variables... and maybe other entities?

Apart from functions, variables can be also exported by a translation unit:

```fibo.c
#include <stdio.h>
int debug;

int fibo(int n)
{
    if(debug != 0)
        printf("n=%d\n", n);
    //...
}
```

```main.c

extern int debug;
int fibo(int);

int main(int argc, char** argv)
{
    debug = 1;
    return fibo(42);
}
```

We have to specify `extern` keyword, because otherwise, the compiler would
think the intention is to define a variable within the current translation
unit.

What about other entities, can we export a struct or an enum?

The short answer, is no, but to understand why, we first need to understand,
that the compiler, in fact, only uses, for example, a struct definition to
generate machine instructions for the code that operates upon the struct.

The struct itself is not a code, and it is also not stored in any kind of a
table as metadata in object files. So this information does no longer
exist when an object file is built.

This is unlike, for example, C# or Java, where classes are stored in a compiled
code, and available for introspection at a later time.

So let's say we have the following routine:

```distance.c
#include <math.h>

struct point
{
    float x, y;
};

float distance(struct point p1, struct point p2)
{
    return sqrt((p2.x-p1.x)*(p2.x-p1.x) + (p2.y-p1.y)*(p2.y-p1.y));
}
```

And we want to call it from `main`:
```main.c

struct point;
float distance(struct point p1, struct point p2);

int main(int argc, char** argv)
{
    struct point p1 = { 11.0, 24.0 }, p2 = { 35.0, 45.0 };
    distance(p1, p2);
}
```

Now, this won't compile because the compiler has to know what is the
size of `struct point` to allocate space for it on stack when passing it
to the `distance()`.
It also has to understand that the struct 2 float fields to be able to
compile the initalization code.

We've already established that we cannot "export" the struct defintion
for the `distance` to `main`, so we're stuck, right?

Actually, know. Because each translation unit is compiled separately,
it won't hurt anyone if we also define `struct point` in `main.c`:

```main.c
struct point
{
    float x, y;
};

float distance(struct point p1, struct point p2);

int main(int argc, char** argv)
{
    struct point p1 = { 11.0, 24.0 }, p2 = { 35.0, 45.0 };
    distance(p1, p2);
}
```

### Doing bad things

What would happen, if we were to define `struct point` in `main.c`
differently from the one in `distance.c`, for example:

```
struct point
{
    float a, b;
};
```

or even:

```
struct point
{
    float x, y, z;
};
```

Wait, we can do even crazier stuff, who told us we have to declare
`distance()` the same? What about:

```
float distance(float x1, float y1, float x2, float y2);
```

The answer is that the compiler will, of course, be able to compile
both of the translation units, and the linker will be able to link them.

However, the resulting applictation would (except for the 1st example)
not work correctly.

What happens is that the compiler will generate instructions for the
struct layout and function prototype that it is given, and it occasionaly
might not crash, or even work correctly if somehow the views of the
translation units correspond, but this cannot be relied upon, because
of differences in compiler, platform, optimization levels etc.

In fact, according to the C standard, the above code has *undefined behaviour*,
meaning, it is such a nonsense that a compiler is allowed to generate
any program, even one that prints 'wtf' and exits.

(it turns out, in C, anything that is said to have or invoke an undefined
behaviour gives a compiler a free pass to do just anything it wants to).

### The use of the header files

It is quite inconvenient to write each and every struct definition, and
funciton declaration, in every module that needs them (and in fact is quite
dangerous due to the undefined behaviour, we could potentially invoke if we
were to make a mistake in the defintions).
Well, we could use the preprocessor for this job, and put all the stuff that
needs be present in all of the translation units, in a common header file, and
then #include it where required:

```distance.h

struct point { float x, y; };

float distance(struct point, struct point);
```

```distance.c
#include "distance.h"

float distance(struct point p1, struct point p2)
{
// ...
}
```

```main.c
#include "distance.h"

int main(int argc, char** argv)
{
// ...
}
```

Now everything works as expected, *and* we do not have to worry about code
duplication, we *reuse* the same struct defintion and function prototype
by putting them in a common header file.

### Conclusion

The pattern above is, in fact, so commonly used, that we tend to conflate
the concept of header file with public api or "public" symbols of translation
unit.

It might be benign for most of the part, unless you start adding nasty stuff:
C++, interop with other languages, mixing Visual Studio versions, etc. etc.

It is then that it becomes important to understand that the header files
are a means of code reuse and not a mechanism to export symbols.

In my opinion, it is also beneficial to understand what are the proper means
to control the information hiding in C.

I hope, this reading was useful and refreshing, next time I would like to
write about the difference between kinds of libraries and their linkage
mechaninsms in C.

