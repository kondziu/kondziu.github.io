# Everything You Always Wanted to Know About SEXPs But Were Afraid to Ask

<!--  Title: Everything You Always Wanted to Know About SEXPs But Were Afraid to Ask
  --  Date: 2018-06-4T18:50:14
  --  Tags: R, by Konrad Siek
  -->

So. 
<!-- My writing teacher in high school repeatedly told me to never start a
sentence with 'So,' but she never said anything about starting an essay with
it. I would illuminate it if I could. -->
 
About a year and a half ago I joined a PL lab where they worked a lot on a
language called R. I joined one of these R-related projects that had to do with
tracing the execution of R programs. I had heard about R before, I even used it
to make graphs for some of my previous papers, but I've never taken a look
inside. 

R is a dynamically typed functional language with lazy semantics that is used
extensively for statistical computing, data analysis, etc. It has a long
history, a large user base, and a plethora of quirks. It can be both
interpreted and byte-code compiled.

I very quickly realized I am in well over my head. The implementation of R is
thousands of lines of impenetrable C code. The documentation is fairly
in-depth, but I often found myself understanding what the documentation said on
any reasonable level only after I had understood the thing it described. I was
even lucky in having access to an R core developer, but often I would find
myself failing to formulate a question whose answer would shed light on my
investigations.

Nevertheless, like all Lovecraftian protagonists before me, I relentlessly
delved deeper into the cryptic writings in front of me. I ended up spending a
lot of time with the debugger, armed with multitudes of examples illustrating
various behaviors. It didn't help that I was really rusty when it came to both
C and R. After some time, I found code inspection tools hidden deep inside R
and started using those to my advantage. Soon some things started to make sense
(clearly a sign of madness) and I stopped groping in the dark constantly. My
code interfacing R became more deliberate and the mysterious SEGFAULTs that
plagued me before started being just a rare occurrence. 

I can't say I'm an expert in R's internals by any stretch of the imagination!
I'm just a bumbling outsider, who seem to learn something new or correct a
misconception every time they give R another look. But I do feel I've become
comfortable with some aspects of the R interpreter now, to the point where I
can get things done. And to the point where I can attempt to share some of my
painfully collected conclusions.

Since you are here, my guess is you are interested in looking into R's
internals and perhaps also find yourself at a loss as to how it all works.  I'd
like  try give you a hand with a few pointers about R interpreter's basic
underlying concept: SEXPs.

<!--SEXPs, which are one of the
first things you will encounter in the bowls of the R interpreter. I should
note though, I am not an expert on R internals (if there even is such a
thing...) just a bumbling researcher just like you, trying to figure out this
infernal machinery. Nevertheless, I have been around the block a few times and
I think I can show you a thing or two about SEXP.-->

# What are SEXPs
<!--the SEXP talk-->

So what is a SEXP? Conceptually, SEXPs represent expressions which take the
form of a tree-like structure that describes expressions in the the R language<sup>9</sup>.


For example a language expression like:

```R
{ x + y + 1 }
```

is translated by the R interpreter into a tree containing SEXPs representing
function applications, symbols, and numeric values, as follows:

