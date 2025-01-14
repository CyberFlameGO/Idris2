.. _sec-views:

*****************************
Views and the “``with``” rule
*****************************

.. warning::

   NOT UPDATED FOR IDRIS 2 YET

Dependent pattern matching
==========================

Since types can depend on values, the form of some arguments can be
determined by the value of others. For example, if we were to write
down the implicit length arguments to ``(++)``, we’d see that the form
of the length argument was determined by whether the vector was empty
or not:

.. code-block:: idris

    (++) : Vect n a -> Vect m a -> Vect (n + m) a
    (++) {n=Z}   []        ys = ys
    (++) {n=S k} (x :: xs) ys = x :: xs ++ ys

If ``n`` was a successor in the ``[]`` case, or zero in the ``::``
case, the definition would not be well typed.

.. _sect-nattobin:

The ``with`` rule — matching intermediate values
================================================

Very often, we need to match on the result of an intermediate
computation. Idris provides a construct for this, the ``with``
rule, inspired by views in ``Epigram`` [#McBridgeMcKinna]_, which takes account of
the fact that matching on a value in a dependently typed language can
affect what we know about the forms of other values. In its simplest
form, the ``with`` rule adds another argument to the function being
defined.

We have already seen a vector filter function. This time, we define it
using ``with`` as follows:

.. code-block:: idris

    filter : (a -> Bool) -> Vect n a -> (p ** Vect p a)
    filter p [] = ( _ ** [] )
    filter p (x :: xs) with (filter p xs)
      filter p (x :: xs) | ( _ ** xs' ) = if (p x) then ( _ ** x :: xs' ) else ( _ ** xs' )

Here, the ``with`` clause allows us to deconstruct the result of
``filter p xs``. The view refined argument pattern ``filter p (x ::
xs)`` goes beneath the ``with`` clause, followed by a vertical bar
``|``, followed by the deconstructed intermediate result ``( _ ** xs'
)``. If the view refined argument pattern is unchanged from the
original function argument pattern, then the left side of ``|`` is
extraneous and may be omitted:

.. code-block:: idris

    filter p (x :: xs) with (filter p xs)
      | ( _ ** xs' ) = if (p x) then ( _ ** x :: xs' ) else ( _ ** xs' )

``with`` clauses can also be nested:

.. code-block:: idris

    foo : Int -> Int -> Bool
    foo n m with (n + 1)
      foo _ m | 2 with (m + 1)
        foo _ _ | 2 | 3 = True
        foo _ _ | 2 | _ = False
      foo _ _ | _ = False

and left hand sides that are the same as their parent's can be skipped by
using ``_`` to focus on the patterns for the most local ``with``. Meaning
that the above ``foo`` can be rewritten as follows:

.. code-block:: idris

    foo : Int -> Int -> Bool
    foo n m with (n + 1)
      _ | 2 with (m + 1)
        _ | 3 = True
        _ | _ = False
      _ | _ = False

If the intermediate computation itself has a dependent type, then the
result can affect the forms of other arguments — we can learn the form
of one value by testing another. In these cases, view refined argument
patterns must be explicit. For example, a ``Nat`` is either even or
odd. If it is even it will be the sum of two equal ``Nat``.
Otherwise, it is the sum of two equal ``Nat`` plus one:

.. code-block:: idris

    data Parity : Nat -> Type where
       Even : {n : _} -> Parity (n + n)
       Odd  : {n : _} -> Parity (S (n + n))

We say ``Parity`` is a *view* of ``Nat``. It has a *covering function*
which tests whether it is even or odd and constructs the predicate
accordingly. Note that we're going to need access to ``n`` at run time, so
although it's an implicit argument, it has unrestricted multiplicity.

.. code-block:: idris

    parity : (n:Nat) -> Parity n

We’ll come back to the definition of ``parity`` shortly. We can use it
to write a function which converts a natural number to a list of
binary digits (least significant first) as follows, using the ``with``
rule:

.. code-block:: idris

    natToBin : Nat -> List Bool
    natToBin Z = Nil
    natToBin k with (parity k)
       natToBin (j + j)     | Even = False :: natToBin j
       natToBin (S (j + j)) | Odd  = True  :: natToBin j

The value of ``parity k`` affects the form of ``k``, because the
result of ``parity k`` depends on ``k``. So, as well as the patterns
for the result of the intermediate computation (``Even`` and ``Odd``)
right of the ``|``, we also write how the results affect the other
patterns left of the ``|``. That is:

- When ``parity k`` evaluates to ``Even``, we can refine the original
  argument ``k`` to a refined pattern ``(j + j)`` according to
  ``Parity (n + n)`` from the ``Even`` constructor definition. So
  ``(j + j)`` replaces ``k`` on the left side of ``|``, and the
  ``Even`` constructor appears on the right side. The natural number
  ``j`` in the refined pattern can be used on the right side of the
  ``=`` sign.

- Otherwise, when ``parity k`` evaluates to ``Odd``, the original
  argument ``k`` is refined to ``S (j + j)`` according to ``Parity (S
  (n + n))`` from the ``Odd`` constructor definition, and ``Odd`` now
  appears on the right side of ``|``, again with the natural number
  ``j`` used on the right side of the ``=`` sign.

Note that there is a function in the patterns (``+``) and repeated
occurrences of ``j`` - this is allowed because another argument has
determined the form of these patterns.

Defining ``parity``
===================

The definition of ``parity`` is a little tricky, and requires some knowledge of
theorem proving (see Section :ref:`sect-theorems`), but for completeness, here
it is:

.. code-block:: idris

    parity : (n : Nat) -> Parity n
    parity Z = Even {n = Z}
    parity (S Z) = Odd {n = Z}
    parity (S (S k)) with (parity k)
      parity (S (S (j + j))) | Even
          = rewrite plusSuccRightSucc j j in Even {n = S j}
      parity (S (S (S (j + j)))) | Odd
          = rewrite plusSuccRightSucc j j in Odd {n = S j}

For full details on ``rewrite`` in particular, please refer to the theorem
proving tutorial, in Section :ref:`proofs-index`.

.. [#McBridgeMcKinna] Conor McBride and James McKinna. 2004. The view from the
       left. J. Funct. Program. 14, 1 (January 2004),
       69-111. https://doi.org/10.1017/S0956796803004829
