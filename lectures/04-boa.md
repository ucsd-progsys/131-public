# Boa: Branches & Binary Operators

Next, lets add

* Branches (`if`-expressions)
* Binary Operators (`+`, `-`, etc.)

In the process of doing so, we will learn about

* **Intermediate Forms**
* **Normalization**

## Branches

Lets start first with branches (conditionals).

We will stick to our recipe of:

1. Build intuition with **examples**,
2. Model problem with **types**,
3. Implement with **type-transforming-functions**,
4. Validate with **tests**.

### Examples

First, lets look at some examples of what we mean by branches.

* For now, lets treat `0` as "false" and non-zero as "true"

### Example: `If1`

```haskell
if 10:
  22
else:
  sub1(0)
```  

* Since `10` is _not_ `0` we evaluate the "then" case to get `22`


### Example: `If2`

```haskell
if sub(1):
  22
else:
  sub1(0)
```

* Since `sub(1)` _is_ `0` we evaluate the "else" case to get `-1`


### Example: `If3`

`if-else` is also an _expression_ so we can nest them:

```haskell
let x = if sub(1):
          22
        else:
          sub1(0)
in
  if x:
    add1(x)
  else:
    999
```  

* `x` is bound to `-1`...
* ... which is _non-zero_ so we evaluate `add1(x)` yielding `0`

### Control Flow in Assembly

To compile branches, we will use:

* **labels** of the form

```asm
our_code_label:
  ...
```

are "landmark" from which execution (control-flow)
can be started, or to which it can be diverted,

* **comparisons** of the form

```asm
cmp a1, a2
```

do a (numeric) comparison between the values
referred to by `a1` and `a2`, storing the result
in a special processor **flag**,

* **Jump** operations of the form

```asm
jmp LABEL     # jump unconditionally (i.e. always)
je  LABEL     # jump if previous comparison result was EQUAL  
jne LABEL     # jump if previous comparison result was NOT-EQUAL  
```

use the result of the _flag_ set by the most recent `cmp` to
*transfer control flow* to the given `LABEL`

### Strategy

To compile an expression of the form

```haskell
if eCond:
  eThen
else:
  eElse
```

We will:

1. Evaluate `eCond`
2. Compare the result (in `eax`) against `0`
3. Jump if the result _is zero_ to a **special** `"IfFalse"` label
  * At which we will evaluate `eElse`,
  * Ending with a special `"IfExit"` label.
4. (Otherwise) continue to evaluate `eTrue`
  * And then jump (unconditionally) to the `"IfExit"` label.

### Example: If-Expressions to `Asm`

Lets see how our strategy works by example:

### Example: if1

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/if-1-to-asm.png" height="200">

### Example: if2

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/if-2-to-asm.png" height="200">

### Example: if3

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/if-3-to-asm.png" height="300">

Oops, **cannot reuse labels** across if-expressions!

* Can't use same label in two places (invalid assembly)

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/if-3-to-asm-bad.png" height="450">

Oops, need **distinct labels** for each branch!

* Require **distinct tags** for each `if-else` expression

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/if-3-to-asm-tag.png" height="450">

### Types: Source

Lets modify the _Source Expression_

```haskell
data Expr a
  = Number Int                        a
  | Add1   (Expr a)                   a
  | Sub1   (Expr a)                   a
  | Let    Id (Expr a) (Expr a)       a   
  | Var    Id                         a
  | If     (Expr a) (Expr a) (Expr a) a         
```

* Add `if-else` expressions and
* Add **tags** of type `a` for each sub-expression
  * Tags are polymorphic `a` so we can have _different types_ of tags
  * e.g. Source-Position information for error messages

Lets define a name for `Tag` (just integers).

```haskell
type Tag   = Int
```

We will now use:

```haskell
type BareE = Bare ()     -- AST after parsing   
type TagE  = Expr Tag    -- AST with distinct tags
```

### Types: Assembly

Now, lets extend the _Assembly_ with labels, comparisons and jumps:

```haskell
data Label
  = BranchFalse Tag
  | BranchExit  Tag          

data Instruction
  = ...
  | ICmp   Arg   Arg  -- Compare two arguments
  | ILabel Label      -- Create a label
  | IJmp   Label      -- Jump always
  | IJe    Label      -- Jump if equal  
  | IJne   Label      -- Jump if not-equal
```

### Transforms

We can't expect _programmer_ to put in tags (yuck.)

* Lets squeeze in a `tagging` transform into our pipeline

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-tag.png" height="200">

### Transforms: Parse

Just as before, but now puts a dummy `()` into each position

```haskell
ghci> parse "if 1: 22 else: 33"
If (Number 1 ()) (Number 22 ()) (Number 33 ()) ()
```

### Transforms: Tag