```
LANG
│
├─SYM `{`
└─LANG 
  │
  ├─SYM `+`
  ├─LANG
  │ │
  │ ├─SYM `+`
  │ ├─SYM `x`
  │ └─SYM `y`
  │
  └─REAL 1
```

Additionally, things like environments, promises, and argument lists are also
represented as SEXPs in R. Furthermore, there are even cases when SEXPs are
created by the R interpreter internally out of thin air for convenience (e.g.
to use as maps or linked lists).

In other words, whenever you interact with the internals of R in any meaningful
way you will end up dealing extensively with SEXPs.

# SEXP inspector

While SEXPs are omnipresent from the point of view of R, they are also mostly
invisible to the R programmer. Nevertheless, you can ask the R interpreter to
give you fairly comprehensive information about them using the internal
`inspect` function. For instance, if we wanted to see the SEXP of the example
expression we talked about before, we would do it like so:

<!--This is a
`srcfile` attribute which records the parameters of the file form which some
particular piece of code came. We can immediately see that the `srcfile`
attribute has its own `class` attribute at the end, specifying this is an
object belonging to classes `srcfilecopy` and `srcfile`. Note that the
inspector prints `ATT` to indicate that the SEXP has attributes. Apart from the
attribute, the SEXP's `tagval` says that this is indeed a `srcfile` attribute.
Finally, the `carval` slot points to a hash table environment which holds some
information about the file from which the data came from. We can represent this
all as follows:-->

```R
x <- 3
y <- 2
.Internal(inspect({ x + y + 1 }))
```

Actually, that would not work, since the expression would get evaluated before
we passed it to inspect, and we would just inspect the result. So let's prevent
it from doing that by running it through `substitute` like so:

```R
x <- 3
y <- 2
.Internal(inspect(substitute({ x + y + 1 })))
```

The `substitute` function will return an unevaluated parse tree for our
expression, and `inspect` will traverse it. 

<!--
This is a
`srcfile` attribute which records the parameters of the file form which some
particular piece of code came. We can immediately see that the `srcfile`
attribute has its own `class` attribute at the end, specifying this is an
object belonging to classes `srcfilecopy` and `srcfile`. Note that the
inspector prints `ATT` to indicate that the SEXP has attributes. Apart from the
attribute, the SEXP's `tagval` says that this is indeed a `srcfile` attribute.
Finally, the `carval` slot points to a hash table environment which holds some
information about the file from which the data came from. We can represent this
all as follows:
-->

When function `inspect` is called via `.Internal` what actually happens is that
the expression given as the argument to inspect is passed into a `SEXP`
structure and the C function `R_inspect(SEXP x)` defined in `inspect.c` is
called. 

This means that we can also run the inspect function from within C code or
`gdb` directly as `R_inspect(SEXP x)` and get the same information. The only
problem is that `R_inspect` is not visible outside of `inspect.c`, so you may
need to make it visible first.<sup>1</sup>

Regardless, when called on our examples, the `inspect` function will spit out
something like this:

```
@55989e8b9be8 06 LANGSXP g0c0 [] 
  @55989d5b0a68 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "{" (has value)
  @55989e8b9bb0 06 LANGSXP g0c0 [] 
    @55989d5c0748 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "+" (has value)
    @55989e8b9b78 06 LANGSXP g0c0 [] 
      @55989d5c0748 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "+" (has value)
      @55989d61fcb0 01 SYMSXP g0c0 [MARK,NAM(2)] "x"
      @55989d7487d0 01 SYMSXP g0c0 [MARK] "y"
    @55989ed492c8 14 REALSXP g0c1 [] (len=1, tl=0) 1
```

As you can see, this mirrors the idealized tree we looked at previously, but
with a lot of other details strewn haphazardly around its nodes. We will go
into the details of this output in the rest of this post, but as an overview
what we see in each line is this:

```
   @55989d5b0a68 01 SYMSXP g0c0 [MARK,LCK,gp=0x7000] "{" (has value)
ʌ  ʌ             ʌ         ʌ    ʌ                    ʌ
│  │             │         │    │                    │
│  │             │         │    │                    └ payload (type-specific)
│  │             │         │    │
│  │             │         │    └ other header flags
│  │             │         │
│  │             │         └ garbage collector header flags
│  │             │
│  │             └ type of the SEXP: numeric and text representation
│  │ 
|  └ address of the SEXP (hexadecimal representation)
|
└ indentation showing where the SEXP falls in the tree
```

Now let's get into the details both of SEXPs and the output of `inspect` for
specific SEXP types.

# Intimate knowledge

From the point of view of the internal C code of the R interpreter, a SEXP is a
pointer to a struct called `SEXPREC` which contains a header and a payload in
the form of a union `u`, all defined in `Rinternals.h` like so:

```
┌────────────────────[SEXPREC_HEADER]─────────────────────┐
│                                                         │                      
v                                                         v
┌──────────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
│                  │            │   gengc    │   gengc    │            │            │            │
│     sxpinfo      │   attrib   │ prev_node  │ next_node  │            │            │            │
│                  │            │            │            │            │            │            │
└─[sxpinfo_struct]─┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┘
                                                          ʌ                                      ʌ
                                                          │                                      │
                                                          └──────────────────[u]─────────────────┘
```

## Header

Let's start by looking at the header. And let's start from the end: slots
`gengc_prev_node` and `gen_gc_next_node` are used by the garbage collector to
iterate over the SEXPs in the same generation, they point other SEXPs. All the
SEXPs form a circular doubly-linked list which has specific perks for node
removal<sup>6</sup>. 

Then `attrib` is also another SEXP that describes add-on attributes of the SEXP
we're analyzing, usually used for metadata such as class or source reference
information. Attributes are used to hang information on an SEXP without
interfering with its contents. We describe them in detail in a separate section
further below. 

Finally, the `sxpinfo` part of the header is a structure defined like this:

```C
struct sxpinfo_struct {
    SEXPTYPE type      :  5; 
    unsigned int obj   :  1;
    unsigned int named :  2;
    unsigned int gp    : 16;
    unsigned int mark  :  1;
    unsigned int debug :  1;
    unsigned int trace :  1;
    unsigned int spare :  1;
    unsigned int gcgen :  1;
    unsigned int gccls :  3;
};
```

We'll ignore most of these for now, with just a few exceptions. For now we will
go through the flags which often appear on the print outs of our inspection
tool. Most of the other flags pertain to specific SEXP types, so we will
explain them as they come up. All the other flags we will discuss at the very
end.

### Type

The `type` member is very important and it informs us what kind of SEXP we're
dealing with and how to extract useful information from its payload. This
member is of type `SEXPTYPE`, which is basically an `unsigned int`. For every
type there is a corresponding constant that's a little more hamster-readable.
Here are the types and their constants as defined in `Rinternals.h`:

```C
typedef unsigned int SEXPTYPE;

#define NILSXP	     0	  /* null */
#define SYMSXP	     1	  /* symbols */
#define LISTSXP	     2	  /* lists (specifically: pairlists) */
#define CLOSXP	     3	  /* closures */
#define ENVSXP	     4	  /* environments */
#define PROMSXP	     5	  /* promises */
#define LANGSXP	     6	  /* language constructs (special lists) */
#define SPECIALSXP   7	  /* special functions */
#define BUILTINSXP   8	  /* builtin non-special functions */
#define CHARSXP	     9	  /* internal string type*/
#define LGLSXP	    10	  /* logical vectors */

#define INTSXP	    13	  /* integer vectors */
#define REALSXP	    14	  /* real variables */
#define CPLXSXP	    15	  /* complex variables */
#define STRSXP	    16	  /* strings/character vectors */
#define DOTSXP	    17	  /* dot-dot-dot object */
#define ANYSXP	    18	  /* make "any" args work. */
#define VECSXP	    19	  /* generic vectors */
#define EXPRSXP	    20	  /* expression vectors */
#define BCODESXP    21    /* byte code */
#define EXTPTRSXP   22    /* external pointer */
#define WEAKREFSXP  23    /* weak reference */
#define RAWSXP      24    /* raw byte vector */
#define S4SXP       25    /* S4 classes, non-vector */

#define NEWSXP      30    /* fresh node created in new page */
#define FREESXP     31    /* node released by GC */
#define FUNSXP      99    /* closure or builtin or special */
```

Most SEXP type constants refer to specific concrete types of SEXPs: symbols,
integer vectors, etc. The type of an SEXP informs us what are the semantics of
the three payload slots in `u` and gives us a hint about which macros to use to
access and modify the SEXP. We'll tackle these types one by one below in depth
as we talk about the payload.

Meanwhile there are some idiosyncrasies here worth noting. First, constants 11
and 12 are left free. These used to be used for internal factors and ordered
factors. They are no longer part of the language. Nevertheless, the gap should
not be re-used, since the ordering of SEXP types matter.

The types `ANYSXP` and `FUNSXP` are internally by the interpreter as aggregated
types for convenience. `FUNSXP` is used to indicate either a SEXP of type
`CLOSXP`, `BUILTINSXP`, or `SPECIALSXP`. `ANYSXP` is used to indicate a SEXP of
any type.  This does not mean that there are SEXPs of types `FUNSXP` or
`ANYSXP` floating around. Rather these are used in various conditions in the
interpreter (see eg. function `findVar1` from `envir.c`).

The type of a SEXP can be checked using the function `typeof` from within R:

```R
> typeof(1)
[1] "double"
> typeof(substitute({ x + y + 1 }))
[1] "language"
```

There is a macro in C that retrieves the type of a SEXP as a `SEXPTYPE` called
`TYPEOF` which can be readily used with the type constants to make conditions.

```C
SEXP s;
if (TYPEOF(s) == NILSXP) {
    // ...
}
```

### Named

The `named` is less significant in our discussion, but it appears in the
inspector quite often, so it deserves a description. This field is used to
indicate how many times this object has been assigned to a variable (how many
times it has been named). Note the following example:

```R
> .Internal(inspect(c(1,2,3)))
@55555b5ec788 14 REALSXP g0c3 [] (len=3, tl=0) 1,2,3
```
Here the vector has the `named` (indicated by the `NAM` marker in square
brackets) field set to zero, because it has not been assigned to any variable.
In other words, there are not references to it.

```R
> x <- c(1,2,3)
> .Internal(inspect(x))
@55555b5ec6e8 14 REALSXP g0c3 [NAM(1)] (len=3, tl=0) 1,2,3
> y <- x
> .Internal(inspect(y))
@55555b5ec6e8 14 REALSXP g0c3 [NAM(2)] (len=3, tl=0) 1,2,3
``` 

If we assign the vector to a variable and inspect that, we see that `named` is
set to `1`, indicating that there is exactly one reference to the vector. If we
then assign `x` to another variable `y`, `named` will be set to `2`, again,
indicating that there are two references to `named`. Note that `named` is
increased also in `x` after the assignment:

```R
> .Internal(inspect(x))
@55555b5ec6e8 14 REALSXP g0c3 [NAM(2)] (len=3, tl=0) 1,2,3
```

However, if we continue the assignment, `named` plateaus at a number specified
by the constant `NAMEDMAX` defined in `Rinternals.h`. In my interpreter that
number is set to `3`. Thus:

```R
> z <- x
> .Internal(inspect(z))
@55555b5ec6e8 14 REALSXP g0c3 [NAM(3)] (len=3, tl=0) 1,2,3
> v <- x
> .Internal(inspect(v))
@55555b5ec6e8 14 REALSXP g0c3 [NAM(3)] (len=3, tl=0) 1,2,3
```

Note also that when we inspect `x`, `y`, `z`, and `v` we retrieve the exact
same object each time (`@55555b5ec6e8`). In fact, the purpose of counting the
references is to provide an illusion of *copy by value* semantics in R while
preventing the interpreter from needlessly performing costly copying
operations. When a SEXP is about to be altered, the interpreter checks the
`named` field. Values within the range of `2` and `NAMEDMAX` signify that the
SEXP must be duplicated before the alteration is applied. A value of `0` means
that it is known that no other SEXP shares data with this one, and so it may
safely be altered in-place. A value of `1` signifies that there are two copies
of the SEXP in principle, but one of them exists only for the duration of
computation, so many operations can avoid copying in this case. There are handy
macros that illustrate the semantics of `named`:

```C
#define MAYBE_SHARED(x) (NAMED(x) > 1)
#define NO_REFERENCES(x) (NAMED(x) == 0)
#define MARK_NOT_MUTABLE(x) SET_NAMED(x, NAMEDMAX)
#define MAYBE_REFERENCED(x) (! NO_REFERENCES(x))
#define NOT_SHARED(x) (! MAYBE_SHARED(x))
```

There are two macros to look up and manipulate the `named` field directly:

```C
#define SET_NAMED(x,v)  (((x)->sxpinfo.named)=(v))
#define NAMED(x)        ((x)->sxpinfo.named)
```

### General purpose

There are 16 general-purpose bits used by various specific SEXPs to indicate
type-specific information. Generally speaking there are a number of bit-masks
that are used within the interpreter to see whether a particular bit is set or
not. For instance:

```C
#define MISSING_MASK	15
#define MISSING(x)	((x)->sxpinfo.gp & MISSING_MASK)
```

<!--
Bit 0 is used by macros DDVAL and SET_DDVAL. This indicates that a SYMSXP is
one of the symbols ..n which are implicitly created when ... is processed, and
so indicates that it may need to be looked up in a DOTSXP.

Bit 0 is used for PRSEEN, a flag to indicate if a promise has already been seen
during the evaluation of the promise (and so to avoid recursive loops).

Bit 0 is used for HASHASH, on the PRINTNAME of the TAG of the frame of an
environment. (This bit is not serialized for CHARSXP objects.)

Bits 0 and 1 are used for weak references (to indicate ‘ready to finalize’,
‘finalize on exit’).

Bit 0 is used by the condition handling system (on a VECSXP) to indicate a
calling handler.

Bit 4 is turned on to mark S4 objects.

Bits 1, 2, 3, 5 and 6 are used for a CHARSXP to denote its encoding. Bit 1
indicates that the CHARSXP should be treated as a set of bytes, not necessarily
representing a character in any known encoding. Bits 2, 3 and 6 are used to
indicate that it is known to be in Latin-1, UTF-8 or ASCII respectively.

Bit 5 for a CHARSXP indicates that it is hashed by its address, that is
NA_STRING or is in the CHARSXP cache (this is not serialized). Only
exceptionally is a CHARSXP not hashed, and this should never happen in end-user
code.

#define S4_OBJECT_MASK ((unsigned short)(1<<4))
#define NOJIT_MASK ((unsigned short)(1<<5))
#define GROWABLE_MASK ((unsigned short)(1<<5))

-->

We will point out specific masks and their uses as we discuss specific SEXPs.

### GC

There are a few flags which pertain to the GC which will make recurring
appearances. These are: `mark`, `gcgen`, and `gccls`. R has a generational
garbage collector. 

<!--I think getting into the specifics of the GC is probably beyond the scope
of this report, so let us gloss over these. -->

The `mark` flag is used by the GC to mark which objects are currently in use.
The value of `1` signifies that the the object is in use and `0` signifies that
it is not.

The `gcgen` specifies which generation of old objects this SEXP belongs to. This
can either be `0` or `1`.

Finally, `gccls` specifies the class of objects to which this object belongs.
There are up to 8 classes (`0`--`7`) among which there is a special class for
large nodes, nodes with custom initializers, and multiple classes for small
nodes. 

Let's take a look at some examples using the `inspect` function in R

```R
> x <- 1
> .Internal(inspect(x))
@55555d1687c0 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
```

Here the inspector tells us through `g0c1` that object `x` belongs to
generation `0` and class `1`. It also tells us that `x` is not marked.
Otherwise we would see a flag `MARK` in the square brackets.

Let's take a look at another example.

```R
> y <- c(1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16)
> .Internal(inspect(y))
@55555d021bb8 14 REALSXP g0c5 [NAM(1)] (len=16, tl=0) 1,2,3,4,5,...
```

Object `y` is also unmarked and belongs to generation `0`, but we can see that
since it is larger it falls into a different class. Let's try aging the objects
by invoking the garbage collector manually:

```R
> gc()
          used (Mb) gc trigger  (Mb) max used  (Mb)
Ncells  481097 25.7     930328  49.7   821881  43.9
Vcells 8445555 64.5   16253847 124.1 14849172 113.3
> .Internal(inspect(x))
@55555d1687c0 14 REALSXP g1c1 [MARK,NAM(3)] (len=1, tl=0) 1
> .Internal(inspect(y))
@55555d021bb8 14 REALSXP g1c5 [MARK,NAM(1)] (len=16, tl=0) 1,2,3,4,5,...
```

Here we can see that both x and y were moved to generation `1`, and both of
them were marked as in use, as indicated by the `MARK` notification.

<!--sxpinfo allocates 3 bits for the node class, so at most 8 are allowed 

Node Classes.  Non-vector nodes are of class zero. Small vector
   nodes are in classes 1, ..., NUM_SMALL_NODE_CLASSES, and large
   vector nodes are in class LARGE_NODE_CLASS. Vectors with
   custom allocators are in CUSTOM_NODE_CLASS. For vector nodes the
   node header is followed in memory by the vector data, offset from
   the header by SEXPREC_ALIGN

#define LARGE_NODE_CLASS  (NUM_NODE_CLASSES - 1)
#define CUSTOM_NODE_CLASS (NUM_NODE_CLASSES - 2)
#define NUM_SMALL_NODE_CLASSES (NUM_NODE_CLASSES - 2)

#define NUM_OLD_GENERATIONS 2

#define NODE_GENERATION(s) ((s)->sxpinfo.gcgen)
#define SET_NODE_GENERATION(s,g) ((s)->sxpinfo.gcgen=(g))-->

## Payload

Apart from the header the SEXP contains the following union `u` which contains
the meat of the SEXP:

```C
union {
    struct primsxp_struct primsxp;
    struct symsxp_struct  symsxp;
    struct listsxp_struct listsxp;
    struct envsxp_struct  envsxp;
    struct closxp_struct  closxp;
    struct promsxp_struct promsxp;
} u;
```

The component of the union on which to operate is chosen depending on the type:
`closxp` will have members that make sense for function definitions, `promsxp`
will have members representing a promise, etc. There's rarely a need to look
into this union directly either, since there are ample macros that retrieve
particular members of particular SEXP types. 

Let's break down how to use the payload by SEXP type.

### Symbols

`SYMSXP`s represent symbols, which are anything from operators, to names of
variables and function arguments. As such they also often appear as part of
other SEXPs.

`SYMSXP`s' payloads are best represented using the `symsxp_struct`. This
structure is the `symsxp` field of union `u` in `SEXPREC`. It looks as follows:

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│   pname    │   value    │  internal  │
│            │            │            │
└───[SEXP]───┴───[SEXP]───┴───[SEXP]───┘
```

The `pname` slot contains a pointer to a `CHARSXP` which represents the
printable name of the symbol. This can be accessed by the C macro `PRINTNAME`.
Since the result of `PRINTNAME` is an SEXP of type `CHARSXP`, which is a vector
type (described further below). We can retrieve the contents of a `CHARSXP` as
a C string using the `CHAR` macro.

The `value` slot contains, well, a value associated with the symbol. This is
often not used (most uses in the source set the value to the constant
`R_Undefined`). We can see it being used in the case of symbols for built-in
and special functions, which store the FUNSXPs (discussed below) in the `value`
slot. Another example are symbols for active bindings, which store the `FUNSXP`
of the function bound to the symbol in the `value` slot.

If the `value` slot points to an `.Internal` function, the `internal` slot
should point to a SEXP that represents the appropriate internal function. It
can be retrieved via the `INTERNAL` macro. It is often the case that when this
field is not set, it is uninitialized, so reference it with care.

A brief digression about `R_Undefined`. There are a few pre-defined SEXPs that
usually contain no useful content, but their address/identity are used for
marking specific cases. One such example is `R_Undefined` which represents an
undefined SEXP. `R_Undefined` is also a SYMSXP.

Let's see some symbols from the inside using the `inspect` function in R:

```R
> x <- 1
> .Rinternal(inspect(substitute(x)))
@558fa07afe20 01 SYMSXP g0c0 [MARK] "x"
```

Inspect gives us the pointer address of this particular SEXP.  Upon further
inspection we also see this is a SEXP of type `01`, ie. indeed a `SYMSXP`. It
holds the value of `x` which represents the printable name of this symbol. In
order to extract that value in C we can use the `PRINTNAME` macro. This macro
returns another SEXP, a `CHARSXP`. `CHARSXP`s are the internal cache-able string
representation of R. You don't see them from R level, but you see them a lot of
them when extracting information from other types of SEXPs. You can retrieve a
C-string from a `CHARSXP` with the `CHAR` macro. Example:

```C
SEXP s;
const char *symbol_printname = CHAR(PRINTNAME(s));
```

We can also observe the value associated with the symbol:

```C
if (SYMVALUE(s) != R_Undefined) {
    // ... 
}
```

### Nil

Here's a simple one: the `NILSXP`. It is a singleton representing R's `NULL`
value, which is represented by the `R_NilValue` singleton from the point of
view of a C programmer.  `R_NilValue` is sometimes used to fill in an empty
slot in a SEXP, although often that is done with the `R_Undefined` symbol.
Internally, the singleton is self-referential:

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│ R_NilValue │ R_NilValue │ R_NilValue │
│            │            │            │
└──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┘
```

Inspection reveals nothing of particular interest:

```R
> .Internal(inspect(substitute(NULL)))
@558fa0740d78 00 NILSXP g0c0 [MARK,NAM(2)] 
```

### Lists

Another type of SEXPs are lists (of other SEXPs). They are usually not directly
used by R programmers, but instead they make up other SEXPs. The name "list" is
also a misnomer. A `LISTSXP` is really a *cons cell* containing a `carval`
slot, a `tagval` slot, and a `cdrval` slot. The `carval` and `tagval` slots
contain a pair of pointers to values constituting an element of the list, and
`cdrval` is a pointer to another `LISTSXP` or `NILSXP`. Since there is a pair
of values in each member, we refer to these lists as *pairlists*.

The payload of a `LISTSXP` is defined by the `listsxp_struct`. It can be
accessed via the `listsxp` field of union `u` in `SEXPREC` and looks as
follows: 

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└───[SEXP]───┴───[SEXP]───┴───[SEXP]───┘
```

Let's inspect one example:

```R
> l <- pairlist(1,2,3)
> .Internal(inspect(l))
@55ba3e32f9d0 02 LISTSXP g0c0 [NAM(1)] 
  @55ba3e0c8fc0 14 REALSXP g0c1 [] (len=1, tl=0) 1
  @55ba3e0c8f88 14 REALSXP g0c1 [] (len=1, tl=0) 2
  @55ba3e0c8f50 14 REALSXP g0c1 [] (len=1, tl=0) 3
```

We create a pairlist by coercing a vector into one. Alternatively, we could
have used the `vector(mode="pairlist")` constructor or the
`as.vector(mode="pairlist", ...)` coercion to create an one.  The inspect tool
shows the list as flat, but the real structure is composed of multiple cons
cells as follows:

```
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │ R_NilValue │            │
│            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     2      │ R_NilValue │            │
│            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     3      │ R_NilValue │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴──[NILSXP]──┘
               
```

The `carval` slots all point to another SEXP that holds a vector of real
numbers (described further below) and the `tagval` slots all point to
`R_NilValue`, also another SEXP. The `cdrval` points to the next element of the
pairlist, or in the case of the last element, to `R_NilValue`.

The example does not set any interesting `tagval` values. Let us try setting
those:

```R
> lt <- pairlist(x=1, y=2, z=3)
> .Internal(inspect(lt))
@5555570d2980 02 LISTSXP g0c0 [NAM(1)] 
  TAG: @555555c92920 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
  @55555a3c2d50 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
  TAG: @555555d7e508 01 SYMSXP g1c0 [MARK] "y"
  @55555a3c2d18 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 2
  TAG: @555555d62f68 01 SYMSXP g1c0 [MARK,NAM(3)] "z"
  @55555a3c2ce0 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 3
```

We can see the `tagval` slot used to keep the names (ie. *tags*) of vectors that
we assigned. This is indicated by a line prefixed `TAG:` just before the line
describing `carval`. We see that the `tagval` slots point to `SYMSXP`s. On a
diagram, this would be represented as follows:

```
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │            │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     2      │    `y`     │            │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     3      │    `z`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
               
```

There are a number of accessor macros designed to make working with pairlists
easier in `C`:

```C
#define TAG(e)   ((e)->u.listsxp.tagval)
#define CAR(e)   ((e)->u.listsxp.carval)
#define CDR(e)   ((e)->u.listsxp.cdrval)
#define CDAR(e)  CDR(CAR(e))
#define CDDR(e)  CDR(CDR(e))
#define CDDDR(e) CDR(CDR(CDR(e)))
```

Let us illustrate what they access using the diagram of one of the previous
examples:

```
      ┌─────────────────────────────────────── CAR(s)
      │            ┌────────────────────────── TAG(s)
      │            │            ┌───────────── CDR(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │            │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 │    ┌─────────────────────────────────────── CDAR(s)
 │    │            ┌────────────────────────── TAG(CDR(s))
 │    │            │            ┌───────────── CDDR(s)
 v    v            v            v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     2      │    `y`     │            │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 │    ┌─────────────────────────────────────── CAR(CDDR(s))
 │    │            ┌────────────────────────── TAG(CDDR(s))
 │    │            │            ┌───────────── CDDDR(s)
 v    v            v            v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     3      │    `z`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
               
```

It is worth noting that there is no macro that would return the length of the
pairlist. Indeed, one must implement it as follows (this can also be used as a
template to traversing pairlists):

```C
SEXP s;
int length = 0;  
for(SEXP cons = s; cons != R_NilValue; cons = CDR(cons)) { 
    length++;
}
``` 

Pairlists can also be used to make trees. We can construct a (very simple) tree
as follows:

```R
> tree <- pairlist(pairlist(1, 2, 3), 1, 2, 3)
> .Internal(inspect(tree))
@5555570bbc68 02 LISTSXP g0c0 [NAM(1)] 
  @5555570bb8e8 02 LISTSXP g0c0 [NAM(3)] 
    @55555a3c2880 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
    @55555a3c2848 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 2
    @55555a3c2810 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 3
  @55555a3c27d8 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
  @55555a3c27a0 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 2
  @55555a3c2768 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 3
```

There are accessors that can be used specifically with for trees:

```C
#define CAAR(e)   CAR(CAR(e))                    
#define CADR(e)   CAR(CDR(e))                    
#define CADDR(e)  CAR(CDR(CDR(e)))               
#define CADDDR(e) CAR(CDR(CDR(CDR(e))))          
#define CAD4R(e)  CAR(CDR(CDR(CDR(CDR(e)))))
```

Then, the tree and its accessors look like this:

```
      ┌────────────────────────────────── CAR(s)
      │            ┌───────────────────── TAG(s)
      │            │            ┌──────── CDR(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│            │ R_NilValue │            │
│            │            │            │
└─[LISTSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
      │                         │
 ┌────┘                         └────────────────────────┐
 │    ┌────────────────────────────────── CAAR(s)        │    ┌────────────────────────────────── CDAR(s)
 │    │            ┌───────────────────── TAG(CAR(s))    │    │            ┌───────────────────── TAG(CDR(s))
 │    │            │            ┌──────── CADR(s)        │    │            │            ┌──────── CDDR(s)
 v    v            v            v                        v    v            v            v
┌────────────┬────────────┬────────────┐                ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │                │   carval   │   tagval   │   cdrval   │
│     1      │ R_NilValue │            │                │     1      │ R_NilValue │            │
│            │            │            │                │            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘                └─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
                                 │                                                      │
 ┌───────────────────────────────┘                       ┌──────────────────────────────┘
 │    ┌────────────────────────────────── CAR(CADR(s))   │    ┌────────────────────────────────── CAR(CDDR(s))
 │    │            ┌───────────────────── TAG(CADR(s))   │    │            ┌───────────────────── TAG(CDDR(s))
 │    │            │            ┌──────── CADDR(s)       │    │            │            ┌──────── CDDDR(s)
 v    v            v            v                        v    v            v            v
┌────────────┬────────────┬────────────┐                ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │                │   carval   │   tagval   │   cdrval   │
│     2      │ R_NilValue │            │                │     2      │ R_NilValue │            │
│            │            │            │                │            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘                └─[REALSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘ 
                                 │                                                      │
 ┌───────────────────────────────┘                       ┌──────────────────────────────┘
 │    ┌────────────────────────────────── CAR(CADDR(s))  │    ┌────────────────────────────────── CAR(CDDDR(s))
 │    │            ┌───────────────────── TAG(CADDR(s))  │    │            ┌───────────────────── TAG(CDDDR(s))
 │    │            │            ┌──────── CAD4R(s)       │    │            │            ┌──────── CDR(CDDDR(s))
 v    v            v            v                        v    v            v            v
┌────────────┬────────────┬────────────┐                ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │                │   carval   │   tagval   │   cdrval   │
│     3      │ R_NilValue │ R_NilValue │                │     3      │ R_NilValue │ R_NilValue │
│            │            │            │                │            │            │            │
└─[REALSXP]──┴──[NILSXP]──┴──[NILSXP]──┘                └─[REALSXP]──┴──[NILSXP]──┴──[NILSXP]──┘ 
```

There are also a lot of analogous setters for these fields: `SETCAR`, `SETCDR`,
`SETCADR`, `SETCADDR`, `SETCADDDR`, `SETCAD4R`, `SET_TAG`.

There are only a few places where we can openly interface with pairlists in
their natural environments, and that's when inspecting function arguments
(formals), language expressions and SEXP attributes. We will discuss those
when dealing with closures, language expressions and expression vectors.

### Language expressions

Language expressions are a type of SEXPs used to create ASTs and one that
specifically signifies function applications.  They do not have their own
structure in the payload union, so we will use `listsxp_struct` to represent
it.

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[SYMSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
```

The structure contains three slots. The `carval` slot points to a `SYMSXP` that
represents the function being called. the `cdrval` slot points to a list of
arguments with which the function is supposed to be applied. The list is a
`LISTSXP` whose `carval`s represent the values passed by argument, and
`tagval`s represent the names of the arguments, if they are passed by name. If
there are no arguments, `cdrval` points to `R_NilValue`.

I have never seen a `LANGSXP`'s `tagval` to be anything except `R_NilValue`.

We can retrieve an unevaluated AST by passing an expression to substitute:

```R
> ast <- substitute(f(x) + f(x=y))
```

This creates the following structure:

```
                          ┌────────────┬────────────┬────────────┐
                          │   carval   │   tagval   │   cdrval   │
                          │    `+`     │ R_NilValue │            │
                          │            │            │            │
                          └──[SYMSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
                                                          │
                           ┌──────────────────────────────┘
                           v
                          ┌────────────┬────────────┬────────────┐
                          │   carval   │   tagval   │   cdrval   │
                          │            │ R_NilValue │            │
                          │            │            │            │
                          └─[LANGSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘
                                │                         │
 ┌──────────────────────────────┘                         │
 │                         ┌──────────────────────────────┘
 │                         v
 │                        ┌────────────┬────────────┬────────────┐
 │                        │   carval   │   tagval   │   cdrval   │
 │                        │            │ R_NilValue │ R_NilValue │
 │                        │            │            │            │
 │                        └─[LANGSXP]──┴──[NILSXP]──┴──[NILSXP]──┘
 │                              │
 │                              └────────────────────┐
 v                                                   v
┌────────────┬────────────┬────────────┐            ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │            │   carval   │   tagval   │   cdrval   │
│    `f`     │ R_NilValue │            │            │    `f`     │ R_NilValue │            │
│            │            │            │            │            │            │            │
└──[SYMSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘            └──[SYMSXP]──┴──[NILSXP]──┴─[LISTSXP]──┘ 
                                │                                                   │
 ┌──────────────────────────────┘                    ┌──────────────────────────────┘
 v                                                   v
┌────────────┬────────────┬────────────┐            ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │            │   carval   │   tagval   │   cdrval   │
│    `x`     │ R_NilValue │            │            │    `y`     │    `x`     │            │
│            │            │            │            │            │            │            │
└──[SYMSXP]──┴──[NILSXP]──┴──[NILSXP]──┘            └──[SYMSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘ 

```

Inspecting the AST with the `inspect` function provides self-explanatory
output:

``` R
> .Internal(inspect(ast))
@55d96b9d0900 06 LANGSXP g0c0 [NAM(3)] 
  @55d96a453b80 01 SYMSXP g1c0 [MARK,LCK,gp=0x5000] "+" (has value)
  @55d96b9d0a88 06 LANGSXP g0c0 [] 
    @55d96a5c7868 01 SYMSXP g1c0 [MARK] "f"
    TAG: @55d96a4c5880 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @55d96a4c5880 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
  @55d96b9d0bd8 06 LANGSXP g0c0 [] 
    @55d96a5c7868 01 SYMSXP g1c0 [MARK] "f"
    @55d96a5b1228 01 SYMSXP g1c0 [MARK] "y"
```

There are no special macros for accessing `LANGSXP`s, but their members can be
retrieved using `LISTSXP` macros.

### Vectors

Vector types are a whole family of types used mainly to store data. Most of the
vector types represent vectors available to the R programmer: `character`,
`numeric`, `logical`, etc. The one exception is a type of vector used
internally to store strings called `CHARSXP`.

Vector types are actually a little uncharacteristic, since they don't use the
SEXP structure, and instead need to be cast to VECSEXP (do not confuse with
`VECSXP`) if we want to make sense of their internals. A `VECSEXP` is a pointer
to a structure called `VECTOR_SEXPREC` defined as follows:

```
┌────────────────────[SEXPREC_HEADER]─────────────────────┐                         ┌──[align and data]─---
│                                                         │                         │                      
v                                                         v                         v
┌──────────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬───────────────────---
│                  │            │   gengc    │   gengc    │            │            │            
│     sxpinfo      │   attrib   │ prev_node  │ next_node  │   length   │ truelength │         ...        
│                  │            │            │            │            │            │             
└─[sxpinfo_struct]─┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┴─[R_xlen_t]─┴─[R_xlen_t]─┴───────────────────---
                                                          ʌ                         ʌ
                                                          │                         │
                                                          └────────[vecsxp]─────────┘
```

They also have a standard SEXP header structure first, but then they have a
payload of type `vecsxp_struct`. The payload contains two pieces of information: the
length of the vector and its true length. 

The length represents how many elements the vector currently holds. Apparently,
vector length used to be limited to $2^{31}-1$, but currently the maximum
length can be up to $2^{64}-1$. So, currently `R_xlen_t` is just an alias for
`int`. The length of a vector can be retrieved using the `XLENGTH` macro.

True length is mostly unused. It is used in when hash tables are implemented
using VECSXPs to indicate the number of primary slots in use, and for the
reference hash tables in serialization, where truelength is the number of slots
in use.

<!--
True length is the number of bytes allocated to carry this vector's data in
memory. In other words it's the maximum number of bytes of data the vector can
hold without being expanded. True length is rounded up to a multiple of 8
bytes. 
-->

That data itself is located in a block of bytes that follows `vecsxp` in
memory. The data is aligned as required, so there may be an artificial gap
between the end of the VECSEXP structure and the beginning of data. There is a
convenience structure to help us find the beginning of the data called
SEXPREC_ALIGN defined as follows, which aligns the structure above with a
`double`:

```C
typedef union { VECTOR_SEXPREC s; double align; } SEXPREC_ALIGN;
```

We can then get a pointer to the data as follows:

```C
((SEXPREC_ALIGN *) (x)) + 1
```

We should cast the pointer into whatever data type the vector contains, or in
the general case:

```C
((void *) ((SEXPREC_ALIGN *) (x)) + 1)
```

This is encapsulated in the macro `DATAPTR`.

Now onto specific sub-types of vectors!

#### Vector sub-types

One of the most common types of vectors are those exposed to the programmers as
R atomic vectors:

<!--
┌─────────┬─────────────────────┬───────────────────────────┬──────────┬──────────────────────────────────┐
│ type    │ constructor (in R)  │ coercion (in R)           │ contains │ description                      │
├─────────┼─────────────────────┼───────────────────────────┼──────────┼──────────────────────────────────┤
│ REALSXP │ numeric(N)          │ as.numeric(V)             │ double   │ vector of reals/floating points  │
│ INTSXP  │ integer(N)          │ as.integer(V)             │ int      │ vector of integers               │
│ LGLSXP  │ logical(N)          │ as.logical(V)             │ Rboolean │ vector of logical/boolean values │
│ CPLXSXP │ complex(N)          │ as.complex(V)             │ Rcomplex │ vector of complex numbers        │
│ STRSXP  │ character(N)        │ as.character(V)           │ CHARSXP  │ vector of character strings      │
│ RAWSXP  │ raw(N)              │ as.raw(V)                 │ Rbyte    │ vector of raw bytes              │
│ VECSXP  │ vector(mode="list") │ as.vector(mode="list", V) │ SEXP     │ generic vector of SEXPs          │
│ EXPRSXP │ vector(             │ as.vector(                │ SEXP     │ vector of expressions (usually   │
│         │  mode="expression") │  mode="expression", V)    │          │ LANGSXPs)                        |
└─────────┴─────────────────────┴───────────────────────────┴──────────┴──────────────────────────────────┘
            N: initial length     V: another vector
-->

```
type     constructor (in R)         coercion (in R)                contains  description                      
══════════════════════════════════════════════════════════════════════════════════════════════════════════════
REALSXP  numeric(N)                 as.numeric(V)                  double    vector of reals/floating points  
INTSXP   integer(N)                 as.integer(V)                  int       vector of integers              
LGLSXP   logical(N)                 as.logical(V)                  Rboolean  vector of logical/boolean value
CPLXSXP  complex(N)                 as.complex(V)                  Rcomplex  vector of complex numbers      
STRSXP   character(N)               as.character(V)                CHARSXP   vector of character strings      
RAWSXP   raw(N)                     as.raw(V)                      Rbyte     vector of raw bytes              
VECSXP   vector(mode="list")        as.vector(mode="list", V)      SEXP      generic vector of SEXPs          
EXPRSXP  vector(mode="expression")  as.vector(mode="expression, V) SEXP      vector of expressions (usually 
                                                                             LANGSXPs)
N: initial length
V: another vector
```

We explain the type CHARSXP further below, and the types Rcomplex and Rboolean
are defined as follows:

```C
typedef char                           Rbyte;
typedef enum { FALSE = 0, TRUE }       Rboolean;
typedef struct { double r; double i; } Rcomplex;
```

Let's take a look at an example vector:

```R
> c(1, 2, 3)
[1] 1 2 3
```

We could represent this vector as follows (without the header):

```
  VECSXP:                           REAL:                                   
┌────────────┬────────────┬─-----─┬────────────┬────────────┬────────────┐
│   length   │ truelength │       │    [0]     │    [1]     │    [2]     │          
│     3      │     0      │       │    1.0     │    2.0     │    3.0     │
│            │            │       │            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴──[double]──┴──[double]──┴──[double]──┘
```

We can observe the vectors with inspect as follows:

```R
> .Internal(inspect(c(1,2,3)))
@55e9105aa398 14 REALSXP g0c3 [] (len=3, tl=0) 1,2,3
```

The inspector prints out the length and true length of the vector in
parentheses as well as the contents of the vector (possibly truncated for
readability). The same applies for integer, logical, and raw vectors:

```R
> .Internal(inspect(as.integer(c(1,2,3))))
@55e9105cecf8 13 INTSXP g0c2 [] (len=3, tl=0) 1,2,3
> .Internal(inspect(c(TRUE, FALSE, NA)))
@55e9105cecb8 10 LGLSXP g0c2 [] (len=3, tl=0) 1,0,-2147483648
> .Internal(inspect(as.raw(c(1,2,3))))
@5562052b1aa8 24 RAWSXP g0c1 [] (len=3, tl=0) 01,02,03
```

Complex vectors' contents are not listed by the inspector:

```R
> .Internal(inspect(complex(real=c(1,2,3), imaginary=c(1,2,3))))
@55e910924ac8 15 CPLXSXP g0c4 [] (len=3, tl=0)
```
Whereas, the contents of a character string vector and a generic vector are
displayed per-line as children of the vector, since they are each an
independent SEXPs:

```R
> .Internal(inspect(c("a", "b", "c")))
@55e9105aa258 16 STRSXP g0c3 [] (len=3, tl=0)
  @55e90a16fec0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "a"
  @55e90a4e47f8 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "b"
  @55e909e4ce48 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "c"
> .Internal(inspect(as.vector(mode="list", c(1,2,3))))
@55ba41b45148 19 VECSXP g0c3 [] (len=3, tl=0)
  @55ba3c70d410 14 REALSXP g0c1 [] (len=1, tl=0) 1
  @55ba3c70d598 14 REALSXP g0c1 [] (len=1, tl=0) 2
  @55ba3c70d640 14 REALSXP g0c1 [] (len=1, tl=0) 3
```

Note that in the generic vector all elements are individual REALSXP vectors.

On the C side of things, the block of bytes containing the elements are
accessible through the `DATAPTR` macro. The results have to be coerced into the
appropriate types, and there are macros that do this too:

```C
#define LOGICAL(x)      ((int *)            DATAPTR(x))
#define INTEGER(x)      ((int *)            DATAPTR(x))
#define RAW(x)          ((Rbyte *)          DATAPTR(x))
#define COMPLEX(x)      ((Rcomplex *)       DATAPTR(x))
#define REAL(x)         ((double *)         DATAPTR(x))
```

We can also cast the data into read-only types:

```C
#define LOGICAL_RO(x)   ((const int *)      DATAPTR_RO(x))
#define INTEGER_RO(x)   ((const int *)      DATAPTR_RO(x))
#define RAW_RO(x)       ((const Rbyte *)    DATAPTR_RO(x))
#define COMPLEX_RO(x)   ((const Rcomplex *) DATAPTR_RO(x))
#define REAL_RO(x)      ((const double *)   DATAPTR_RO(x))
```

These return an array of elements that we can traverse as follows:

```C
SEXP s;
for (int i = 0; i < XLENGTH(s); i++) {
    INTEGER(s)[i] += 1;
}
```

There are also macros and functions for reading or writing just one element of
the vector: `VECTOR_ELT` and `SET_VECTOR_ELT` for the general case, and then
`LOGICAL_ELT` and `SET_LOGICAL_ELT`, `INTEGER_ELT` and `SET_INTEGER_ELT`, etc.
for specific sub-types.

```C
SEXP s;
for (int i = 0; i < XLENGTH(s); i++) {
    SET_INTEGER_ELT(s, i, INTEGER_ELT(s, i) + 1);
}
```

These are implemented by coercing the results of `DATA_PTR` into appropriate
types.

#### Scalars

One of the flags in SEXP headers is the `scalar` flag. this flag is used to
indicate that a numeric (or logical) vector is of length one. So:

```R
> v1 <- 1       # scalar flag = 1
> v2 <- c(1)    # scalar flag = 1
> v3 <- c(1,1)  # scalar flag = 0
```

There is a macro that allows us to access the scalar flag from C which checks
whether a particular SEXP is a scalar vector of a particular type.

```C
#define IS_SCALAR(x, t) (((x)->sxpinfo.type == (t)) && (x)->sxpinfo.scalar)
```

The knowledge about whether a vector is scalar is used by the interpreter for
optimizations. For instance, the `do_arith` function that, as the name
suggests, does arithmetic, will perform a much cheaper addition, say, right
there and then, rather than firing up the machinery to add two vectors
together.

As of the time of writing the inspect function does not print the scalar flag.

```R
> .Internal(inspect(1))
@555638fa87e8 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
> .Internal(inspect(c(1,2)))
@55563a2e7928 14 REALSXP g0c2 [] (len=2, tl=0) 1,2
```

#### Internal character vectors

Vectors of type CHARSXP are used internally by other SEXPs to store character
strings. For example, they are used to store the elements of STRSXP, as well as
the printable names of a SYMSXPs. They can only be seen as parts of other
SEXPs and are transparent to programmers.

We can retrieve the contents of a CHARSXP via the `CHAR` macro, which returns a
C string.

If inspected, CHARSXPs yield the following output, which informs the encoding
of the character string, whether or not it was cached, and, of course, the
contents.

```
@555555c62998 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "x"
```

the idea of encoding should be fairly self-explanatory, but what is `cached`?
By default R maintains a (hashed) global cache of ‘variables’ (that is symbols
and their bindings) which have been found, and this refers only to environments
which have been marked to participate, which consists of the global environment
(aka the user workspace), the base environment plus environments which have
been attached to the search path (see `attach` function). When an environment
is either attached or detached, the names of its symbols are flushed from the
cache. The cache is used whenever searching for variables from the global
environment (possibly as part of a recursive search).

The encoding and caching are specified in the general purpose bits of the
header. There are bit-masks defined for specific encodings and a bit mask for
the cached flag:

```C
#define BYTES_MASK  (1<<1)
#define LATIN1_MASK (1<<2)
#define UTF8_MASK   (1<<3)
                             /* (1<<4) is taken by S4_OBJECT_MASK */
#define CACHED_MASK (1<<5)
#define ASCII_MASK  (1<<6)
```

They can be checked and set by using macros like `IS_UTF8` and `SET_UTF8` which
compare or set the appropriate `gp` bits against specific masks, eg.:

```C
# define IS_UTF8(x)  ((x)->sxpinfo.gp & UTF8_MASK)
# define SET_UTF8(x) (((x)->sxpinfo.gp) |= UTF8_MASK)
``

#### Expression vectors

Expression vectors are similar to generic vectors, except that there is an
expectation that a character vector contains LANGSXPs (although is not enforced
by the SEXP itself).

When inspected, each expression in an expression vector is displayed on a
separate line, since they are independent SEXPs. 

```R
> .Internal(inspect(as.vector(mode="expression", substitute(c(1,2,3)))))
@55ba3e0be080 20 EXPRSXP g0c1 [] (len=1, tl=0)
  @55ba3e35b180 06 LANGSXP g0c0 [NAM(3)] 
    @55ba3985aa38 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x6000] "c" (has value)
    @55ba3e0be208 14 REALSXP g0c1 [] (len=1, tl=0) 1
    @55ba3e0be160 14 REALSXP g0c1 [] (len=1, tl=0) 2
    @55ba3e0be128 14 REALSXP g0c1 [] (len=1, tl=0) 3
```

In the example below we use `substitute` to obtain a LANGSXP, but we could
technically put any SEXP into an expression vector:

```R
> .Internal(inspect(as.vector(mode="expression", c(1,2,3))))
@55ba419757d8 20 EXPRSXP g0c3 [] (len=3, tl=0)
  @55ba3e0c18b0 14 REALSXP g0c1 [] (len=1, tl=0) 1
  @55ba3e0c1878 14 REALSXP g0c1 [] (len=1, tl=0) 2
  @55ba3e0c1760 14 REALSXP g0c1 [] (len=1, tl=0) 3
``` 

Expressions are used primarily as the result of the `parse` function, which
parses text into R ASTs represented as a `EXPRSXP` vector containing `LANGSXP`s
for each parsed item: 

```R
> expr <- parse(text=c("1 + 1", "2 + 2")))
> expr
expression(1 + 1, 2 + 2)
> .Internal(inspect(expr)))
@55555a3bfc78 20 EXPRSXP g0c2 [ATT] (len=2, tl=0)
  @5555569c3a28 06 LANGSXP g0c0 [] 
    @555555c26ce0 01 SYMSXP g1c0 [MARK,LCK,gp=0x5000] "+" (has value)
    @55555a3c5890 14 REALSXP g0c1 [] (len=1, tl=0) 1
    @55555a3c5858 14 REALSXP g0c1 [] (len=1, tl=0) 1
  @5555569c3c90 06 LANGSXP g0c0 [] 
    @555555c26ce0 01 SYMSXP g1c0 [MARK,LCK,gp=0x5000] "+" (has value)
    @55555a3c57e8 14 REALSXP g0c1 [] (len=1, tl=0) 2
    @55555a3c57b0 14 REALSXP g0c1 [] (len=1, tl=0) 2
ATTRIB:
  @5555569c4358 02 LISTSXP g0c0 [] 
    TAG: @555555c1ae80 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "srcref" (has value)
    @55555a3bfc38 19 VECSXP g0c2 [] (len=2, tl=0)
      @55555a3c37a8 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 1,1,1,5,1,...
      ATTRIB:
	@5555569c3b08 02 LISTSXP g0c0 [] 
	  TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
	  @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>
	  ENCLOS:
	    @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
	  HASHTAB:
	    @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @5555569c76e0 02 LISTSXP g0c0 [] 
		TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"
		@55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)
		  @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Workspace/R-dyntrace"
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      ...
	  ATTRIB:
	    @5555569c5d70 02 LISTSXP g0c0 [] 
	      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	      @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
		@555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
		@555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
	  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	  @55555a3c5820 16 STRSXP g0c1 [] (len=1, tl=0)
	    @555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
      @55555a3c3758 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 2,1,2,5,1,...
      ATTRIB:
	@5555569c3cc8 02 LISTSXP g0c0 [] 
	  TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
	  @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>
	  ENCLOS:
	    @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
	  HASHTAB:
	    @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      @5555569c76e0 02 LISTSXP g0c0 [] 
		TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"
		@55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)
		  @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Workspace/R-dyntrace"
	      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	      ...
	  ATTRIB:
	    @5555569c5d70 02 LISTSXP g0c0 [] 
	      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	      @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
		@555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
		@555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
	  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	  @55555a3c5778 16 STRSXP g0c1 [] (len=1, tl=0)
	    @555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
    TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
    @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>
    ENCLOS:
      @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
    HASHTAB:
      @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)
	@555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	@555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	@555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	@5555569c76e0 02 LISTSXP g0c0 [] 
	  TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"
	  @55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)
	    @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Workspace/R-dyntrace"
	@555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	...
    ATTRIB:
      @5555569c5d70 02 LISTSXP g0c0 [] 
	TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	@55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	  @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
	  @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
    TAG: @555555c1ae10 01 SYMSXP g1c0 [MARK] "wholeSrcref"
    @55555a3c3708 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 1,0,3,0,0,...
    ATTRIB:
      @5555569c43c8 02 LISTSXP g0c0 [] 
	TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
	@5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>
	ENCLOS:
	  @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
	HASHTAB:
	  @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)
	    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	    @5555569c76e0 02 LISTSXP g0c0 [] 
	      TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"
	      @55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)
		@55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Workspace/R-dyntrace"
	    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
	    ...
	ATTRIB:
	  @5555569c5d70 02 LISTSXP g0c0 [] 
	    TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	    @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	      @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
	      @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
	TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	@55555a3c56d0 16 STRSXP g0c1 [] (len=1, tl=0)
	  @555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
