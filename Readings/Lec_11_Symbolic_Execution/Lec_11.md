# EXE: Automatically Generating Inputs of Death


Link: <https://css.csail.mit.edu/6.858/2022/readings/exe.pdf>

## Introduction

Attacker-exposed code must be able to safeguard against a wide variety of attacks, which often requires awareness of subtle issues like arithmetic and buffer overflow conditions, which historically shows that programmers reason about quite poorly.

They are also difficult to test for (even with fuzzing), and dynamic tools requires test cases to drive them, which can have the same coverage problems as both random and manual testing.

EXE ("EXecution generated Executions") is a dynamic bug-finding tool to deeply check real code. EXE's contribution is that it can generate its own test cases. It uses *symbolic* input that is initially allowed to be "anything". It uses STP to let it reason about all possible values that the path could be run with, rather than a single set of concrete values from an individual test case.

**Novel features:** STP primitives allow EXE to build constraints for all C expressions with perfect accuracy, down to a single bit. (One exception is floating point).

## Overview

For developer to check code with EXE, they mark the memory locations that should be treated as containing **symbolic data** whose values are initially entirely unconstrained. These memory locations are typically the input to the program.

As program runs, EXE executes each feasible path, tracking all constraints. When the program terminates, EXE call STP to solve the path's constraints for concrete values. A path is considered to terminate when it:
 1. calls exit()
 2. crashes
 3. an assertion fails, or
 4. EXE detects an error

Constraint solutions are the concrete bit values of an input that will cause the given path to execute. When it finds the response generates an error, EXE provides a concrete attack to be launched against tested system.

The EXE compiler (exe-cc) has three main jobs:
 1. First, it inserts check around every assignment, expression, and branch in the test program to determine if operands are concrete or symbolic. We consider an operand to be concrete **iff** all of its constituent bits are concrete (known + doesn't change). If any operand is symbolic, the operation is not performed, but instead passed to the EXE runtime system, which adds it as a constant for the current path
 2. The compiler also insert codes to fork program execution when it reaches a symbolic branch point, so that it can explore each possibility. Every time exe-cc adds a branch constraint, EXE queries STP to check that there exists at least one solution for the current path's constraints. If not, the branch is impossible and the EXE stops executing it.
 3. exe-cc inserts code that calls to check if a symbolic expression could have any value that could cause either a null or out-of-bounds memory reference or a division or modulo by zero. If so, EXE forks execution and 
    1. on the truth path asserts that the condition does occur, emit a test case, and terminates
    2. on the false path assert that the condition does not occur, and continues execution

As a side note, EXE turns any **assert(e)** statement on a symbolic expression *e* into a universal check of *e* simply because it wants to exhaust both paths of if-statements.

### STP Key Features

STP, which is EXE's constraint solver, is more precisely a *decision procedure* for bitvectors and arrays. Decision procedures are programs that can determine the falsifiability of a logical formula that can express constraints relevant to software and hardware.

STP views memory as untyped bytes. It only provides three data types, which are boolean, bitvectors, and arrays of bit vectors. A bitvector is fixed length sequence of bits. In the implementation terms (like `x+y`) are converted into vectors of boolean formulas consisting entirely of single bit operations (AND, XOR, etc.). For example, a 32-bit add is implemented as a ripple-carry adder. Formulas (`x+y<z`) are converted into DAGs of single bit operations, where expressions with identical structures are represented uniquely.

EXE represents each symbolic data block as an array of 8-bit bitvectors. EXE emulates symbolic pointer expressions by mapping them as an array reference at some offset.

## Optimizations

One of the slow parts of the reasoning is dealing with arrays. STP eliminates array expression by translating them to bivector primitives, which it then translates to SAT. This is accomplished through two logic transformations. In addition, EXE also caches the result of satisfiability queries and constraint solutions to avoid calling STP when possible. 

EXE also does **constraint independence**, which exploits the fact that we can often divide the set of constraint EXE tracks into multiple independent subset of constraints. This can help us consider only the groups of sub-constraints that contribute to the path decision

## Search Heuristics

By default, EXE uses depth-first-search, picking randomly between the two branches. However, if EXE encounters a loop with a symbolic variable as a bound, DFS can get stuck as it attempts to execute the loop as many time as possible. To overcome this, they use search heuristics to drive the execution along "interesting" execution paths (e.g. paths that cover unexplored statements). Their current heuristics uses a mixture of best-first and depth-first search.