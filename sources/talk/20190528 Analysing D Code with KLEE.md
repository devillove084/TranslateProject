[#]: collector: (lujun9972)
[#]: translator: ( )
[#]: reviewer: ( )
[#]: publisher: ( )
[#]: url: ( )
[#]: subject: (Analysing D Code with KLEE)
[#]: via: (https://theartofmachinery.com/2019/05/28/d_and_klee.html)
[#]: author: (Simon Arneaud https://theartofmachinery.com)

Analysing D Code with KLEE
======

[KLEE][1] is symbolic execution engine that can rigorously verify or find bugs in software. It’s designed for C and C++, but it’s just an interpreter for LLVM bitcode combined with theorem prover backends, so it can work with bitcode generated by `ldc2`. One catch is that it needs a compatible bitcode port of the D runtime to run normal D code. I’m still interested in getting KLEE to work with normal D code, but for now I’ve done some experiments with `-betterC` D.

### How KLEE works

What makes KLEE special is its support for two kinds of variables: concrete and symbolic. Concrete variables are just like the normal variables in normal code: they have a deterministic value at any given point in the program. On the other hand, symbolic variables contain a bundle of logical constraints instead of values. Take this code:

```
int x = klee_int("x");
klee_assume(x >= 0);
if (x > 42)
{
    doA(x);
}
else
{
    doB(x);
    assert (3 * x != 21);
}
```

`klee_int("x")` creates a symbolic integer that will be called “`x`” in output reports. Initially it has no contraints and can imply any value that a 32b signed integer can have. `klee_assume(x >= 0)` tells KLEE to add `x >= 0` as a constraint, so now we’re only analysing the code for non-negative 32b signed integers. On hitting the `if`, KLEE checks if both branches are possible. Sure enough, `x > 42` can be true or false even with the constraint `x >= 0`, so KLEE has to _fork_. We now have two processes being interpreted on the VM: one executing `doA()` while `x` holds the constraints `x >= 0, x > 42`, and another executing `doB()` while `x` holds the contraints `x >= 0, x <= 42`. The second process will hit the `assert` statement, and KLEE will try to prove or disprove `3 * x != 21` using the assumptions `x >= 0, x <= 42` — in this case it will disprove it and report a bug with `x = 7` as a crashing example.

### First steps

Here’s a toy example just to get things working. Suppose we have a function that makes an assumption for a performance optimisation. Thankfully the assumption is made explicit with `assert` and is documented with a comment. Is the assumption valid?

```
int foo(int x)
{
    // 17 is a prime number, so let's use it as a sentinel value for an awesome optimisation
    assert (x * x != 17);
    // ...
    return x;
}
```

Here’s a KLEE test rig. The KLEE function declarations and the `main()` entry point need to have `extern(C)` linkage, but anything else can be normal D code as long as it compiles under `-betterC`:

```
extern(C):

int klee_int(const(char*) name);

int main()
{
    int x = klee_int("x");
    foo(x);
    return 0;
}
```

It turns out there’s just one (frustrating) complication with running `-betterC` D under KLEE. In D, `assert` is handled specially by the compiler. By default, it throws an `Error`, but for compatibility with KLEE, I’m using the `-checkaction=C` flag. In C, `assert` is usually a macro that translates to code that calls some backend implementation. That implementation isn’t standardised, so of course various C libraries work differently. `ldc2` actually has built-in logic for implementing `-checkaction=C` correctly depending on the C library used.

KLEE uses a port of [uClibc][2], which translates `assert()` to a four-parameter `__assert()` function, which conflicts with the three-parameter `__assert()` function in other implementations. `ldc2` uses LLVM’s (target) `Triple` type for choosing an `assert()` implementation configuration, but that doesn’t recognise uClibc. As a hacky workaround, I’m telling `ldc2` to compile for Musl, which “tricks” it into using an `__assert_fail()` implementation that KLEE happens to support as well. I’ve opened [an issue report][3].

Anyway, if we put all that code above into a file, we can compile it to KLEE-ready bitcode like this:

```
ldc2 -g -checkaction=C -mtriple=x86_64-linux-musl -output-bc -betterC -c first.d
```

`-g` is optional, but adds debug information that can be useful for later analysis. The KLEE developers recommend disabling compiler optimisations and letting KLEE do its own optimisations instead.

Now to run KLEE:

```
$ klee first.bc
KLEE: output directory is "/tmp/klee-out-1"
KLEE: Using Z3 solver backend
warning: Linking two modules of different target triples: klee_int.bc' is 'x86_64-pc-linux-gnu' whereas 'first.bc' is 'x86_64--linux-musl'

KLEE: ERROR: first.d:4: ASSERTION FAIL: x * x != 17
KLEE: NOTE: now ignoring this error at this location

KLEE: done: total instructions = 35
KLEE: done: completed paths = 2
KLEE: done: generated tests = 2
```

Straight away, KLEE has found two execution paths through the program: a happy path, and a path that fails the assertion. Let’s see the results:

```
$ ls klee-last/
assembly.ll
info
messages.txt
run.istats
run.stats
run.stats-journal
test000001.assert.err
test000001.kquery
test000001.ktest
test000002.ktest
warnings.txt
```

Here’s the example that triggers the happy path:

```
$ ktest-tool klee-last/test000002.ktest
ktest file : 'klee-last/test000002.ktest'
args       : ['first.bc']
num objects: 1
object 0: name: 'x'
object 0: size: 4
object 0: data: b'\x00\x00\x00\x00'
object 0: hex : 0x00000000
object 0: int : 0
object 0: uint: 0
object 0: text: ....
```

Here’s the example that causes an assertion error:

```
$ cat klee-last/test000001.assert.err
Error: ASSERTION FAIL: x * x != 17
File: first.d
Line: 4
assembly.ll line: 32
Stack:
    #000000032 in _D5first3fooFiZi () at first.d:4
        #100000055 in main (=1, =94262044506880) at first.d:16
$ ktest-tool klee-last/test000001.ktest
ktest file : 'klee-last/test000001.ktest'
args       : ['first.bc']
num objects: 1
object 0: name: 'x'
object 0: size: 4
object 0: data: b'\xe9&\xd33'
object 0: hex : 0xe926d333
object 0: int : 869476073
object 0: uint: 869476073
object 0: text: .&.3
```

So, KLEE has deduced that when `x` is 869476073, `x * x` does a 32b overflow to 17 and breaks the code.

It’s overkill for this simple example, but `run.istats` can be opened with [KCachegrind][4] to view things like call graphs and source code coverage. (Unfortunately, coverage stats can be misleading because correct code won’t ever hit boundary check code inserted by the compiler.)

### MurmurHash preimage

Here’s a slightly more useful example. D currently uses 32b MurmurHash3 as its standard non-cryptographic hash function. What if we want to find strings that hash to a given special value? In general, we can solve problems like this by asserting that something doesn’t exist (i.e., a string that hashes to a given value) and then challenging the theorem prover to prove us wrong with a counterexample.

Unfortunately, we can’t just use `hashOf()` directly without the runtime, but we can copy [the hash code from the runtime source][5] into its own module, and then import it into a test rig like this:

```
import dhash;

extern(C):

void klee_make_symbolic(void* addr, size_t nbytes, const(char*) name);
int klee_assume(ulong condition);

int main()
{
    // Create a buffer for 8-letter strings and let KLEE manage it symbolically
    char[8] s;
    klee_make_symbolic(s.ptr, s.sizeof, "s");

    // Constrain the string to be letters from a to z for convenience
    foreach (j; 0..s.length)
    {
        klee_assume(s[j] > 'a' && s[j] <= 'z');
    }

    assert (dHash(cast(ubyte[])s) != 0xdeadbeef);
    return 0;
}
```

Here’s how to compile and run it. Because we’re not checking correctness, we can use `-boundscheck=off` for a slight performance boost. It’s also worth enabling KLEE’s optimiser.

```
$ ldc2 -g -boundscheck=off -checkaction=C -mtriple=x86_64-linux-musl -output-bc -betterC -c dhash.d dhash_klee.d
$ llvm-link -o dhash_test.bc dhash.bc dhash_klee.bc
$ klee -optimize dhash_test.bc
```

It takes just over 4s:

```
$ klee-stats klee-last/
-------------------------------------------------------------------------
|   Path   |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|
-------------------------------------------------------------------------
|klee-last/|     168|     4.37|    87.50|    50.00|     160|       99.95|
-------------------------------------------------------------------------
```

And it actually works:

```
$ ktest-tool klee-last/test000001.ktest
ktest file : 'klee-last/test000001.ktest'
args       : ['dhash_test.bc']
num objects: 1
object 0: name: 's'
object 0: size: 8
object 0: data: b'psgmdxvq'
object 0: hex : 0x7073676d64787671
object 0: int : 8175854546265273200
object 0: uint: 8175854546265273200
object 0: text: psgmdxvq
$ rdmd --eval 'writef("%x\n", hashOf("psgmdxvq"));'
deadbeef
```

For comparison, here’s a simple brute force version in plain D:

```
import std.stdio;

void main()
{
    char[8] buffer;

    bool find(size_t idx)
    {
        if (idx == buffer.length)
        {
            auto hash = hashOf(buffer[]);
            if (hash == 0xdeadbeef)
            {
                writeln(buffer[]);
                return true;
            }
            return false;
        }

        foreach (char c; 'a'..'z')
        {
            buffer[idx] = c;
            auto is_found = find(idx + 1);
            if (is_found) return true;
        }

        return false;
    }

    find(0);
}
```

This takes ~17s:

```
$ ldc2 -O3 -boundscheck=off hash_brute.d
$ time ./hash_brute
aexkaydh

real    0m17.398s
user    0m17.397s
sys    0m0.001s
$ rdmd --eval 'writef("%x\n", hashOf("aexkaydh"));'
deadbeef
```

The constraint solver implementation is simpler to write, but is still faster because it can automatically do smarter things than calculating hashes of strings from scratch every iteration.

### Binary search

Now for an example of testing and debugging. Here’s an implementation of [binary search][6]:

```
bool bsearch(const(int)[] haystack, int needle)
{
    while (haystack.length)
    {
        auto mid_idx = haystack.length / 2;
        if (haystack[mid_idx] == needle) return true;
        if (haystack[mid_idx] < needle)
        {
            haystack = haystack[mid_idx..$];
        }
        else
        {
            haystack = haystack[0..mid_idx];
        }
    }
    return false;
}
```

Does it work? Here’s a test rig:

```
extern(C):

void klee_make_symbolic(void* addr, size_t nbytes, const(char*) name);
int klee_range(int begin, int end, const(char*) name);
int klee_assume(ulong condition);

int main()
{
    // Making an array arr and an x to find in it.
    // This time we'll also parameterise the array length.
    // We have to apply klee_make_symbolic() to the whole buffer because of limitations in KLEE.
    int[8] arr_buffer;
    klee_make_symbolic(arr_buffer.ptr, arr_buffer.sizeof, "a");
    int len = klee_range(0, arr_buffer.length+1, "len");
    auto arr = arr_buffer[0..len];
    // Keeping the values in [0, 32) makes the output easier to read.
    // (The binary-friendly limit 32 is slightly more efficient than 30.)
    int x = klee_range(0, 32, "x");
    foreach (j; 0..arr.length)
    {
        klee_assume(arr[j] >= 0);
        klee_assume(arr[j] < 32);
    }

    // Make the array sorted.
    // We don't have to actually sort the array.
    // We can just tell KLEE to constrain it to be sorted.
    foreach (j; 1..arr.length)
    {
        klee_assume(arr[j - 1] <= arr[j]);
    }

    // Test against simple linear search
    bool has_x = false;
    foreach (a; arr[])
    {
        has_x |= a == x;
    }

    assert (bsearch(arr, x) == has_x);

    return 0;
}
```

When run in KLEE, it keeps running for a long, long time. How do we know it’s doing anything? By default KLEE writes stats every 1s, so we can watch the live progress in another terminal:

```
$ watch klee-stats --print-more klee-last/
Every 2.0s: klee-stats --print-more klee-last/

---------------------------------------------------------------------------------------------------------------------
|   Path   |  Instrs|  Time(s)|  ICov(%)|  BCov(%)|  ICount|  TSolver(%)|  States|  maxStates|  Mem(MB)|  maxMem(MB)|
---------------------------------------------------------------------------------------------------------------------
|klee-last/|    5834|   637.27|    79.07|    68.75|     172|      100.00|      22|         22|    24.51|          24|
---------------------------------------------------------------------------------------------------------------------
```

`bsearch()` should be pretty fast, so we should see KLEE discovering new states rapidly. But instead it seems to be stuck. [At least one fork of KLEE has heuristics for detecting infinite loops][7], but plain KLEE doesn’t. There are timeout and batching options for making KLEE work better with code that might have infinite loops, but let’s just take another look at the code. In particular, the loop condition:

```
while (haystack.length)
{
    // ...
}
```

Binary search is supposed to reduce the search space by about half each iteration. `haystack.length` is an unsigned integer, so the loop must terminate as long as it goes down every iteration. Let’s rewrite the code slightly so we can verify if that’s true:

```
bool bsearch(const(int)[] haystack, int needle)
{
    while (haystack.length)
    {
        auto mid_idx = haystack.length / 2;
        if (haystack[mid_idx] == needle) return true;
        const(int)[] next_haystack;
        if (haystack[mid_idx] < needle)
        {
            next_haystack = haystack[mid_idx..$];
        }
        else
        {
            next_haystack = haystack[0..mid_idx];
        }
        // This lets us verify that the search terminates
        assert (next_haystack.length < haystack.length);
        haystack = next_haystack;
    }
    return false;
}
```

Now KLEE can find the bug!

```
$ klee -optimize bsearch.bc
KLEE: output directory is "/tmp/klee-out-2"
KLEE: Using Z3 solver backend
warning: Linking two modules of different target triples: klee_range.bc' is 'x86_64-pc-linux-gnu' whereas 'bsearch.bc' is 'x86_64--linux-musl'

warning: Linking two modules of different target triples: memset.bc' is 'x86_64-pc-linux-gnu' whereas 'bsearch.bc' is 'x86_64--linux-musl'

KLEE: ERROR: bsearch.d:18: ASSERTION FAIL: next_haystack.length < haystack.length
KLEE: NOTE: now ignoring this error at this location

KLEE: done: total instructions = 2281
KLEE: done: completed paths = 42
KLEE: done: generated tests = 31
```

Using the failing example as input and stepping through the code, it’s easy to find the problem:

```
/// ...
if (haystack[mid_idx] < needle)
{
    // If mid_idx == 0, next_haystack is the same as haystack
    // Nothing changes, so the loop keeps repeating
    next_haystack = haystack[mid_idx..$];
}
/// ...
```

Thinking about it, the `if` statement already excludes `haystack[mid_idx]` from being `needle`, so there’s no reason to include it in `next_haystack`. Here’s the fix:

```
// The +1 matters
next_haystack = haystack[mid_idx+1..$];
```

But is the code correct now? Terminating isn’t enough; it needs to get the right answer, of course.

```
$ klee -optimize bsearch.bc
KLEE: output directory is "/tmp/kee-out-3"
KLEE: Using Z3 solver backend
warning: Linking two modules of different target triples: klee_range.bc' is 'x86_64-pc-linux-gnu' whereas 'bsearch.bc' is 'x86_64--linux-musl'

warning: Linking two modules of different target triples: memset.bc' is 'x86_64-pc-linux-gnu' whereas 'bsearch.bc' is 'x86_64--linux-musl'

KLEE: done: total instructions = 3152
KLEE: done: completed paths = 81
KLEE: done: generated tests = 81
```

In just under 7s, KLEE has verified every possible execution path reachable with arrays of length from 0 to 8. Note, that’s not just coverage of individual code lines, but coverage of full pathways through the code. KLEE hasn’t ruled out stack corruption or integer overflows with large arrays, but I’m pretty confident the code is correct now.

KLEE has generated test cases that trigger each path, which we can keep and use as a faster-than-7s regression test suite. Trouble is, the output from KLEE loses all type information and isn’t in a convenient format:

```
$ ktest-tool klee-last/test000042.ktest
ktest file : 'klee-last/test000042.ktest'
args       : ['bsearch.bc']
num objects: 3
object 0: name: 'a'
object 0: size: 32
object 0: data: b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x10\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
object 0: hex : 0x0000000000000000000000000000000001000000100000000000000000000000
object 0: text: ................................
object 1: name: 'x'
object 1: size: 4
object 1: data: b'\x01\x00\x00\x00'
object 1: hex : 0x01000000
object 1: int : 1
object 1: uint: 1
object 1: text: ....
object 2: name: 'len'
object 2: size: 4
object 2: data: b'\x06\x00\x00\x00'
object 2: hex : 0x06000000
object 2: int : 6
object 2: uint: 6
object 2: text: ....
```

But we can write our own pretty-printing code and put it at the end of the test rig:

```
char[256] buffer;
char* output = buffer.ptr;
output += sprintf(output, "TestCase([");
foreach (a; arr[])
{
    output += sprintf(output, "%d, ", klee_get_value_i32(a));
}
sprintf(output, "], %d, %s),\n", klee_get_value_i32(x), klee_get_value_i32(has_x) ? "true".ptr : "false".ptr);
fputs(buffer.ptr, stdout);
```

Ugh, that would be just one format call with D’s `%(` array formatting specs. The output needs to be buffered up and printed all at once to stop output from different parallel executions getting mixed up. `klee_get_value_i32()` is needed to get a concrete example from a symbolic variable (remember that a symbolic variable is just a bundle of constraints).

```
$ klee -optimize bsearch.bc > tests.d
...
$ # Sure enough, 81 test cases
$ wc -l tests.d
81 tests.d
$ # Look at the first 10
$ head tests.d
TestCase([], 0, false),
TestCase([0, ], 0, true),
TestCase([16, ], 1, false),
TestCase([0, ], 1, false),
TestCase([0, 0, ], 0, true),
TestCase([0, 0, ], 1, false),
TestCase([1, 16, ], 1, true),
TestCase([0, 0, 0, ], 0, true),
TestCase([16, 16, ], 1, false),
TestCase([1, 16, ], 3, false),
```

Nice! An autogenerated regression test suite that’s better than anything I would write by hand. This is my favourite use case for KLEE.

### Change counting

One last example:

In Australia, coins come in 5c, 10c, 20c, 50c, $1 (100c) and $2 (200c) denominations. So you can make 70c using 14 5c coins, or using a 50c coin and a 20c coin. Obviously, fewer coins is usually more convenient. There’s a simple [greedy algorithm][8] to make a small pile of coins that adds up to a given value: just keep adding the biggest coin you can to the pile until you’ve reached the target value. It turns out this trick is optimal — at least for Australian coins. Is it always optimal for any set of coin denominations?

The hard thing about testing optimality is that you don’t know what the correct optimal values are without a known-good algorithm. Without a constraints solver, I’d compare the output of the greedy algorithm with some obviously correct brute force optimiser, run over all possible cases within some small-enough limit. But with KLEE, we can use a different approach: comparing the greedy solution to a non-deterministic solution.

The greedy algorithm takes the list of coin denominations and the target value as input, so (like in the previous examples) we make those symbolic. Then we make another symbolic array that represents an assignment of coin counts to each coin denomination. We don’t specify anything about how to generate this assignment, but we constrain it to be a valid assignment that adds up to the target value. It’s [non-deterministic][9]. Then we just assert that the total number of coins in the non-deterministic assignment is at least the number of coins needed by the greedy algorithm, which would be true if the greedy algorithm were universally optimal. Finally we ask KLEE to prove the program correct or incorrect.

Here’s the code:

```
// Greedily break value into coins of values in denominations
// denominations must be in strictly decreasing order
int greedy(const(int[]) denominations, int value, int[] coins_used_output)
{
    int num_coins = 0;
    foreach (j; 0..denominations.length)
    {
        int num_to_use = value / denominations[j];
        coins_used_output[j] = num_to_use;
        num_coins += num_to_use;
        value = value % denominations[j];
    }
    return num_coins;
}

extern(C):

void klee_make_symbolic(void* addr, size_t nbytes, const(char*) name);
int klee_int(const(char*) name);
int klee_assume(ulong condition);
int klee_get_value_i32(int expr);

int main(int argc, char** argv)
{
    enum kNumDenominations = 6;
    int[kNumDenominations] denominations, coins_used;
    klee_make_symbolic(denominations.ptr, denominations.sizeof, "denominations");

    // We're testing the algorithm itself, not implementation issues like integer overflow
    // Keep values small
    foreach (d; denominations)
    {
        klee_assume(d >= 1);
        klee_assume(d <= 1024);
    }
    // Make the smallest denomination 1 so that all values can be represented
    // This is just for simplicity so we can focus on optimality
    klee_assume(denominations[$-1] == 1);

    // Greedy algorithm expects values in descending order
    foreach (j; 1..denominations.length)
    {
        klee_assume(denominations[j-1] > denominations[j]);
    }

    // What we're going to represent
    auto value = klee_int("value");

    auto num_coins = greedy(denominations[], value, coins_used[]);

    // The non-deterministic assignment
    int[kNumDenominations] nd_coins_used;
    klee_make_symbolic(nd_coins_used.ptr, nd_coins_used.sizeof, "nd_coins_used");

    int nd_num_coins = 0, nd_value = 0;
    foreach (j; 0..kNumDenominations)
    {
        klee_assume(nd_coins_used[j] >= 0);
        klee_assume(nd_coins_used[j] <= 1024);
        nd_num_coins += nd_coins_used[j];
        nd_value += nd_coins_used[j] * denominations[j];
    }

    // Making the assignment valid is 100% up to KLEE
    klee_assume(nd_value == value);

    // If we find a counterexample, dump it and fail
    if (nd_num_coins < num_coins)
    {
        import core.stdc.stdio;

        puts("Counterexample found.");

        puts("Denominations:");
        foreach (ref d; denominations)
        {
            printf("%d ", klee_get_value_i32(d));
        }
        printf("\nValue: %d\n", klee_get_value_i32(value));

        void printAssignment(const ref int[kNumDenominations] coins)
        {
            foreach (j; 0..kNumDenominations)
            {
                printf("%d * %dc\n", klee_get_value_i32(coins[j]), klee_get_value_i32(denominations[j]));
            }
        }

        printf("Greedy \"optimum\": %d\n", klee_get_value_i32(num_coins));
        printAssignment(coins_used);

        printf("Better assignment for %d total coins:\n", klee_get_value_i32(nd_num_coins));
        printAssignment(nd_coins_used);
        assert (false);
    }

    return 0;
}
```

And here’s the counterexample it found after 14s:

```
Counterexample found.
Denominations:
129 12 10 3 2 1
Value: 80
Greedy "optimum": 9
0 * 129c
6 * 12c
0 * 10c
2 * 3c
1 * 2c
0 * 1c
Better assignment for 8 total coins:
0 * 129c
0 * 12c
8 * 10c
0 * 3c
0 * 2c
0 * 1c
```

Note that this isn’t proven to be the new optimum; it’s just a witness that the greedy algorithm isn’t always optimal. There’s a well-known [dynamic programming][10] [solution][11] that always works.

### What’s next?

As I said, I’m interesting in getting this to work with full D code. I’m also interested in using [one of the floating point forks of KLEE][12] on some D because floating point is much harder to test thoroughly than integer and string code.

--------------------------------------------------------------------------------

via: https://theartofmachinery.com/2019/05/28/d_and_klee.html

作者：[Simon Arneaud][a]
选题：[lujun9972][b]
译者：[译者ID](https://github.com/译者ID)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]: https://theartofmachinery.com
[b]: https://github.com/lujun9972
[1]: https://klee.github.io/
[2]: https://www.uclibc.org/
[3]: https://github.com/ldc-developers/ldc/issues/3078
[4]: https://kcachegrind.github.io/html/Home.html
[5]: https://github.com/dlang/druntime/blob/4ad638f61a9b4a98d8ed6eb9f9429c0ef6afc8e3/src/core/internal/hash.d#L670
[6]: https://www.calhoun.io/lets-learn-algorithms-an-intro-to-binary-search/
[7]: https://github.com/COMSYS/SymbolicLivenessAnalysis
[8]: https://en.wikipedia.org/wiki/Greedy_algorithm
[9]: http://people.clarkson.edu/~alexis/PCMI/Notes/lectureB03.pdf
[10]: https://www.algorithmist.com/index.php/Dynamic_Programming
[11]: https://www.topcoder.com/community/competitive-programming/tutorials/dynamic-programming-from-novice-to-advanced/
[12]: https://github.com/srg-imperial/klee-float