.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.

.. trac-ticket:: Leave blank. This will eventually be filled with the Trac
                 ticket number which will track the progress of the
                 implementation of the feature.

.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.

.. highlight:: haskell

This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_. **After creating the pull request, edit this file again, update the number in the link, and delete this bold sentence.**

.. contents::

Notes on reStructuredText - delete this section before submitting
==================================================================

The proposals are submitted in reStructuredText format.  To get inline code, enclose text in double backticks, ``like this``.  To get block code, use a double colon and indent by at least one space

::

 like this
 and

 this too

To get hyperlinks, use backticks, angle brackets, and an underscore `like this <http://www.haskell.org/>`_.


Linear Types
============

This proposal introduces a notion of *linear function* to GHC.
Linear functions are regular functions, which guarantee that they will
use their argument exactly once. Whether a function ``f`` is linear or
not is called the *multiplicity* of ``f``. We propose a new language
extension, ``-XLinearTypes``. When turned on, the user can enforce
a given multiplicity for ``f`` using a type annotation.

The theory behind this proposal has been fully developed in a peer
reviewed conference publication that will be presented at POPL'18. See
the `extended version of the paper
<https://arxiv.org/abs/1710.09756>`_.

Motivation
----------

Haskell, along with a few other languages, heralded the notion of
*type safety* into mainstream programming. That is, *well-typed
programs do not go wrong*. Well-typed programs do sometimes crash, or
fail to terminate, but they do not segfault. But the system resources
that these programs manipulate have changing states, need to be
initialized before use and conversely, must be freed in a timely
manner. We want not just type safety in Haskell, but also *resource
safety*. We want well-typed programs that do not go wrong in the sense
that they might still crash, but they do not rewind the state of I/O
resources, these resources are never used before they are initialized,
are guaranteed to be freed by the time control flow exits user defined
scopes, and never used after being freed.

This proposal hits another goal as a side benefit. In Haskell, impure
computations are typically structured as a sequence of steps, be it in
the ``IO`` monad or in ``ST``. The latter in particular serves to
precisely control which effects are possible and the scope within
which they are visible. But using monads to write "locally impure"
computations that still look pure from the outside has an unfortunate
consequence: computations are oversequentialized, making it hard for
the compiler to recover lost opportunities for parallelism.

Linear types enable better solutions to both problems: using types to
guarantee resource safety, and using types to control the scope of
effects without forcing an unnatural sequencing of mutually
independent effects.

The following example illustrates both points. Using linear types, we
express a pure API for mutable array construction (the type ``a ->. b``
is the type of linear functions, ``Unrestricted`` is such that
``Unrestricted a ->. b`` is isomorphic to ``a -> b``):

::

  data MArray a
  data Array a
  newMArray :: Int -> (MArray a ->. Unrestricted b) ->. Unrestricted b
  write :: MArray a ->. (Int, a) -> MArray a
  read :: MArray a ->. Int -> (MArray a, Unrestricted a)
  freeze :: MArray a ->. Unrestricted (Array a)

The types in this interface ensure that values of type ``MArray a``
are always *unique* references to a mutable array. As a consequence,
mutations cannot be observed by the context, because references
aliasing each other is ruled out. Referencial transparency is
preserved.

The two main benefits of this API are:

- reads and writes on distinct arrays are not sequenced. This means
  that the compiler is free to reorder them, e.g. as an optimisation.
  We could go further and introduce `fork-join parallelism
  <https://en.wikipedia.org/wiki/Fork%E2%80%93join_model>`_ primitives
  where disjoint slices can be mutated in parallel, *e.g.* by
  different cores.
- The ``freeze`` function consumes the unique ``MArray`` by turning it
  into a non-unique immutable array. ``freeze`` does not, in fact,
  copy the array, it just changes its (static!) state. In the ``ST``
  implementation of ``MArray``, the primitive is ``unsafeFreeze``
  because it is up to the programmer to promise that they won't ever
  mutate the frozen ``MArray`` again. This shrinks the trusted code
  base (TCB). Or to put it another way: the user can now write more
  efficient code even when keeping to safe primitives only.

Section 5 of the `companion article
<https://arxiv.org/abs/1710.09756>`_ is dedicated to more advanced
examples, such as tracking the typestate of sockets, and using
destination-passing-style to work efficiently with serialised
data. Linear functions are also known to be able to encode
communication protocol (*e.g.* complex RPC calls). Yet an other
envisionned application is in manual memory management: much like in
the array example, pointers to the C heap are enforced to be unique,
so yielding a pure API, and ``free`` consumes the pointer, so that it
cannot be used after ``free``.