```

#### Attributes

The vector in the example above is heavily annotated with attributes, so let's
take a look at what they are before proceeding. As we initially went through
the header, there was a field there called `attrib` which points to another
SEXP: 

```
┌────────────────────[SEXPREC_HEADER]─────────────────────┐
│                                                         │                      
v                                                         v
┌──────────────────┬────────────┬────────────┬────────────┬──---
│                  │            │   gengc    │   gengc    │
│     sxpinfo      │   attrib   │ prev_node  │ next_node  │
│                  │            │            │            │
└─[sxpinfo_struct]─┴───[SEXP]───┴───[SEXP]───┴───[SEXP]───┴──---
```

The SEXP that `attrib` points to is a `LISTSXP` pairlist where every element's
`carval` is a separate attribute represented as an arbitrary SEXP and an
element's `tagval` is a name of that attribute represented by a `SYMSXP`. The
`cdrval` slot points to the next attribute. In the example above the attributes
have their own attributes, which makes the structure a little involved. Let's
separate some patterns and look at those first, then put it all together.

The simplest attribute we see in that mess of code is a `class` attribute. It
is also fairly popular among R objects. It simply specifies the name of classes
that this SEXP belongs to. We can see them in the output of the inspector, e.g:

```R
@5555569c5d70 02 LISTSXP g0c0 [] 
  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
  @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
    @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
    @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
```

Let's depict the first of them:

```
 ATTRIB:
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│            │ R_ClassS.. │            │
│            │  ..ymbol   │            │
└──[STRSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
 ┌────┘
 v                                 STRING:
┌────────────┬────────────┬─-----─┬────────────┬────────────┐
│   length   │ truelength │       │    [0]     │    [1]     │
│     2      │     0      │       │ "srcfile.. │ "srcfile"  │
│            │            │       │  ..copy"   │            │ 
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴─[CHARSXP]──┴─[CHARSXP]──┘
```

So the attribute is an element of a pairlist, whose `tagval` is the symbol at
available as the constant `R_ClassSymbol` whose print name is `symbol`. The
`carval` points to a 2-element string vector containing the strings
`"srcfilecopy"` and `"srcfile"`, which specifies the two classes this object
belongs to.

Let's take a slightly more complicated example from our mess:

```R
@5555569c43c8 02 LISTSXP g0c0 [] 
  TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
  @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>
  ENCLOS:
    @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
  HASHTAB:
    @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)
      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
      @5555569c76e0 02 LISTSXP g0c0 [] 
      TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"
      @55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)
      @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Workspace/R-dyntrace"
      @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
      ...
  ATTRIB:
    @5555569c5d70 02 LISTSXP g0c0 [] 
    TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
      @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
        @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
        @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
```

This is a `srcfile` attribute which records the parameters of the file form
which some particular piece of code came. We can immediately see that the
`srcfile` attribute has its own `class` attribute at the end, specifying this
is an object belonging to classes `srcfilecopy` and `srcfile`. Note that the
inspector prints `ATT` to indicate that the SEXP has attributes. Apart from the
attribute, the SEXP's `tagval` says that this is indeed a `srcfile` attribute.
Finally, the `carval` slot points to a hash table environment which holds some
information about the file from which the data came from. We can represent this
all as follows:


```
 ATTRIB:
--─┬────────────┬─--─┬────────────┬────────────┬────────────┐
   │   attrib   │    │   carval   │   tagval   │   cdrval   │
   │            │    │    ...     │ R_Srcfil.. │            │
   │            │    │            │ ..eSymbol  │            │
--─┴─[LISTSXP]──┴─--─┴──[ENVSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
 ┌──────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│            │ R_ClassS.. │            │
│            │  ..ymbol   │            │
└──[STRSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
 ┌────┘
 v                                 STRING:
┌────────────┬────────────┬─-----─┬────────────┬────────────┐
│   length   │ truelength │       │    [0]     │    [1]     │
│     2      │     0      │       │ "srcfile.. │ "srcfile"  │
│            │            │       │  ..copy"   │            │ 
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴─[CHARSXP]──┴─[CHARSXP]──┘
```

Since the `srcfile` attribute has a `class` that we've already analyzed we will
represent it using a much more concise notation for reasons of sanity
retention:

```
 ATTRIB:
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│    ...     │ R_Srcfil.. │            │
│            │ ..eSymbol  │            │
└──[ENVSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
 ATTRIB:
 class=c("srcfilecopy", "srcfile")
```

There is also the mystery of an environment as `carval`. We will talk about
environments in a separate section, but let us just briefly present the
contents of this one, to show what sort of information is associated with
`srcfile` attributes:

```
index  TAG(HASHTAB)    CAR(HASHTAB)
═══════════════════════════════════════════════════════════
3      wd            ↦ "/home/kondziu/Workspace/R-dyntrace"
5      lines         ↦ c("1 + 1", "2 + 2")
6      Enc           ↦ "unknown"
8      isfile        ↦ FALSE
9      timestamp     ↦ 1.5313e+09
13     filename      ↦ "<text>"
27     fixedNewlines ↦ TRUE
28     parseData     ↦ c(1,1,1,1,1,...)
```

The last attribute pattern we see in that horrible mess is a source reference.
It looks big and scary, but that is only because it has its arguments, which
are the ones we've seen before:

```
@5555569c4358 02 LISTSXP g0c0 [] 
  TAG: @555555c1ae80 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "srcref" (has value)
  @55555a3bfc38 19 VECSXP g0c2 [] (len=2, tl=0)
    @55555a3c37a8 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 1,1,1,5,1,...
    ATTRIB:
        @5555569c3b08 02 LISTSXP g0c0 []                                                                ┐
          TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)           │
          @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>                                │
          ENCLOS:                                                                                       │
            @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>                                     │
          HASHTAB:                                                                                      │
            @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)                                              │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @5555569c76e0 02 LISTSXP g0c0 []                                                          │ srcfile
                TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"                                           │
                @55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)                                     │
                  @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Worksp... │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              ...                                                                                       │
          ATTRIB:                                                                                       │
            @5555569c5d70 02 LISTSXP g0c0 []                                                   ┐        │
              TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)       │        │
              @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)                              │ class  │ 
                @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"    │        │
                @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"        ┘        ┘
          TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)                    ┐
          @55555a3c5820 16 STRSXP g0c1 [] (len=1, tl=0)                                                 │ class
            @555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"                      ┘
    @55555a3c3758 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 2,1,2,5,1,...                                 
    ATTRIB:
        @5555569c3cc8 02 LISTSXP g0c0 []                                                                ┐
          TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)           │
          @5555569ca358 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x5555569ca358>                                │
          ENCLOS:                                                                                       │
            @555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>                                     │
          HASHTAB:                                                                                      │
            @5555573f4dd0 19 VECSXP g0c7 [] (len=29, tl=8)                                              │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              @5555569c76e0 02 LISTSXP g0c0 []                                                          │ srcfile
                TAG: @5555565c8540 01 SYMSXP g1c0 [MARK] "wd"                                           │
                @55555a3c5bd8 16 STRSXP g0c1 [NAM(1)] (len=1, tl=0)                                     │
                  @55555a3c1c48 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "/home/kondziu/Worksp... │
              @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)]                                                │
              ...                                                                                       │
          ATTRIB:                                                                                       │
            @5555569c5d70 02 LISTSXP g0c0 []                                                   ┐        │
              TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)       │        │
              @55555a387558 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)                              │ class  │
                @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"    │        │
                @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"        ┘        ┘
          TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)                    ┐
          @55555a3c5778 16 STRSXP g0c1 [] (len=1, tl=0)                                                 │ class
            @555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"                      ┘
