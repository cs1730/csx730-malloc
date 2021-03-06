# csx730-malloc

> Without memory, there is no culture. 
> Without memory, there would be no civilization, no society, no future.
> __--Elie Wiesel__

**DUE MON 2019-04-01 (Apr 01) @ 11:55 PM**

This repository contains the skeleton code for the `csx730-malloc` project
assigned to the students in the Spring 2019 CSCI 4730/6730 class
at the University of Georgia.

## Table of Contents

1. [Academic Honesty](#academic-honesty)
1. [Memory Allocation](#memory-allocation)
   1. [The User API](#the-user-api)
   1. [The Developer API](#the-developer-api)
   1. [How to Manage your Heap](#how-to-manage-your-heap)
   1. [Block Metadata](#block-metadata)
   1. [Examples](#examples)
1. [How to Get the Skeleton Code](#how-to-get-the-skeleton-code)
1. [Project Requirements](#project-requirements)
   1. [Functional Requirements](#functional-requirements)
   1. [6730 Requirements](#6730-requirements)
   1. [Non-Functional Requirements](#non-functional-requirements)
1. [Submission Instructions](#submission-instructions)

## Academic Honesty

You agree to the Academic Honesty policy as outlined in the course syllabus. 
In accordance with this notice, I must caution you **not** to fork this repository
on GitHub if you have an account. Doing so will more than likely make your copy of
the project publicly visible. Please follow the instructions contained in the 
[How to Get the Skeleton Code](#how-to-get-the-skeleton-code) section below in
order to do your development on Nike. Furthermore, you must adhere to the copyright
notice and licensing information at the bottom of this document.

## Memory Allocation

The heap or free store is the memory area of a process reserved by dynamic allocation.
In most C programs, this area is managed by `malloc(3)` and `free(3)`. 
In this project, you are tasked with implementing the `malloc(3)` family of functions
in C using a non-cryptographic block chain! Essentially, your code will be in charge
of managing the process's heap by manually changing the program break and keeping
track of what parts of your heap are available (i.e., free) and unavailable (i.e.,
used) via a linked list. Since the linked list elements refer to memory "blocks", many 
memory allocator enthusiasts refer to this implementation as a block chain. This term
should not be confused with the cryptographic block chains used for cryptocurrencies.
Mixing your implementation with GLIBC's `malloc(3)` and `free(3)` is undefined. 
Some starter code is provided. Other project details are provided below.

### The User API

The primary User API provides two functions:

* __`void * csx730_malloc(size_t size);`__<br>
  Allocates `size` bytes and returns a pointer to the allocated memory. The memory is not cleared.
  If `size` is `0`, then `csx730_malloc` returns either `NULL`, or a unique pointer value that 
  can later be successfully passed to the `csx730_free` function. On failure, returns a `NULL` 
  pointer. **NOTE:** Unlike `malloc(3)`, the underlying memory block is _not required_ to be 
  suitably aligned for any object type with 
  [fundamental alignment](https://en.cppreference.com/w/c/language/object#Alignment). 
  
* __`void csx730_free(void * ptr);`__<br>
  Frees the memory space pointed to by `ptr`, which must have been returned by a previous call to
  the `csx730_malloc`, `csx730_calloc`, or `csx730_realloc` function. If `ptr` is `NULL`, then
  no operation is performed. If `csx730_free(ptr)` has already been called, then the behavior is
  undefined.

### The Developer API

The Developer API provides three functions:

* __`void csx730_pheapstats(void);`__<br>
  Prints a summary of the heap to standard output. At a minimum, the following information
  should be printed: the pagesize, the original program break, the current program break,
  the total heap size, free size, used size, and the size of a block's metadata structure. 
  Sizes denote a number of bytes and should be presented in decimal. Optionally, they
  may also be presented in hexadecimal. 
  
  * Here is an annoted example (`//` comments should be omitted in actual output):
    
    ```
    csx730_pheapstats()
    {
      .initialized = TRUE
      .page_size   = 4096 (0x1000)  // return value of getpagesize(2) 
      .brk0        = 0x24dc000      // original program break address
      .brk         = 0x24de000      // actual program break address
      .total_size  = 8192 (0x2000)  // total heap size
      .used_size   = 5273 (0x1499)  // bytes used
      .free_size   = 2775 (0xad7)   // bytes free
      .head_meta   = 0x24dc000      // address of first block meta
      .meta_size   = 24 (0x18)      // size of block metadata structure
    }
    ```
  
  * More examples are [provided below](#examples).
  
  * I reccommend storing this in a "private" global anonymous structure, e.g., 
  
    ```c
    struct {
      size_t page_size;
    } _heap;
    ```

* __`void csx730_pheapmap(void);`__<br>
  Prints a memory map of the heap to standard outpt. Each non-separator line in the output 
  denotes a memory address. If the memory address is a multiple of the pagesize, then that
  line is prefixed with a `P`. The address itself is printed in 16-digit hexadeximal, padded 
  with zeros. To the right of each address is a description, which should be present for
  the following: the original program break, the current program break, the first address
  in the metadata for a memory block, the start of a memory block's allocated memory, and the
  end of a memory block's allocated memory. Here, the term "allocated" refers to the memory allocated
  for use by the user. If an allocated region crosses a page boundary, then that address
  should be printed. 
  
  * Here is an example:
  
    ```
    csx730_pheapmap()
      ------------------
    P 0x00000000024dc000 original program break
      ------------------
    P 0x00000000024dc000 used block
      0x00000000024dc018 start (100 bytes)
      0x00000000024dc07c end
      ------------------
      0x00000000024dc07c used block
      0x00000000024dc094 start (100 bytes)
      0x00000000024dc0f8 end
      ------------------
      0x00000000024dc0f8 used block
      0x00000000024dc110 start (1025 bytes)
      0x00000000024dc511 end
      ------------------
      0x00000000024dc511 free block
      0x00000000024dc529 start (2775 bytes)
    P 0x00000000024dd000 end
      ------------------
    P 0x00000000024dd000 used block
      0x00000000024dd018 start (4048 bytes)
      0x00000000024ddfe8 end
      ------------------
      0x00000000024ddfe8 free block
    P 0x00000000024de000 start (0 bytes)
    P 0x00000000024de000 end
      ------------------
    P 0x00000000024de000 program break
    ```
    
  * More examples are [provided below](#examples).
  
### How to Manage your Heap 

Your code will be in charge of the process's heap by manually adjusting the program break,
as needed, using `brk(2)` and `sbrk(2)`. As such, mixing your implementation with GLIBC's 
`malloc(3)` and `free(3)` is undefined. 
For this project, we will define the term "heap" as the virtual memory between the original 
program break and the new program break.
Your heap should consist of a sequence of "blocks", which are defined here as
contiguous areas of virtual memory within the heap space.  
The blocks themselves should also be contiguous within the heap
are some requirements. 
Each block consists of two parts, a metadata structure and a region of contiguous memory
that is either "used" or "free". A "used" block is one in which `csx730_malloc` has
"allocated" for use by a user. A "free" block is one that is available to be "allocated"
for use by a user.

Your heap must be aligned to memory pages. On most systems, the original program break
should be a multiple of the page size (e.g., from `getpagesize(2)`) and thus at the 
beginning of a page. Here, page alignment specifically means that your total heap 
size needs to be a multiple of the page size. Futhermore, multipage blocks must also
be aligned to memory pages -- that is, the address of such a block's metadata structure
should be a multiple of the page size.

Your heap must also minimize its heap size in addition to being aligned to memory pages.
This means that whenever `csx730_malloc` and `csx730_free` are used, you should ensure
that the total heap size is as small as it can be given the alignment property. 
Multiple examples are [provided below](#examples).

### Block Metadata  

The metadata for each block usually consists of the block's state (i.e., used or free)
as well as some pointers to other blocks. Many implementations use one pointer to maintain
a list of all blocks and the other pointer to maintain a list of all free blocks. When
a linked list of free blocks is implemented, the implementation is referred to as a
non-cryptographic free block chain. In the examples [provided below](#examples), each
block's metadata structure is `24` bytes.

### Examples

Initially, the process has no heap:

```
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 program break
csx730_pheapstats()
{
  .initialized = FALSE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24dc000
  .total_size  = 0 (0x0) 
  .used_size   = 0 (0x0) 
  .free_size   = 0 (0x0) 
  .head_meta   = (nil)
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate `32` bytes of memory using `csx730_malloc`.
Since the heap is required to be aligned to memory pages, this results
in the automatic creation of a free block. In general, a free block
is always created at the end of the heap -- sometimes this results in
additional page.

```
csx730_malloc(32) = 0x24dc018 [append]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (32 bytes)
  0x00000000024dc038 end
  ------------------
  0x00000000024dc038 free block
  0x00000000024dc050 start (4016 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24dd000
  .total_size  = 4096 (0x1000) 
  .used_size   = 32 (0x20) 
  .free_size   = 4016 (0xfb0) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate `4048` bytes of memory using `csx730_malloc`.
Since the heap free block at the end of the heap is not big
enough, the total heap size is increased.

```
csx730_malloc(4048) = 0x24dd018 [append]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (32 bytes)
  0x00000000024dc038 end
  ------------------
  0x00000000024dc038 free block
  0x00000000024dc050 start (4016 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24de000
  .total_size  = 8192 (0x2000) 
  .used_size   = 4080 (0xff0) 
  .free_size   = 4016 (0xfb0) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate `16384` bytes of memory using `csx730_malloc`.
On this particular system, that is larger that the page size.
Therefore, the total heap size is increased and the block's
metadata structure is aligned to a page. In the output
of `csx730_pheapmap`, you can see that multiple pages exist
in the used block's allocated memory region.

```
csx730_malloc(16384) = 0x24de018 [append]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (32 bytes)
  0x00000000024dc038 end
  ------------------
  0x00000000024dc038 free block
  0x00000000024dc050 start (4016 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 used block
  0x00000000024de018 start (16384 bytes)
P 0x00000000024df000 
P 0x00000000024e0000 
P 0x00000000024e1000 
P 0x00000000024e2000 
  0x00000000024e2018 end
  ------------------
  0x00000000024e2018 free block
  0x00000000024e2030 start (4048 bytes)
P 0x00000000024e3000 end
  ------------------
P 0x00000000024e3000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24e3000
  .total_size  = 28672 (0x7000) 
  .used_size   = 20464 (0x4ff0) 
  .free_size   = 8064 (0x1f80) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Now we free the memory at address `0x24dc018`
using `csx730_free`. Instead of having two
contiguous free blocks, the implementation
joined them together into a block of size
`4096` with `4072` bytes of memory available for
future allocation.

```
csx730_free(0x24dc018)
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 free block
  0x00000000024dc018 start (4072 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 used block
  0x00000000024de018 start (16384 bytes)
P 0x00000000024df000 
P 0x00000000024e0000 
P 0x00000000024e1000 
P 0x00000000024e2000 
  0x00000000024e2018 end
  ------------------
  0x00000000024e2018 free block
  0x00000000024e2030 start (4048 bytes)
P 0x00000000024e3000 end
  ------------------
P 0x00000000024e3000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24e3000
  .total_size  = 28672 (0x7000) 
  .used_size   = 20432 (0x4fd0) 
  .free_size   = 8120 (0x1fb8) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Now we free the memory at address `0x24de018`
using `csx730_free`. Since that frees up the
multipage block at the end of the heap, the
implementation shrinks the heap by moving the
program break accordingly.

```
csx730_free(0x24de018)
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 free block
  0x00000000024dc018 start (4072 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24de000
  .total_size  = 8192 (0x2000) 
  .used_size   = 4048 (0xfd0) 
  .free_size   = 4072 (0xfe8) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate `100` bytes of memory using `csx730_malloc`.
Since there is enough space available in the first free block,
the implementation splits that free block and converts the first
one into a block, adjusting the sizes of both blocks accordingly.
The output shows that `csx730_malloc` is performing an "emplace"
operation instead of an "append" operation on the heap. 

```
csx730_malloc(100) = 0x24dc018 [emplace]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (100 bytes)
  0x00000000024dc07c end
  ------------------
  0x00000000024dc07c free block
  0x00000000024dc094 start (3948 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24de000
  .total_size  = 8192 (0x2000) 
  .used_size   = 4148 (0x1034) 
  .free_size   = 3948 (0xf6c) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate another `100` bytes of memory using `csx730_malloc`.
Since there is enough space available in the first free block,
the implementation splits that free block and converts the first
one into a block, adjusting the sizes of both blocks accordingly.
The output shows that `csx730_malloc` is performing an "emplace"
operation instead of an "append" operation on the heap. 

```
csx730_malloc(100) = 0x24dc094 [emplace]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (100 bytes)
  0x00000000024dc07c end
  ------------------
  0x00000000024dc07c used block
  0x00000000024dc094 start (100 bytes)
  0x00000000024dc0f8 end
  ------------------
  0x00000000024dc0f8 free block
  0x00000000024dc110 start (3824 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24de000
  .total_size  = 8192 (0x2000) 
  .used_size   = 4248 (0x1098) 
  .free_size   = 3824 (0xef0) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Next, we allocate `1025` bytes of memory using `csx730_malloc`.
Since there is enough space available in the first free block,
the implementation splits that free block and converts the first
one into a block, adjusting the sizes of both blocks accordingly.
The output shows that `csx730_malloc` is performing an "emplace"
operation instead of an "append" operation on the heap. 

```
csx730_malloc(1025) = 0x24dc110 [emplace]
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 used block
  0x00000000024dc018 start (100 bytes)
  0x00000000024dc07c end
  ------------------
  0x00000000024dc07c used block
  0x00000000024dc094 start (100 bytes)
  0x00000000024dc0f8 end
  ------------------
  0x00000000024dc0f8 used block
  0x00000000024dc110 start (1025 bytes)
  0x00000000024dc511 end
  ------------------
  0x00000000024dc511 free block
  0x00000000024dc529 start (2775 bytes)
P 0x00000000024dd000 end
  ------------------
P 0x00000000024dd000 used block
  0x00000000024dd018 start (4048 bytes)
  0x00000000024ddfe8 end
  ------------------
  0x00000000024ddfe8 free block
P 0x00000000024de000 start (0 bytes)
P 0x00000000024de000 end
  ------------------
P 0x00000000024de000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24de000
  .total_size  = 8192 (0x2000) 
  .used_size   = 5273 (0x1499) 
  .free_size   = 2775 (0xad7) 
  .head_meta   = 0x24dc000
  .meta_size   = 24 (0x18) 
}
```

Now we free all currently allocated memory using multiple
calls to `csx730_free`.
The implementation recognizes that no blocks are needed
and moves the program break back to its original position.

```
csx730_free(0x24dd018)
csx730_free(0x24dc018)
csx730_free(0x24dc094)
csx730_free(0x24dc110)
csx730_pheapmap()
  ------------------
P 0x00000000024dc000 original program break
  ------------------
P 0x00000000024dc000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x24dc000
  .brk         = 0x24dc000
  .total_size  = 0 (0x0) 
  .used_size   = 0 (0x0) 
  .free_size   = 0 (0x0) 
  .head_meta   = (nil)
  .meta_size   = 24 (0x18) 
}
```

In this final example, we allocate `4072` bytes of memory using `csx730_malloc`.
On this system and implementation, this causes the used block that is created 
to be the same size as a page. Theoretically, this would only require a single
block, however, the implementation chooses to create a free block at the end
of the heap, requiring an additional page.

```
csx730_malloc(4072) = 0x1f34018 [append]
csx730_pheapmap()
  ------------------
P 0x0000000001f34000 original program break
  ------------------
P 0x0000000001f34000 used block
  0x0000000001f34018 start (4072 bytes)
P 0x0000000001f35000 end
  ------------------
P 0x0000000001f35000 free block
  0x0000000001f35018 start (4072 bytes)
P 0x0000000001f36000 end
  ------------------
P 0x0000000001f36000 program break
csx730_pheapstats()
{
  .initialized = TRUE
  .page_size   = 4096 (0x1000)
  .brk0        = 0x1f34000
  .brk         = 0x1f36000
  .total_size  = 8192 (0x2000)
  .used_size   = 4072 (0xfe8)
  .free_size   = 4072 (0xfe8)
  .head_meta   = 0x1f34000
  .meta_size   = 24 (0x18)
}

```

## How to Get the Skeleton Code

On Nike, execute the following terminal command in order to download the project
files into a subdirectory within your present working directory:

```
$ git clone https://github.com/cs1730/csx730-malloc.git
```

This should create a directory called `csx730-malloc` in your present working directory that
contains the project files. For this project, the only files that are included with the project
download are listed near the top of the page [here](https://github.com/cs1730/csx730-malloc).

Here is a table that briely outlines each file in the skeleton code:

| File                      | Description                                                      |
|---------------------------|------------------------------------------------------------------|
| `Doxyfile`                | Configuration file for `doxygen`.                                |
| `Makefile`                | Configuration file for `make`.                                   |
| `README.md`               | This project description.                                        |
| `SUBMISSION.md`           | Student submission information.                                  |
| `csx730_malloc.c`         | Where you will put most of your implementation.                  |
| `csx730_malloc.h`         | Thread structures, function prototypes, and macros.              |

If any updates to the project files are announced by your instructor, then you can
merge those changes into your copy by changing into your project directory
on Nike and issuing the following terminal command:

```
$ git pull
```

If you have any problems with any of these procedures, then please contact
your instructor via Piazza. 

## Project Requirements

This assignment is worth 100 points. The lowest possible grade is 0, and the 
highest possible grade is 120 (if you are enrolled in CSCI 4730).

### Functional Requirements

A functional requirement is *added* to your point total if satisfied.
There will be no partial credit for any of the requirements that simply 
require the presence of a function related a particular functionality. 
The actual functionality is tested using test cases.

1. __(40 points) Project Compiles.__ Your submission compiles and can successfully
   link with object files expecting the symbols defined in `csx730_malloc.h`. 
   Please be aware that the __Build Compliance__ non-functional requirement still
   applies.

1. __(60 points) Implement `csx730_malloc.h` functions in `csx730_malloc.c`.__
   Each of the functions whose prototype appears in the header and does not require
   the `_CS6760_SOURCE` feature test macro must be implemented correctly in the
   corresponding `.c` file. Here is a list of the functions forming the user API:

   * __(15 points)__ `void * csx730_malloc(size_t size);`
   * __(15 points)__ `void csx730_free(void * ptr);`

   Here is a list of functions forming the developer API:
   
   * __(15 points)__ `void csx730_pheapstats(void);`
   * __(15 points)__ `void csx730_pheapmap(void);`

   The documentation for each function is provided directly in
   the header. You may generate an HTML version of the corresponding
   documentation using the `doc` target provided by the project's `Makefile`.
   Students should not modify the prototypes for these functions in any way--doing
   so will cause the tester used by the grader to fail.

   You are free,  _actively encouraged_, and will likely need to write other functions, as needed,
   to support the required set of functions. It is recommended that you give private function
   names a `_csx730_` prefix. 

### 6730 Requirements

For the purposes of grading, a 6730 requirement is treated as a non-functional requirement
for students enrolled in CSCI 6730 and a functional requirement for students enrolled in
CSCI 4730. This effectively provides an extra credit opportunity to the undergraduate
students and a non-compliance penalty for the graduate students.

1. __(20 points) [CS6730] Implement `_CS6730_SOURCE` features__
   The `_CS6730_SOURCE` feature test macro should enable the following addtional 
   functions to the public API:

   * __(10 points)__ `void * csx730_calloc(size_t nmemb, size_t size);`
   * __(10 points)__ `void * csx730_realloc(void * ptr, size_t size);`

   The documentation for each function is provided directly in
   the header. You may generate an HTML version of the corresponding
   documentation using the `doc` target provided by the project's `Makefile`.
   Students should not modify the prototypes for these functions in any way--doing
   so will cause the tester used by the grader to fail.

### Non-Functional Requirements

A non-functional requirement is *subtracted* from your point total if not satisfied. In order to
emphasize the importance of these requirements, non-compliance results in the full point amount
being subtracted from your point total. That is, they are all or nothing.

1. __(100 points) Build Compliance:__ Your project must compile on Nike
   using the provided `Makefile` and `gcc` (GCC) 8.2.0. Your code must compile,
   assemble, and link with no errors or warnings. The required compiler
   flags for this project are already included in the provided `Makefile`.

   The grader will compile and test your code using the `all` and `test` targets in
   the provided `Makefile`. The `test` target will not work until the test driver
   is provided during grading. If your code compiles, assembles, and links
   with no errors or warnings usign the `all` target, then it will very likely
   do the same with the `test` target.

1. __(100 points) Libraries:__ You are allowed to use any of the C standard library
   functions, unless they are explicitly forbidden below. A general reference for
   the C standard library is provided [here](https://en.cppreference.com/w/c).
   No other libraries are permitted. You are also **NOT**
   allowed to use any of the following: 
   * `malloc(3)`, 
   * `free(3)`,
   * `mmap(2)`, 
   * `calloc(3)`, and 
   * `realloc(3)`.

1. __(100 points) `SUBMISSION.md`:__ Your project must include a properly formatted 
   `SUBMISSION.md` file that includes, at a minimum, the following information:

   * __Author Information:__ You need to include your full name, UGA email, and
     which course you are enrolled in (e.g., CSCI 4730 or CSCI 6730).

   * __Implementation Overview:__ You need to include a few sentences that provide an overview
     of your implementation.
     
   * __6730 Requirements:__ You need to include a list of the 6730 requirements that you
     would like the grader to check. These requirements will only be checked if you
     include them in this list. 
     
   * __Reflecton:__ You need to provide answers to the following questions:

     1. What do you think was the motivation behind assigning this project in this class?
     1. What is the most important thing you learned in this project?
     1. What do you wish you had spent more time on or done differently?
     1. What part of the project did you do your best work on?
     1. What was the most enjoyable part of this project?
     1. What was the least enjoyable part of this project?
     1. How could your instructor change this project to make it better next time?

   A properly formatted sample `SUBMISSION.md` file is provided that may be modified to
   help fulfill this non-functional requirement.

1. __(25 points) Code Documentation:__ Any new functions or macros must be properly documented
   using Javadoc-style comments. An example of such comments can be seen in the souce code
   provided with the project. Please also use inline documentation, as needed, to explain
   ambiguous or tricky parts of your code.

## Submission Instructions

You will still be submitting your project via Nike. Make sure your project files
are on `nike.cs.uga.edu`. Change into the parent directory of your
project directory. If you've followed the instructions provided in earlier in this
document, then the name of your project directory is likely `csx730-malloc`.
While in your project parent directory, execute the following command: 

```
$ submit csx730-malloc csx730
```

If you have any problems submitting your project then please make a private Piazza
post to "Instructors" as soon as possible. 

<hr/>

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg)](http://creativecommons.org/licenses/by-nc-nd/4.0/)

<small>
Copyright &copy; Michael E. Cotterell and the University of Georgia.
This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-nd/4.0/">Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License</a> to students and the public.
The content and opinions expressed on this page do not necessarily reflect the views of nor are they endorsed by the University of Georgia or the University System of Georgia.
</small>