Proposed Change Specification
-----------------------------

We introduce a new language extension. Types with a linearity
specification are syntactically legal anywhere in a module if and only
if ``-XLinearTypes`` is turned on.

Definition
~~~~~~~~~~

We say that a function ``f`` is *linear* when ``f u`` is consumed
exactly once implies that ``u`` is *consumed exactly once* (defined
as follows).

- Consuming a value of a data type exactly once means evaluating it to
  head normal form, then consuming its fields exactly once
- Consuming a function exactly once means applying it and consuming
  its result exactly once

*TODO: specify diverging case*

The type of linear function from type ``A`` to type ``B`` is written
``A ->. B`` (see syntax below).

Linearity is a strengthening of the contract that a function must
usually enforce. The regular function type ``A -> B`` will be called
the type of *unrestricted* functions.

Polymorphism
~~~~~~~~~~~~

In order for linear functions and unrestricted functions not to live
in completely distinct worlds, hence avoid code duplication, we
introduce a notion of polymorphism, dubbed multiplicity polymorphism,
over whether a function is linear.

A linear function is said to have multiplicity ``1`` while an
unrestricted function is said to have multiplicity ``ω``. Multiplicity
polymorphic function may have variable multiplicity, *e.g.*

::

  map :: (a ->: p b) -> [a] ->: p [b]

Syntax
~~~~~~

*The syntax in this section is non-definitive, feel free to come up
 with better ideas*

The new primary constructs are: multiplicities and the multiplicity
indexed arrow.

- Multiplicity literal are lexically distinct from type constants to
  avoid collisions. Literals starting with the character ``~`` are
  multiplicity literals

  - Multiplicity ``1`` is written ``~1``
  - Multiplicity ``ω`` is written ``~u`` (for unrestricted) in ASCII
    syntax, and ``~ω`` in Unicode syntax

- Multiplicity variables are type variables of kind ``Multiplicity``.
- We will also need to write sums and products of multiplicities (see
  formalism below)

  - ``p ~+ q``
  - ``p ~* q``

- The multiplicity annotated arrow is written ``a ->: p q``. The type
  constructor is ``(->: p)`` for each multiplicity ``p``.
- In addition, in type annotations in binders, the ``::`` be followed
  by an optional multiplicity. So that ``\ (x :: ~1 A) -> x`` has type
  ``A ->: ~1 A`` (*i.e.* ``A ->. A``), while ``\ (x :: ~u A) -> x`` has
  type ``A ->: ~u A`` (*i.e.* ``A->A``).

The linear and unrestricted arrows are aliases:

- ``(->)`` is an alias for ``(->: ~u)``
- ``(->.)`` (ASCII syntax) and ``(⊸)`` (Unicode syntax) are aliases
  for ``(->: ~1)``

Because of the unicode syntax is based on the syntax from the
published literature, the rest of the proposal will primarily use the
unicode syntax.


Constructors & pattern-matching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Constructors of data types defined with the Haskell 98 syntax

::

  data Foo
    = Bar A B
    | Baz C

Have linear function types, that is ``Bar :: A ->. B ->. Foo``. This
implies that most types in ``base`` (``Maybe``, ``[]``, etc…) have
linear constructors. We also make primivitive tuples ``(,)`` have
linear constructors.

With the GADT syntax, multiplicity of the arrows is honored:

::

  data Foo2 where
    Bar2 :: A ->. B -> C

then ``Bar2 :: A ->. B -> C``

The definition of consuming a value in a data type exactly once must
be refined to take the multiplicities of field into account:
- Consuming a value in a datatype is exactly once means evaluating it
  to head normal form and consuming its *linear* fields exactly once

When pattern macthing a linear argument, linear fields are introduced
as linear variables, and unrestricted fields as unrestricted
variables:

::

  f :: Foo2 ->. A
  f (Bar2 x y) = x  -- y is unrestricted, hence does not need to be consumed


Base
~~~~

Because linear functions are only strengthen the contract of
unrestricted function, a number of functions of ``base`` can get a
more precise type. However, for pedagogical reason, to prevent linear
types from interfering with newcomers' understanding the ``Prelude``,
this proposal introduces a new ``Linear`` namespace to hold the new
types. For instance ``Linear.Prelude`` will export a strengthened
version of ``Prelude``, ``Linear.Data.List`` a strenghtened version of
``Data.List``.

