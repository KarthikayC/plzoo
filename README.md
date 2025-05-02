# Handling Exceptions in MiniML

## Introduction

### What is MiniML?

MiniML is an implementation of an eager statically typed functional language with a compiler and abstract machine.

It has the following constructs:

- Integers with arithmetic operations `+`, `-`, `*`
- Since there is no exceptions defined in the language by default, there is no division operation
- Booleans with conditional control flows and integer comparisons (`=`, `<`)
- Recursive functions and function application
- Toplevel definitions

### Aim of the Project

The main aim of this project is

- Extend the functionality of the MiniML language
- Introduce division to the language
- Add exceptions and exception handling

## Features

- The language now supports integer division
- `try-with` construct had been added to handle exceptions
- Custom exceptions have been added
  - `DivisionByZero` - Raised by the machine when the divisor is zero
  - `GenericException` - A Generic Exception type that takes an integer input

## Logic

### compile.ml

This module defines the compile function, which compiles MiniML expressions into machine instructions understood by the MiniML abstract machine

- Added `IDivi` - Instruction for division, added to enable division with runtime exception handling
- Added `ITry` - Instruction for handling exceptions using the try-with construct at runtime

### lexer.mll

The lexer (written using OCaml’s ocamllex) is responsible for converting raw input text into a stream of tokens that can be consumed by the parser. Each token corresponds to a meaningful syntactic unit in the MiniML language.

- Added tokens for DIVISION ('/'), LCURLY ('{'), and RCURLY ('}')
- Added tokens for TryWith constructor, TRY ('try') and WITH ('with')
- Added tokens for exceptions,  DIVISIONBYZERO ('DivisionByZero') and GENERIC ('GenericException')

### machine.ml

This module implements an abstract stack machine to execute programs compiled from MiniML, a simple purely functional language. The machine uses instructions, frames, environments, and stacks to evaluate expressions

- Added instr for Division and Try-With `ITry of frame * Syntax.except * frame`
- In exec block, the ITry part has no functionality. All the functionality is handled in the run loop block `ITry (_,_,_) -> failwith "ITry is handled by run loop`
- In the run loop block, there is an extra matching case for any `ITry (try_frame, exception_given, catch_frame)` in the instruction frame `frm`. Upon finding an ITry frame, the code enters a try-with block to evaluate the try_frame instructions. These are the instructions meant to be executed under the protection of the try-with construct. If no exception is raised, execution proceeds normally by continuing with try_frame. However, if an exception is raised:
  - If it's DivisionByZero, the code checks whether the exception_given in the TryWith clause matches DivisionByZero. If it matches, it executes the catch_frame, i.e., the handler block of the try-with. If it doesn't match, the original exception is re-raised to propagate upward
  - If a Machine_error is raised (used here as a catch for GenericException), the code similarly checks if the provided exception_given is GenericException. If it matches, the catch_frame is executed. Otherwise, the exception is re-raised

### parser.mly

This parser is built using Menhir, the parser generator for OCaml. It handles expressions and definitions in a MiniML-like functional language, supporting core constructs as well as exception handling with try-with blocks

- Added the parser for custom exceptions
- Added the parser for division - `e1 = expr DIVISION e2 = expr`
- Added the parser for try-with block - `TRY LCURLY e1 = expr RCURLY WITH LCURLY e2 = exception_name TARROW e3 = expr RCURLY`
- Added the parser and type for `{ ... }`

### syntax.ml

This defines the core data structures (Abstract Syntax Tree or AST) for a small functional programming language, written in OCaml.

- Added syntax of custom exceptions
- Added the syntax of try-with block `TryWith of expr * except * expr`

### type_check.ml

- Added custom exception, `Type_error`
- Changed the check block to raise Type_error instead of printing the error directly (To allow GenericException handling)
- In the type_of block, the Try-With block is matched with exn:
  - If it is DivisionByZero, it checks if type_of t1 = type_of t2 and returns t1 if no exception, else, it prints the exception and stops
  - If it is GenericException, it checks if type_of t1 exists. If it raises Type_error, the code returns t2, else, it check is type_of t1 = type_of t2 and returns t1 if no exception, else, it prints the exception and stops

## Build and Execution

1. Install OCaml and set up the OCaml Development Environment
2. Activate the opam switch
3. Clean previous build:
   ```bash
   dune clean
   ```
4. Rebuild the project:
   ```bash
   dune build src/miniml
   ```
5. Run the executable:
   ```bash
   ./miniml.exe
   ```

## Examples

  ```ocaml
miniML> 35 / 24;; 
- : int = 1

miniML> 2 + 18 / 0;;
Fatal error: exception Dune__exe__Machine.DivisionByZero

miniML> 2 * false;;
Fatal error: exception Dune__exe__Type_check.Type_error

miniML> let double = fun f (n : int) : int is 2 * n;;
double : int -> int = <fun>
miniML> double 81;;
- : int = 162

miniML> try { 15 / 0 } with { DivisionByZero -> -12 };;
- : int = -12

miniML> try { 825 / 0 } with { GenericException -> 0 };;                
Fatal error: exception Division_by_zero

miniML> try { 1000 / 101 + false } with { GenericException -> 10 };; 
- : int = 10

miniML> let safeprime = fun f (n : int) : int is if n = 0 then 0 else 333/n;;
safeprime : int -> int = <fun>
miniML> try { safeprime true } with { GenericException -> 0 };;              
- : int = 0

miniML> try { try { true + false / false } with { DivisionByZero -> 100 } } with { GenericException -> 12 };;                        
- : int = 12

miniML> try { 81 / 0 } with { | DivisionByZero -> 10 | GenericException -> 0 };;
Syntax error at line 0, characters 21-22:
unrecognised symbol
  ```

## Conclusion

### Key Learnings

- Learned about lexer, parser, type checker, and evaluator and how to integrate them to extend the functional language
- Understanding and modifying parser.mly and lexer.mll to support new constructs
- Bypassing type-checking to support runtime errors
- Understood debugging in functional languages

### Verification Approach
- A large number of test cases were manually designed and executed to verify correctness
- Outputs were verified by visual inspection

### Limitations
- With block can have only one exception
- Negative integer inputs for functions fail
- Entire code must be on one line
