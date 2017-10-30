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

This proposal adds a notion of *linear function* to Haskell. Linear functions are regular functions which guarantee that they will use their argument exactly once. Whether a function ``f`` is linear or not is called the *multiplicity* of ``f``. This proposal also include multiplicity polymorphism. Further details and examples can be found in the `companion article <https://arxiv.org/abs/1710.09756>`_.

Motivation
------------

Linear function make it possible to encode invariants which are inaccessible without them. They tend to fall into two categories: "making more things pure" and "typestate".

A fairly typical example is a pure API for mutable array (the type ``a ⊸ b`` is the type of linear functions, ``Unrestricted`` is such that ``Unrestricted a ⊸ b`` is isomorphic to ``a -> b``):

::

  type MArray a
  type Array a
  newMArray :: Int -> (MArray a ⊸ Unrestricted b) ⊸ Unrestricted b
  write :: MArray a ⊸ (Int, a) -> MArray a
  read :: MArray a ⊸ Int -> (MArray a, Unrestricted a)
  freeze :: MArray a ⊸ Unrestricted (Array a)

The gist of this API is that the linear functions ensure that values of type ``MArray a`` are always unique references to a mutable array. As a consequence mutations cannot be observed by the context. Referencial transparency is preserved.

There are a number of benefits to this API
- falling in the category of making more things pure: reads and writes on distinct arrays are not sequenced. This means that the compiler is free to find better optimisation. We could go further and and introduce `fork-join parallelism <https://en.wikipedia.org/wiki/Fork%E2%80%93join_model>`_ primitives where disjoint slices can be mutated in parallel, *e.g.* by different cores.
- falling in the category of typestate: the ``freeze`` function consumes the unique ``MArray`` by turning it into a non-unique immutable array. ``freeze`` does not, in fact, copy the array, it just changes its (static!) state. In the ``ST`` implementation of ``MArray``, the primitive is ``unsafeFreeze`` because it is up to the programmer to promise that they won't ever mutate the frozen ``MArray`` again. This shrinks the code base.

Section 5 of the `companion article <https://arxiv.org/abs/1710.09756>`_ is dedicated to more advanced examples, such as tracking the typestate of sockets, and using destination-passing-style to work efficiently with serialised data. Linear functions are also known to be able to encode communication protocol (*e.g.* complex RPC calls). Yet an other envisionned application is in manual memory management: much like in the array example, pointers to the C heap are enforced to be unique, so yielding a pure API, and ``free`` consumes the pointer, so that it cannot be used after ``free``.

Proposed Change Specification
-----------------------------

Note, however, that this section need not describe details of the implementation of the feature. The proposal is merely supposed to give a conceptual specification of the new feature and its behavior.

The use of linear functions is enabled with the language extension ``-XLinearTypes``.

Definition
~~~~~~~~~~

We say that a function ``f`` is *linear* if when ``f u`` is consumed exactly once, then ``u`` is consumed exactly once. And define consume exactly once as

- Consuming a value of a data type exactly once means evaluating it to head normal form, then consume its fields exactly once
- Consuming a function exactly once means applying it and consuming its result exactly once

*TODO: specify diverging case*

The type of linear function from type ``A`` to type ``B`` is written ``A ⊸ B`` (see syntax below).

Linearity is a strengthening of the contract that a function must usually enforce. The regular function type ``A->B`` will be called the type of *unrestricted* functions.

Polymorphism
~~~~~~~~~~~~

In order for linear functions and unrestricted functions not to live in completely distinct worlds, hence avoid code duplication, we introduce a notion of polymorphism, dubbed multiplicity polymorphism, over whether a function is linear.

A linear function is said to have multiplicity ``1`` while an unrestricted function is said to have multiplicity ``ω``. Multiplicity polymorphic function may have variable multiplicity, *e.g.*

::
  map :: (a ->: p b) -> [a] ->: p [b]

Syntax
~~~~~~

*The syntax in this section is non-definitive, feel free to come up with better ideas*

The new primary constructs are: multiplicities and the multiplicity indexed arrow.