In practice, ``Linear.Prelude`` will contain the actual implementation
while ``Prelude`` will merely re-export the functions strengthening
the types of a few of them (*i.e.* there is no need to duplicate
implementations).

Familiar functions with a new type
++++++++++++++++++++++++++++++++++

*Some important functions are probably still missing here, do propose
 to add more*

Here are functions from ``base`` which are exported in the ``Linear``
namespace, with their types:

- ``($) :: (a ->: p q) -> a ->: p q``
- ``const :: a ->. b -> b``
- ``swap :: (a,b) ->. (b,a)``
- ``flip :: (a ->: p b ->: q -> c) ->. (b ->: q a ->: p ->: c)``
- ``seq :: a -> b ->. b`` (note that the first argument of ``seq``
  cannot be linear as it is only evaluated to head normal forms, it it
  has fields, they are not consumed)
- ``(.) :: (b ->: p c) ->. (a ->: q c) ->. a ->: (p ~* q) c``
- ``map :: (a ->: p b) -> [a] ->: p [b]``
- ``(++) :: [a] ->. [a] ->. [a]``
- ``reverse :: [a] ->. [a]``
- ``trace :: String ->. a ->. a``
- ``error :: String ->. a`` (simplified type)
  - The goal here is to be able to consume and return linear variables
    from the context. This requires a linear variant of ``show``.


New definitions
+++++++++++++++

Note that the type of type class methods cannot be strengthened
without breaking backwards compatibility as a stronger type means
fewer instances. So all type class changes introduce new type classes.

The following list are additional functions for ``Linear.Prelude``:

- ``data Unrestricted a where { Unrestricted :: a -> Unrestricted a
  }`` is the primary way to return an unrestricted value (consuming a
  value of type ``Unrestricted a`` exactly once means evaluating it to
  head normal form)