```

In effect a `srcref` is an attribute that has one or more 8-element `INTSXP`
vectors, each with `srcfile` and `class` attributes, in conjunction
representing a location in source code. We can represent a `srcref` attribute as follows:

```
 ATTRIB:
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│    ...     │ R_Srcref.. │            │
│            │  ..Symbol  │            │
└──[VECSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
 ┌──────┘
 v                                 VECTOR_ELT:
┌────────────┬────────────┬─-----─┬────────────┬────────────┐
│   length   │ truelength │       │    [0]     │    [1]     │
│     2      │     0      │       │            │            │
│            │            │       │            │            │ 
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴──[INTSXP]──┴──[INTSXP]──┘
 ┌──────────────────────────────────────┘            │
 │  ┌────────────────────────────────────────────────┘
 │  v
 │ ┌────────────┬────────────┬────────────┬─---
 │ │   length   │ truelength │   align    │
 │ │     8      │     0      │            │ 
 │ │            │            │            │
 │ └─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
 │  ATTRIB:  
 │  srcfile=<text>
 │  class="srcref"
 │            
 │  INTEGER:
 │ ┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
 │ │    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │    [7]     │
 │ │     2      │     1      │     2      │     5      │     1      │     5      │     2      │     2      │ 
 │ │            │            │            │            │            │            │            │            │
 │ └───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘
 └──┐                   
    v
   ┌────────────┬────────────┬────────────┬─---
   │   length   │ truelength │   align    │
   │     8      │     0      │            │ 
   │            │            │            │
   └─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
    ATTRIB:  
    srcfile=<text>
    class="srcref"   

    INTEGER:
   ┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
   │    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │    [7]     │
   │     1      │     1      │     1      │     5      │     1      │     5      │     1      │     1      │ 
   │            │            │            │            │            │            │            │            │
   └───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘
```

How to read a `srcref`? Specifically `class` tells us that what we're looking
at is a `srcref` object, and `srcfile` tells us the location of the file where
the code came from. Then each vector specifies the location of one R expression
as follows:

```
index  meaning                            
═══════════════════════════════════════════════════════════
0      first line of expression
1      first byte of expression
2      first character of expression
3      last line of expression
4      last byte of expression
5      last character of expression
6      first parsed line of expression⁵
7      last parsed line of expression⁵
```

There are functions in R that can retrieve attributes from objects called
`attr` and `attributes`. Let us try to get the `srcref` from our example above
via R:

```R
> s <- attr(expr, "srcref")
> s
[[1]]
1 + 1

[[2]]
2 + 2
```

R behaves as if `srcref` contained code of the expression, but this is just a
trick of the pretty printer. When we ask R to print our `s`, it looks up the
`class` attribute of `s` which is NULL to an appropriate pretty printer
function for lists without class. That pretty printer function looks up the
class attribute for each element and again redirects to an appropriate pretty
printer function for that class. In the case of `srcref` objects, this prints
out the source rather than the less human-readable vector of integers.

```R
> attr(s, "class")                     # or just class(s)
NULL
> attr(s[[1]], "class")                # or just class(s[[1]])
[1] "srcref"
> attr(s[[2]], "class")                # or just class(s[[2]])
[1] "srcref"
```

We can remove the class attribute from our `srcref`s though to remove pretty
printing:

```R
> attr(s[[1]], "class") <- NULL        # or just unclass(s[[1]])
> attr(s[[2]], "class") <- NULL        # or just unclass(s[[2]])
> s
[[1]]
[1] 1 1 1 5 1 5 1 1

[[2]]
[1] 2 1 2 5 1 5 2 2
```

There is also a macro on the C side that retrieves a SEXP's attribute list. 

```C
#define ATTRIB(x)	((x)->attrib)
```

Finally, there is also a function that searches through the list of attributes
looking for a specific one:

```C
SEXP getAttrib(SEXP vec, SEXP name);
```

This can be used as follows:

```C
SEXP attrib1 = getAttrib(sexp, R_ClassSymbol);
SEXP attrib2 = getAttrib(sexp, install("customAttribute"));  // installs a string as a symbol
```

If no attribute is found this function returns `R_NilValue`.





#### Alternative representation

The vectors also have an *alternative representation* (*altrep*). This is a new
inclusion into R (3.5.0) meant to speed up the implementation and clean up the
interface. Currently, altrep vectors are created in very limited cases and I
don't have a lot of experience with them. For these reasons and given that this
adventure in writing is already a lot longer than I initially planned I will
leave altrep for later. In the meantime there are a number of sources out there
introducing altrep and otherwise discussing it. <sup>4</sup>

I will only mention that there is a header flag `alt` which is set to `1` if
the vector uses altrep and `0` otherwise. This flag can be looked up and
modified via the following macro:

```C
#define ALTREP(x)       ((x)->sxpinfo.alt)
#define SETALTREP(x, v) (((x)->sxpinfo.alt) = (v))
```
### Environments

Environments are structures that store bindings from variables (`SYMSXP`s) to
values. They are expressed by `ENVSXP`s. Their payload can be accessed by the
`envsxp` field of the payload union, which is specified by `envsxp_struct`.

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│   frame    │   enclos   │  hashtab   │
│            │            │            │
└─[LISTSXP]──┴──[ENVSXP]──┴──[VECSXP]──┘
```

The `enclos` slot points to the enclosing environment. Functions create new
environments where the `enclos` slot will point to the enclosing environment of
the function. This creates trees of environments. There is an empty environment
defined to be used as roots of environment trees as `R_EmptyEnv`. `R_EmptyEnv`
is defined so that its `frame`, `enclos`, and `hashtab` are all set to
`R_NilValue`.

The slots of the environments can be accessed by the following macros:

```C
#define FRAME(x)	((x)->u.envsxp.frame)
#define ENCLOS(x)	((x)->u.envsxp.enclos)
#define HASHTAB(x)	((x)->u.envsxp.hashtab)
```

There are two basic types of environments. 

#### Hash table environments

Usually environments store bindings in a hash table. An environment's hash
table is implemented using a vector in the `hashtab` slot. In that case the
`frame` slot points to `R_NilValue`.

The `VECSXP` implementing the hash table is a generic vector containing SEXPs.
The vector uses `length` to indicate the allocated size of the hash table and
`truelength` as the number of elements that are currently in use. An element
that is not in use points to a `NILSXP`. An element that is in use points to a
pairlist (`LISTSXP`) containing bindings.

Before looking at how exactly bindings are stored in the vector let us first
take a look at how hashes are calculated. Every binding contains a symbol and a
value.  The symbol is a `SYMSXP`. A symbol's hash is calculated using the
`CHARSXP` that is the symbol's `pname`. The `CHARSXP` is rendered as a C
character string and then the hashing is performed on that string using the
algorithm from the Dragon Book:

```C
int attribute_hidden R_Newhashpjw(const char *s)
{
    char *p;
    unsigned h = 0, g;
    for (p = (char *) s; *p; p++) {
	h = (h << 4) + (*p);
	if ((g = h & 0xf0000000) != 0) {
	    h = h ^ (g >> 24);
	    h = h ^ g;
	}
    }
    return h;
}
```

`R_Newhashpjw` yields an integer hash which is stored inside the `CHARSXP` as
`truelength`. This is shown in the example below. In addition, the
`HASHASH_FLAG` is set in the `gp` field of the SEXP's header.

```
┌────────────┬────────────┬────────────┐
│   pname    │    value   │  external  │
│    "x"     │ R_NilValue │ R_NilValue │
│            │            │            │
└─[CHARSXP]──┴──[NILSXP]──┴──[NILSXP]──┘
      │                 
 ┌────┘            ┌───────────────────────────────── hash (from R_Newhashpjw)
 │                 │
 v                 v               CHAR:
┌────────────┬────────────┬─-----─┬────────────┐
│   length   │ truelength │       │    [0]     │
│     1      │    121     │       │    'x'     │
│            │            │       │            │  
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴───[int]────┘
```

In order to place a binding of some value to some symbol, we create a new  SEXP
of type `LISTSXP` whose `tagval` will point to the SEXP representing the symbol
and whose `carval` will point to the SEXP representing the value. For instance:

```
┌────────────┬────────────┬────────────┐ 
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
```

This binding will become a part of a `chain` of bindings stored as one of the
elements of the `hashtab` vector. The position in the vector is determined by a
`hashcode` which we calculate by modulo dividing the `hash` of the symbol in
the binding by the length of `hashtab`. By default `hashtab` length is 29
(resized by a factor of 1.2 whenever `truelength` grows to be 85% or more of
`length`).

So, for the example above, the `hashcode` is `121 % 29 = 4`. This means that in
a 0-indexed vector, we would index the vector with 4 to get the `chain` that's
appropriate for this binding.  <!--However, since R vectors are 1-indexed,-->
The appropriate chain is therefore located at index 5 (`VECTOR_ELT(table, 5)`).

The position in `hashtab` where we want to insert our binding will point to
some existing `chain` which will be either a `NILSXP` or a `LISTSXP`. Our
binding will be pre-pended to the existing `chain`: the position in `hashtab`
will now point to our binding, and our binding's `cdrval` will point to the old
`chain`.

Let's go through an example. First let's create an environment. We use
`new.env` to do that. If we don't specify any parameters, we will get a
hash table environment with the initial size of 29. The enclosing environment is
going to be the global environment: `R_GlobalEnv`.

```R
e <- new.env()      # defaults: hash=TRUE, size=29
```

This structure can be illustrated as follows:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│ R_NilValue │ R_Global.. │            │
│            │            │            │
└──[NILSXP]──┴──[ENVSXP]──┴──[VECSXP]──┘
                                │ 
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┬─---
│   length   │ truelength │   align    │
│     29     │     0      │            │
│            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
 
 VECTOR_ELT:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬─-----─┬────────────┐
│    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │       │    [28]    │
│ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │       │ R_NilValue │
│            │            │            │            │            │            │            │       │            │
└──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴─-----─┴──[NILSXP]──┘
```

The structure is also reflected under inspection:

```R
> .Internal(inspect(e))
@555558bee978 04 ENVSXP g0c0 [NAM(3)] <0x555558bee978>
ENCLOS:
  @555555c4bc60 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
HASHTAB:
  @555558b4a4f0 19 VECSXP g0c7 [] (len=29, tl=0)
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
```

We subsequently add two bindings to the environment:

``` R
e$x <- 1 
e$y <- 2
```

The hashes of `x` and `y` respectively are `120` and `121`, which for a
29-element hash table will produce hash codes of `120 % 29 = 4` and `121 % 29 =
5`. This means that `x` and `y` will be placed in `chain`s located at positions
4 and 5 in the `hashtab` vector:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│ R_NilValue │ R_Global.. │            │
│            │            │            │
└──[NILSXP]──┴──[ENVSXP]──┴──[VECSXP]──┘
                                │ 
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┬─---
│   length   │ truelength │   align    │
│     29     │     2      │            │
│            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
 
 VECTOR_ELT:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬─-----─┬────────────┐
│    (1)     │    (2)     │    (3)     │    (4)     │    (5)     │    (6)     │    (7)     │       │    (29)    │
│ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │            │            │ R_NilValue │       │ R_NilValue │
│            │            │            │            │            │            │            │       │            │
└──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴─[LISTSXP]──┴─[LISTSXP]──┴──[NILSXP]──┴─-----─┴──[NILSXP]──┘
                                                          │            │
 ┌────────────────────────────────────────────────────────┘            │
 │                                                   ┌─────────────────┘
 v                                                   v
┌────────────┬────────────┬────────────┐            ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │            │   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │ R_NilValue │            │     2      │    `y`     │ R_NilValue │
│            │            │            │            │            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘            └──[REALSXP]─┴──[SYMSXP]──┴──[NILSXP]──┘ 
```

The inspector will show the new binding for `x` but not the one for `y`, since it is truncated:

```R
> .Internal(inspect(e))
@555558bee978 04 ENVSXP g0c0 [NAM(3)] <0x555558bee978>
ENCLOS:
  @555555c4bc60 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
HASHTAB:
  @555558b4a4f0 19 VECSXP g0c7 [] (len=29, tl=3)
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555558bef008 02 LISTSXP g0c0 [] 
      TAG: @555555c956d0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
      @555558ac5130 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
```

Finally, let's add one more binding for `dt`:

```R
e$dt <- 3
```

The hash of `dt` is `1716`, which means its hashcode is `1716 % 29 = 5`, the same
as for `y`. This means that the binding for `dt` will be pre-pended to the chain
at location `5` in `hashtab`:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│ R_NilValue │ R_Global.. │            │
│            │            │            │
└──[NILSXP]──┴──[ENVSXP]──┴──[VECSXP]──┘
                                │ 
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┬─---
│   length   │ truelength │   align    │
│     29     │     3      │            │
│            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
 
 VECTOR_ELT:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬─-----─┬────────────┐
│    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │       │    [28]    │
│ R_NilValue │ R_NilValue │ R_NilValue │ R_NilValue │            │            │ R_NilValue │       │ R_NilValue │
│            │            │            │            │            │            │            │       │            │
└──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴──[NILSXP]──┴─[LISTSXP]──┴─[LISTSXP]──┴──[NILSXP]──┴─-----─┴──[NILSXP]──┘
                                                          │            │
 ┌────────────────────────────────────────────────────────┘            │
 │                                                   ┌─────────────────┘
 v                                                   v
┌────────────┬────────────┬────────────┐            ┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │            │   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │ R_NilValue │            │     3      │    `dt`    │            │
│            │            │            │            │            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘            └──[REALSXP]─┴──[SYMSXP]──┴─[LISTSXP]──┘ 
                                                                                    │
                                                     ┌──────────────────────────────┘
                                                     v
                                                    ┌────────────┬────────────┬────────────┐
                                                    │   carval   │   tagval   │   cdrval   │
                                                    │     2      │    `y`     │ R_NilValue │
                                                    │            │            │            │
                                                    └─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
```

The inspector show as follows:

```R
> .Internal(inspect(e))
@555558bee978 04 ENVSXP g0c0 [NAM(3)] <0x555558bee978>
ENCLOS:
  @555555c4bc60 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
HASHTAB:
  @555558b4a4f0 19 VECSXP g0c7 [] (len=29, tl=3)
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c189e0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555558bef008 02 LISTSXP g0c0 [] 
      TAG: @555555c956d0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
      @555558ac5130 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
```

Note that `truelength` increased to 3.

Some useful macros for hash table environments. The following can be used to
retrieve the `hashtab` vector and information about it:

```C
#define IS_HASHED(x)            (HASHTAB(x) != R_NilValue)
#define HASHTAB(x)              ((x)->u.envsxp.hashtab)
#define HASHSIZE(x)             ((int) STDVEC_LENGTH(x))
#define HASHPRI(x)              ((int) STDVEC_TRUELENGTH(x))
#define HASHTABLEGROWTHRATE     1.2
#define HASHMINSIZE             29
```

These macros can be used to retrieve chains and bindings from a hashtab
(including cases of active bindings which are out of scope for this
already lengthy discussion):

```C
#define HASHCHAIN(table, i)     ((SEXP *) STDVEC_DATAPTR(table))[i]
#define BINDING_VALUE(b)        ((IS_ACTIVE_BINDING(b) ? getActiveValue(CAR(b)) : CAR(b)))
#define SYMBOL_BINDING_VALUE(s) ((IS_ACTIVE_BINDING(s) ? getActiveValue(SYMVALUE(s)) : SYMVALUE(s)))
#define SYMBOL_HAS_BINDING(s)   (IS_ACTIVE_BINDING(s) || (SYMVALUE(s) != R_UnboundValue))
```

Finally, these macros can be used to retrieve and manipulate hashes of
individual SEXPs:

```C
#define HASHASH_MASK 1
#define HASHASH(x)         ((x)->sxpinfo.gp & HASHASH_MASK)
#define HASHVALUE(x)       ((int) TRUELENGTH(x))
#define SET_HASHASH(x,v)   ((v) ? (((x)->sxpinfo.gp) |= HASHASH_MASK) : (((x)->sxpinfo.gp) &= (~HASHASH_MASK)))
#define SET_HASHVALUE(x,v) SET_TRUELENGTH(x, ((int) (v)))
```

<!--
e <- new.env()      # defaults: hash=TRUE, size=29
e$x  <- 1           # hash of `x`  is  120 % 29 = 4 -> lives at VECTOR_ELT(5)
e$ai <- 2           # hash of `ai` is 1657 % 29 = 4 -> lives at VECTOR_ELT(5)
e$y  <- 2           # hash of `y`  is  121 % 29 = 5 -> lives at VECTOR_ELT(6)
e$dt <- 3           # hash of `dt` is 1716 % 29 = 5 -> lives at VECTOR_ELT(6)
-->

#### List environments

List environments are environments that use internal lists for storing binding
rather than hash tables. Such environments will ave their `hashtab` slot
pointing to `R_NilValue` and their `frame` slot pointing to either
a`R_NilValue` (signifying an empty environment) or a pairlist (`LISTSXP`). Such
a pairlist would contain symbols under `tagval`and values under `carval`.

List environments are fairly straightforward. We'll follow a simple example of
creating and populating one:

```R
e <- new.env(hash=FALSE)
```

This creates an empty environment with both `hashtab` and `frame` pointing to
`R_NilValue`:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│ R_NilValue │ R_Global.. │ R_NilValue │
│            │            │            │
└──[NILSXP]──┴──[ENVSXP]──┴──[NILSXP]──┘
```

We subsequently add a binding to the environment:

```R
e$x <- 1
```

As with hash table environments, in order to place a binding of some value to
some symbol, we create a new SEXP of type `LISTSXP` whose `tagval` will point
to the SEXP representing the symbol and whose `carval` will point to the SEXP
representing the value. Our binding will be pre-pended to the existing `frame`:
`frame` will now point to our binding, and our binding's `cdrval` will point to
the old `frame`:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│            │ R_Global.. │ R_NilValue │
│            │            │            │
└─[LISTSXP]──┴──[ENVSXP]──┴──[NILSXP]──┘
      │
 ┌────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
```

If we then add another binding:

```R
e$y <- 2
```

That binding will also be pre-pended to `frame`:

```
┌────────────┬────────────┬────────────┐
│   frame    │   enclos   │   hashtab  │
│            │ R_Global.. │ R_NilValue │
│            │            │            │
└─[LISTSXP]──┴──[ENVSXP]──┴──[NILSXP]──┘
      │
 ┌────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     2      │    `y`     │            │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘
```

We can inspect our list environment to yield the following:

```R
> .Internal(inspect(e))
@55555711cbf0 04 ENVSXP g0c0 [NAM(1)] <0x55555711cbf0>
FRAME:
  @555557112870 02 LISTSXP g0c0 [] 
    TAG: @555555d812b8 01 SYMSXP g1c0 [MARK] "y"
    @55555a3bd050 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 2
    TAG: @555555c956d0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @55555a3bd130 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
ENCLOS:
  @555555c4bc60 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
```

### Functions

From the point of view of SEXPs, there are three types of functions found in R:
closures, built-ins, specials. Closures are lazy functions defined in R (eg.
`print`). Built-ins are strict functions defined in C or other languages (eg.
`+`). Specials are functions defined in C whose arguments are not interpreted,
but passed in as ASTs (eg. `if`). They are collectively grouped under the
synthetic type `FUNSXP`.

#### Closures

A closure is represented by a `CLOSXP`. The payload of a closure's SEXP is
specified by a `closxp_struct`, which contains pointers to three other SEXPs:
the closure's formals (argument definitions), its body, and its enclosing
environment: 

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│  formals   │    body    │    env     │
│            │            │            │
└─[LISTSXP]──┴─[LANGSXP]──┴──[ENVSXP]──┘
```

The formals are a list of formal arguments that the function accepts expressed
as a pairlist (`LISTSXP`). If the function has no formal arguments, `formals`
points to `R_NilValue`. If there are formal arguments, there is a list where
the `tagval`s point to symbols representing the names of arguments, and
`carval`s point to their values, if the arguments have default values. In
arguments that do not have values, `carval` is set to `R_UnboundValue`.  Let us
illustrate with the following example:

```R
> f <- function(x, y, z=1) x + y + z
```

Its formals are structured as follows:

```
┌────────────┬────────────┬────────────┐
│  formals   │    body    │    env     │
│            │ x + y + z  │ R_Global.. │
│            │            │            │
└─[LISTSXP]──┴─[LANGSXP]──┴──[ENVSXP]──┘
      │
 ┌────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│ R_Unboun.. │    `x`     │            │
│            │            │            │
└──[SYMSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│ R_Unboun.. │    `y`     │            │
│            │            │            │
└──[SYMSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `z`     │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘

```

The body is an AST representing the body of the function. The enclosing
environment is the environment in which the function operates. These follow the
structure we have laid out earlier.

All three elements can be accessed in R via their eponymous functions:

```R
> f <- function(x) x + 1
> formals(f)
$x
> body(f)
x + 1
> environment(f)
<environment: R_GlobalEnv>
```

They can also be accessed via macros in C:

```C
#define FORMALS(x)	((x)->u.closxp.formals)
#define BODY(x)		((x)->u.closxp.body)
#define CLOENV(x)	((x)->u.closxp.env)
```

Finally, we can inspect a closure to get some useful information about its constituent elements:

```R
> .Internal(inspect(f))
@555634d373b8 03 CLOSXP g0c0 [NAM(1),ATT] 
FORMALS:
  @555634d36cb8 02 LISTSXP g0c0 [] 
    TAG: @555634aea660 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @555634a6d890 01 SYMSXP g1c0 [MARK] "" (has value)
    TAG: @555634bd6008 01 SYMSXP g1c0 [MARK] "y"
    @555634a6d890 01 SYMSXP g1c0 [MARK] "" (has value)
    TAG: @555634bbaca8 01 SYMSXP g1c0 [MARK,NAM(3)] "z"
    @555639318558 14 REALSXP g0c1 [] (len=1, tl=0) 1
BODY:
  @555634d36e78 06 LANGSXP g0c0 [] 
    @555634a78960 01 SYMSXP g1c0 [MARK,LCK,gp=0x5000] "+" (has value)
    @555634d36dd0 06 LANGSXP g0c0 [] 
      @555634a78960 01 SYMSXP g1c0 [MARK,LCK,gp=0x5000] "+" (has value)
      @555634aea660 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
      @555634bd6008 01 SYMSXP g1c0 [MARK] "y"
    @555634bbaca8 01 SYMSXP g1c0 [MARK,NAM(3)] "z"
CLOENV:
  @555634aa0bf0 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
ATTRIB:
  @555634d37428 02 LISTSXP g0c0 [] 
    TAG: @555634a6cb00 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "srcref" (has value)
    @555639307758 13 INTSXP g0c3 [OBJ,ATT] (len=8, tl=0) 1,6,1,34,6,...
    ATTRIB:
      @555634d36f58 02 LISTSXP g0c0 [] 
	TAG: @555634a6cb70 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
	@555634d36ba0 04 ENVSXP g0c0 [OBJ,ATT] <0x555634d36ba0>
	FRAME:
	  @555634d37230 02 LISTSXP g0c0 [] 
	    TAG: @555635131d08 01 SYMSXP g1c0 [MARK] "lines"
	    @555639318478 16 STRSXP g0c1 [] (len=1, tl=0)
	      @55563930bca8 09 CHARSXP g0c4 [gp=0x60,ATT] [ASCII] [cached] "f <- function(x, y, z=1) x + y + z
"
	    TAG: @55563513f3a0 01 SYMSXP g1c0 [MARK] "filename"
	    @5556393184b0 16 STRSXP g0c1 [] (len=1, tl=0)
	      @555634a6f7e8 09 CHARSXP g1c1 [MARK,gp=0x60] [ASCII] [cached] ""
	ENCLOS:
	  @555634a6d938 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
	ATTRIB:
	  @555634d37268 02 LISTSXP g0c0 [] 
	    TAG: @555634a6d660 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	    @555639309e48 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	      @555634b29848 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
	      @555634a6f2e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
	TAG: @555634a6d660 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "class" (has value)
	@555639318520 16 STRSXP g0c1 [] (len=1, tl=0)
	  @555634a6f2a8 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
```

#### Built-ins and specials

Built-ins are represented by SEXPs of type `BUILTINSXP` and specials by
`SPECIALSXP`. These both have a very simple structure described by the
`primsxp_struct` structure accessible via the `primsxp` field of the payload
structure. Unlike the other payloads, `primsxp_struct` contains only a single
integer:

```C
struct primsxp_struct {
    int offset;
};
```

The R interpreter has a table specifying internal and primitive functions. It
is defined in `names.c` as `R_FunTab`. The `offset` field in `primsxp_struct`
is an index for that table. This, given a `BUILTINSXP` or a `SPECIALSXP`, we
can get more information about the function it describes by reaching for its
offset and retrieving the element specified by that offset from `R_FunTab`:

```C
SEXP s;
int index = s->u.primsxp.offset;
FUNTAB function_info = R_FunTab[index];
```

The information about a function is described by the `FUNTAB` structure:
```C
typedef struct {
    char   *name;    /* print name */
    CCODE  cfun;     /* c-code address */
    int	   code;     /* offset within c-code */
    int	   eval;     /* evaluate args? */
    int	   arity;    /* function arity */
    PPinfo gram;     /* pretty-print info */
} FUNTAB;
```

The `name` and `arity` of the function are self-explanatory. The other fields
require some interpretation.  The `cfun` field contains the address of the
function. The `CCODE` type is a pointer to a four argument function defined as
follows:

```C
typedef SEXP (*CCODE)(SEXP,  /* call expression     -- LANGSXP */
                      SEXP,  /* function expression -- CLOSXP  */
                      SEXP,  /* argument list       -- LISTSXP */
                      SEXP); /* environment (rho)   -- ENSXP   */
```

Further, the `code` variable specifies the variant of the function which should
be used. For instance, the operators `<-`, `<<-`, and `=` are all defined
within the single function `do_set`, but their `code` values are 1, 2, and 3,
respectively. This is used within the `do_set` function, as follows:

```C
SEXP attribute_hidden do_set(SEXP call, SEXP op, SEXP args, SEXP rho)
{
    /* ... */

    rhs = eval(CADR(args), rho);
    INCREMENT_NAMED(rhs);
    if (PRIMVAL(op) == 2)                           /* <<- */
        setVar(lhs, rhs, ENCLOS(rho));              /* define value in enclosing environment */
    else                                            /* <-, = */
        defineVar(lhs, rhs, rho);                   /* define value in current environment */
    R_Visible = FALSE;
    return rhs;
    
   /* ... */
}
```

<!--
{"<-",		do_set,		1,	100,	-1,	{PP_ASSIGN,  PREC_LEFT,	  1}},
{"=",		do_set,		3,	100,	-1,	{PP_ASSIGN,  PREC_EQ,	  1}},
{"<<-",		do_set,		2,	100,	-1,	{PP_ASSIGN2, PREC_LEFT,	  1}},
-->

The `eval` field contains the information about how the function and its
arguments are supposed to be evaluated. This single three digit integer and
each digit contains one separate piece of information, summarized as follows:

```
            ┌───────────────────────────────────── 1: force R_Visible off
            │                                      0: force R_Visible on
            │                                      2: switch R_Visible on but
            │                                         allow C to override this
            │
            │            ┌──────────────────────── 1: internal
            │            │
            │            │            ┌─────────── 1: evaluate arguments
            │            │            │            0: do not evaluate arguments
            v            v            v
      ┌────────────┬────────────┬────────────┐
eval: │     X      │     Y      │     Z      │
      └────────────┴────────────┴────────────┘
```

If the least significant digit is 1, then the arguments are evaluated before
calling the function (ie. built-in semantics). Alternatively, if the least
significant digit is 0, the arguments are passed without evaluating (ie.
special semantics). If the middle digit is 1, then the function must be
executed via the `.Internal` function in R. The most significant digit can
either be 0, 1, or 2. It specifies whether `R_Visible` should be turned on or
off for this function. If `R_Visible` is turned on, the function prints its
result if called from the interpreter. If it is turned off, the function's
result is not printed on the interpreter. If the most significant digit is set
to 1, the `R_Invisible` is forced on, and for 0 it is forced off. If the digit
is 2, then `R_Invisible` is turned on, but it can subsequently be modified by C
code.

<!--
The structure contains the name of the function and its location in memory. It
also has the information about whether its arguments should be evaluated before
executing it, and its arity. The pretty-print info offers information about the
type of deparsing that should be used, the precedence of the function, and its
associativity.
-->

Last field! `gram`! It is a structure that is used to guide the pretty printer:

```C
typedef struct {
    PPkind kind; 	     /* deparse kind */
    PPprec precedence;       /* operator precedence */
    unsigned int rightassoc; /* right associative? */
} PPinfo;
```

`PPkind` and `PPprec` are enums which specify different kinds of expressions
(eg. function, binary call, if statement) and different precedence values (17
specific precedence levels). The `rightassoc` is 1 if the expression associates
to the right, and 0 if it does not. 

As an example, here are some rows from the primitives and internals table:

```C
{"if",    do_if,     0,      200, -1, {PP_IF,      PREC_FN,  1}},
{"+",     do_arith,  1,      1,   -1, {PP_BINARY,  PREC_SUM, 0}},
{"length",do_length, 0,      1,    1, {PP_FUNCALL, PREC_FN,  0}},
```

The inspector does not really provide any information apart from the most
rudimentary:

```R
> .Internal(inspect(`+`))
@555634a78998 08 BUILTINSXP g1c0 [MARK,NAM(3)] 
> .Internal(inspect(`if`))
@555634a6c0b8 07 SPECIALSXP g1c0 [MARK,NAM(3)]
```

There are macros that provide an easy way to access both the value of the
offset for a given `BUILTINSXP` or `SPECIALSXP`, as well as to access the
fields of the `FUNTAB` structure, including some interpretation:

```C
#define PRIMOFFSET(x)                ((x)->u.primsxp.offset)
#define SET_PRIMOFFSET(x,v)         (((x)->u.primsxp.offset)=(v))
#define PRIMFUN(x)          (R_FunTab[(x)->u.primsxp.offset].cfun)
#define PRIMNAME(x)         (R_FunTab[(x)->u.primsxp.offset].name)
#define PRIMVAL(x)          (R_FunTab[(x)->u.primsxp.offset].code)
#define PRIMARITY(x)        (R_FunTab[(x)->u.primsxp.offset].arity)
#define PPINFO(x)           (R_FunTab[(x)->u.primsxp.offset].gram)
#define PRIMPRINT(x)      (((R_FunTab[(x)->u.primsxp.offset].eval)/100)%10)
#define PRIMINTERNAL(x)   (((R_FunTab[(x)->u.primsxp.offset].eval)%100)/10)
```

### Promises

When arguments are passed to functions the argument expression is usually
wrapped in a promise. This gives R its lazy semantics. Despite being created
whenever most functions are called, promises are generally invisible to R
programmers, and some do not even know they exist. Structure-wise promises are
specified by SEXPs of type `PROMSXP` and their payload is described by
`promsxp_struct` accessible via the `promsxp` field of the payload union. The
`promsxp_struct` structure has three slots: `value`, `expr`, and `env`:

```
┌────────────┬────────────┬────────────┐
│            │            │            │
│   value    │    expr    │    env     │
│            │            │            │
└───[SEXP]───┴───[SEXP]───┴──[ENVSXP]──┘
```

In the typical case, when a promise is created, it wraps some expression passed
to a function. The promise retains the original unevaluated expression in the
`expr` slot. It also retains the environment in `env` which should be used to
evaluate the expression in `expr`. In the case of expressions passed to some
function, `env` will be the enclosing environment of that function. In the case
of expressions defined as default arguments, `env` will be the environment
defined within the function. The `value` slot initially points to the
`R_UnboundValue` symbol. The following example shows an example of a promise at
creation:

```
┌────────────┬────────────┬────────────┐
│   value    │    expr    │    env     │
│ R_Unboun.. │   2 + 2    │ R_Global.. │
│            │            │            │
└──[SYMSXP]──┴─[LANGSXP]──┴──[ENVSXP]──┘
```

Whenever the interpreter requires the value of a promise, it performs a check
to see whether the promise has previously been seen by checking if its `value`
points to `R_UnboundValue`.  If that is the case, the promise is *forced*. This
means its `expr` is evaluated using the environment pointed to by `env` and the
result of the evaluation is stored in `value`. For the duration of forcing the
youngest bit of the `gp` field in the header is set (accessible by the
`PRSEEN`/`SET_PRSEEN` macros), to detect and prevent recursion. After the
promise finishes evaluating the expression, `env` is set to `R_NilValue`.

Forcing the example above will yield the following:

```
┌────────────┬────────────┬────────────┐
│   value    │    expr    │    env     │
│     4      │   2 + 2    │ R_NilValue │
│            │            │            │
└─[REALSXP]──┴─[LANGSXP]──┴──[NILSXP]──┘
```

A promise force is triggered whenever the argument holding the promise is
assigned or returned from the function. Promises can also be forced as a
result of being passed to another function: always in the case of a built-in.
When a promise is passed to a special function, it depends on the function
whether it will evaluate the promise or not. In the case of a being passed to a
closure it may be the case that the promise is wrapped in another promise and
then forced whenever the wrapper promise get forced (although it is my
understanding that these cases are being eliminated by R core devs in the
current version). 


Below is an example illustrating these cases

```R
f <- function(a, b, c, d, e) { 
    tmp <- a + 1                  # forces promise bound to `a`
    print(b)                      # forces promise bound to `b` before entering print
    c                             # forces promise bound to `c`
    g(d)                          # forces promise bound to `d` inside `g` 
}                                 # promise bound to `e` is not forced
g <- function(x) {
    x                             # forces promise bound to `x`: this will force any nested promises inside `x`
}
```

Apart from the typical case we also see promises that are created pre-forced.
In that case, the `value` is set to something else than `R_UnboundValue` from
the outset, and other fields may or may not be forced as required by the use
case.

#### Scrutinizing promises

Inspecting promises is fairly difficult since they are not visible from the R
interpreter. What we need to do is to run the interpreter with the debugger and
stop it when it evaluates a function:

```sh
$ R --debugger=gdb
(gdb) r
```

Then when the interpreter hands you back control, press `CTRL+C` to interrupt
the interpreter and go back into the debugger to set a breakpoint on the
function that is responsible for executing a closure. (We do this now and not
earlier, because we don't want to deal with all the function calls that happen
at startup).

```GDB
>^C 
Thread 1 "R" received signal SIGINT, Interrupt.
0x00007ffff619903f in __GI___select (nfds=1, readfds=0x555555c030e0 <readMask.14046>, writefds=0x0, exceptfds=0x0, timeout=0x0) at ../sysdeps/unix/sysv/linux/select.c:41
41	../sysdeps/unix/sysv/linux/select.c: No such file or directory.
(gdb) break R_execClosure
(gdb) c
Continuing.
```

(My line numbers will not fit yours, since I'm using a modified development
version of the interpreter for this example.)

Then define a suitable function:

```R
> f <- function(x, y) x + y
> f(1,2)
```

This should trigger a breakpoint:

```GDB
Thread 1 "R" hit Breakpoint 1, R_execClosure (call=0x555559504310, newrho=0x555559504578, sysparent=0x555555c4c780, rho=0x555555c4c780, arglist=0x555559504460, op=0x555559507f10)
    at eval.c:1733
1733	{
(gdb)
```

From this position we can inspect SEXPs by address as follows:

```GDB
(gdb) print R_inspect((SEXP) 0x555559504310)
@555559504310 06 LANGSXP g0c0 [] 
  @555555d98418 01 SYMSXP g1c0 [MARK] "f"
  @555558af4b78 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
  @555558af4bb0 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 2
$1 = (struct SEXPREC *) 0x555559504310
```

Now, when we inspect the argument list `arglist` at `0x555559504460` we will
find that it contains two promises:

```GDB
(gdb) print R_inspect((SEXP) 0x555559504460)
@555559504460 02 LISTSXP g0c0 [gp=0x1] 
  @555559504428 05 PROMSXP g0c0 [] 
  @555559504498 05 PROMSXP g0c0 [] 
$2 = (struct SEXPREC *) 0x555559504460
```

We can then try inspecting one of those promises, but it doesn't yield any more
information:

```GDB
(gdb) p R_inspect((SEXP) 0x555559504428)
@555559504428 05 PROMSXP g0c0 [] 
$3 = (struct SEXPREC *) 0x555559504428
```

(We can conveniently shorten `print` to `p`.)

But we can use our knowledge of a promise's structure to take a look at its
constituent components anyway:

```GDB
(gdb) p ((struct SEXPREC *) 0x555559504428)->u.promsxp
$4 = {value = 0x555555c19458, expr = 0x555558af4b78, env = 0x555555c4c780}
```

Let us inspect these components:

```GDB
(gdb) p R_inspect($.expr)
@555558af4b78 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
$5 = (struct SEXPREC *) 0x555558af4b78
(gdb) p R_inspect($4.env)
@555555c4c780 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
ENCLOS:
  @5555595c3428 04 ENVSXP g0c0 [NAM(3),LCK,GL,gp=0xc000,ATT] <package:stats>
  ATTRIB:
    @5555595c3460 02 LISTSXP g0c0 [] 
HASHTAB:
  @555555c4fc10 19 VECSXP g1c7 [MARK] (len=29, tl=4)
    @555555c19500 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c19500 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c19500 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c19500 00 NILSXP g1c0 [MARK,NAM(3)] 
    @5555573b8c60 02 LISTSXP g1c0 [MARK] 
      TAG: @555555c961f0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
      @555556f93ed0 14 REALSXP g1c1 [MARK] (len=1, tl=0) 1
    ...
$6 = (struct SEXPREC *) 0x555559504428
```

We can also look at the value, but since it's `R_UnboundValue` which doesn't
define all of its fields, `R_inspect` will explode, so let's not. Well, I will,
but you shouldn't.

```GDB
(gdb) p $4.value == R_UnboundValue
$7 = 1
(gdb) p R_inspect($4.value)
@555555c19458 01 SYMSXP g1c0 [MARK,NAM(3)] Error: 'getCharCE' must be called on a CHARSXP
$8 = 0x555558af4b78
```

We can try forcing the promise:

```GDB
(gdb) p forcePromise(0x555559504428)
$9 = (struct SEXPREC *) 0x555558af4b78
(gdb) p R_inspect($9)
@555558af4b78 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
```

The force returns a value that resulted from executing `expr`. It also causes
the `value` field of the promise to be set and `env` to be set to `R_NilValue`.

```GDB
(gdb) p $4->u.promsxp
$10 = {value = 0x555558af4b78, expr = 0x555558af4b78, env = 0x555555c19500}
(gdb) p $.value
@555558af4b78 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
$11 = (struct SEXPREC *) 0x555558af4b78
(gdb) p R_inspect($10.expr)
@555558af4b78 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 1
$12 = (struct SEXPREC *) 0x555558af4b78
(gdb) p R_inspect($10.env)
@555555c19500 00 NILSXP g1c0 [MARK,NAM(3)]
$13 = (struct SEXPREC *) 0x555555c19500
```

There are convenient macros facilitating retrieving the payload slots and
header information of promises:

```C
#define PRCODE(x)       ((x)->u.promsxp.expr)
#define PRENV(x)        ((x)->u.promsxp.env)
#define PRVALUE(x)      ((x)->u.promsxp.value)
#define PRSEEN(x)       ((x)->sxpinfo.gp)
#define SET_PRSEEN(x,v) (((x)->sxpinfo.gp)=(v))
```

### Dot-dot-dot

Since we have all those breakpoints set up, there is one specific type of SEXP
which is a list, but we really only see it in conjunction with promises, and
that's `DOTSXP`.  

To inspect it, let's make a function that takes variable arguments and returns
them via a vector, then let's run that function with some arguments:

```GDB
> f <- function(...) c(...)
> f(x=1, y=2, 3)
Thread 1 "R" hit Breakpoint 1, R_execClosure (call=0x55555950e080, newrho=0x55555950e3c8, sysparent=0x555555c4ef70, rho=0x555555c4ef70, arglist=0x55555950e1d0, op=0x55555950d9f0)
    at eval.c:1733
(gdb) 
```

At this point let us look at the formal parameters of our function:

```GDB
(gdb) p R_inspect(FORMALS(op))
@55555950d440 02 LISTSXP g0c0 [] 
  TAG: @555555c1b7b0 01 SYMSXP g1c0 [MARK,NAM(3)] "..."
  @555555c1bc10 01 SYMSXP g1c0 [MARK] "" (has value)
$14 = (struct SEXPREC *) 0x55555950d440
```

We see that the formals are a single-element list containing a pointer to a
symbol describing our variable argument list. This symbol is actually the
predefined singleton `R_DotsSymbol`, by the way. We can get the values passed
as arguments by looking up the bindings of symbols from formals in the
function's environment `newrho`. We can do this using `findVar`, although one
should always be careful using that function, since it can potentially have
side effects. So, let us instead do it the old-fashioned way, by inspecting the
environment. 

```GDB
(gdb) p R_inspect(newrho)
@55555950e3c8 04 ENVSXP g0c0 [gp=0x1000] <0x55555950e3c8>
FRAME:
  @55555950e2e8 02 LISTSXP g0c0 [] 
    TAG: @555555c1b7b0 01 SYMSXP g1c0 [MARK,NAM(3)] "..."
    @55555950e390 17 DOTSXP g0c0 [] 
ENCLOS:
  @555555c4ef70 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
$15 = (struct SEXPREC *) 0x55555950e3c8
```

And finally, we have come across a `DOTSXP` list, which is helpfully rendered
by the inspector with no information whatsoever:

```GDB
(gdb) p R_inspect(0x55555950e390)
@55555950e390 17 DOTSXP g0c0 [] 
$16 = (struct SEXPREC *) 0x55555950e390
```

Well, it is a sub-type of `LISTSXP` that contains as `carval` promises that wrap
the arguments passed via `...` and as `tagval` (optionally) symbols that were
used to pass them. For our example the structure is the following boring list:

```
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     1      │    `x`     │            │
│            │            │            │
└─[PROMSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     2      │    `y`     │            │
│            │            │            │
└─[PROMSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘
                                │
 ┌──────────────────────────────┘
 v
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│     3      │            │ R_NilValue │
│            │            │            │
└─[PROMSXP]──┴──[NILSXP]──┴──[NILSXP]──┘
```

We can also delve in deeper with the inspector and get similar information:

```GDB
(gdb) p R_inspect($16->u.listsxp.carval)
@55555950e198 05 PROMSXP g0c0 [] 
$17 = (struct SEXPREC *) 0x55555950e198
(gdb) p R_inspect($20->u.listsxp.cdrval)
@55555950e358 02 LISTSXP g0c0 [] 
  TAG: @555555d84588 01 SYMSXP g1c0 [MARK] "y"
  @55555950e208 05 PROMSXP g0c0 [] 
  @55555950e278 05 PROMSXP g0c0 [] 
$18 = (struct SEXPREC *) 0x55555950e358
(gdb) p R_inspect($20->u.listsxp.tagval)
@555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
$19 = (struct SEXPREC *) 0x555555c989a0
```

### Byte code

R is an interpreted language, but the interpreter comes in two forms. There's
the AST interpreter which runs SEXPs as they are, and there's the the byte code
interpreter, which runs byte code. Byte code is generated from ASTs using the
byte code compiler. The compiler is a just-in-time compiler that is turned on by
default in the newest versions of R. Otherwise you can turn it on by setting
the environmental variable `R_ENABLE_JIT` to `3` before running R (and to `0`
to turn the JIT off and `1`--`2` for intermediate settings<sup>8</sup>). When
the JIT is running whenever you execute a function twice, it gets byte code
compiled before the second execution:

```R
> f <- function(x) x + 1
> f
function(x) x + 1
> f(1)
[1] 2
> f(1)
[1] 2
> f
function(x) x + 1
<bytecode: 0x55555ae04870>
```

We see that after the second execution the interpreter says the function
contains bytecode. We can also bytecode compile functions and arbitrary
expressions on demand using the `compiler` package. eg.:

```R
> library(compiler)
> f <- cmpfun(function(x) x + 1)
> f
function(x) x + 1
<bytecode: 0x55555b4d0c00>
> expr <- compile(42)
> expr
<bytecode: 0x5555573ff110>
```

Let us concentrate on expressions for now. Internally, a bytecode compiled
expression is represented by a SEXP of type `BCODESXP`. There is no dedicated
payload type that would represent bytecode expressions. Instead, we use
`listsxp_struct` to represent them, which we will access via the `listsxp`
field of the payload union. However, there are macros for accessing `BCODESXP`s
that more clearly specify the contents of the structure:

```C
#define BCODE_CODE(x)   CAR(x)
#define BCODE_EXPR(x)   TAG(x)
#define BCODE_CONSTS(x) CDR(x)
```

Thus we represent a `BCODESXP` as follows:

```
      ┌────────────────────────────────── BCODE_CODE(s)
      │            ┌───────────────────── BCODE_EXPR(s)
      │            │            ┌──────── BCODE_CONSTS(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[INTSXP]──┴──[NILSXP]──┴──[VECSXP]──┘
```

The `BCODE_CODE` slot points to a vector of operation codes and their
arguments. The `BCODE_CONSTS` slot points to a vector of constant expressions
describing various elements of the SEXP including usually: constants referenced
to by the expression prior to compilation (the AST), the operations' arguments,
the index of current expressions, and the source reference. 

The `BCODE_EXPR` slot looks like it was initially intended to store the AST of
the uncompiled expression, but this does not seem to be the case currently. The
byte code compiler (function `do_mkcode` in `eval.c` simply omits this field
when compiling an expression and the macro is not used anywhere in the code to
access it.

#### Simple example

Let us first take a look at both `BCODE_CODE` and `BCODE_CONSTS` and explain
how it works using a very simple example:

```R 
> expr <- compile(42)
> expr
<bytecode: 0x5555573ff110>
```

If we were to decompile `expr` using some pretty printer (alas I don't know of
any good ones) we would get the following:

```
LDCONST 42
RETURN
```

Let's first take a look at `BCODE_CODE` of this expression. We can represent it like this:

```
      ┌────────────────────────────────── BCODE_CODE(s)
      │            ┌───────────────────── BCODE_EXPR(s)
      │            │            ┌──────── BCODE_CONSTS(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[INTSXP]──┴──[NILSXP]──┴──[VECSXP]──┘
      │ 
 ┌────┘ 
 v     
┌────────────┬────────────┬────────────┬─---
│   length   │ truelength │   align    │
│     8      │     0      │            │ 
│            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---
 
 INTEGER_ELT:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
│    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │    [7]     │
│    10      │     0      │ 1432968343 │   21845    │     0      │     0      │ 1432961375 │   21845    │ 
│            │            │            │            │            │            │            │            │
└───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘
``` 

Let's try to make sense of this. It's a vector containing 8 elements which
somehow map to either operations or operations' arguments. This does not make
sense, since there are only 2 operations in our code, one of which has 1
argument. So we would expect there to be 3 elements not 8.

Well, the vector is a vector of type `INTSXP`, containing integers, but
operations and arguments are both expressed as the following union, where `v`
can point to an operation and `i` can be used to express an argument.

```C
typedef union { void *v; int i; } BCODE;
```

This union has a potentially different size than `int`, so every element in the
`INTSXP` vector can take up more than one element. The number of elements that
is used by a single byte code operation is defined as follows.

```C
int m = (sizeof(BCODE) + sizeof(int) - 1) / sizeof(int);
```

On my system this number is `2`, so each operation will take up two vector
elements. So, it's a lie that this vector should be read as an integer vector.
Instead, what we need to do, is to cast it to a BCODE vector:

```
 INTEGER_ELT:
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
│    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │    [7]     │
│    10      │     0      │ 1432968343 │   21845    │     0      │     0      │ 1432961375 │   21845    │ 
│            │            │            │            │            │            │            │            │
└───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘

 (BCODE *):
┌─────────────────────────┬─────────────────────────┬─────────────────────────┬─────────────────────────┐
│           [0]           │          [1]            │           [2]           │           [3]           │
│  { i = 10, v = 0x10 }   │  { i = 1432968343       │   { i = 0, v = 0x0 }    │  { i = 1432961375,      │ 
│                         │    v = 0x555555695c97 } │                         │    v = 0x55555569415f } │
└─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┘
```

This makes more sense, but we still have 4 elements. But the first element in
the `BCODE` vector is actually neither an operation nor an argument. Instead it
is the version of the byte code compiler that was used to compile this SEXP. In
our case, this is version `10`. The other three elements are the operations and
their arguments, as expected.

How do we decode these? There is an array of operations called `opinfo` which
contains the address of the operation, the number of arguments it takes, and
its printable name:

```C
volatile
static struct { void *addr; int argc; char *instname; } opinfo[OPCOUNT];
```

The `addr` field of the array corresponds to the `v` fields of `BCODE`
operations. So for each operation we can search through `opinfo` to find such
`addr` that equals `v` and thus figure out what operation it is. (There is a
function in `eval.c` that does it called `R_bcDecode`. It traverses the BCODE
vector looking up operations with the `findOp` function.) We can also read the
number of arguments from `argc` which points tells us how to treat subsequent
elements in the vector. In the case of the example a search through
`opinfo` would yield the following operations for element 1:

```C
{addr = 0x555555695c97 <bcEval+7996>, argc = 1, instname = 0x55555580bd12 "LDCONST"}
```

We see that this is the `LDCONST` (load constant) operation. It also tells us
that element 2 is an argument for that operation, and element 3 is another
operation, which is defined as follows:

```C
{addr = 0x55555569415f <bcEval+1028>, argc = 0, instname = 0x55555580bc60 "RETURN"}
```

Let us go back to the argument to LDCONST though. The argument is not a value,
but instead it is an index to `BCODE_CONSTS` that will yield the actual value
of the argument. Here, the argument's index is `0`. So let's take a look at
`BCODE_CONSTS`. 

```
      ┌────────────────────────────────── BCODE_CODE(s)
      │            ┌───────────────────── BCODE_EXPR(s)
      │            │            ┌──────── BCODE_CONSTS(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[INTSXP]──┴──[NILSXP]──┴──[VECSXP]──┘
                                │ 
 ┌──────────────────────────────┘
 v                                INTEGER_ELT:
 ┌────────────┬────────────┬─---─┬────────────┬────────────┐
 │   length   │ truelength │     │    [0]     │    [1]     │
 │     2      │     0      │     │    42      │            │
 │            │            │     │            │            │
 └─[R_xlen_t]─┴─[R_xlen_t]─┴─---─┴─[REALSXP]──┴──[INTSXP]──┘
                                                    │
 ┌──────────────────────────────────────────────────┘
 v                                INTEGER_ELT:
 ┌────────────┬────────────┬─---─┬────────────┬────────────┬────────────┬────────────┐
 │   length   │ truelength │     │    [0]     │    [1]     │    [2]     │    [3]     │
 │     4      │     0      │     │-2147483648 │     0      │     0      │     0      │
 │            │            │     │    NA      │            │            │            │
 └─[R_xlen_t]─┴─[R_xlen_t]─┴─---─┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘
```

In this example that vector is fairly simple: it only contains two entries. The
first entry (index `0`) of this vector is always the entire expression that
underwent compilation. So, if we wanted to 'decompile' a SEXP, we just need to
reach into `BCODE_CONSTS` and take out the first element to do it. In our case
this is a `REALSXP` vector containing the single element `42`. Apart from
representing the entire compiled expression in our simple example this element
is also pointed to by the argument of the `LDCONST` operation---the entire
expressions as well as the loaded constant are the same, and the byte-code
compiler tries not to duplicate entries in `BCODE_CONSTS`.

The second element in `BCODE_CONSTS` is an *expressions index*. This is an
integer vector that contains one entry per `(BCODE *)` element in `BCODE_CODE`.
In our example we had 4 elements in `BCODE_CODE`, so there are 4 elements in
the expressions index. Each element of this vector can be used to index
`BCODE_CONSTS` to map operations in `BCODE_CODE` to uncompiled expressions from
which they originated. This is trivial in the current example since we only
really have one expression in `BCODE_CONSTS` but we will show a more
complicated example of this further below. The first element is (always) the
integer value of `NA`, since the compiler version should not be mapped to a
particular expression.

In summary, the entire expression can be represented as follows:

```
      ┌────────────────────────────────── BCODE_CODE(s)
      │            ┌───────────────────── BCODE_EXPR(s)
      │            │            ┌──────── BCODE_CONSTS(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[INTSXP]──┴──[NILSXP]──┴──[VECSXP]──┘           
      │                         │                  ┌────────────────────────────────────────────────────────┐
 ┌────┘       ┌─────────────────┘                  ├──────────────────────────────────────┐                 │
 │            v                                    v                                      │                 │
 │           ┌────────────┬────────────┬─---─┬────────────┬────────────┐                  │                 │
 │           │   length   │ truelength │     │    [0]     │    [1]     │                  │                 │
 │           │     2      │     0      │     │    42      │            │                  │                 │
 │           │            │            │     │            │            │                  │                 │
 │           └─[R_xlen_t]─┴─[R_xlen_t]─┴─---─┴─[REALSXP]──┴──[INTSXP]──┘                  │                 │
 │                                                              │                         │                 │
 │            ┌─────────────────────────────────────────────────┘                         │                 │
 │            │                                                 ┌────────────┬────────────┤                 │
 │            v                               INTEGER:          │            │            │                 │
 │           ┌────────────┬────────────┬─---─┬────────────┬────────────┬────────────┬────────────┐          │
 │           │   length   │ truelength │     │    [0]     │    [1]     │    [2]     │    [3]     │          │
 │           │     4      │     0      │     │-2147483648 │     0      │     0      │     0      │          │
 │           │            │            │     │    NA      │            │            │            │          │
 │           └─[R_xlen_t]─┴─[R_xlen_t]─┴─---─┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘          │
 │                                                                                                          │
 │                                            ATTRIB: class="expressionsIndex"                              │
 v                                                                                                          │
┌────────────┬────────────┬────────────┬─---                                                                │
│   length   │ truelength │   align    │                                                                    │
│     8      │     0      │            │                                                                    │ 
│            │            │            │                                                                    │
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---                                                                │
                                                                                                            │
 INTEGER_ELT:                                                                                               │
┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐   │
│    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]     │    [6]     │    [7]     │   │
│    10      │     0      │ 1432968343 │   21845    │     0      │     0      │ 1432961375 │   21845    │   │
│            │            │            │            │            │            │            │            │   │
└───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┘   │
                                                                                                            │
 (BCODE *):                                                                                                 │
┌─────────────────────────┬─────────────────────────┬─────────────────────────┬─────────────────────────┐   │
│           [0]           │          [1]            │           [2]           │           [3]           │   │
│  { i = 10, v = 0x10 }   │  { i = 1432968343       │   { i = 0, v = 0x0 }    │  { i = 1432961375,      │   │ 
│                         │    v = 0x555555695c97 } │                         │    v = 0x55555569415f } │   │
└─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┘   │
                                       │                         │                         │                │
                                       │                         │                         └──────> RETURN  │  
                                       │                         └──────────────────────────────────────────┘
                                       └──────────────────────────────────────────────────────────> LDCONST
```

#### Inspection

If we inspect our bytecode expression we will not get anything worthwhile, just
the type of the SEXP:

```R
> .Internal(inspect(expr))
@5555573bb1a8 21 BCODESXP g1c0 [MARK,NAM(1)]
```

However, we can interrupt our session and open up the debugger to get further
information about the constituent bits of the SEXP. Let's take a look at the
`BCODE_CODE` vector and its elements then:

```GDB
> .Internal(inspect(expr))
@5555573bb1a8 21 BCODESXP g1c0 [MARK,NAM(1)] 
> ^C
Thread 1 "R" received signal SIGINT, Interrupt.
0x00007ffff619903f in __GI___select (nfds=1, readfds=0x555555c030e0 <readMask.14046>, writefds=0x0, exceptfds=0x0, timeout=0x0) at ../sysdeps/unix/sysv/linux/select.c:41
41	in ../sysdeps/unix/sysv/linux/select.c
(gdb) p R_inspect(((SEXP) 0x5555573bb1a8)->u.listsxp.carval) 
@555556194f08 13 INTSXP g1c3 [MARK] (len=8, tl=0) 10,0,1432968343,21845,0,...
$1 = (struct SEXPREC *) 0x555556194f08
(gdb) p INTEGER($1)[0]
$2 = 10
(gdb) p INTEGER($1)[1]
$3 = 0
(gdb) p INTEGER($1)[2]
$4 = 1432968343
(gdb) p INTEGER($1)[3]
$5 = 21845
(gdb) p INTEGER($1)[4]
$6 = 0
(gdb) p INTEGER($1)[5]
$7 = 0
(gdb) p INTEGER($1)[6]
$8 = 1432961375
(gdb) p INTEGER($1)[7]
$9 = 21845
(gdb) p ((BCODE *) INTEGER($1))[0]
$10 = {v = 0xa, i = 10}
(gdb) p ((BCODE *) INTEGER($1))[1]
$11 = {v = 0x555555695c97 <bcEval+7996>, i = 1432968343}
(gdb) p ((BCODE *) INTEGER($1))[2]
$12 = {v = 0x0, i = 0}
(gdb) p ((BCODE *) INTEGER($1))[3]
$13 = {v = 0x55555569415f <bcEval+1028>, i = 1432961375}
```

That's pretty obvious though. The `BCODE_CONSTS` vector is more interesting:

```GDB
(gdb) p R_inspect(((SEXP) 0x5555573bb1a8)->u.listsxp.cdrval) 
@555557130918 19 VECSXP g1c2 [MARK] (len=2, tl=0)
  @555556f966f0 14 REALSXP g1c1 [MARK] (len=1, tl=0) 3
  @5555571309d8 13 INTSXP g1c2 [OBJ,MARK,ATT] (len=4, tl=0) -2147483648,0,0,0
  ATTRIB:
    @5555573bb1e0 02 LISTSXP g1c0 [MARK] 
      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
      @555556f96760 16 STRSXP g1c1 [MARK] (len=1, tl=0)
	@555555cd5828 09 CHARSXP g1c3 [MARK,gp=0x60] [ASCII] [cached] "expressionsIndex"
$14 = (struct SEXPREC *) 0x555557130918
```

We can see the vector of length 2 containing one `REALSXP` representing the
expression and one `LISTSXP` used as an expressions index. We also see that the
expressions index has a `class` attribute set to `expressionsIndex`, allowing
us to identify it.

Inspecting `BCODESXP`s is a little cumbersome, but we can get some basic
information about the byte code by disassembling it:

```R
> disassemble(expr)
list(.Code, list(10L, LDCONST.OP, 0L, RETURN.OP), list(42, structure(c(NA,
0L, 0L, 0L), class = "expressionsIndex")))
```

This returns a simplified, slightly cleaned-up version of the expression. The
list contains a `.Code` symbol to indicate that this is byte-code, then a list
of operations expressing the contents of `BCODE_CODE`, and finally a list
representing `BCODE_CONSTS`.

#### A byte-code compile function example

Now that we have the details of `BCODESXP` down and a full toolkit, let's try
looking at a function.

```R
> f <- cmpfun( function(x, y) x + y )
> f
function(x, y) x + y
<bytecode: 0x55555b3b9130>
```

This function is still represented by a `CLOSXP` like all other user-defined R
functions. The only difference is that the body of the function is not a
`LANGSXP`, but instead it is a `BCODESXP`. The whole thing can be represented
as follows: 

```
┌────────────┬────────────┬────────────┐
│  formals   │    body    │    env     │
│            │            │ R_Global.. │
│            │            │            │
└─[LISTSXP]──┴─[BCODESXP]─┴──[ENVSXP]──┘
 ┌────┘            └───────────────────────┐
 v                                         │
┌────────────┬────────────┬────────────┐   │
│   carval   │   tagval   │   cdrval   │   │
│ R_Unboun.. │    `x`     │            │   │
│            │            │            │   │
└──[SYMSXP]──┴──[SYMSXP]──┴─[LISTSXP]──┘   │
 ┌──────────────────────────────┘          │
 v                                         │
┌────────────┬────────────┬────────────┐   │
│   carval   │   tagval   │   cdrval   │   │
│ R_Unboun.. │    `y`     │ R_NilValue │   │
│            │            │            │   │
└──[SYMSXP]──┴──[SYMSXP]──┴──[NILSXP]──┘   │
 ┌─────────────────────────────────────────┘
 │    ┌────────────────────────────────────── BCODE_CODE(s)
 │    │            ┌───────────────────────── BCODE_EXPR(s)
 │    │            │            ┌──────────── BCODE_CONSTS(s)
 v    v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[INTSXP]──┴──[NILSXP]──┴──[VECSXP]──┘   
 ┌────┘       ┌─────────────────┘                  
 │            v                                    
 │           ┌────────────┬────────────┬─---
 │           │   length   │ truelength │ 
 │           │     6      │     0      │  
 │           │            │            │   
 │           └─[R_xlen_t]─┴─[R_xlen_t]─┴─---
 │ 
 │            VECTOR_ELT:     
 │                 ┌────────────────────────────────────────────────────────────────────────────────────────────┐
 │                 │            ┌─────────────────────────────────────────────────────────────────────────────┐ │
 │                 │            │                         ┌───────────────────────────────────────────────────┼┐│
 │                 v            v                         v                                                   │││
 │           ┌────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐                  │││
 │           │    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │     [5]    │                  │││
 │           │   x + y    │     x      │ 1,13,1,32, │     y      │            │ NA,2,2,2   │                  │││
 │           │            │            │ 13,32,1,1  │            │            │  2,2,2,2   │                  │││
 │           └─[LANGSXP]──┴──[SYMSXP]──┴──[INTSXP]──┴──[SYMSXP]──┴──[INTSXP]──┴──[INTSXP]──┘                  │││
 │                                      ATTRIB:                        │       ATTRIB:                        │││
 │                                      class="srcref"                 │       class="srcref"                 │││
 │                                      srcfile=<R_GlobalEnv>          │                                      │││
 │   ┌─────────────────────────────────────────────────────────────────┘                                      │││
 │   v                                                                                                        │││
 │  ┌────────────┬────────────┬─---                                                                           │││
 │  │   length   │ truelength │                                                                               │││
 │  │     8      │     0      │                                                                               │││
 │  │            │            │                                                                               │││
 │  └─[R_xlen_t]─┴─[R_xlen_t]─┴─---                                                                           │││
 │                                                                                                            │││
 │   INTEGER:                                                                                                 │││
 │  ┌────────────┬────────────┬────────────┬────────────┬────────────┬───────────┬────────────┬────────────┐  │││
 │  │    [0]     │    [1]     │    [2]     │    [3]     │    [4]     │    [5]    │    [6]     │    [7]     │  │││
 │  │-2147483648 │     1      │     1      │     3      │     3      │     0     │     0      │     0      │  │││
 │  │    NA      │            │            │            │            │           │            │            │  │││
 │  └───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]────┴───[int]───┴───[int]────┴───[int]────┘  │││
 │                                                                                                            │││
 │   ATTRIB: class="expressionsIndex"                                                                         │││
 v                                                                                                            │││
┌────────────┬────────────┬────────────┬─---                                                                  │││
│   length   │ truelength │   align    │                                                                      │││
│     13     │     0      │            │                                                                      │││
│            │            │            │                                                                      │││
└─[R_xlen_t]─┴─[R_xlen_t]─┴────────────┴─---                                                                  │││
                                                                                                              │││
 (BCODE *):                                                                                                   │││
┌─────────────────────────┬─────────────────────────┬─────────────────────────┬─────────────────────────┐     │││
│           [0]           │          [1]            │           [2]           │           [3]           │     │││
│  { i = 10, v = 0x10 }   │  { i = 1432969285,      │   { i = 1, v = 0x1 }    │  { i = 1432969285,      │     │││
│                         │    v = 0x555555696045 } │                         │    v = 0x555555696045 } │     │││
└─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┘     │││
                                       │                         │                         └──────────> LDVAR │││
                                       │                         └────────────────────────────────────────────┘││
                                       └──────────────────────────────────────────────────────────────> LDVAR  ││
 (BCODE *) + 4:                                                                                                ││
┌─────────────────────────┬─────────────────────────┬─────────────────────────┬─────────────────────────┐      ││
│           [4]           │          [5]            │           [6]           │           [7]           │      ││
│   { i = 3, v = 0x3 }    │  { i = 1432987413,      │   { i = 0, v = 0x0 }    │  { i = 1432961375,      │      ││
│                         │    v = 0x55555569a715 } │                         │    v = 0x55555569415f } │      ││
└─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┴─────────[BCODE]─────────┘      ││
             │                         │                         │                         └──────────> RETURN ││
             │                         │                         └─────────────────────────────────────────────┘│
             │                         └──────────────────────────────────────────────────────────────> ADD     │
             └──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

The structure of the `BCODESXP` is the same as we've seen before, but there are
a few additional elements that show up in `BCODE_CONSTS`. Position `0` still
holds the entire uncompiled expression, which here is a `LANGSXP` representing
`2 + 2`. However, we also have two more expressions in positions `1` and `3`
representing variables.  We see that `LDVAR` operations in the `BCODE_CODE`
vector have arguments pointing these variables. We also see that the `ADD`
operation's argument points to element `0` in `BCODE_CONSTS`. The
`BCODE_CONSTS` vector also contains an `expressionsIndex` vector just as
before, but it is no longer trivial. Let us therefore follow the mapping
between `BCODE_CODE` and `BCODE_CONSTS` that `expressionsIndex` describes:

```
BCODE_CODE   expressionsIndex   BCODE_CONSTS
════════════════════════════════════════════
0 ↦ 10       0 ↦ NA             NA ↦ NA
1 ↦ LDVAR    1 ↦ 1              1  ↦ x
2 ↦ 1        2 ↦ 1              1  ↦ x
3 ↦ LDVAR    3 ↦ 3              3  ↦ y
4 ↦ 3        4 ↦ 3              4  ↦ y
5 ↦ ADD      5 ↦ 0              0  ↦ x + y
6 ↦ 0        6 ↦ 0              0  ↦ x + y
7 ↦ RETURN   7 ↦ 0              0  ↦ x + y
```

The mapping shows us that `LDVAR` operations come about by compiling the
individual variables. The `ADD`  operation come from compiling the plus
operation. Finally, `RETURN` is associated with the last expression we compiled
(because we will return the value we obtain from the last expression).

Finally, the `BCODE_CONSTS` contains two more `INTSXP` vectors, both with the
class attribute `srcref`. In a `BCODE_CONSTS` vector all `INTSXP` vectors of
class `srcref` are source references for byte-code operations, *except* for the
very last one. The *last* `srcref` is a mapping from elements in the `BCODE_CODE`
vector to source references defined in `BCODE_CONSTS`. In our example we only
have one source reference, so that mapping is trivial though. Nevertheless, let
us follow the mapping in detail:

```
BCODE_CODE   srcref  BCODE_CONSTS
═════════════════════════════════════════════
0 ↦ 10       0 ↦ NA  NA ↦ NA
1 ↦ LDVAR    1 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1
2 ↦ 1        2 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1
3 ↦ LDVAR    3 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1 
4 ↦ 3        4 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1
5 ↦ ADD      5 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1    
6 ↦ 0        6 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1   
7 ↦ RETURN   7 ↦ 2   2  ↦ 1,13,1,32,13,32,1,1   
```

From this we see that of the operations in our code come from source code
located in the environment, starting at line 1, byte 13 and ending at line 1
byte 32.

We can inspect the function using the `inspect` function, but we run into the
same problem as we did with inspecting `BCODESXP` previously---the `inspector`
does not reach inside it, so we have to inspect a `BCODESXP`'s elements by
manually feeding an address to the `inspect` function, eg:

```GDB
> .Internal(inspect(f))
@55555b3b8c28 03 CLOSXP g0c0 [NAM(3),ATT] 
FORMALS:
  @555555dfe548 02 LISTSXP g0c0 [NAM(3)] 
    TAG: @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @555555c1bc10 01 SYMSXP g1c0 [MARK,NAM(3)] "" (has value)
    TAG: @555555d84588 01 SYMSXP g1c0 [MARK,NAM(3)] "y"
    @555555c1bc10 01 SYMSXP g1c0 [MARK,NAM(3)] "" (has value)
BODY:
  @55555b3b9130 21 BCODESXP g0c0 [NAM(3)] 
CLOENV:
  @555555c4ef70 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
ATTRIB:
  @55555b3b8bf0 02 LISTSXP g0c0 [] 
...
> ^C
Thread 1 "R" received signal SIGINT, Interrupt.
0x00007ffff619903f in __GI___select (nfds=1, readfds=0x555555c030e0 <readMask.14046>, writefds=0x0, exceptfds=0x0, timeout=0x0) at ../sysdeps/unix/sysv/linux/select.c:41
41	in ../sysdeps/unix/sysv/linux/select.c
(gdb) p R_inspect(((SEXP) 0x55555b3b9130)->u.listsxp.cdrval)
@55555c824af8 19 VECSXP g0c4 [] (len=6, tl=0)
  @555555dfea50 06 LANGSXP g0c0 [NAM(3)] 
    @555555c26ce0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x5000] "+" (has value)
    @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @555555d84588 01 SYMSXP g1c0 [MARK,NAM(3)] "y"
  @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
  @55555b4ce088 13 INTSXP g0c3 [OBJ,NAM(3),ATT] (len=8, tl=0) 1,13,1,32,13,...
  ATTRIB:
    @555555dfea88 02 LISTSXP g0c0 [] 
      TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
      @555555dfe4d8 04 ENVSXP g0c0 [OBJ,NAM(3),ATT] <0x555555dfe4d8>
      FRAME:
	@55555b3bca58 02 LISTSXP g0c0 [] 
	  TAG: @55555645df40 01 SYMSXP g1c0 [MARK] "fixedNewlines"
	  @555555c1dc10 10 LGLSXP g1c1 [MARK,NAM(3)] (len=1, tl=0) 1
	  TAG: @5555562dffd8 01 SYMSXP g1c0 [MARK] "lines"
	  @55555d1c1b88 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	    @55555cdcdf98 09 CHARSXP g0c4 [gp=0x60] [ASCII] [cached] "f <- cmpfun(function(x, y) x + y)"
	    @555555c1db68 09 CHARSXP g1c1 [MARK,gp=0x60] [ASCII] [cached] ""
	  TAG: @5555562ed670 01 SYMSXP g1c0 [MARK] "filename"
	  @55555b49fb48 16 STRSXP g0c1 [] (len=1, tl=0)
	    @555555c1db68 09 CHARSXP g1c1 [MARK,gp=0x60] [ASCII] [cached] ""
      ENCLOS:
	@555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
      ATTRIB:
	@555555dff9a0 02 LISTSXP g0c0 [] 
	  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
	  @55555d153d08 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	    @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
	    @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
      @55555b49fbb8 16 STRSXP g0c1 [NAM(3)] (len=1, tl=0)
	@555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
  @555555d84588 01 SYMSXP g1c0 [MARK,NAM(3)] "y"
  @55555c749548 13 INTSXP g0c3 [OBJ,NAM(3),ATT] (len=8, tl=0) -2147483648,1,1,3,3,...
  ATTRIB:
    @55555b3b5190 02 LISTSXP g0c0 [] 
      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
      @555557fceb50 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
	@555555cd5828 09 CHARSXP g1c3 [MARK,gp=0x60] [ASCII] [cached] "expressionsIndex"
  ...
$14 = (struct SEXPREC *) 0x55555c824af8
```

We can also disassemble the function to retrieve most of that information more
compactly:

```R
> disassemble(f)
list(.Code, list(10L, GETVAR.OP, 1L, GETVAR.OP, 3L, ADD.OP, 0L, 
    RETURN.OP), list(x + y, x, structure(c(1L, 13L, 1L, 32L, 
13L, 32L, 1L, 1L), srcfile = <environment>, class = "srcref"), 
    y, structure(c(NA, 1L, 1L, 3L, 3L, 0L, 0L, 0L), class = "expressionsIndex"), 
    structure(c(NA, 2L, 2L, 2L, 2L, 2L, 2L, 2L), class = "srcrefsIndex")))
```

### S4 objects

The type `S4SXP` is used to express classes and objects in R's S4
object-oriented model. <sup>7</sup> S4 allows us to create classes, supports
inheritance and multiple dispatch. It also tends to be a little convoluted for
unaccustomed users. And for me.

Actually, while `S4SXP`s express S4 objects, other types of SEXPs can also
express S4 objects. What matters whether something is considered an S4 object
is not the type, but whether the S4 mask is set in the `gp` field of the SEXP's
header. The following snippets shows the definition of the mask and the basic
operations that set and re-set an object's S4 status.

```C
#define S4_OBJECT_MASK     ((unsigned short)(1<<4))
#define IS_S4_OBJECT(x)    ((x)->sxpinfo.gp & S4_OBJECT_MASK)
#define SET_S4_OBJECT(x)   (((x)->sxpinfo.gp) |= S4_OBJECT_MASK)
#define UNSET_S4_OBJECT(x) (((x)->sxpinfo.gp) &= ~S4_OBJECT_MASK)
```

There is no dedicated payload type that would represent S4 objects. There
aren't even helpful macros like we've used for `BCODESXP`s. Instead, we are
stuck using `listsxp_struct` to represent `S4SXP`s.  Fortunately, there is not
much to look at:

```
┌────────────┬────────────┬────────────┐
│   carval   │   tagval   │   cdrval   │
│            │ R_NilValue │            │
│            │            │            │
└───[SEXP]───┴──[NILSXP]──┴───[SEXP]───┘
```

What we see inside is two dangling pointers in `carval` and `cdrval` and a
pointer to `R_NilValue` in `tagval`. This perplexing structure escapes any
insight I have developed so far, but I am *pretty sure* neither slot is used
for anything, since the `duplicate` function for `S4SXP`s seems to just
recreate this state exactly without copying anything. Instead, all the
information related to operating S4 objects is stored as attributes. I imagine
this is what allows S4 objects to be of any type, not only `S4SXP`.

So what attributes would we see in an S4 object? Let's investigate an example.
In order to do that we should first create a class:

```R
> setClass("X", slots=c(x = "numeric", y = "numeric"))
```

This creates a class called `X` in the current environment with two slots
(fields), `x` and `y`, both of the numeric. What the interpreter actually does
is that it creates an object called `.__C__X` in the current environment that
represents our class and can be used as a factory for instantiating objects.
Let's fish it out from the environment and look at it:

```R
> x_class <- globalenv()$.__C__X
> x_class
Class "X" [in ".GlobalEnv"]

Slots:
                      
Name:        x       y
Class: numeric numeric
```

The pretty printer shows us the basic information about the class and its
fields. Let us now inspect it:

```R
> .Internal(inspect(x_class))
@5555572a1438 25 S4SXP g0c0 [OBJ,NAM(3),S4,gp=0x10,ATT] 
ATTRIB:
  @5555572a1b70 02 LISTSXP g0c0 [] 
    TAG: @555556714638 01 SYMSXP g1c0 [MARK,NAM(3)] "slots"
    @55555b2eddc8 19 VECSXP g0c2 [NAM(3),ATT] (len=2, tl=0)
      @555558ac04f0 16 STRSXP g0c1 [NAM(3),ATT] (len=1, tl=0)
	@555555c4c6c0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "numeric"
      ATTRIB:
	@555557315cf8 02 LISTSXP g0c0 [] 
	  TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	  @555558047928 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
	    @555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
      @555558ac0598 16 STRSXP g0c1 [NAM(3),ATT] (len=1, tl=0)
	@555555c4c6c0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "numeric"
      ATTRIB:
	@555557311dd8 02 LISTSXP g0c0 [] 
	  TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	  @555558047928 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
	    @555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
    ATTRIB:
      @5555573158d0 02 LISTSXP g0c0 [] 
	TAG: @555555c1b510 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "names" (has value)
	@55555b2d3db8 16 STRSXP g0c2 [NAM(3)] (len=2, tl=0)
	  @555555c6a298 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "x"
	  @555555d72528 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "y"
    TAG: @5555567148a0 01 SYMSXP g1c0 [MARK,NAM(3)] "contains"
    @555557526ab0 19 VECSXP g0c0 [NAM(3)] (len=0, tl=0)
    TAG: @5555568319a0 01 SYMSXP g1c0 [MARK,NAM(3)] "virtual"
    @555555c1dbd8 10 LGLSXP g1c1 [MARK,NAM(3)] (len=1, tl=0) 0
    TAG: @5555563bbab0 01 SYMSXP g1c0 [MARK,NAM(3)] "prototype"
    @555557500120 25 S4SXP g0c0 [NAM(3),S4,gp=0x10,ATT] 
    ATTRIB:
      @555557500200 02 LISTSXP g0c0 [] 
	TAG: @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
	@555557577618 14 REALSXP g0c0 [NAM(3)] (len=0, tl=0)
	TAG: @555555d84588 01 SYMSXP g1c0 [MARK] "y"
	@55555754bb80 14 REALSXP g0c0 [NAM(3)] (len=0, tl=0)
    TAG: @555556831850 01 SYMSXP g1c0 [MARK,NAM(3)] "validity"
    @5555568318f8 01 SYMSXP g1c0 [MARK,NAM(3)] "\001NULL\001"
    TAG: @5555568317a8 01 SYMSXP g1c0 [MARK] "access"
    @55555748e368 19 VECSXP g0c0 [NAM(3)] (len=0, tl=0)
    TAG: @5555563b3928 01 SYMSXP g1c0 [MARK,NAM(3)] "className"
    @555558ae3060 16 STRSXP g0c1 [NAM(3),ATT] (len=1, tl=0)
      @555555d44958 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "X"
    ATTRIB:
      @55555750c860 02 LISTSXP g0c0 [] 
	TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	@55555d6718c0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
	  @555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
    TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
    @55555d6718c0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
      @555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
    TAG: @555556839b88 01 SYMSXP g1c0 [MARK,NAM(3)] "subclasses"
    @5555574ce2a0 19 VECSXP g0c0 [NAM(3)] (len=0, tl=0)
    TAG: @555556839aa8 01 SYMSXP g1c0 [MARK] "versionKey"
    @55555748e528 22 EXTPTRSXP g0c0 [NAM(3)] 
    TAG: @5555568399c8 01 SYMSXP g1c0 [MARK] "sealed"
    @55555a8f48c8 10 LGLSXP g1c1 [MARK,NAM(3)] (len=1, tl=0) 0
    TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
    @555555ff92e8 16 STRSXP g1c1 [MARK,NAM(3),ATT] (len=1, tl=0)
      @5555567a64e8 09 CHARSXP g1c3 [MARK,gp=0x61,ATT] [ASCII] [cached] "classRepresentation"
    ATTRIB:
      @555556d6b8d0 02 LISTSXP g1c0 [MARK] 
	TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	@555555ff92b0 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
	  @555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
```

We see that `x_class` is an `S4SXP` type object. The `inspect` function also
indicates that the S4 bit is set in the `gp` part of the header by printing
`S4` in the square brackets. The payload of `S4SXP` is not shown. We could peer
into it regardless, but it contains only those two dangling pointers and a
pointer to `R_NilValue` so it wouldn't be productive.

Instead we can take a look at the mess of attributes. There is a lot of stuff
in the print out, but we summarize the attributes as follows. (We could also
list them using `attributes` in R).

```
attr         value                         
═══════════════════════════════════════════
slots        list(x="numeric", y="numeric")
contains     list()                        
virtual      FALSE                            
prototype    an S4 object where x=0, y=0   
validity     "\001NULL\001"                
access       list()
className    "X"
package      R_GlobalEnv
subclasses   list()
versionKey   an EXPRPTRSXP object
sealed       FALSE
class        "classRepresentation"
```

Most of these fields are self-explanatory and it is probably beyond the scope of
this report to go into the details of all of them, but let's boringly discuss a few key
attributes.

The `slots` attribute contains the definitions of slots (fields) of class `X`.
The `contains` attribute lists the classes from which `X` inherits. The
`prototype` attribute gives us an object that will be used as a basis to create
instances of this class. The `validity` attribute optionally points to a
function for checking the validity of an object of class `X`. The `className`
attribute tells us the name of class `X`, which is contrasted with the `class`
attribute, which tells us what is the role of the object we're looking at: it's
a class definition. The `sealed` attribute says whether the class can be
modified or not. The `package` attribute specifies which environment this
object belongs to. 

Now let's define a method for the class. First we define a generic method which
we will call `concatenate` and which will take one argument called `object`:

```R
> setGeneric("concatenate", function(object) standardGeneric("concatenate"))
[1] "concatenate"
```

This creates a function called `concatenate` in the current environment:

```R
> globalenv()$concatenate
standardGeneric for "concatenate" defined from package ".GlobalEnv"

function (object) 
standardGeneric("concatenate")
<environment: 0x55555d2328f0>
Methods may be defined for arguments: object
Use  showMethods("concatenate")  for currently available ones.
``` 

Further inspection would reveal that this is an ordinary `CLOSXP` closure which
invokes the dispatch mechanism for S4. Another thing that happens is that an
object called `.__T__concatenate:.GlobalEnv` is created in the environment.
Let's take a closer look:

```R
> .Internal(inspect(globalenv()$`.__T__concatenate:.GlobalEnv`))   # ticks around the name to prevent it from
                                                                   # being interpreted as a function call of :
@55555d3445b0 04 ENVSXP g0c0 [MARK,NAM(3)] <0x55555d3445b0>
ENCLOS:
  @55555d2328f0 04 ENVSXP g0c0 [MARK,NAM(3)] <0x55555d2328f0>
HASHTAB:
  @55555a756110 19 VECSXP g0c7 [MARK] (len=29, tl=0)
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    ...
```

For now the object is an empty hash table environment. Let's see how it changes
as we proceed to define the actual method. 

```R
> setMethod("concatenate", signature(object = "X"), function(object) paste(object@x, object@y))
> concatenate(x)
[1] "42 63"
```

We define the signature as an object of class `X`---this will be used to guide
the dispatch. The body we will concatenate the two field of `object`, `x` and
`y`, using the standard `paste` function. Nothing surprising. We then test out
the method by calling it on our object `x` of class `X` and see it indeed
works. 

Adding a method impacts that `.__T__concatenate:.GlobalEnv` object in the
environment: 

```R
> .Internal(inspect(g$`.__T__concatenate:.GlobalEnv`))
@55555d3445b0 04 ENVSXP g0c0 [MARK,NAM(3)] <0x55555d3445b0>
ENCLOS:
  @55555d2328f0 04 ENVSXP g0c0 [MARK,NAM(3)] <0x55555d2328f0>
HASHTAB:
  @55555a756110 19 VECSXP g0c7 [MARK] (len=29, tl=1)
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @55555a7d6fd8 02 LISTSXP g0c0 [] 
      TAG: @555555d51050 01 SYMSXP g1c0 [MARK,NAM(3)] "X"
      @55555a93e280 03 CLOSXP g0c0 [OBJ,NAM(3),S4,gp=0x50,ATT] 
      FORMALS:
	@55555d343cb8 02 LISTSXP g0c0 [MARK,NAM(3)] 
	  TAG: @555555d4b560 01 SYMSXP g1c0 [MARK,NAM(3)] "object"
	  @555555c1bc10 01 SYMSXP g1c0 [MARK,NAM(3)] "" (has value)
      BODY:
	@55555d3439a8 06 LANGSXP g0c0 [MARK] 
	  @555555c31c78 01 SYMSXP g1c0 [MARK,LCK,gp=0x6000] "paste" (has value)
	  @55555d343c10 06 LANGSXP g0c0 [MARK,NAM(3)] 
	    @555555c24c70 01 SYMSXP g1c0 [MARK,LCK,gp=0x6000] "@" (has value)
	    @555555d4b560 01 SYMSXP g1c0 [MARK,NAM(3)] "object"
	    @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
	  @55555d343a88 06 LANGSXP g0c0 [MARK,NAM(3)] 
	    @555555c24c70 01 SYMSXP g1c0 [MARK,LCK,gp=0x6000] "@" (has value)
	    @555555d4b560 01 SYMSXP g1c0 [MARK,NAM(3)] "object"
	    @555555d84588 01 SYMSXP g1c0 [MARK] "y"
      CLOENV:
	@555555c4ef70 04 ENVSXP g1c0 [MARK,NAM(3),GL,gp=0x8000] <R_GlobalEnv>
      ATTRIB:
	@55555a93e398 02 LISTSXP g0c0 [] 
	  TAG: @555555febbf0 01 SYMSXP g1c0 [MARK,NAM(3)] "target"
	  @55555a4c2588 16 STRSXP g0c1 [OBJ,NAM(3),S4,gp=0x10,ATT] (len=1, tl=0)
	    @555555d44958 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "X"
	  ATTRIB:
	    @55555a945710 02 LISTSXP g0c0 [] 
	      TAG: @555555c1b510 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "names" (has value)
	      @55555d35b4a0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555d454b8 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "object"
	      TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	      @55555dc2bf70 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
	      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
	      @555557fd4348 16 STRSXP g1c1 [MARK,NAM(3),ATT] (len=1, tl=0)
		@5555562a2958 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "signature"
	      ATTRIB:
		@555557055e50 02 LISTSXP g1c0 [MARK] 
		  TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
		  @555557fd4310 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
		    @555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
	  TAG: @555556644a08 01 SYMSXP g1c0 [MARK,NAM(3)] "defined"
	  @55555a4c2588 16 STRSXP g0c1 [OBJ,NAM(3),S4,gp=0x10,ATT] (len=1, tl=0)
	    @555555d44958 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "X"
	  ATTRIB:
	    @55555a945710 02 LISTSXP g0c0 [] 
	      TAG: @555555c1b510 01 SYMSXP g1c0 [MARK,LCK,gp=0x4000] "names" (has value)
	      @55555d35b4a0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555d454b8 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "object"
	      TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	      @55555dc2bf70 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
	      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
	      @555557fd4348 16 STRSXP g1c1 [MARK,NAM(3),ATT] (len=1, tl=0)
		@5555562a2958 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "signature"
	      ATTRIB:
		@555557055e50 02 LISTSXP g1c0 [MARK] 
		  TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
		  @555557fd4310 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
		    @555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
	  TAG: @555555d4b4f0 01 SYMSXP g1c0 [MARK,NAM(3)] "generic"
	  @55555d34aab8 16 STRSXP g0c1 [MARK,NAM(3),ATT] (len=1, tl=0)
	    @55555b36a1a8 09 CHARSXP g0c2 [MARK,gp=0x61,ATT] [ASCII] [cached] "concatenate"
	  ATTRIB:
	    @55555d231e70 02 LISTSXP g0c0 [MARK] 
	      TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	      @55555d6718c0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
	  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
	  @555556297fe0 16 STRSXP g1c1 [MARK,NAM(3),ATT] (len=1, tl=0)
	    @5555566b5b88 09 CHARSXP g1c3 [MARK,gp=0x61,ATT] [ASCII] [cached] "MethodDefinition"
	  ATTRIB:
	    @555557043368 02 LISTSXP g1c0 [MARK] 
	      TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	      @555556298018 16 STRSXP g1c1 [MARK,NAM(3)] (len=1, tl=0)
		@555555df08e0 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "methods"
	  TAG: @555555c1ae80 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcref" (has value)
	  @55555d8394a8 13 INTSXP g0c3 [OBJ,MARK,NAM(3),ATT] (len=8, tl=0) 1,51,1,92,51,...
	  ATTRIB:
	    @55555d343970 02 LISTSXP g0c0 [MARK] 
	      TAG: @555555c1aef0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "srcfile" (has value)
	      @55555d343fc8 04 ENVSXP g0c0 [OBJ,MARK,ATT] <0x55555d343fc8>
	      FRAME:
		@55555d347490 02 LISTSXP g0c0 [MARK] 
		  TAG: @5555562dffd8 01 SYMSXP g1c0 [MARK] "lines"
		  @55555d5d1580 16 STRSXP g0c1 [MARK] (len=1, tl=0)
		    @55555ceb2028 09 CHARSXP g0c5 [MARK,gp=0x60,ATT] [ASCII] [cached] "setMethod("concatenate", signature(object = "X"), function(object) paste(object@x, object@y))
"
		  TAG: @5555562ed670 01 SYMSXP g1c0 [MARK] "filename"
		  @55555d5d15f0 16 STRSXP g0c1 [MARK] (len=1, tl=0)
		    @555555c1db68 09 CHARSXP g1c1 [MARK,gp=0x60] [ASCII] [cached] ""
	      ENCLOS:
		@555555c1bcb8 04 ENVSXP g1c0 [MARK,NAM(3)] <R_EmptyEnv>
	      ATTRIB:
		@55555d347458 02 LISTSXP g0c0 [MARK] 
		  TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
		  @55555d83c308 16 STRSXP g0c2 [MARK,NAM(3)] (len=2, tl=0)
		    @555555cd7b88 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] "srcfilecopy"
		    @555555c1d660 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcfile"
	      TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
	      @55555d5d16d0 16 STRSXP g0c1 [MARK] (len=1, tl=0)
		@555555c1d628 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "srcref"
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    @555555c1bcf0 00 NILSXP g1c0 [MARK,NAM(3)] 
    ...