The key work is done by `doTag i e`
* Recursively walk over the `BareE` named `e` starting tagging at counter `i`
* Returning a pair `(i', e')` of _updated counter_ `i'` and  tagged expr `e'`

```haskell
doTag :: Int -> BareE -> (Int, TagE)
doTag i (Number n _)    = (i + 1 , Number n i)
doTag i (Var    x _)    = (i + 1 , Var     x i)
doTag i (If e1 e2 e3 i) = (i3 + 1, If e1' e2' e3' i3)
  where
    (i1, e1')           = doTag i  e1
    (i2, e2')           = doTag i1 e2
    (i3, e3')           = doTag i2 e3
doTag i (Let x e1 e2 i) = (i2 + 1, Let x e1' e2' i2)
  where
    (i1, e1')           = doTag i  e1
    (i2, e2')           = doTag i1 e2
```

(**ProTip:** Use `mapAccumL`)

We can now tag the whole program by
* calling `doTag` with  the initial counter (e.g. `0`),
* throwing away the final counter.

```haskell
tag :: BareE -> TagE
tag e = e'  where  (_, e') = doTag 0 e
```

### Transforms: CodeGen

Now that we have the tags we lets implement our [compilation strategy](#strategy)

```haskell
compile env (If eCond eTrue eFalse i)    
  =   compile env eCond ++
    [ ICmp (Reg EAX) (Const 0)
    , IJe (BranchFalse i)
    ]
   ++ compile env eTrue  ++
    [ IJmp   lExit
    , ILabel (BranchFalse i)
    ]
   ++ compile env eFalse ++
    [ ILabel (BranchExit i) ]
```

### Recap: Branches

* `Tag` each sub-expression,
* Use tag to generate control-flow labels implementing branch.

**Lesson:** Tagged program representation simplifies compilation...

* Next: another example of how intermediate representations help.

## Binary Operations

You know the drill.

1. Build intuition with **examples**,
2. Model problem with **types**,
3. Implement with **type-transforming-functions**,
4. Validate with **tests**.

### Compiling Binary Operations

Lets look at some expressions and figure out how they would get compiled.

* Recall: We want the result to be in `eax` after the instructions finish.


#### Example: Bin1

Lets start with some easy ones. The source:

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/bin-1-to-asm.png" height="120">

**Strategy:** Given `n1 + n2`

* Move `n1` into `eax`,
* Add `n2` to `eax`.

#### Example: Bin2

What if the first operand is a variable?

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/bin-2-to-asm.png" height="200">

Simple, just copy the variable off the stack into `eax`

**Strategy:** Given `x + n`

* Move `x` (from stack) into `eax`,
* Add `n` to `eax`.


#### Example: Bin3

Same thing works if the second operand is a variable.

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/bin-3-to-asm.png" height="275">

Strategy: Given `x + n`

* Move `x` (from stack) into `eax`,
* Add `n` to `eax`.

### Example: Bin4

But what if we have _nested_ expressions

```haskell
(1 + 2) * (3 + 4)
```

* Can compile `1 + 2` with result in `eax` ...
* .. but then need to _reuse_ `eax` for `3 + 4`

Need to **save** `1 + 2` somewhere!

**Idea** How about use _another_ register for `3 + 4`?
* But then what about `(1 + 2) * (3 + 4) * (5 + 6)` ?
* In general, may need to _save_ more sub-expressions than we have registers.

### Idea: Immediate Expressions

Why were `1 + 2` and `x + y` so easy to compile but `(1 + 2) * (3 + 4)` not?

Because `1` and `x` are **immediate expressions**

* Their values don't require any computation,
* Either a constant, or,
* Can be read off the stack.

### Idea: Administrative Normal Form

> An expression is in **Administrative Normal Form (ANF)**
> if all **primitive operations** have **immediate** arguments.

**Primitive Operations:** Those whose values we _need_ for computation to proceed.

* `v1 + v2`
* `v1 - v2`
* `v1 * v2`
* `if v: e1 else: e2`

Examples:

```haskell
2 + 3

let x = 12
in
    x + 1

let x = 12
  , y = 18
in
    x + y

let x = 12
  , y = 18
  , t = x + y
in
  if t: 7 else: 9
```

Unfortunately, the below is _not_ in ANF

```haskell
(1 + 2) * (3 + 4)
```

* As the `*` has _non-immediate_ arguments.

However, note the following variant _is in ANF_

```haskell
let t1 = 1 + 2
  , t2 = 3 + 4
in  
    t1 * t2
```

### Binary Operations: Strategy

We can convert _any_ expression to ANF

* By adding "temporary" variables for sub-expressions

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-anf.png" height="200">

* **Step 1:** Compiling ANF into Assembly             
* **Step 2:** Converting Expressions into ANF

### Types: Source

Lets add binary primitive operators

```haskell
data Prim2
  = Plus | Minus | Times
```

and use them to extend the source language:

```haskell
data Expr a
  = ...
  | Prim2 Prim2  (Expr a) (Expr a) a
```

So, for example, `2 + 3` would be parsed as:

```haskell
Prim2 Plus (Number 2 ()) (Number 3 ()) ()
```

### Types: Assembly

Need to add X86 instructions for primitive arithmetic:

```haskell
data Instruction
  = ...
  | IAdd Arg Arg
  | ISub Arg Arg
  | IMul Arg Arg
```

### Types: ANF

We _can_ define a separate type for ANF (try it!)
* but cumbersome as it requires duplicating a bunch of code.

Instead, lets write a _function_ that describes **immediate expressions**

```haskell
isImm :: Expr a -> Bool
isImm (Number _ _) = True
isImm (Var    _ _) = True
isImm _            = False
```

We can now think of **immediate** expressions as:

> The _subset_ of `Expr` _such that_ `isImm` returns `True`

Similarly, lets write a function that describes **ANF** expressions

```haskell
isAnf :: Expr a -> Bool
isAnf (Number  _     _) = True
isAnf (Var     _     _) = True
isAnf (Prim2 _ e1 e2 _) = isImm e1 && isImm e2
isAnf (If e1 e2 e3   _) = isImm e1 && isAnf e2 && isAnf e3
isAnf (Let x e1 e2   _) = isAnf e1 && isAnf e2
```

We can now think of **ANF** expressions as:

> The _subset_ of `Expr` _such that_ `isAnf` returns `True`

We can use the above to **test** our ANF conversion.


### Types & Strategy

Writing the type aliases:

```haskell
type BareE   = Expr ()
type AnfE    = Expr ()  -- such that isAnf is True
type AnfTagE = Expr Tag -- such that isAnf is True
type ImmTagE = Expr Tag -- such that isImm is True
```  

we get the overall pipeline:

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-anf-types.png" height="200">


### Transforms: Compiling `AnfTagE` to `Asm`

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-anf-to-asm.png" height="200">

The compilation from ANF is easy, lets recall our examples and strategy:

Strategy: Given `v1 + v2` (where `v1` and `v2` are **immediate expressions**)

* Move `v1` into `eax`,
* Add `v2` to `eax`.

```haskell
compile :: Env -> TagE -> Asm
compile env (Prim2 o v1 v2)
  = [ IMov      (Reg EAX) (immArg env v1)
    , (prim2 o) (Reg EAX) (immArg env v2)
    ]
```

where we have a helper to find the `Asm` variant of a `Prim2` operation

```haskell
prim2 :: Prim2 -> Arg -> Arg -> Instruction
prim2 Plus  = IAdd
prim2 Minus = ISub
prim2 Times = IMul
```

and another to convert an _immediate expression_ to an x86 argument:

```haskell
immArg :: Env -> ImmTag -> Arg
immArg _   (Number n _) = Const n
immArg env (Var    x _) = RegOffset ESP i
  where
    i                   = fromMaybe err (lookup x env)
    err                 = error (printf "Error: Variable '%s' is unbound" x)
```

### Transforms: Compiling `Bare` to `Anf`

Next lets focus on **A-Normalization** i.e. transforming expressions into ANF

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-bare-to-anf.png" height="200">

### A-Normalization

We can fill in the base cases easily

```haskell
anf (Number n)      = Number n
anf (Var x)         = Var x
```

Interesting cases are the binary operations

#### Example: Anf-1

Left operand is not immediate

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/anf-1.png" height="200">

**Key Idea: Helper Function**

```haskell
imm :: BareE -> ([(Id, AnfE)], ImmE)
```

`imm e` returns `([(t1, a1),...,(tn, an)], v)` where

* `ti, ai` are new temporary variables bound to ANF exprs,
* `v` is an **immediate value** (either a constant or variable)

Such that `e` is _equivalent to_

```haskell
let t1 = a1
  , ...
  , tn = an
in
   v
```

Lets look at some more examples.

#### Example: Anf-2

Left operand is not internally immediate

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/anf-2.png" height="250">


#### Example: Anf-3

Both operands are not immediate

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/anf-3.png" height="250">


### ANF: General Strategy

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/anf-strategy.png" height="250">

1. **Invoke** `imm` on both the operands
2. **Concat** the `let` bindings
3. **Apply** the binop to the immediate values

### ANF: Implementation

Lets implement the above strategy

```haskell
anf (Prim2 o e1 e2) = lets (b1s ++ b2s)
                        (Prim2 o (Var v1) (Var v2))
  where
    (b1s, v1)       = imm e1
    (b2s, v2)       = imm e2

lets :: [(Id, AnfE)] -> AnfE -> AnfE
lets [] e'         = e
lets ((x,e):bs) e' = Let x e (lets bs e')
```

Same principle applies to `If`

* use `imm` to make the condition _immediate_
* use `anf` to recursively transform the branches.

```haskell
anf (If e1 e2 e3)   = lets bs (If t e1' e2')  
  where
    (bs, t)         = imm e1
    e1'             = anf e1
    e2'             = anf e2
```

Finally, for `Let` just make sure we recursively `anf`
the sub-expressions.

```haskell
anf (Let x e1 e2)   = Let x e1' e2'
  where
    e1'             = anf e1
    e2'             = anf e2
```

### ANF: Making Arguments Immediate via `imm`

The workhorse is the function

```haskell
imm :: BareE -> ([(Id, AnfE)], ImmE)
```

which creates temporary variables to crunch an
arbitrary `Bare` into an _immediate_ value.

No need to create an variables if the expression
is _already_ immediate:

```haskell
imm v@(Number _)    = ( [], v )
imm v@(Id     _)    = ( [], v )
```

The tricky case is when the expression has a primop:

```haskell
imm (Prim2 o e1 e2) = ( b1s ++ b2s ++ [(t,  Prim2 o v1 v2)]
                      , Id t  )
  t                 = FreshVar
  (b1s, v1)         = imm e1
  (b2s, v2)         = imm e2  
```

Oh, what shall we do when:

```haskell
imm (If e1 e2 e3)   = ???
imm (Let x e1 e2)   = ???
```

Lets look at an example for inspiration.

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/anf-4.png" height="200">

That is, simply

* `anf` the relevent expressions,
* bind them to a fresh variable.

```haskell
imm e@(If _ _ _) = immExp e
imm e@(If _ _ _) = immExp e

immExp :: AnfE -> ([(Id, AnfE)], ImmE)
immExp e = [(t, e')], t
  where
    t    = FreshVar
    e'   = anf e
```

### One last thing: Whats up with `FreshVar` ?

Wait a minute, what is this magic **FRESH** ?

How can we create **distinct** names out of thin air?

What's that? Global variables? Increment a counter?

![Get thee behind me Satan!](/resources/dr-evil.jpg)

We will use a counter, but will have to **pass its value around**

* Just like `tag`

```haskell
anf :: Int -> BareE -> (Int, AnfE)

anf i (Number n l)      = (i, Number n l)

anf i (Id     x l)      = (i, Id     x l)

anf i (Let x e b l)     = (i'', Let x e' b' l)
  where
    (i',  e')           = anf i e
    (i'', b')           = anf i' b

anf i (Prim2 o e1 e2 l) = (i'', lets (b1s ++ b2s) (Prim2 o e1' e2' l))
  where
    (i' , b1s, e1')     = imm i  e1
    (i'', b2s, e2')     = imm i' e2

anf i (If c e1 e2 l)    = (i''', lets bs  (If c' e1' e2' l))
  where
    (i'  , bs, c')      = imm i   c
    (i'' ,     e1')     = anf i'  e1
    (i''',     e2')     = anf i'' e2
```

and

```haskell
imm :: Int -> AnfE -> (Int, [(Id, AnfE)], ImmE)

imm i (Number n l)        = (i  , [], Number n l)

imm i (Var x l)           = (i  , [], Var x l)

imm i (Prim2 o e1 e2 l) = (i''', bs, Var v l)
  where
    (i'  , b1s, v1)     = imm i  e1
    (i'' , b2s, v2)     = imm i' e2
    (i''', v)           = fresh i''
    bs                  = b1s ++ b2s ++ [(v, Prim2 o v1 v2 l)]

imm i e@(If _ _ _  l)   = immExp i e

imm i e@(Let _ _ _ l)   = immExp i e

immExp :: Int -> BareE -> (Int, [(Id, AnfE)], ImmE)
immExp i e l  = (i'', bs, Var v ())
  where
    (i' , e') = anf i e
    (i'', v)  = fresh i'
    bs        = [(v, e')]
```

where now, the `fresh` function returns a _new counter_ and a variable

```haskell
fresh :: Int -> (Int, Id)
fresh n = (n+1, "t" ++ show n)
```

**Note** this is super clunky. There _is_ a really slick way to write the above
code without the clutter of the `i` but thats too much of a digression,
[but feel free to look it up yourself][monad-230]

## Recap and Summary

Just created `Boa` with

* Branches (`if`-expressions)
* Binary Operators (`+`, `-`, etc.)

In the process of doing so, we will learned about

* **Intermediate Forms**
* **Normalization**

Specifically,

<img src="https://github.com/ucsd-progsys/131/blob/master/resources/compiler-pipeline-anf-types.png" height="200">

[monad-230]: https://cseweb.ucsd.edu/classes/wi12/cse230-a/lectures/monads.html
