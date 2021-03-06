
# Software Power On Self-Test

>_Abstract: A 'Power On Self Test (POST)' framework for the C Programming Language is described wherein the framework allows the developer to create unit test routines in the same translation unit as the code, assign testing routines to dependency layers, all while ensuring the testing code does not affect the size of the program at runtime._

>_Reference: The code for this framework and a sample program can be found [here](https://github.com/c-linkage/software-post)._

After reading yet another article on Test Driven Development (TDD) or Unit Testing or some such thing, my fur was up.  Just _thinking_ about all of the effort that goes into testing gave me hives: evaluating different test frameworks and down-selecting, integrating the test framework into the build and deployment systems, developing the tests and mocks, making sure the actual code implements the same behavior of the mocks, scanning the testing logs for failures, etc.  It all just seemed to be just so much ... _activity_ ... that could be better spent, you know, _developing_.

Now, don't get me wrong: **testing is important**.  But having been in development for 20 years before [Kent Beck popularized the concept of Test Driven Development](https://en.wikipedia.org/wiki/Test-driven_development), I had to come up with my own system for making correct software: a rapid development cycle that interleaved coding and testing, sometimes including throw-away test harnesses.

I realized later that the reason I got mad was because my project had hit a level of complexity where unit testing had become necessary, and the article just happend to poke a sore spot. Back-filling tests for half a million lines of C and C# code was not my idea of a good time.

But then I was struck by an idea:

> **Hardware has a [Power On Self Test (POST)](https://en.wikipedia.org/wiki/Power-on_self-test) where successive layers are validated using the capabilities and functions of other already validated layers.  So why can't I have a "power on self test" for software? If I validate the low-level subsystems first, then I could validate successively higher level systems using those lower systems. This would give me both unit testing _and_ integration testing without having to write any mocks.**

Oh yeah.  [Let's do this](https://youtu.be/K2-3YacveiQ)!

## Requirements

I quickly hashed out my requirements:

1. Running the tests should be optional.
2. The testing code should not affect the size of the program at runtime, but it _could_ inflate the size of the binary.
3. I should be able to create tests in whatever translation unit I wanted.
4. Test functions should not require "registration" with some master test coordinator.
5. The framework should be cross-platform (Windows and Linux), but it's okay if it only works on x86 or x64 hardware.

Item (1) was easy: pass a command-line option to the program that would trigger the Software POST.

Item (2) should be easy as well: put all of the testing code into its own [section](https://en.wikipedia.org/wiki/Data_segment) in the compiled binary and the operating system will take care of loading the tests during execution and discarding them when completed.

Item (3) ended up being a bit of a bonus.  I originally set that requirement so I could put the tests into a translation unit _other_ than the one containing the code under test. But I realized during development that it was actually better to put the test and the code in the same file. Doing so meant it was trivial to keep the code and the tests synchronized over time. Combining that with always running the tests when debugging meant I was forced to update the test when out of sync, otherwise the program wouldn't even run!

Item (4) was going to be tricky. But I recalled reading in [Raymond Chen](https://devblogs.microsoft.com/oldnewthing/author/oldnewthing)'s blog [The Old New Thing](https://devblogs.microsoft.com/oldnewthing/)  a series of articles
([1](https://devblogs.microsoft.com/oldnewthing/20181107-00/?p=100155), [2](https://devblogs.microsoft.com/oldnewthing/20181108-00/?p=100165), [3](https://devblogs.microsoft.com/oldnewthing/20181109-00/?p=100175)) about using linker sections to arrange data in a way that could help for developing the framework on Windows.

Item (5) was going to be challenging.  Although I cut my C programming teeth on UNIX, my career path has had me programming Windows for many years. Figuring out the [incantations](https://i.imgur.com/aqg21UI.jpeg) needed to get **gcc** and **ld** to produce an "initializer" style section was going to take some doing. Fortunately, my pass-time activity of browsing the [Linux source code](https://elixir.bootlin.com/linux/latest/source) gave me some hints.

## Final Product 

Before delving into the implementation details, let's see the final product first.

The sample program I developed to test the framework is a simple number guessing game where the user tries to guess one of the ten numbers selected at random from the range of 1 to 100. The numbers are selected and stored in a singly-linked list whose elements are managed on the heap. 

The heap uses a custom allocator to make it possible to detect memory leaks, so it requires a self-test. And since the list implementation relies on the custom allocator, the allocator must be tested before the list.

### Running the Self Test

The skeleton of the main program is below. It parses the command line to see if a self-test is requested, and, if so, runs the self-test using the `self_test_run()` function.  For this example, the default self-test output stream for the platform (Windows or Linux) is used, and the flags that control the self-test behavior are set to  `SELF_TEST_FLAG_NONE` to use the default options.

The program exits with a return value of -1 when the self-test fails. Returning on self-test failure is important for two reasons. First, a self-test failed, so the program likely won't run correctly anyway. Second, the failed self-test likely left the application in an inconsistent state, so continuing execution will likely result in a crash.

```c
int main(int argc, char **argv)
{
    struct list *list;
 
    // Scan parameters to see if f_self_test flag should be set
    parse_args(argc, argv);  

    if (f_self_test)
    {
        // Use the standard error reporting for the platform: 
	// stderr for Linux and OutputDebugString() for Windows.  
	// If the self test fails return -1.

        if (!self_test_run(SELF_TEST_SYSTEM_REPORT, SELF_TEST_FLAG_NONE))
	    return -1;
    }

    // Initialize the allocator and create a list

    mem_init();
    list = mem_create(struct list);
    list_init(list);

    // ...Run the program...

    list_clear(list);
    mem_free(list);

    // Uninitialize allocator and report leaks using mem_leak_detected

    mem_uninit(mem_leak_detected, NULL);

    return 0;
}
```

### Self-Test Template

Every self-test follows the same basic template using the `SELF_TEST()` macro, where the first parameter is the name of the system under test and the second parameter is the "level" at which the test should be performed.  The first parameter must be a valid C identifier since the macro constructs a function using that name.

The `SELF_TEST_ASSERT()` macro evaluates an expression, the value of which must be non-zero for the assertion to be true.

```
SELF_TEST(<name-of-subsystem-under-test>, <self-test-level>)
{
    int rc = 0;
        
    // Run one or more tests
    // Assertion failure causes a jump to the 'failure' label
        
    SELF_TEST_ASSERT(<test-expression>);

    rc = 1;
    
failure:
    return rc;
}
```

_Yes_, the self-test framework requires using `goto`, but that is pretty standard for a C program where exceptions don't exist.

### The List Self-Test

The self-test for the list module is reviewed first; the memory allocator is too complex introdctory purposes and will be reviewed later.

To start with, consider the programming interface for `list`.

```c
// Initialize a list 
void list_init(struct list *list);

// Clear all items from the list and reset it to the initialized state
void list_clear(struct list *list);

// Get a count of items in the list
size_t list_count(struct list *list);

// Return non-zero if the list contains a specific value
int list_contains(struct list *list, int value);

// Add a value to the list and return 1 on success; 0 means out of memory
int list_add(struct list *list, int value);

// Remove a value from the list; do nothing if the value is not present
void list_remove(struct list *list, int value);
```

The six functions above will each need to be tested. And because some of the functions can't be tested without first using other functions (you can't delete an item before you've added one) there will likely be many more tests to validate the interactions between the functions.

Using the template from above, the self-test for the list uses the `SELF_TEST()` macro with  `list` as the name of the subsystem  being tested and `SELF_TEST_DEFAULT_LEVEL` to indicate that this test should be run at the default testing level -- not too high, and not too low.  Temporary objects required for the test are declared and initialized, and the expected results of each function are verified using the `SELF_TEST_ASSERT()` macro.

For this test, I chose to allocate the list on the stack instead of on the heap because I only want to test the list _implementation_ for memory leaks; the `list` object will get cleaned up automatically when the test completes.

```c
SELF_TEST(list, SELF_TEST_LEVEL_DEFAULT)
{
    struct list l_list;             // On the stack to avoid heap use
    struct list *list = &l_list;    // Convenience pointer
    int rc = 0;

    // Initialize already-tested memory subsystem
    mem_init();

    // Verify list initialization
    list_init(list);
    SELF_TEST_ASSERT(list_count(list) == 0);
    
    // ...Do a lot more tests...

    list_clear(list);
    SELF_TEST_ASSERT(list_count(list) == 0);
    
    // Verify the memory allocator resets properly.
    // Failure here could corrupt the program!
    SELF_TEST_ASSERT(mem_uninit(NULL, NULL) == 1);

    rc = 1;

failure:
    return rc;
}
```

The self-test for the  `list` object relies on the memory allocator for creating elements on the heap. This means that there are some important things to consider:

1. The memory allocator must be _initialized_ and _uninitialized_ during the test.
2. The memory allocator must be tested _before_ the list object, otherwise the `list` self-test may be invalid.
3. The memory allocator must be written so that when it is uninitialized it returns to the system to the same state it would be if the self-test had never been run.

### The Memory Allocator Self-Test - Test Support Functions

The self-test for the list shows a relatively simple test using only the default self-test options.  But what if a test needs something more, like special reporting requirements? Or maybe it requires supporting function needed only during the test?  The self-test framework has your back here, too.  

As with the list, planing the self-test for the allocator begins by looking at the interface.

```c
// Function pointer definition for reporting the location of a memory
// leak. An opaque 'data' pointer is also passed in from the 
// mem_uninit() call in case your specific implementation needs extra 
// data like a file handle.

typedef void (*mem_report_pf)(const char *file, int line, void *data);

// Initialize and uninitalize the memory subsystem. Use the reporting
// function during uninitialization to output the leak locations.

extern void mem_init(void);
extern int mem_uninit(mem_report_pf report, void *data);

// Macros that allocate and release memory that wrap the internal 
// implementation

#define mem_alloc(s) mem_alloc_internal((s), __FILE__, __LINE__)
#define mem_free(p) mem_free_internal(p)
#define mem_create(s) (s *)mem_alloc(sizeof(s))

// Internal implementations: DO NOT CALL DIRECTLY

extern void *mem_alloc_internal(size_t size, const char *file, int line);
extern void mem_free_internal(void *ptr);
```

The use of a reporting function in `mem_uninit()` to show leak locations complicates this self-test. But before worrying about that, it's best to build a self-test stub that can be updated as each complexity point is addressed.

To start, the `SELF_TEST()` macro gets `memory` as the name of the subsystem being tested and `SELF_TEST_LEVEL_1` as the test level, indicating that test is on the lowest level and  should be run before all other higher level tests.

The first test in the self-test is the simplest: allocate some memory and free it, verifying that nothing has leaked. The parameters to `mem_uninit()`  and the condition in the `SELF_TEST_ASSERT()` macro are left blank for now because it's not yet clear what _should_ be passed.

```c
SELF_TEST(memory, SELF_TEST_LEVEL_1)
{
    int *p;
    int rc = 0;
    
    // Test for no leaks

    mem_init();
    p = mem_create(int);
    SELF_TEST_ASSERT(p != NULL);
    mem_free(p);
    mem_test_uninit(???,???)
    SELF_TEST_ASSERT(???);
    
    rc = 1;
    
failure:
    return rc;
}
```
 
The `mem_uninit()` interface requires a pointer to a function that will be used to report  memory leaks. Writing leak information to `STDERR` is a bad idea because this is a test and leaks may be expected.  Besides, the self-test must somehow validate the leak information using the  `SELF_TEST_ASSERT()` macro.

Fortunately, the `mem_uninit()` function also has an opaque  `data` parameter that gets passed to the reporting function along with the leak location information. This extra parameter can be used to pass information between the self-test and the memory leak reporting function.

To capture the leak data, a structure is needed that can hold the leak location information.

```c
struct mem_self_test_data {
    const char *file;
    int line;
};
```

Next, the reporting function is defined. In it, the leak location information -- the filename and line number of where the original allocation took place -- are captured into the `mem_self_test_data` structure.

```c
static void SELF_TEST_FUNC mem_report_self_test(
    const char *file, int line, void *data)
{
    struct mem_self_test_data *mstd;
    
    mstd = (struct mem_self_test_data *)data;

    mstd->file = file;
    mstd->line = line;
}
```

Note the `SELF_TEST_FUNC` macro used to decorate the function. Because this function is only used during testing, its important that its code not mix with production code. The `SELF_TEST_FUNC` macro is used here to make sure the code gets installed into the code segment set aside for testing code.

With the reporting function and leak location structures now defined, the missing parameters to `mem_uninit()` can be filled in and the captured leak information can be tested.  Since the expectation is that no memory is leaked, the `file` and `line` fields of the `mem_self_test_data` structure should remain unchanged.

```c
SELF_TEST(memory, SELF_TEST_LEVEL_1)
{
    struct mem_self_test_data mstd;
    int *p;
    int rc = 0;

    //  Initialize the leak capture structure

    mstd.file = NULL;
    mstd.line = 0;

    // Test for no leaks

    mem_init();
    p = mem_create(int);
    SELF_TEST_ASSERT(p != NULL);
    mem_free(p);
    mem_uninit(mem_report_self_test, (void *)&mstd);
    SELF_TEST_ASSERT(mstd.file == NULL); 
    SELF_TEST_ASSERT(mstd.line == 0);
    
    rc = 1;
        
failure:
    return rc;
}
```    
    
### The Memory Allocator Self-Test - Special Reporting

Earlier, you saw the self-test system kicked off by this call:

```c
self_test_run(SELF_TEST_SYSTEM_REPORT, SELF_TEST_FLAG_NONE))
```

What isn't clear from this call is that `SELF_TEST_SYSTEM_REPORT` is actually a sentinel value (technically an alias for a `NULL` pointer) indicating that the default reporting function for your platform should be used. Naturally, you can override the default use your own function if you want. But whatever self-test reporting function is used gets passed as the hidden argument `self_test_report` to the body of the `SELF_TEST()`.  

The self-test reporting function has three arguments: a string message, a file name, and a line number; this should be enough information to zero in on whichever test failed.

```c
typedef void (SELF_TEST_DECL *self_test_report_pf)(
    const char *msg,     // Message to report
    const char *file,    // Filename containing error
    size_t line          // Line number in file
);
```

At any time during a self-test you can call the self-test report function to emit a message into the self-test log.

For example, given the following self-test 

```c
SELF_TEST(jokes, SELF_TEST_LEVEL_DEFAULT)
{
    self_test_report("What's the difference between Dubai and Abu Dabi?", __FILE__, __LINE__);
    self_test_report("The people in Dubai don't like the Flintstones...", __FILE__, __LINE__);
    self_test_report("...but the people in Abu Dabi do!", __FILE__, __LINE__);
    return 1;
}
```    

the output written to the self-test log might look something like this:

    self-test: info: starting self test...
    self-test: info: test jokes
    jokes.c(15): What's the difference between Dubai and Abu Dabi?
    jokes.c(16): The people in Dubai don't like the Flintstones...
    jokes.c(17): ...but the people in Abu Dabi do!
    self-test: info: self-test complete
   
Note that each reported line is designed to look like a compiler-emitted error or warning message suitable for parsing by your IDE of preference.  This helps in Visual Studio, for example, where the use can double-click the message in the Output window and the IDE will transport you directly to the file and line generating the message.

The self-test framework was designed to be relatively quiet, outputting only the name of the self-test being run and any detected errors.  But if you want to emit a messsage that doesn't look like an error or warning, you can pass in `NULL` for the file parameter

```c
SELF_TEST(jokes, SELF_TEST_LEVEL_DEFAULT)
{
    self_test_report("What's the difference between Dubai and Abu Dabi?", NULL, 0);
    self_test_report("The people in Dubai don't like the Flintstones...", NULL, 0);
    self_test_report("...but the people in Abu Dabi do!", NULL, 0);
    return 1;
}
```

and the message will be emitted without file or line information

    self-test: info: starting self test...
    self-test: info: test jokes
    What's the difference between Dubai and Abu Dabi?
    The people in Dubai don't like the Flintstones...
    ...but the people in Abu Dabi do!
    self-test: info: self-test complete

Here is an updated memory self-test that reports the name of the test being run.

```c
SELF_TEST(memory, SELF_TEST_LEVEL_1)
{
    struct mem_self_test_data mstd;
    int *p;
    int rc = 0;

    // Initialize the leak capture structure

    mstd.file = NULL;
    mstd.line = 0;

    // Test for no leaks
    
    self_test_report("memory: initializing the memory subsystem", NULL, 0);
    mem_init();
    self_test_report("memory: allocating an integer on the heap", NULL, 0);
    p = mem_create(int);
    SELF_TEST_ASSERT(p != NULL);
    self_test_report("memory: freeing the integer on the heap", NULL, 0);
    mem_free(p);
    self_test_report("memory: uninitializing the memory subsystem", NULL, 0);
    mem_uninit(mem_report_self_test, (void *)&mstd);
    SELF_TEST_ASSERT(mstd.file == NULL); 
    SELF_TEST_ASSERT(mstd.line == 0);

    rc = 1;
    
failure:
    return rc;
}
```

The output of this self-test might look something like this:

    self-test: info: starting self test...
    self-test: info: test memory
    memory: initializing the memory subsystem
    memory: allocating an integer on the heap
    memory: freeing the integer on the heap
    memory: unintializing the memory subsystem
    self-test: info: self-test complete

## Implementation

The self-test implementation uses a mixture of platform-dependent and platform-independent functions.

The platform-independent section is a very thin wrapper around the platform-dependent functions and, as such, is not that interesting: it mainly does things like parameter validation and emitting some basic messages about beginning and ending the self-test.

The more interesting parts are the platform-dependent functions. This article will discuss the Windows-specific functions as I am primarily a Windows developer and I feel more comfortable discussing the details.  But fear not: the Linux-specific functions work quite similarly, except for the compiler- and linker-specific commands needed to place code into the correct sections.

### Windows Self-Test Driver

Sections in a [Portable Excutable file](https://en.wikipedia.org/wiki/Portable_Executable) are each given names. The standard names you might see are "text" for the machine code, "bss" for uninitialized data (typically filled in with zeros by the operating system loader), "data" for initialized data, and "rdata" for read-only data such as constants.

The self-test framework requires multiple sections for its code and data, so I chose  group them together with the prefix `slftst` -- short for "self test".

There are three primary code sections created under the Microsoft tool chain:

* `slftsti` = "self test initializer" - array of function pointers to self test descriptors
* `slftstt` = "self test text" - executable code for all self tests
* `slftstr` = "self test read-only" - constants (e.g. strings, self-test descriptors) used during the self-test

The `slftsti` section is divided into twelve subsections, ten of which contain the arrays of pointers to the self tests. The last two subsections act as "bookends" that bound that section in memory allowing the self-test driver to scan memory looking for self-test descriptor pointers.

```c
// Always use C linkage

#define SELF_TEST_DECL __cdecl

// Define the "bookends" for the initializer section

#pragma section("slftsti$a", read, shared)
    __declspec(allocate("slftsti$a")) 
    const struct self_test *win32_self_test_start = (struct self_test *)1;

#pragma section("slftsti$z", read, shared)
    __declspec(allocate("slftsti$z")) 
    const struct self_test *win32_self_test_end = (struct self_test *)1;

// Define the names of the sections for each test level

#define SELF_TEST_LEVEL_1  "slftsti$b"
#define SELF_TEST_LEVEL_2  "slftsti$e"
#define SELF_TEST_LEVEL_3  "slftsti$i"
#define SELF_TEST_LEVEL_4  "slftsti$k"
#define SELF_TEST_LEVEL_5  "slftsti$m"
#define SELF_TEST_LEVEL_6  "slftsti$o"
#define SELF_TEST_LEVEL_7  "slftsti$q"
#define SELF_TEST_LEVEL_8  "slftsti$s"
#define SELF_TEST_LEVEL_9  "slftsti$u"
#define SELF_TEST_LEVEL_10 "slftsti$w"

// Set the default test level

#define SELF_TEST_LEVEL_DEFAULT SELF_TEST_LEVEL_5

// Forward declare all sections.
// Unused sections are removed by the linker

#pragma section(SELF_TEST_LEVEL_1, read, shared)
#pragma section(SELF_TEST_LEVEL_2, read, shared)
#pragma section(SELF_TEST_LEVEL_3, read, shared)
#pragma section(SELF_TEST_LEVEL_4, read, shared)
#pragma section(SELF_TEST_LEVEL_5, read, shared)
#pragma section(SELF_TEST_LEVEL_6, read, shared)
#pragma section(SELF_TEST_LEVEL_7, read, shared)
#pragma section(SELF_TEST_LEVEL_8, read, shared)
#pragma section(SELF_TEST_LEVEL_9, read, shared)
#pragma section(SELF_TEST_LEVEL_10, read, shared)

// Forward feclare the section that will contain 
// read-only data, mostly the strings used to print 
// out test names and error messages.

#pragma section("slftstr", read, shared)
```

The sections declared for each level require a bit of explanation.

The Microsoft tool chain allows section names with `$` in them. The part of the identifier before the `$` is the final name of the section, and the part after the `$` is used to order the contents of that section alphabetically.  For example, if there are two sections called `test$a` and `test$b` then the linker will create one section called `test` with the contents of `test$a` placed before the contents of `test$b` in the section.

A challenge when dealing with the Microsoft tool chain is that there are no padding guarantees when the section is finalized. That means that there could be a gap of arbitrary size in section `test` between the contents of `test$a` and `test$b`". What we _are_ guaranteed is that the padding will always be zero.  This is why the bookend pointers `win32_self_test_start` and `win32_self_test_end` are initialized to the "illegal" non-NULL value of 1 to distinguish them from the padding.

Before we can run a self-test, a self-test must be defined and installed into the correct section using the following macros:

```c
#define SELF_TEST_FUNC __declspec(code_seg("slftstt")) SELF_TEST_DECL
#define SELF_TEST_RO __declspec(allocate("slftstr"))
#define SELF_TEST_LEVEL(l) __declspec(allocate(l))

#define SELF_TEST(n,l) \
    extern int SELF_TEST_FUNC self_test_##n(self_test_report_pf); \
    static const char SELF_TEST_RO self_test_msg_##n[] = "self-test: info: test " #n; \
    static const struct self_test SELF_TEST_RO self_test_desc_##n = { self_test_##n, self_test_msg_##n }; \
    struct self_test SELF_TEST_LEVEL(l) *self_test_ptr_##n = &self_test_desc_##n; \
    int SELF_TEST_FUNC self_test_##n(self_test_report_pf self_test_report)
```

The first three macros encapsulate the compiler-specific text and serve to make the `SELF_TEST()` macro easier to read.  

* `SELF_TEST_FUNC` decorates a function to give it the calling convention `SELF_TEST_DECL` and to place it into the self-test code segment `slftstt`.
* `SELF_TEST_RO` decorates a variable to place it into the self-test read-only section`slftstr`. In almost all cases, variables decorated with `SELF_TEST_RO` should also be  decorated with `const` to avoid section permission mismatches.
* `SELF_TEST_LEVEL(l)` decorates the pointer to the self-test descriptor to place it into the self-test initializer section appropriate for the testing level.

Given the first three macros, the `SELF_TEST()` macro is a little easier to understand:

1. Forward-declare the self-test function, whose name is composed of `self_test_` and the name of the subsystem being tested; this is needed later when defining the self-test descriptor.
2. Define a text string to be printed when the self-test is executed, and install that string into the self-test read-only section.
3. Create the self-test descriptor -- a structure containing the self-test message and a pointer to the self-test code -- and install it into the self-test read-only section.
4. Install a pointer to the descriptor into the self-test initialization section for the requested test level
5. Begin the definition of the self-test function.

To complete the example, consider the following extract from the `jokes.c` example earlier:

```c
SELF_TEST(jokes, SELF_TEST_LEVEL_DEFAULT)
{
    self_test_report("What's the difference between Dubai and Abu Dabi?", NULL, 0);
    self_test_report("The people of Dubai don't like the Flintstones...", NULL, 0);
    self_test_report("...but the people of Abu Dabu do!", NULL, 0);
    return 1;
}
```

After expanding all of the macros and adding some comments, the full implementation becomes clear:

```c
// Forward-declare the test
extern int __declspec(code_seg("slftstt")) __cdecl self_test_jokes(self_test_report_pf);

// Define the test message and install it into the self-test read-only section
static const char __declspec(allocate("slftstr")) self_test_msg_jokes[] = "self-test: info: test " "jokes"; 

// Define the test descriptor and install it into the self-test read-only section
static struct self_test __declspec(allocate("slftstr")) self_test_desc_jokes = { self_test_jokes, self_test_msg_jokes }; 

// Install a pointer to the descriptor into the section for the requested self-test initialization level
struct self_test __declspec(allocate("slftsti$m")) *self_test_ptr_jokes = &self_test_desc_jokes; 

// Define the self-test function and install it into the self-test code section
int __declspec(code_seg("slftstt")) __cdecl self_test_jokes(self_test_report_pf self_test_report)
{
    self_test_report("What's the difference between Dubai and Abu Dabi?", ((void *)0), 0);
    self_test_report("The people of Dubai don't like the Flintstones...", ((void *)0), 0);
    self_test_report("...but the people of Abu Dabu do!", ((void *)0), 0);
    return 1;
}
```

It should be clear now that the initializer section `slftsti` gets populated with pointers to self-test descriptors. The decriptor pointers within the section are partially ordered: all Level 1 pointers are installed before all Level 2 pointers, but the order of descriptor pointers _within_ a level is arbitrary. The reason for installing pointers in the initializer rather than the self-test descriptors themselves is that the alignment and padding requirements for pointers are well known, and this makes scanning the section much easier.

For the sample program, verifying that the descriptor pointers are ordered correctly in the right section can be confirmed by using `dumpbin`:

    SECTION HEADER #6
     slftsti name
          10 virtual size
       26000 virtual address (00426000 to 0042600F)
         200 size of raw data
       22800 file pointer to raw data (00022800 to 000229FF)
           0 file pointer to relocation table
           0 file pointer to line numbers
           0 number of relocations
           0 number of line numbers
    50000040 flags
             Initialized Data
             Shared
             Read Only
    
    RAW DATA #6
      00426000: 01 00 00 00 68 4F 42 00 1C 40 42 00 01 00 00 00  ....hOB..@B.....

Since this example was compiled for the 32-bit x86 architecture, each pointer takes four bytes. The first four bytes and the last four bytes are the bookend pointers used to bound the section. Bytes 5 through 8 form the pointer 0x00424F68 that points to the memory system self-test, and bytes 9 through 12 form the pointer 0x0042401C that points to the list system self-test.  It just so happens that there are no gaps in the initializer section, so there are no NULL pointers to skip. But having gaps in the initializer section is quite common, so the code must be prepared for that.

Actually _running_ the self-tests is rather simple: scan the self-test initializer section `slftsti` between the bookends looking for non-NULL pointers. When a non-NULL pointer is found, interpret that pointer as the address of a self-test descriptor. Print the descriptor message and then run the self-test.

```c
int sys_self_test_run(self_test_report_pf report, unsigned flags)
{
    const struct self_test **test;
    int rc;
    
    rc = 1;                         // Assume success
    test = &win32_self_test_start;  // Scan from the first bookend
        
    while(++test < &win32_self_test_end)  // Scanning to second bookend
    {
        // Look for non-null pointer to a descriptor
        
        if ((*test) != NULL)  
        {
            // Print the test message if it's non NULL
            
            if ((*test)->name != NULL) 
                report((*test)->name, NULL, 0);

            // Run the test and if the flags ask to stop on failure
	    // then return immediately
            
            if (!(*test)->func(report)) 
            {
                if (flags & SELF_TEST_FLAG_STOP_ON_FAILURE) 
                    return 0;
                rc = 0;
            }
            
            // Use the built-in Microsoft C runtime memory-leak 
	    // detector to report and leaks that occured during 
	    // self-test
            
            _CrtDumpMemoryLeaks();
        }
    }

    return rc;
}

```

## Conclusion

My dislike of writing mocks combined with the dread of having to select from the many different unit-testing frameworks inspired me to take the concept of a hardware power-on self test (POST) and apply it to software. The Software POST integrates quite easily into both new and existing code because there is no need to manually register test functions in the project and because there is no need to configure an external test driver. Integration testing is virtually free because tests can take advantage of subsystems verified earlier in the POST. Taking advantage of the compiler directives to prevent mixing test and production code means you can leave the tests in a production release without worring about the tests taking up space at runtime.  Finally, opting to run the tests each time the program is executed forces the programmer to keep the code and the tests synchronized.