```

What happens is that the closure we created by defining a function is stored it
in the hash table under `X`, the class name used for multiple dispatch. In
addition, there are a number of attributes attached to the closure. There's a
lot there, so let's summarize them:

```
attr         value                         
════════════════════════════════════════════════════
target       S4 object of class "signature"
defined      S4 object of class "signature"
generic      "concatenate"
class        "methodDefinition"
srcref       source reference c(1,51,1,92,51,92,1,1)
``` 

These attributes can be used to direct multiple dispatch using the types of all
objects passed to a generic function, since we have the signatures of methods
preserved right there.

Let's now create an object of class `X` and take a look at it.

```R
> x <- new("X", x=42, y=63)
> x
An object of class "X"
Slot "x":
[1] 42

Slot "y":
[1] 63
```

So far so good. Now let's look at it harder:

```R
> .Internal(inspect(x))
@55555c84e4e0 25 S4SXP g0c0 [OBJ,NAM(3),S4,gp=0x10,ATT] 
ATTRIB:
  @55555c84e438 02 LISTSXP g0c0 [] 
    TAG: @555555c989a0 01 SYMSXP g1c0 [MARK,NAM(3)] "x"
    @55555cb3e978 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 42
    TAG: @555555d84588 01 SYMSXP g1c0 [MARK] "y"
    @55555cb52070 14 REALSXP g0c1 [NAM(3)] (len=1, tl=0) 63
    TAG: @555555c1b9e0 01 SYMSXP g1c0 [MARK,NAM(3),LCK,gp=0x4000] "class" (has value)
    @555558ae3060 16 STRSXP g0c1 [NAM(3),ATT] (len=1, tl=0)
      @555555d44958 09 CHARSXP g1c1 [MARK,gp=0x61] [ASCII] [cached] "X"
    ATTRIB:
      @55555750c860 02 LISTSXP g0c0 [] 
	TAG: @555555c1b430 01 SYMSXP g1c0 [MARK,NAM(3)] "package"
	@55555d6718c0 16 STRSXP g0c1 [MARK,NAM(3)] (len=1, tl=0)
	  @555555c654e8 09 CHARSXP g1c2 [MARK,gp=0x61] [ASCII] [cached] ".GlobalEnv"
