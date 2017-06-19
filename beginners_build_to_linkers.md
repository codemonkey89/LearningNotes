
# Beginner's Guide to Linkers
Original Link:  *http://www.lurklurk.org/linkers/linkers.html*

[TOC]

### Declaration vs. Definition
#### Declaration
A *declaration* tells the C compiler that a definition of something (with a particular name) exists elsewhere in the program, probably in a different C file. 

* ***global variables***,  which exist for the whole lifetime of the program ("**static** extent"), and which are usually accessible in lots of different functions
* ***local variable***, which only exist while a particular function is being executed ("**local** extent") and are only accessible within that function

#### Definition
A *definition* associates a name with an implementation of that name, which could be either data or code. A definition of a variable induces the compiler to

* reserve some space for that variable, and possibly fill that space with a particular value.
* generate code for that function.
 
Phase | Code| Global Initialized| Global uninitialized| Local Initialized | Local Uninitialized| Dynamic
:-|:-|:-|:-|:-|:-|:-|
 Declaration| `int fn(int x);`|`extern int x;`|`extern int x;`| N/A | N/A | N/A
 Definition|`int fn(int x) { ... }`|`int x = 1;` | `int x;`|`int x = 1;`|`int x;`|`(int* p = malloc(sizeof(int));)`

c_parts.c
```c
 /* This is the definition of a uninitialized global variable */
int x_global_uninit;

/* This is the definition of a initialized global variable */
int x_global_init = 1;

/* This is the definition of a uninitialized global variable, albeit
 * one that can only be accessed by name in this C file */
static int y_global_uninit;

/* This is the definition of a initialized global variable, albeit
 * one that can only be accessed by name in this C file */
static int y_global_init = 2;

/* This is a declaration of a global variable that exists somewhere
 * else in the program */
extern int z_global;

/* This is a declaration of a function that exists somewhere else in
 * the program (you can add "extern" beforehand if you like, but it's
 * not needed) */
int fn_a(int x, int y);

/* This is a definition of a function, but because it is marked as
 * static, it can only be referred to by name in this C file alone */
static int fn_b(int x)
{
  return x+1;
}

/* This is a definition of a function. */
/* The function parameter counts as a local variable */
int fn_c(int x_local)
{
  /* This is the definition of an uninitialized local variable */
  int y_local_uninit;
  /* This is the definition of an initialized local variable */
  int y_local_init = 3;

  /* Code that refers to local and global variables and other
   * functions by name */
  x_global_uninit = fn_a(x_local, x_global_init);
  y_local_uninit = fn_a(x_local, y_local_init);
  y_local_uninit += fn_b(z_global);
  return (y_global_uninit + y_local_uninit);
}
```
> **Note:** There is no 'extern' keyword before the fn_a() function. This is because all function declarations in the header are 'extern' by default. https://stackoverflow.com/questions/856636/effects-of-the-extern-keyword-on-c-functions

### What The C Compiler Does
Convert from C file to machine code in *.o* or *.obj* suffix that contains

* *code*, corresponding to definitions of functions in the C file
* *data*, corresponding to definitions of **global** variables in the C file 

> **Note:** Wherever the code refers to a variable or function, the compiler only allows this if it has previously seen a declaration for that variable or function.

The job of the linker is to make good on these promises that a definition exists somewhere else in the whole program. When generating an object file, the compiler leaves a *blank*, a "reference" that has a name associated to it, but the value corresponding to that name is not yet known. 

