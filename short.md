---
title: Making OCaml Safe for Performance Engineering
author: Yaron Minsky
theme:
  name: tokyonight-storm
options:
  implicit_slide_ends: true
  incremental_lists: true
  auto_render_languages:
    - mermaid
    - typst
    - latex
---

Abstract
--------

Over the last few years, Jane Street has started developing major extensions of OCaml's type system, with the primary goal of making OCaml a better language for writing high-performance systems.

This talk will provide a developer's-eye view of these changes. While giving a sketch of the overall direction, we'll focus specifically on the use of **modes** to provide lightweight control over how memory is used, to enable features like stack allocation and data-race free parallelism.

In all of this, I'll focus less on the type theory, and more on how these features are surfaced to users, the practical problems that they help us solve, and the place in the design space of programming languages that this leaves us in.

Making OCaml Safe for Performance Engineering
---------------------------------------------

A 50,000-foot view of changes Jane Street is working on to make OCaml into a better language for performance engineering

What is OCaml like?
-------------------

## Expressive static type system, w/type inference

<!-- pause -->

```ocaml
let rec map l f =
   match l with
   | [] -> []
   | first :: rest -> f first :: map rest f
```

<!-- pause -->

will have its type inferred as:

```ocaml
val map : 'a list -> ('a -> 'b) -> 'b list
```

<!-- pause -->

Polymorphism is simple and pervasive!

What is OCaml like?
-------------------

## Uniform representation of values

<!-- pause -->

Either *immediate* or pointer to a *block*

<!-- pause -->

*immediates* fit inside a machine word, minus tag bit
- Examples: int, char, bool

<!-- pause -->

*blocks* are heap-allocated values
- one header word
- one word per nested value
- Examples: string, array, record

<!-- pause -->

Important for GC...

<!-- pause -->

and for fast separate compilation of polymorphic functions

What is OCaml like?
-------------------

## How can we implement polymorphism?

Compile each function just once, rely on uniform memory representation!

<!-- pause -->

There are other ways!

- C++: compile once per type
- C#: runtime code generation

## Parallelism

<!-- pause -->

Pre 5.0: no parallelism, global runtime lock

<!-- pause -->

5.0 and beyond: Multicore GC
- with sane memory model
- but no race-free programming model

So, what's not to love?
-----------------------

## Minimal control over memory representation

- hard to keep data compact
  - byte takes 8 bytes
  - int64 (8 bytes) takes 24!
- pointer indirection defeats prefetcher
- All non-trivial data must be exposed to GC

## Unsafe parallelism is no fun

- Even with a good memory model, it's incredibly error-prone
- and good memory models are expensive!

Design goals
------------

<!-- pause -->

**Safe**, **convenient**, **predictable** control over performance-critical aspects of program behavior,

<!-- pause -->

but **only where you need it**.

<!-- pause -->

And...in OCaml!
So changes must be **backwards-compatible**.

What we're building
-------------------

Three major user-facing features:

- Narrow and flat data layouts
- Stack allocation
- Race-free parallel programming

All type-safe, built on two type-system features:

<!-- pause -->

**kinds** and **modes**

Heap vs Stack allocation
-------------------------

- Heap allocation is expensive
  - Especially major heap allocation
  - Minor is better, but still cache-inefficent

- Stack allocation is better!
  - Similar to minor-heap allocation
  - But values are collected faster, cheaper
  - Touch fewer cache lines

Making stack allocation safe
----------------------------

- Follow a *stack discipline*
- Mainly:
  - don't create pointers from heap to stack
  - don't return stack values

Can't we Rust?
--------------

- Why not use Rust-style lifetimes?
  - Functions take (often implicit) *lifetime* parameter
  - Values under polymorphic lifetimes can be stack-allocated

- But,
  - You often trip in to higher-order polymorphism
  - Inference is undecidable!
  - Very un-ocaml, and arguably unergonomic

Instead, Modes!
---------------

Modes are:

- Properties that can be applied to any type
- That by default apply deeply

Global and Local
-----------------

<!-- pause -->

In this case, we add a pair of modes:

- **global** is the default, unconstrained
- **local** values must follow the stack discipline

<!-- pause -->

There's sub-moding!

can pass a global where a local is expected

An example of stack allocation
------------------------------

```ocaml
let rec map l f =
  match l with
  | [] -> []
  | hd :: tl -> f hd :: map tl f
```

<!-- pause -->

```ocaml
val map : 'a list -> ('a -> 'b) -> 'b list
       @@ .       -> local      -> .
```

<!-- pause -->

```ocaml
let multiply_by l mult =
  map l (fun x -> mult * x)
```

Smart constructors
------------------

functions that can return local values if they don't create a stack frame.

<!-- pause -->

```ocaml
type pos = { x: float; y: float }
let create_pos x y = exclave { x; y }
```

<!-- pause -->

```ocaml
val create_pos
  : float -> float -> pos
 @@ local -> local -> local
```

## Resource allocation

```ocaml
val with_file
  : string -> (In_channel.t -> 'a) -> 'a
 @@ .      -> (local        ->  .) ->  .
```

## Mode polymorphism

<!-- pause -->

Instead of this:

```ocaml
val hd
  : 'a list -> 'a
 @@ .       ->  .

val hd_local
  : 'a list -> 'a
 @@ local   -> local
```

<!-- pause -->

Write this:

```ocaml
val hd : 'a list -> 'a
      @@ 'm      -> 'm
```

## Modal kinds

- Who cares if your immediate is local?
- "value mod global" is a kind that tracks this

Data-race freedom
-----------------

## Modes are a natural fit

<!-- pause -->

Things you can do to any value:

- Make an alias
- Return from a function
- Create a pointer to it
- Pass to another thread

These operations are all *deep*.

## A new mode dimension: contention

Contention tracks whether a value *has been shared across threads*

<!-- pause -->

Values can be **contended** or **uncontended**.

- **uncontended** values have not been shared
- **contended** have been

<!-- pause -->

You are not allowed to read or write the mutable fields of a contended value.

## A new mode dimension: portability

Portability tracks whether a value *is safe to share between threads*

<!-- pause -->

Values can be **portable** or **nonportable**.

- **portable** values can be shared between threads
- **unportable** values cannot

<!-- pause -->

So, what values are portable?

- Almost everything!
- Only functions that close over **uncontended** state are not.

## A bestiary of modes

15 modes, in 5 dimensions.

Two varieties of modes:

- **future**: What you can do with a value in the future
- **past**: What has happened to a value in the past

| dimension   | variety | min mode        |           | max mode        |
|-------------|---------|-----------------|-----------|-----------------|
| Locality    | future  | **global**      | regional  | local           |
| Uniqueness  | past    | unique          | exclusive | **aliased**     |
| Linearity   | future  | **many**        | separate  | once            |
| Contention  | past    | **uncontended** | shared    | contended       |
| Portability | future  | portable        | observing | **nonportable** |

## Spawning threads

- function run by thread must be portable
- returned value doesn't have to be

```ocaml
val spawn
  :          (unit -> 'a) -> 'a thread
 @@ portable (.    ->  .) ->  .
```

<!-- pause -->

```ocaml
val join
  : 'a thread -> 'a
 @@ .         ->  .
```

## Manipulating pointers to shared memory

```ocaml
module Ptr : sig

  (* A pointer to shared memory holding an ['a], with "key" ['k]. *)
  type ('a, 'k) t

  (* Create a shared memory cell protected by key ['k] *)
  val create : (unit -> 'b) -> ('b, 'k) t
            @@ portable     -> .

  (* Manipulate data in shared memory.
     Keys are considered mutable data and can't be used when [contended].
     This ensures data-race freedom. *)
  val map :
      'k Key.t -> ('a -> 'b) -> ('a, 'k) t -> ('b, 'k) t
   @@ .        -> portable   -> .          -> .

  val extract :
      'k Key.t -> ('a -> 'b)               -> ('a, 'k) t -> 'b
   @@ .        -> (. -> portable) portable -> .          -> contended

  ...
end
```