```

We see again an `S4SXP` type object and some arguments that are much fewer in
number than in the class definition. We could summarize them as follows:

```
attr       value                         
════════════════
x          42
y          63
class      "X"
```

These are straightforward and represent the state of the object. We also see
that the `class` attribute has its own attribute that points to the package of
the class.

<!--### Other types

There are a few types which I omitted. These are some simple types which have
very specific applications, and, if I'm honest, I neither understand them
completely, nor did they come up in my work thus far. Let's do a quick rundown
of them.-->

### External pointers

External pointers act as handles to C structures. They have a dedicated SEXP
type: `EXTPTRSXP`. There is no dedicated
payload type that would represent external pointers. Thus, we use
`listsxp_struct` to represent them, which we will access via the `listsxp`
field of the payload union. However, there are macros for accessing external pointer slots
that more clearly specify the contents of the structure:

```C
#define EXTPTR_PTR(x)   CAR(x)
#define EXTPTR_TAG(x)   TAG(x)
#define EXTPTR_PROT(x)  CDR(x)
```

The `EXTPTR_PTR` macro returns a pointer to some C structure. `EXTPTR_TAG` is
used to provide some sort of identification ti the pointer, while `EXTPTR_PROT`
provides an object for protecting from GC the memory that the external pointer
is allocated in case it is allocated from the R heap. `EXTRPTR_TAG` and
`EXTRPTR_PROT` often point to `R_NilValue` though. Thus we represent a
`EXTPTRSXP` as follows:

```
      ┌────────────────────────────────── EXTPTR_PTR(s)
      │            ┌───────────────────── EXTPTR_TAG(s)
      │            │            ┌──────── EXTPTR_PROT(s)
      v            v            v