- Multiplicity literal are lexically distinct from type constants to avoid collisions. Literals starting with the character ``~`` are multiplicity literals
  - Multiplicity ``1`` is written ``~1``
  - Multiplicity ``ω`` is written ``~u`` (for unrestricted) in ASCII syntax, and ``~ω`` in Unicode syntax
- Multiplicity variables are type variables of kind ``Multiplicity``.
- We will also need to write sums and products of multiplicities (see formalism below)
  - ``p ~+ q``
  - ``p ~* q``
- The multiplicity annotated arrow is written ``a ->: p q``. The type constructor is ``(->: p)`` for each multiplicity ``p``.
- In addition, in type annotations in binders, the ``::`` be followed by an optional multiplicity. So that ``\ (x :: ~1 A) -> x`` has type ``A ->: ~1 A`` (*i.e.* ``A ⊸ A``), while ``\ (x :: ~u A) -> x`` has type ``A ->: ~u A`` (*i.e.* ``A->A``).

The linear and unrestricted arrows are aliases:
- ``(->)`` is an alias for ``(->: ~u)``
- ``(->.)`` (ASCII syntax) and ``(⊸)`` (Unicode syntax) are aliases for ``(->: ~1)``

Because of the unicode syntax is based on the syntax from the published literature, the rest of the proposal will primarily use the unicode syntax.


Constructors & pattern-matching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Constructors of data types defined with the Haskell 98 syntax

::
  data Foo
    = Bar A B
    | Baz C

Have linear function types, that is ``Bar :: A ⊸ B ⊸ Foo``. This implies that most types in ``base`` (``Maybe``, ``[]``, etc…) have linear constructors. We also make primivitive tuples ``(,)`` have linear constructors.

With the GADT syntax, multiplicity of the arrows is honored:

::
  data Foo2 where
    Bar2 :: A ⊸ B -> C

then ``Bar2 :: A ⊸ B -> C``

The definition of consuming a value in a data type exactly once must be refined to take the multiplicities of field into account:
- Consuming a value in a datatype is exactly once means evaluating it to head normal form and consuming its *linear* fields exactly once

When pattern macthing a linear argument, linear fields are introduced as linear variables, and unrestricted fields as unrestricted variables:

::
  f :: Foo2 ⊸ A
  f (Bar2 x y) = x  -- y is unrestricted, hence does not need to be consumed


Base
~~~~

Because linear functions are only strengthen the contract of unrestricted function, a number of functions of ``base`` can get a more precise type. However, for pedagogical reason, to prevent linear types from interfering with newcomers' understanding the ``Prelude``, this proposal introduces a new ``Linear`` namespace to hold the new types. For instance ``Linear.Prelude`` will export a strengthened version of ``Prelude``, ``Linear.Data.List`` a strenghtened version of ``Data.List``.

In practice, ``Linear.Prelude`` will contain the actual implementation while ``Prelude`` will merely re-export the functions strengthening the types of a few of them (*i.e.* there is no need to duplicate implementations).

Familiar functions with a new type
++++++++++++++++++++++++++++++++++

*Some important functions are probably still missing here, do propose to add more*

Here are functions from ``base`` which are exported in the ``Linear`` namespace, with their types:
- ``($) :: (a ->: p q) -> a ->: p q``
- ``const :: a ⊸ b -> b``
- ``swap :: (a,b) ⊸ (b,a)``
- ``flip :: (a ->: p b ->: q -> c) ⊸ (b ->: q a ->: p ->: c)``
- ``seq :: a -> b ⊸ b`` (note that the first argument of ``seq`` cannot be linear as it is only evaluated to head normal forms, it it has fields, they are not consumed)
- ``(.) :: (b ->: p c) ⊸ (a ->: q c) ⊸ a ->: (p ~* q) c``
- ``map :: (a ->: p b) -> [a] ->: p [b]``
- ``(++) :: [a] ⊸ [a] ⊸ [a]``
- ``reverse :: [a] ⊸ [a]``
- ``trace :: String ⊸ a ⊸ a``
- ``error :: String ⊸ a`` (simplified type)
  - The goal here is to be able to consume and return linear variables from the context. This requires a linear variant of ``show``.