![Graph](http://www.lurklurk.org/linkers/c_parts.png)
> **Note:** x_global_uninit, y_global_uninit, fn_a and z_global are the blanks that linker will fill in later.

#### Dissecting An Object File
Command line tool *nm* gives information about symbols in an object file.


Symbols from c_parts.o:

Name |Value |Class| Type| Size| Line | Section
:-|
fn_a                |        |   U  |            NOTYPE|        |     |*UND*
z_global            |        |   U  |            NOTYPE|        |     |*UND*
fn_b                |00000000|   t  |              FUNC|00000009|     |.text
x_global_init       |00000000|   D  |            OBJECT|00000004|     |.data
y_global_uninit     |00000000|   b  |            OBJECT|00000004|     |.bss
x_global_uninit     |00000004|   C  |            OBJECT|00000004|     |*COM*
y_global_init       |00000004|   d  |            OBJECT|00000004|     |.data
fn_c                |00000009|   T  |              FUNC|00000055|     |.text

Explanation of **Class** column:

* **U** indicates undefined references (a.k.a. blanks); equal to **UND** in Section column.
* **t** or **T** indicates where code is defined; equal to **text** in Section column.
	-  **t** means function is local to this file, or function is declared with `static`
		- ***Important!*** `static` keyword has different meaning in C and C++ ([stackoverflow](https://stackoverflow.com/questions/558122/what-is-a-static-function)).
	-  **T** means otherwise
* **d** or **D** indicates an initialized global variable; this particular class indicates whether the variable is local(**d**) or not (**D**). equal to **data** in Section column.
*  For an uninitialized global variable, **b** indicates if it's static/local, and **B** or **C** when it's not. equal to **bss** or **COM** in Section column.

### What The Linker Does: Part 1

```c
/* Initialized global variable */
int z_global = 11;
/* Second global named y_global_init, but they are both static */
static int y_global_init = 2;
/* Declaration of another global variable */
extern int x_global_init;

int fn_a(int x, int y)
{
  return(x+y);
}

int main(int argc, char *argv[])
{
  const char *message = "Hello, world";

  return fn_a(11,12);
}
```
![](http://www.lurklurk.org/linkers/c_rest.png)

Combining the two graphs together:

![](http://www.lurklurk.org/linkers/sample1.png)

#### Duplicate Symbols
Linker will throw an error if the definition for a symbol cannot be found. But what if there are *two* definitions for a symbol when it comes to link time?

In C++, the language has a constraint known as **one definition rule**.
In C, there has to be exactly *one* definition of any functions or initialized global variables, but C allows **multiple definitions of an uninitialized global variable** and will pick one of the definitions.

### What The Operating System Does
In short, OS need to store both **code** (e.g. functions) and **data** (e.g. variable values) in memory.

Running the program obviously involves executing the machine code, so the operating system clearly has to transfer the machine code from the executable file on the hard disk into the computer's *memory*, known as the **code segment** or **text segment**.

Code is nothing without data, so all of the global variables need to have some space in the computer's memory too. However, there's a difference between *initialized* and *uninitialized* global variables. 

* Initialized variables have particular values that need to be used to begin with, and these values are stored in the object files and in the executable file. When the program is started, the OS copies these values into  the **data segment**.
* Uninitialized variables are initialized to 0 by OS in memory directly, stored in **bss segment** and do not need to be saved in the object files (i.e. it saves space on disk).

![](http://www.lurklurk.org/linkers/os_map1.png)
> **Note:** Local variables (allocated on stack) and dynamically allocated memory (allocated on heap) don't need linker involvement because their lifetime only occur when the program is running.
 
### What The Linker Does: Part 2
A **Library** is a collection of related compiled object files (i.e. `.o` files), that can be reused by different programs.

#### Static Libraries
On UNIX, `ar` is commonly used to produce static libraries (usually with `.a` extension). These library files are normally prefixed with "`lib`" and passed to the linker with a "`-l`" option followed by the name of the library, without prefix or extension.

For example, "`-lfred`" will pick up the static library called "`libfred.a`".  

* The library is only consulted **after** the normal linking is done and they are processed *in order*. If an object pulled in from a library late in the link line needs a symbol from a library earlier in the link time, the linker won't automatically find it.
* When a symbol is needed, the **whole** object that contains the symbol definition is included and the linker needs to resolve new undefined symbols in that object. (i.e. the granularity of what the linker pulls in from the library is at the *object* level). 

In the example, the linker pulls in `a.o`, `b.o`, `-lx` and `-ly`.

<table frame="box" rules="all" border="1">
  <tbody><tr>
   <th>File</th>
   <th><code>a.o</code></th>
   <th><code>b.o</code></th>
   <th colspan="3" align="center"><code>libx.a</code></th>
   <th colspan="3" align="center"><code>liby.a</code></th>
  </tr>
  <tr>
   <th>Object</th>
   <th><code>a.o</code></th>
   <th><code>b.o</code></th>
   <th><code>x1.o</code></th>
   <th><code>x2.o</code></th>
   <th><code>x3.o</code></th>
   <th><code>y1.o</code></th>
   <th><code>y2.o</code></th>
   <th><code>y3.o</code></th>
  </tr>
  <tr>
   <th>Definitions</th>
   <th><code>a1, a2, a3</code></th>
   <th><code>b1, b2</code></th>
   <th><code>x11, x12, x13</code></th>
   <th><code>x21, x22, x23</code></th>
   <th><code>x31, x32</code></th>
   <th><code>y11, y12</code></th>
   <th><code>y21, y22</code></th>
   <th><code>y31, y32</code></th>
  </tr>
  <tr>
   <th>Undefined references</th>
   <th><code>b2, x12</code></th>
   <th><code>a3, y22</code></th>
   <th><code>x23, y12</code></th>
   <th><code>y11</code></th>
   <th><code></code></th>
   <th><code>y21</code></th>
   <th><code></code></th>
   <th><code>x31</code></th>
  </tr>
 </tbody></table>

1. When `a.o` and `b.o` are processed, `b2` and `a3` are resolved; `x12` and `y22` are undefined.
2. When `libx.a` is processed, `x1.o` is pulled in to resolve `x12` but that adds `x23` and `y12` to the unresolved list.
3. `x2.o` is then pulled in to resolve `x23`, adding `y11` to the unresolved list.
4. The linker then processes `liby.a`,  and similar logic applies.
 
#### Shared Libraries
Disadvantages of static library:

* take up unnecessary disk space by pulling in popular common libraries (e.g. C standard library, `libc`)
* cannot get updated code from the statically linked libraries (e.g. bug fixes)

Shared libraries normally have a `.so` extension on UNIX systems or `.dll` extension on Windows. At linking time, the linker doesn't include the definition of the symbol in the final executable but instead records the name of the symbol and which library it is supposed to come from in the executable file.

Another big difference between static library and shared library is the granularity of the link. If a symbol is pulled from a particular shared library, the *whole* library (for static library, it would have been the *whole* object) is mapped into the address space of the program.

### Adding C++ To The Picture
#### Function Overloading & Name Mangling
```cpp
int max(int x, int y)
{
  if (x>y) return x;
  else return y;
}
float max(float x, float y)
{
  if (x>y) return x;
  else return y;
}
double max(double x, double y)
{
  if (x>y) return x;
  else return y;
}
```
When some other code refers to `max`, which one does it mean?

Because the function signature is known at compile time, different overloaded functions get mangled to different names (i.e. **name mangling**), for example, the names that start with '_Z3max' below.

Name |Value |Class| Type| Size| Line | Section
:-|
__gxx_personality_v0|        |   U  |            NOTYPE|        |     |*UND*
_Z3maxii            |00000000|   T  |              FUNC|00000021|     |.text
_Z3maxff            |00000022|   T  |              FUNC|00000029|     |.text
_Z3maxdd            |0000004c|   T  |              FUNC|00000041|     |.text

> **Note:** When C and C++ code is intermingled, all symbols produced by C++ compiler are mingled; all the symbols produced by C compiler are not. To get around this, the C++ language allows you to put `extern "C"` around the declaration & definition of a function, which tells the C++ compiler not to mangle such functions.
> 
> This is necessary when some C code needs to call C++ functions or vice versa.

#### Initialization of Statics (i.e. global static variables)

In C++, the construction process is allowed to be much more complicated than just copying in a fixed value; all of the code in the various constructors for the class hierarchy has to be run, before the program itself starts running properly.

To deal with this, the compiler includes some extra information in the object files for each C++ file; specifically, the list of constructors that need to be called for this particular file. At link time, the linker combines all of these individual lists into one big list, and includes code that goes through the list one by one, calling all of these global object constructors.

Note that the order in which all of these constructors for global objects get called is not defined—it's entirely at the mercy of what the linker chooses to do. (See Scott Meyers' Effective C++ for more details—Item 4 in the third edition).

#### Templates

Each of these different instantiations of the template involves different actual machine code, so by the time that the program is finally linked, the compiler and linker need to make sure that every instantiation of the template that is used has code included into the program (and no unused template instantiations are included to bloat the program size).

How does the compiler and linker figure that out?

1. **Duplicate initialization**
Each object file contains the code for all of the templates that it uses. These definitions are listed as ***weak symbols***, and this means that when the linker produces the final executable program, it can throw away all *but* *one* of these duplicate definitions.  <br />
The most significant **downside** of this approach is that all of the individual object files take up much more room on the hard disk.

2.  **Deferring instantiation until link time**
None of the template definitions are stored in the object files, and are all left as undefined symbols. At link time, the linker can collect together all of the undefined symbols that actually correspond to template instantiations, and generate the machine code for these instantiations then. <br />
- **Advantage**: saves space in the individual object files
- **Disadvantage**: the linker needs to keep track of where the header file containing the source code, and needs to be able to invoke the C++ compiler at link time (which may slow down the link).

### Dynamically Linked Libraries

A previous section described how using shared libraries means that the final link is deferred until the moment when the program is run. On modern systems, it's possible to defer linking to even later than that via a pair of system calls, `dlopen` and `dlsym`.

... (TBC)