┌────────────┬────────────┬────────────┐
│            │            │            │
│   carval   │   tagval   │   cdrval   │
│            │            │            │
└──[void *]──┴───[SEXP]───┴───[SEXP]───┘
```

These pointers are not really visible from the point of view of R, but we can
see them if we use some external libraries. For instance, let's create a
connection to an SQLite database:

```R
> library(RSQLite)
> db <- dbConnect(RSQLite::SQLite(), "/tmp/db.sqlite")
> db
<SQLiteConnection>
  Path: /tmp/db.sqlite
  Extensions: TRUE
```

Now, `db` itself is an S4 object, but it does contain an external pointer as
one of its fields that is used to connect to the db in C. We can see that if we
inspect it:

```R
> .Internal(inspect(db))
@555556c39c50 25 S4SXP g1c0 [OBJ,MARK,NAM(2),S4,gp=0x10,ATT] 
...ATTRIB:
  @555556c39e10 02 LISTSXP g1c0 [MARK] 
    TAG: @555555add0a0 01 SYMSXP g1c0 [MARK,NAM(2)] "ptr"
    @555556c41df0 22 EXTPTRSXP g1c0 [MARK,NAM(2)] 
    TAG: @555555add1b8 01 SYMSXP g1c0 [MARK,NAM(2)] "dbname"
    @555558914588 16 STRSXP g1c1 [MARK,NAM(2)] (len=1, tl=0)
      @555555ca5198 09 CHARSXP g1c2 [MARK,gp=0x60] [ASCII] [cached] "/tmp/db.sqlite"
