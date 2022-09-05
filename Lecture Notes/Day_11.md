# Symbolic Execution

One of the topics we haven't talked about much so far is finding vulnerabilities in software. We consider `exploits \in bugs`. The CVE database publishes known vulnerabilities with categorization. There are programs like "bug bounty" to look for potential exploits, along with established procedures like testing, fuzzing, and symbolic execution that allow us to find bugs in softwares (which can mitigate exploits upstream).


## Finding bugs

We want to determine, given a set of inputs, does it cause any unexpected behavior / bug in the code? First, we need to consider the class and types of bugs that we are looking for. Also, we need to plan out which inputs we'll be considering in our space of control.

**What counts as a bug?** Example behaviors includes crashes, calling unreachable functions + showing unwarranted behavior (more app-specific), failing assertions, dividing by zero, excessive resource use (loops, memory allocation), and out of bounds memory accesses.

### Current Testing Methods

* Manual testing
  * Provide a specific input and the expected outcome. With manual testing you can define very precise bugs that you are looking for / trying to eliminate. Also, creating test suites allow for *regression testing*.
  * However, manual written tests only get at bugs that the developer is thinking about. There are relatively few inputs, thus limiting your test coverage, especially for large programs. You might miss some corner cases, and also the effort to write them is relatively high.
* Fuzzing
  * Provides random inputs to the code.
  * With this method, you can run your code against many, many inputs. This approach turns out to be very good at finding corner cases, plus it works as a black-box testing (don't really need to look at the code at all).
  * However, fuzzing testing requires high CPU usage, and as mentioned by the paper, it usually yields in weaker coverage (as it may take the fuzzer lots of time to get through conditions). There are improvements on fuzzing using *generators*, which let it know what expected inputs / structure is operate and operate on that instead of truly random cases.
* Coverage-Guided Fuzzing
  * This variant of fuzzing gives us back info about code coverage from test case to inform us on how successful the test case was at exploring new areas.
  * Additionally, we also iteratively pull from known "interesting" test cases (called the **corpus**) and mutate them. If a test case gets add new parts of the code, it is added back the corpus.
  * One good tool is `afl-fuzz`.

## Symbolic Execution

The goal of symbolic execution is to find inputs that cover all execution paths (focused on code coverage). The idea is that instead of representing DS with concrete values, we instead reason over them using *symbolic values*. Along this, we use a SAT solver to decide which values are possible.

**What exactly is a symbolic value?**

Consider the following code:
```java
int x = 5;
int y = al;
int z = y*z;
if(10*x == z) {
    panic();
}
```
Here we consider `al` variable to be something that is supplied from the user of the program. One way we can represent the memory of this program symbolically is:
```text
| x | y  | z   |
| 5 | al | 2al |
```

Note that the symbolic representation / expression is indeed somewhat limited (for example, if you write in the code of `SHA_256(al)` the STP might just timeout and then ask you the tester to deal with it). Many SAT solvers, like `z3`, use many heuristics and smart calculations to help give you useful solutions. It doesn't guarantee solutions - for example satisfiability problems are NP-complete - but is surprisingly robust in the problems that it can solve.

## EXE

EXE converts .c code to .c, adding assertions, instrumentation, and code to maintain data structures & to register symbolic values + expressions. During runtime, if at any point we get to a control flow structure on symbolic expressions, it will be sent over the STP with the **path constraints** to see if there is satisfiable solution or if the STP timed-out when trying to find a solution. We will then run fork to try one or both of the branches. Forks are managed by a scheduler which will switch between processes to try to run processes that seem to be the most promising.

Graphic representation of these relationships:
```text

->  [ application process ] -------> path condition ----> [constraint solver (STP)]
|     at if statement,
|     construct path condition                        is this path condition satisfiable?
|     for each branch
|                      <--------- YES/NO/DON'T KNOW (timeout) ---
|		      
|           |
|	    |
|   If unsatisfiable, don't explore branch  
|   If satisfiable, then fork to explore branch
|           |
|  	    |
|	    V
|--- [ scheduler process ]
    selects which application process to explore
```

EXE executes with symbolic values (i.e. variable/memory holds an expression in terms of inputs). EXE remembers which memory locations holds symbolic values and what each location's current symbolic value. It also remembers **constraints** imposed by executed if statements and *path constraints* (pc). EXE views execution as a tree, which splits at each if-statement or other control flow structures. It will attempt to traverse both the "success" and "failed" conditions.

Also one surprising thing about the input is that z3 solver also used the constraint on bit numbers (e.g. 8-bit numbers) to do things like integer overflow to get a certain result.

### Scale

With more branching, we get a large number of memory paths that we can take. For optimizations, we can consider trying to *merge* multiple paths together (representing multiple paths as one complicated state) after a number of control flow statements, however, this approach adds complexity for the SAT solver. Also it should be noted that EXE was one of the early iterations of symbolic execution, so there have been many iterations and improvements made to these customized SAT solvers since then.

If we get a SAT solver timeout, we can consider that branch as effectively unreachable. This will limit the code coverage that EXE gets to. Also, if we have complicated logic in our code that is hard to reason about (like SHA-1), it might be harder for our current version of SAT solvers to find a satisfiable solution.

**How does EXE handle all the fork()ed processes?**  EXE search server uses "best-first" heuristic: line of code that's been run the fewest times (much like breadth-first). It use DFS on that process and children "for a while"

**How does EXE manage with arrays?** This turns out to be a fairly difficult part of the problem. EXE turns many C constructs into arrays (strings, ptrs, structs). For concrete indexing, like `s[c]`, where c is some for loop counter variable, STP can simply treat `s[c]` as a specific symbolic value (symbolic value inside symbolic value). However, `c[s]` could refer to any element, since `s` is symbolic. In this case we treat it as a big disjunction (can be `c[0], c[1], etc..`) This expands into very complicated symbolic operations. The hardest part is probably dereferencing of symbolic pointer `*s`. Depending on what `s` is, it can be referring to almost anything in memory (for EXE they just don't handle this case at all) 

In general, EXE is very slow at a base level, so optimizations are critical for performance. Tricks to speed up execution include:
  - ordinary concrete operations/operands when possible
  - don't bother with if-branch if no solution
  - cache+share constraint solutions (4.1)
  - solve and cache independent constraint fragments (4.2)
  - solver knows a lot about arrays (3.3)


### Effectiveness

Overall EXE can find real bugs in smallish UNIX utility code like those parsing packet filter, udhcpd, regular expression, kernel file system, etc. In this regard it is quite impressive, e.g. finding real bugs in real C programs! Most of these bugs usually turn out to be mostly buffer overflow and illegal memory references. To make the checks more program specific we will need programmers to add in a lot of permission checks and assertions.