New definitions
+++++++++++++++

Note that the type of type class methods cannot be strengthened without breaking backwards compatibility as a stronger type means fewer instances. So all type class changes introduce new type classes.

The following list are additional functions for ``Linear.Prelude``:

- ``data Unrestricted a where { Unrestricted :: a -> Unrestricted a }`` is the primary way to return an unrestricted value (consuming a value of type ``Unrestricted a`` exactly once means evaluating it to head normal form)
- A few type classes help navigate between the unrestricted and restricted world
  - ``class Dropable a where { drop :: a ⊸ () }``
  - ``class Dropable a => Dupable a where { dup :: a ⊸ (a,a) }``
    - The laws of the ``Dupable`` class are duals to those of monoid
  - ``class Dupable a => Movable a where { move :: a ⊸ Unrestricted a }``
    - ``move`` can be used to define ``drop`` and ``dup``. The laws of ``Movable`` state that this redefinition yields the same functions.
    - Remark: all first-order data types (``Bool``, ``[]``, ``Either``, …) are ``Movable``. *e.g.* the instance for lists (ignoring the ``Dropable`` and ``Dupable`` constraint for conciseness)
      ::
        instance Movable a => Movable [a] where
          move [] = Unrestricted []
          move (a:l) = case (move a, move l) of
            (Unrestricted a', Unrestricted l') -> Unrestricted (a:l')
    - Primitive data types like ``Int`` are sufficiently like data types that they should be ``Movable`` as well. We can make ``Int`` movable for free by declaring ``data Int where { Int# :: Int# -> Int }`` (*i.e.* a linear variable of type ``Int`` contains an unrestricted ``Int#``). But it may also make sense to export enough primitive to make ``Int#`` movable.
- As mentioned above, ``seq`` is not linear in its first argument. But it is easy to define a variant that is, only it requires the first argument to be of type ``()``
  ::
    lseq0 :: () ⊸ b ⊸ b
    lseq0 () b = b

  This is a common enough idiom to deserve its own ``Linear.Prelude`` function. For convenience, let us generalise a ``Dropable`` argument:
  ::
    lseq :: Dropable a => a ⊸ b ⊸ b
    lseq a b = lseq0 (drop a) b
- Another extremely common idiom which deserves inclusion in ``Linear.Prelude`` is returning a pair of a linear state and an unrestricted value: ``(s, Unrestricted a)``. ``Linear.Prelude`` exports the following data type, abstracting over this pattern:
  ::
    data Res s a where
      Res :: s ⊸ a -> Res s a

The interaction of type state and IO (*e.g.* in communication protocol) is one of the motivations of linear types. In order to make it convenient to work with, we introduce in ``Linear.IO`` an ``IO`` type in which the multiplicity can vary
::
  data IORes (p :: Multiplicity) a where  -- it should really be an unboxed pair
    IORes :: State# RealWorld ⊸ a ->: p IORes p a
  type IO p a = State# RealWorld ⊸ IORes p a

This ``IO`` type does not form a monad, as the multiplicity may change at every bind, but it fits the following pattern:
::
  class MMonad m where
    return :: a ->:p m p a
    (>>=) :: m p a ⊸ (a ->: p m q b) ⊸ m q b

Unresolvesd question: is there useful ``Functor`` and ``Applicative`` variants to add below this monad-like class?

The ``Foldable`` type class is generalised in ``Linear.Data.Foldable``
::
  class Foldable (p :: Multiplicity) (q :: Multiplicity) t where
    foldr :: (a ->: p b ->: q b) -> b ->: q t a ->: p b

Unresolved question: is there a similar notion of ``Traversable``?

New unsafe constructions
++++++++++++++++++++++++

Beyond the fact that ``unsafeCoerce`` can be given a linear type. This proposal adds a the following unsafe coercions in ``Linear.Unsafe.Coerce``:

- ``unsafeCoerceMultiplicity :: (a ->:p b) ⊸ (a ->: q b)`` to claim to the compiler that the multiplicity of a function can be, in fact, strengthened.
- ``unsafeUnrestricted :: a ⊸ Unrestricted a`` to turn a linear value into an unrestricted value, without copy.

Formalism
~~~~~~~~~

This section describes the changes required in Core.

For ease of type-checking, we add another multiplicity: ``~0`` representing definite absence of consumption of an argument (``~0`` has also been used by `Conor McBride <https://link.springer.com/chapter/10.1007/978-3-319-30936-1_12>`_ to handle dependent types, which may matter for Dependent Haskell).

The only requirement on multiplicities is that they form a sup-semi-lattice-ordered semi-ring. That is: there is a sum and a product with the usual distributivity laws, a (computable) order compatible with the sum and product, such that each pair of multiplicities has a (computable) join. Even if there is only three multiplicities in this proposal, the proposal is structured to allow future extensions.

Variables are added and multiplied symbolically. Therefore multiplicity expressions are multi-variate polynomials in the multiplicity semi-ring.

In Core, every variable is labelled, with its multiplicity (just like it is with its type prior to this proposal). This multiplicity is used to infer the multiplicity in the type of functions.

In order to cope with the fact that
::
  fst :: (a, b) -> a
  fst (a, _) = a

is well-typed but
::
  fst :: (a, b) ⊸ a
  fst (a, _) = a

isn't, ``case`` expressions are annotated with a multiplicity as well. This multiplicity scales the multiplicity of constructors' fields. The latter example is elaborated into a ``case_1`` so the ``_`` pattern is linear, which is prohibited, in the former we have a ``case_ω`` so the multiplicity of both fields are scaled by ``ω`` (in particular ``_`` is an unrestricted pattern) and the expression typechecks.

This has one consequences: ``case_0`` cannot be allowed, as it would break the definition of ``~0`` (it would force something which is, by definition, definitely not consumed, for instance, it would allow computing the length of a list with multiplicity ``~0``). But we do want to accept ``case_p`` when ``p`` is a variable. Therefore me must take the convention that variables never stand for the ``~0`` multiplicity, and in particular that ``~0`` is not a valid argument for a multiplicity application.

Remark: ``let`` binders are decorated like ``case``, with the restriction that recursive ``let`` binders are necessarily decorated with ``ω``.

Effect and Interactions
-----------------------
Detail how the proposed change addresses the original problem raised in the motivation.

Discuss possibly contentious interactions with existing language or compiler features.

- ``-XRebindableSyntax`` renders ``if`` useless


Costs and Drawbacks
-------------------
Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.

- Exceptions
- What's this weird `->.`
- ``newtype Unrestricted …`` must be prohibited

Alternatives
------------
List existing alternatives to your proposed change as they currently exist and discuss why they are insufficient.

- No annotated case (simpler multiplicity application, but much less polymorphism)
- Unicity
  - Better with freeze a list of array (this issue is exacerbated when freezing an array of array)
  - Would inform a non-aliasing analysis instead of the cardinality analysis
  - Significantly more complex
- Having linearity-in-the-kind
  - Polymorphism in the same way as we do levity polymorphism
  - Many types will have to live in both words (no clear path to that, or very little polymorphism)
  - Doesn't solve the freeze issue (even with no native way to promote )
  - CPS interfaces have the benefit of being able to handle exceptions
  - The only way I know to do dependent type with it (if dependent Haskell eventually exists) is to disallow linear things from being dependent arguments
- Subtyping instead of polymorphism

Future work
~~~~~~~~~~~

- Toplevel linear binders

Unresolved questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.

Syntax
~~~~~~

- Syntax of multiplicity-annotated arrow?
- Syntax of multiplicities

Inference
~~~~~~~~~

- Type annotations
- inference of annotations on case and let

Formalism
~~~~~~~~~

- The seq thing

Base
~~~~

- Should we change the internal representation of ``IO`` (and ``ST``)
- ``MMonad`` hierarchy
- There is another ``Monad`` hierarchy parametrised like ``Fold``
- Related: is there a multiplicity-parametric version of ``Traversable``
- Should ``Int#`` and such be ``Movable``

Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?