```

### Weak references

Weak references are references that are not protected from garbage collection,
but are associated with a finalizer that is run when they become unreachable.

Internally, a weak reference It is represented by it's own type: `WEAKREFSXP`.
In actuality, it is a special sub-type of `VECSXP` which always has length 4.
The elements of the vector represent a `key`, a `value`, a `finalizer`, and
`next`. While there is no special struct to retrieve the information from
`WEAKREFSXPs`, there are macros which aid retrieving the elements of the
vector:

```C
#define WEAKREF_KEY(w)       VECTOR_ELT(w, 0)
#define WEAKREF_VALUE(w)     VECTOR_ELT(w, 1)
#define WEAKREF_FINALIZER(w) VECTOR_ELT(w, 2)
#define WEAKREF_NEXT(w)      VECTOR_ELT(w, 3)
```

Thus we can represent weak references as follows:

```
                                   VECTOR_ELT:
┌────────────┬────────────┬─-----─┬────────────┬────────────┬────────────┬────────────┐
│   length   │ truelength │       │    [0]     │    [1]     │    [2]     │    [3]     │
│     4      │     0      │       │            │            │            │            │
│            │            │       │            │            │            │            │
└─[R_xlen_t]─┴─[R_xlen_t]─┴─-----─┴───[SEXP]───┴───[SEXP]───┴──[FUNSXP]──┴[WEAKREFSXP]┘
                                        ʌ            ʌ            ʌ            ʌ   
                                        │            │            │            └────── WEAKREF_NEXT(s)
                                        │            │            └─────────────────── WEAKREF_FINALZIER(s)
                                        │            └──────────────────────────────── WEAKREF_VALUE(s)
                                        └───────────────────────────────────────────── WEAKREF_KEY(s)
```

The key and value represent the relationships between weak references and data
that the weak reference carries. The `key` is either `R_NilValue`, an
environment or an external pointer. A weak reference can be garbage collected
when it is unreachable. The value is reachable is if it either reachable
directly or via weak references with reachable keys. Once a value is determined
to be unreachable during garbage collection, the key and value are set to
`R_NilValue`. Also bit `0` in the `gp` portion of the header is set to `1` to
indicate readiness to finalize. The finalizer is either a function or a pointer
to `R_NilValue`.  This is the function that will be run when the weak reference
is actually garbage collected. The next field is used to connect all weak
references into a single structure. When created, a weak reference sets its
next field to `R_weak_refs` and then overwrites `R_weak_refs` with a pointer to
itself. Finally, the user can also set `gp` bit in position `1` to indicate
that the weak reference should be indicated on exit.

There are macros to facilitate setting those `gp` bits:

```C
#define READY_TO_FINALIZE_MASK 1

#define SET_READY_TO_FINALIZE(s)   ((s)->sxpinfo.gp |= READY_TO_FINALIZE_MASK)
#define CLEAR_READY_TO_FINALIZE(s) ((s)->sxpinfo.gp &= ~READY_TO_FINALIZE_MASK)
#define IS_READY_TO_FINALIZE(s)    ((s)->sxpinfo.gp & READY_TO_FINALIZE_MASK)

#define FINALIZE_ON_EXIT_MASK 2

#define SET_FINALIZE_ON_EXIT(s)    ((s)->sxpinfo.gp |= FINALIZE_ON_EXIT_MASK)
#define CLEAR_FINALIZE_ON_EXIT(s)  ((s)->sxpinfo.gp &= ~FINALIZE_ON_EXIT_MASK)
#define FINALIZE_ON_EXIT(s)        ((s)->sxpinfo.gp & FINALIZE_ON_EXIT_MASK)
```

Unfortunately, I have not seen a weak reference in the wild yet so I have no
examples.

<!--
http://www.hep.by/gnu/r-patched/r-exts/R-exts_122.html
http://homepage.divms.uiowa.edu/~luke/R/references/weakfinex.html

The design of this mechanism is very close to the one described in
   "Stretching the storage manager: weak pointers and stable names in
   Haskell" by Peyton Jones, Marlow, and Elliott (at
   www.research.microsoft.com/Users/simonpj/papers/weak.ps.gz). 
-->

# The rest of the header

We discussed some parts of the SEXP header already, but there are some useful
flags in there that we have thus far omitted. Here is a rundown of what they
mean and how to use them. 

## Debug

The `debug` flag is set only for SEXPs of type CLOSXP and ENVSXP. For CLOSXPs
it indicates whether the function is executed in debug mode. The debug mode
uses the browser which allows to step over instructions and observe values of
objects (like `gdb`). The browser can be turned on on-demand by calling
`browser()`, but functions in debug mode trigger the browser automatically. To
set/unset the debug flag for a closure use functions `debug` and `undebug`,
like so:

```R
> f <- function(x) x + 1
> debug(f)
> f(1)       
...                      # runs with browser
> undebug(f)
> f(1)       
...                      # runs without browser
```

In the case of environments the debug flag indicates whether the environment is
going to be browsed in single-step mode.  

The inspect function shows that the debug flag is set by printing `DBG` in the
flag section, like so:

```
> debug(f)
> .Internal(inspect(f))
@1995b298 03 CLOSXP g0c0 [MARK,NAM(2),DBG] 
```

There are macros that allow to read and modify these flags:

```C
#define RDEBUG(x)       ((x)->sxpinfo.debug)
#define SET_RDEBUG(x,v) (((x)->sxpinfo.debug)=(v))
#define RSTEP(x)        ((x)->sxpinfo.spare)
```

## Spare

The `spare` flag is set on closures to mark them for one-time debugging with
reference counting. The flag can be set using `debugonce` and it is
automatically unset once the function is executed.

```R
> f <- function(x) x + 1
> debugonce(f)
```

The inspect function shows the spare flag as `STP`:

```
> .Internal(inspect(f))
@1995b298 03 CLOSXP g0c0 [MARK,NAM(2),STP] 
```

## Trace

The `trace` flag is set to indicate that we want to keep track of objects being
copied. The tracing can be turned on via function `tracemem` like so:

<!-- footnote: http://adv-r.had.co.nz/memory.html --> 

```R
> d <- c(1:10)
> tracemem(d)
[1] "<0x55d2e3be8fa0>"
```

The inspect function will then print as follows, with the trace flag indicated
as `TR`:

```R
> .Internal(inspect(d))
@1aef6b18 13 INTSXP g0c4 [NAM(2),TR] (len=10, tl=0) 1,2,3,4,5,...
```

With this flag turned on, the interpreter will inform the programmer whenever
copying occurs. For instance, when executing an assignment the tracer will
print something like this:

```R
> d[5] <- 42
tracemem[0x55d2e3be8fa0 -> 0x55d2e20c8680]:
```

# Bonus

As I was writing this blog post I realized I wanted to be able to look directly
at a SEXP without having to go into the debugger every time, so I wrote a very
simple package for R that would make this possible. The package is called
`sexpinspector` and it is available here: https://gitlab.com/kondziu/sexp-inspector

The package just has one function called `what_do_we_have_here` which takes one
argument that can be any SEXP and prints out information about that SEXP to
screen:

```R 
> library(sexpinspector)
> what_have_we_got_here(c(1,2,3,4,5))
0x55555b367ea8

[header]
type          [1]: 14    # REALSXP
scalar        [1]: 0
obj           [1]: 0
alt           [1]: 0
named         [2]: 3
gp           [16]: 0
mark          [1]: 0
debug         [1]: 0
trace         [1]: 0
spare         [1]: 0
gcgen         [1]: 0
gccls         [3]: 4

[s.vecsxp and align]
length           : 5
truelength       : 0
align            : 7.42699e-314

[macros]
IS_LONG_VEC      : 0
XLENGTH          : 5

[elements]
REAL[0]         : 1
REAL[1]         : 2
REAL[2]         : 3
REAL[3]         : 4
REAL[4]         : 5
```

# Footnotes

1. Here's a quick and dirty hack to making the inspect function visible in your
   C code. Find `inspect.c` and add `inspect.h` alongside it with the following
   contents:

    ```C    
    #include <Rinternals.h>
    SEXP R_inspect(SEXP x);
    ```

    Then, redefine the inspect function in `inspect.c` to remove the hidden attribute:

    ```C
    SEXP /*attribute_hidden*/ R_inspect(SEXP x) {
        inspect_tree(0, x, -1, 5);
        return x;
    }
    ```

2. Actually, you don't need `substitute` for inspecting function definitions,
   you will get the same thing by running `.Internal(inspect(function(x)x))`.

3. To the best of my knowledge, the most comprehensive source of information on
   the byte code compiler is currently a report by Luke Tierney: [A Byte Code
   Compiler for R](http://homepage.stat.uiowa.edu/~luke/R/compiler/compiler.pdf).

4. Sources of information about altrep:

    * https://gmbecker.github.io/jsm2017/jsm2017.html
    * https://www.r-project.org/dsc/2017/slides/dsc2017.pdf
    * https://homepage.divms.uiowa.edu/~luke/talks/nzsa-2017.pdf
    * http://blog.revolutionanalytics.com/2017/09/altrep-preview.html

5. According to  [the manual](https://stat.ethz.ch/R-manual/R-devel/library/base/html/srcfile.html): 

    *Lines (elements 1, 3) and parsed lines (elements 7, 8) may differ if a
    #line directive is used in code: the former will respect the directive, the
    latter will just count lines. If only 4 or 6 elements are given, the parsed
    lines will be assumed to match the lines.*

6. The source code sends us to [The Treadmill: Real-Time Garbage Collection
   Without Motion Sickness by Henry G. Baker]
   (http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.23.5878&rep=rep1&type=pdf).

7. R has four different object-oriented systems built in (that I know of): S3,
   S4, Reference Classes, and R6. Although there are also other custom built
   object-oriented systems in R. For instance `ggproto` is an object-oriented
   system built for and used by the `ggplot2` graphical library. From the
   documentation of `ggplot2`:

    *`ggproto` implements a prototype based OO system which blurs the lines
    between classes and instances. It is inspired by the proto package, but it
    has some important differences. Notably, it cleanly supports cross-package
    inheritance, and has faster performance.*

    *In most cases, creating a new OO system to be used by a single package is
    not a good idea. However, it was the least-bad solution for `ggplot`2
    because it required the fewest changes to an already complex code base.*

    I found the naming convention intriguing so I looked up its history. Here's
    a brief for your enjoyment. S3 and S4 were implemented in R's predecessor
    language, the S language. The S3 system was introduced in version 3 of S,
    and the S4 system was introduced in version 4 of S. Hence the names. When
    Reference Classes were being introduced in R ver. 2.12 to fix some of the
    problems with S4 they were jokingly referred to as R5, but the official name
    stuck (sometimes shortened to refclasses). Reference Classes are based
    strongly on S4. There was also an experimental unreleased object oriented
    system started by Simon Urbanek meant to solve some performance and
    usability issues with S4. R6 is named as a successor to those two, since it
    is very similar to refclasses, but more light-weight and without some
    issues of S4.

    More information is available in [Object-Oriented Programming, Functional
    Programming and R]
    (https://projecteuclid.org/download/pdfview_1/euclid.ss/1408368569) 
    by John Chambers.

8. JIT settings:

    ```
R_ENABLE_JIT  meaning                         
═══════════════════════════════════════════════════════════════════════════════════════════
0             JIT compilation is disabled
1             larger closures are compiled before they are used for the first time
2             like above, but some small closures are also compiled before their second use
3             like above, but all top-level loops are compiled before they are executed
```

9. I am not sure where the name SEXP comes from. It is possible that the name
   SEXP comes from s-expressions, or symbolic expressions, which is a type of
   notation for nested lists. I personally also suspect that it comes from the
   name of R's predecessor, the S language: they are S expressions.

# See also

- [R Internals](https://cran.r-project.org/doc/manuals/r-release/R-ints.html)
- [Environments](http://adv-r.had.co.nz/Environments.html)
- [External pointers and weak references](http://www.hep.by/gnu/r-patched/r-exts/R-exts_122.html)
- [Memory](http://adv-r.had.co.nz/memory.html)