- A few type classes help navigate between the unrestricted and
  restricted world

  - ``class Dropable a where { drop :: a ->. () }``
  - ``class Dropable a => Dupable a where { dup :: a ->. (a,a) }``
    - The laws of the ``Dupable`` class are duals to those of monoid
  - ``class Dupable a => Movable a where { move :: a ->. Unrestricted a }``

    - ``move`` can be used to define ``drop`` and ``dup``. The laws of
      ``Movable`` state that this redefinition yields the same
      functions.
    - Remark: all first-order data types (``Bool``, ``[]``,
      ``Either``, …) are ``Movable``. *e.g.* the instance for lists
      (ignoring the ``Dropable`` and ``Dupable`` constraint for
      conciseness) :: instance Movable a => Movable [a] where move []
      = Unrestricted [] move (a:l) = case (move a, move l) of
      (Unrestricted a', Unrestricted l') -> Unrestricted (a:l')
    - Primitive data types like ``Int`` are sufficiently like data
      types that they should be ``Movable`` as well. We can make
      ``Int`` movable for free by declaring ``data Int where { Int# ::
      Int# -> Int }`` (*i.e.* a linear variable of type ``Int``
      contains an unrestricted ``Int#``). But it may also make sense
      to export enough primitive to make ``Int#`` movable.

- As mentioned above, ``seq`` is not linear in its first argument. But
  it is easy to define a variant that is, only it requires the first
  argument to be of type ``()``

  ::

    lseq0 :: () ->. b ->. b
    lseq0 () b = b

  This is a common enough idiom to deserve its own ``Linear.Prelude``
  function. For convenience, let us generalise a ``Dropable``
  argument:

  ::

    lseq :: Dropable a => a ->. b ->. b
    lseq a b = lseq0 (drop a) b

- Another extremely common idiom which deserves inclusion in
  ``Linear.Prelude`` is returning a pair of a linear state and an
  unrestricted value: ``(s, Unrestricted a)``. ``Linear.Prelude``
  exports the following data type, abstracting over this pattern:

  ::

     data Res s a where Res :: s ->. a -> Res s a

The interaction of type state and IO (*e.g.* in communication
protocol) is one of the motivations of linear types. In order to make
it convenient to work with, we introduce in ``Linear.IO`` an ``IO``
type in which the multiplicity can vary

::

  data IORes (p :: Multiplicity) a where  -- it should really be an unboxed pair
    IORes :: State# RealWorld ->. a ->: p IORes p a
  type IO p a = State# RealWorld ->. IORes p a

This ``IO`` type does not form a monad, as the multiplicity may change
at every bind, but it fits the following pattern:

::

  class MMonad m where
    return :: a ->:p m p a
    (>>=) :: m p a ->. (a ->: p m q b) ->. m q b

Unresolvesd question: is there useful ``Functor`` and ``Applicative``
variants to add below this monad-like class?

The ``Foldable`` type class is generalised in ``Linear.Data.Foldable``

::

  class Foldable (p :: Multiplicity) (q :: Multiplicity) t where
    foldr :: (a ->: p b ->: q b) -> b ->: q t a ->: p b

Unresolved question: is there a similar notion of ``Traversable``?

New unsafe constructions
++++++++++++++++++++++++

Beyond the fact that ``unsafeCoerce`` can be given a linear type. This
proposal adds a the following unsafe coercions in
``Linear.Unsafe.Coerce``:

- ``unsafeCoerceMultiplicity :: (a ->:p b) ->. (a ->: q b)`` to claim to
  the compiler that the multiplicity of a function can be, in fact,
  strengthened.
- ``unsafeUnrestricted :: a ->. Unrestricted a`` to turn a linear value
  into an unrestricted value, without copy.

Formalism
~~~~~~~~~

This section describes the changes required in Core.

For ease of type-checking, we add another multiplicity: ``~0``
representing definite absence of consumption of an argument (``~0``
has also been used by `Conor McBride
<https://link.springer.com/chapter/10.1007/978-3-319-30936-1_12>`_ to
handle dependent types, which may matter for Dependent Haskell).

The only requirement on multiplicities is that they form a
sup-semi-lattice-ordered semi-ring. That is: there is a sum and a
product with the usual distributivity laws, a (computable) order
compatible with the sum and product, such that each pair of
multiplicities has a (computable) join. Even if there is only three
multiplicities in this proposal, the proposal is structured to allow
future extensions.

Variables are added and multiplied symbolically. Therefore
multiplicity expressions are multi-variate polynomials in the
multiplicity semi-ring.

In Core, every variable is labelled, with its multiplicity (just like
it is with its type prior to this proposal). This multiplicity is used
to infer the multiplicity in the type of functions.

In order to cope with the fact that

::

  fst :: (a, b) -> a
  fst (a, _) = a

is well-typed but

::

  fst :: (a, b) ->. a
  fst (a, _) = a

isn't, ``case`` expressions are annotated with a multiplicity as
well. This multiplicity scales the multiplicity of constructors'
fields. The latter example is elaborated into a ``case_1`` so the
``_`` pattern is linear, which is prohibited, in the former we have a
``case_ω`` so the multiplicity of both fields are scaled by ``ω`` (in
particular ``_`` is an unrestricted pattern) and the expression
typechecks.

This has one consequences: ``case_0`` cannot be allowed, as it would
break the definition of ``~0`` (it would force something which is, by
definition, definitely not consumed, for instance, it would allow
computing the length of a list with multiplicity ``~0``). But we do
want to accept ``case_p`` when ``p`` is a variable. Therefore me must
take the convention that variables never stand for the ``~0``
multiplicity, and in particular that ``~0`` is not a valid argument
for a multiplicity application.

Remark: ``let`` binders are decorated like ``case``, with the
restriction that recursive ``let`` binders are necessarily decorated
with ``ω``.

Effect and Interactions
-----------------------

A staple of this proposal is that it does not modify Haskell for those
who don't want to use it, or don't know of linear types. Even if an
API exports linear types, they are easy to ignore: just imagine that
the arrows are regular arrows, it will work as expected.

Linear data types are just regular Haskell type, which means its cheap
to get interact with existing libraries.

There is one known unpleasant interaction: with
``-XRebindableSyntax``, ``if u then t else e`` is interpreted as
``ifThenElse u t e``. Unfortunately, these two construct have
differrent typing rules when ``t`` and ``e`` have free linear
variables. Therefore well-typed linearly typed programs can stop
typing when ``-XRebindableSyntax`` is added.

Costs and Drawbacks
-------------------

Give an estimate on development and maintenance costs. List how this
effects learnability of the language for novice users. Define and list
any remaining drawbacks that cannot be resolved.

This proposal tries hard to make the changes invisible to newcomers,
however, if many libraries start adopting it, the new function types
will appear in APIs. They can be safely ignore, but they can still be
considered distracting.

There is a an issue that defining newtypes such as

::

  newtype Unrestricted' a where
    Unrestricted' :: a -> Unrestricted' a

Because of laziness which could cause linear values not to be
consumed.


Alternatives
------------

Subtyping instead of polymorphism
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since ``A ->. B`` is a strengthening of ``A -> B``, it is tempting to
make ``A ->. B`` a subtype of ``A -> B``. But subtyping and polymorphism
don't mesh very well, and would yield a significantly more complex
solution.

In general, subtyping and polymorphism are not comparable, and some
examples will work better with one or the other. Therefore it makes
sense to go for the simplest one.

In this proposal

::

  f :: A ->. B

  g :: A -> B
  g = f

is, in theory, ill-typed. But it would be a problem to reject this
program (especially with all the constructors which have been
converted to linear types). So the type inference mechanism elaborates
this program to the well-typed η-expansion

::

  f :: A ->. B

  g :: A -> B
  g x = f x

No annotation on case
~~~~~~~~~~~~~~~~~~~~~

Instead of having ``case_p`` we could just have the regular ``case``
(which would correspond to ``case_1`` in this proposal's
formalism). This simplify the implementation of polymorphism as we
can't risk writing ``case_0``.

On the other hand, doing this loses the principle that linear data
types and unrestricted data types are one and the same. And sacrifices
much code reuse.

Unicity instead of linearity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Languages like Clean and Rust have a variant of linear types called
uniqueness, or ownership, typing. This is a dual notion: instead of
functions guaranteeing that they use their argument exactly once, and
no restriction being imposed on the caller, with uniqueness type, the
caller must guarantee that it has a non-aliased reference to a value,
and the function has no restriction.

Where unicity really shines, is for in-place mutation: the ``write``
function can take a regular ``Array`` as an argument, it just needs to
require that it is unique. Freezing is really easy: just drop the
constraint that the ``Array`` is unique, it will never be writable
again.

With linear types, we need to have two types ``MArray`` (guaranteed
unique) and ``Array``, just like in Haskell today. This is fine when
we are freezing one array: just call ``freeze``. But what if we are
freezing a list of arrays? Do we need to ``map freeze``? This is
unfortunate (the problem is even more complicated if we start
considering ``MArray (MArray a)``). It has a feel of ``Coercible``,
but it does feel harder.

On the other hand, other examples work better with linear types, such
as fork-join parallelism. This is why Rust has a notion of so-called
mutable borrowed reference, on which constraints are more akin to
linear types (or rather, affine types, technically).

Overall, uniqueness type system are significantly more complex to
specify and implement than linear types systems such as this
proposal's.

Linearity-in-kinds
~~~~~~~~~~~~~~~~~~

Instead of adding a type for linear function, we could classify types
in two kinds: one of unrestricted types and one of linear
types. A value of a linear type must be used in a linear fashion.

This would get rid of the continuation of ``newMArray`` in the
motivating ``MArray`` interface.

The most natural way to do this, in Haskell, is to add a second
parameter to ``TYPE`` (the first one is for levity polymorphism). So,
ignoring the levity polymorphism, we would have ``TYPE ~1`` for linear
types and ``TYPE ~u`` for unrestricted type. We get polymorphism by
abstracting over the multiplicity.

As interesting as it is, there is quite some complication associated
to it. First, because of laziness, you can't have a function of type
``(A :: TYPE ~1) -> (B :: TYPE ~u)`` (because you don't need to
consume the result, hence you may not consume an argument that you
have to consume). So what would be the type of the arrow? Something
like ``forall (p :: Multiplicity) (q ⩽ p). p -> q -> q``. So we're
introducing some kind of bounded polymorphism in our story. This is
quite a bit harder than our proposal.

Most types will live in both kinds, but that would have to be
explicit:

::

  data List (p :: Multiplicity) (a :: TYPE p) :: TYPE p where
    [] :: List p a
    (:) :: a -> List p a -> List p a

Mixing non-linear and linear lists (*e.g.* with ``(++)``) would
require either some subtyping from ``List ~u a`` to ``List ~1 a`` (but
as discussed above, subptyping in presence of polymorphism quickly
becomes hairy) or some conversion function.

It it worth taking into account that the issues with ``MArray`` and
``Array`` (which may be ``Array ~1`` and ``Array ~u`` in this case)
above are not solved by such a situation. Unless there is a subptyping
relation from ``Array ~u`` from ``Array ~1``, which cannot be performed
by an explicit function since this would be equivalent to the
proposal's situation.

On the other hand, the CPS interface to ``newMArray`` delimits a scope
in which the array lives. This gives a perfect opportunity to put
clean-up code to react to exceptions. So it may not be such a bad thing
after all.

So linearity in kind seem to add a lot of complication for very little
gain.

On the matter of dependent Haskell, to the best our knowledge, the only
presentations of dependent types with linearity-in-kinds disallow
linear types as arguments of dependent functions.

Future work
~~~~~~~~~~~

Something that hasn't been touched up by this proposal is the idea of
declaring toplevel linear binders

::

  module Foo where
  token :: ~1 A

Here ``token`` would have be consumed exactly once by the program,
this property is a link-time property. This generalised the
``RealWorld`` token which is currently magically inserted in the
``main`` function (the existence of which is checked at link time).

This would allow libraries to abstract on ``main`` or to provide their
own linearly-threaded token.

Unresolved questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.

Syntax
~~~~~~

Nothing in the syntax is fixed, except the unicode notation ``a ->. b``
which is standard from the literature for linear functions. In
particular the syntax for multiplicity literals could be improved.

Inference
~~~~~~~~~

- There is no systematic account of type inference. Can it be made
  predictable when a type annotation is required? For compatibility
  reasons, we want to infer unrestricted arrows conservatively, but
  experience shows that it can result in very surprising type errors.

- In Core, we case is indexed by a multiplicity: ``case_p`` (and
  similarly ``let_p``). In the surface language, we can deduce the
  multiplicity in equations when their is a type annotation.

  ::

    fst :: (a,b) -> a
    fst (a,_) = a    -- this is elaborated as a case_ω

    swap :: (a,b) ->. (b,a)
    swap (a,b) = (b,a)   -- this is elaborated as a case_1

  But what of explicit ``case`` and ``let`` in the surface language? We
  can annotate them with a multiplicity, but it is generally clear from
  the context which multiplicity is meant. So the multiplicity
  annotation really ought to be inferred. The general idea is: if
  their is any linear variable in the scrutiny, then the case must be
  linear, and if there are only unrestricted variables, it can be
  unrestricted. Is it sound to always pick the highest possible value ?
  What if there are multiplicities with variable multiplicity ?

Formalism
~~~~~~~~~

There's one thing I papered over on the formalism: in Core, ``case``
is of the form ``case u as x of { <alternatives> }`` where ``x``
represents the head normal form of ``u``. It is a widely use tool in
Core to Core passes. It is in particular used to implement the default
alternative is a case:

::

  fmap' :: (a -> a) -> Maybe a -> Maybe a
  fmap' (Just x) = Just (f x)
  fmap' y = y

is elaborated into

::

  \f o -> case o as y of { Just x -> Just (f x) ; WILDCARD -> y }

But it is not obvious what to do for linear cases. The following is a
linearity violation as ``y`` in a sense contains ``x`` (basically, you
could define a function ``a ->. (a,a)`` generically with this).

::
  case_1 o as y of { Just x -> Just (x,y) }

So we need a simple (Core needs to stay fairly simple) story for the
``as`` clause of linear cases.

The easiest thing to do would be to mark ``y`` as dead for linear
cases, and make sure it stays dead throughout the optimiser. But this
is not reasonable: it would prevent default cases, which is probably a
bad idea, and anyway not something we can ensure if we have nested
patterns.

Solving this will also help understand how to handle linear view
patterns, the status of which is also unclear.

Base
~~~~

- It would be nice to change the defintion of the ``IO`` proper to be a
  linear function of ``RealWorld``: this would shrink the trusted code
  base, as even functions which have access to the definition of
  ``IO`` are forced to thread the ``RealWorld`` properly.
  But it would require a way to define unboxed tuples with
  unrestricted constructors.
- Is there a useful hierarchy below the ``MMonad`` class above?
- There is a generalisation of the regular monad type class
  parametrised by a multiplicity:

  ::

    class Monad (p :: Multiplicity) m where
      return :: a ->: p m a
      (>>=) :: m a ->: p (a ->: p m b) ->: p m b

   (technically ``Monad ~u`` is a monad in the usual sense, and
   ``Monad ~1`` a monad in the category of linear functions)

   And a corresponding notion of ``Functor`` and ``Applicative``. We
   could use it to define a generalisation of ``Traversable`` as well,
   in the same spirit of ``Fold`` above.
- Should primitive type such as ``Int#`` be ``Movable``?

Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?
