
.. _tblgen-mirpats:

========================
MIR Patterns in TableGen
========================

.. contents::
   :local:


User's Guide
============

This section is intended for developers who want to use MIR patterns in their
TableGen files.

``NOTE``:
This feature is still in active development. This document may become outdated
over time. If you see something that's incorrect, please update it.

Use Cases
---------

MIR patterns are supported in the following places:

* GlobalISel ``GICombineRule``
* GlobalISel ``GICombinePatFrag``

Syntax
------

MIR patterns use the DAG datatype in TableGen.

.. code-block:: text

  (inst operand0, operand1, ...)

``inst`` must be a def which inherits from ``Instruction`` (e.g. ``G_FADD``)
or ``GICombinePatFrag``.

Operands essentially fall into one of two categories:

* immediates

  * untyped, unnamed: ``0``
  * untyped, named: ``0:$y``
  * typed, unnamed: ``(i32 0)``
  * typed, named: ``(i32 0):$y``

* machine operands

  * untyped: ``$x``
  * typed: ``i32:$x``

Semantics:

* A typed operand always adds an operand type check to the matcher.
* There is a trivial type inference system to propagate types.

  * e.g. You only need to use ``i32:$x`` once in any pattern of a
    ``GICombinePatFrag`` alternative or ``GICombineRule``, then all
    other patterns in that rule/alternative can simply use ``$x``
    (``i32:$x`` is redundant).

* A nammed operand's behavior depends on whether the name has been seen before.

  * For match patterns, reusing an operand name checks that the operands
    are identical (see example 2 below).
  * For apply patterns, reusing an operand name simply copies that operand into
    the new instruction (see example 2 below).

Operands are ordered just like they would be in a MachineInstr: the defs (outs)
come first, then the uses (ins).

Patterns are generally grouped into another DAG datatype with a dummy operator
such as ``match``, ``apply`` or ``pattern``.

Finally, any DAG datatype in TableGen can be named. This also holds for
patterns. e.g. the following is valid: ``(G_FOO $root, (i32 0):$cst):$mypat``.
This may also be helpful to debug issues. Patterns are *always* named, and if
they don't have a name, an "anonymous" one is given to them. If you're trying
to debug an error related to a MIR pattern, but the error mentions an anonymous
pattern, you can try naming your patterns to see exactly where the issue is.

.. code-block:: text
  :caption: Pattern Example 1

  // Match
  //    %imp = G_IMPLICIT_DEF
  //    %root = G_MUL %x, %imp
  (match (G_IMPLICIT_DEF $imp),
         (G_MUL $root, $x, $imp))

.. code-block:: text
  :caption: Pattern Example 2

  // using $x twice here checks that the operand 1 and 2 of the G_AND are
  // identical.
  (match (G_AND $root, $x, $x))
  // using $x again here copies operand 1 from G_AND into the new inst.
  (apply (COPY $root, $x))


Limitations
-----------

This a non-exhaustive list of known issues with MIR patterns at this time.

* Matching intrinsics is not yet possible.
* Using ``GICombinePatFrag`` within another ``GICombinePatFrag`` is not
  supported.
* ``GICombinePatFrag`` can only have a single root.
* Instructions with multiple defs cannot be the root of a ``GICombinePatFrag``.
* Using ``GICombinePatFrag`` in the ``apply`` pattern of a ``GICombineRule``
  is not supported.
* Deleting the matched pattern in a ``GICombineRule`` needs to be done using
  ``G_IMPLICIT_DEF`` or C++.
* Replacing the root of a pattern with another instruction needs to be done
  using COPY.
* We cannot rewrite a matched instruction other than the root.
* Matching/creating a (CImm) immediate >64 bits is not supported
  (see comment in ``GIM_CheckConstantInt``)
* There is currently no way to constrain two register/immediate types to
  match. e.g. if a pattern needs to work on both i32 and i64, you either
  need to leave it untyped and check the type in C++, or duplicate the
  pattern.

GICombineRule
-------------

MIR patterns can appear in the ``match`` or ``apply`` patterns of a
``GICombineRule``.

The ``root`` of the rule can either be a def of an instruction, or a
named pattern. The latter is helpful when the instruction you want
to match has no defs. The former is generally preferred because
it's less verbose.

.. code-block:: text
  :caption: Combine Rule root is a def

  // Fold x op 1 -> x
  def right_identity_one: GICombineRule<
    (defs root:$dst),
    (match (G_MUL $dst, $x, 1)),
    // Note: Patterns always need to create something, we can't just replace $dst with $x, so we need a COPY.
    (apply (COPY $dst, $x))
  >;

.. code-block:: text
  :caption: Combine Rule root is a named pattern

  def Foo : GICombineRule<
    (defs root:$root),
    (match (G_ZEXT $tmp, (i32 0)),
           (G_STORE $tmp, $ptr):$root),
    (apply (G_STORE (i32 0), $ptr):$root)>;


Combine Rules also allow mixing C++ code with MIR patterns, so that you
may perform additional checks when matching, or run additional code after
rewriting a pattern.

The following expansions are available for MIR patterns:

* operand names (``MachineOperand &``)
* pattern names (``MachineInstr *`` for ``match``,
  ``MachineInstrBuilder &`` for apply)

.. code-block:: text
  :caption: Example C++ Expansions

  def Foo : GICombineRule<
    (defs root:$root),
    (match (G_ZEXT $root, $src):$mi),
    (apply "foobar(${root}.getReg(), ${src}.getReg(), ${mi}->hasImplicitDef())")>;

Common Pattern #1: Replace a Register with Another
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The 'apply' pattern must always redefine its root.
It cannot just replace it with something else directly.
A simple workaround is to just use a COPY that'll be eliminated later.

.. code-block:: text

  def Foo : GICombineRule<
    (defs root:$dst),
    (match (G_FNEG $tmp, $src), (G_FNEG $dst, $tmp)),
    (apply (COPY $dst, $src))>;

Common Pattern #2: Erasing a Pattern
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As said before, we must always emit something in the 'apply' pattern.
If we wish to delete the matched instruction, we can simply replace its
definition with a ``G_IMPLICIT_DEF``.

.. code-block:: text

  def Foo : GICombineRule<
    (defs root:$dst),
    (match (G_FOO $tmp, $src), (G_BAR $dst, $tmp)),
    (apply (G_IMPLICIT_DEF $dst))>;

If the instruction has no definition, like ``G_STORE``, we cannot use
an instruction pattern in 'apply' - C++ has to be used.

Common Pattern #3: Emitting a Constant Value
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When an immediate operand appears in an 'apply' pattern, the behavior
depends on whether it's typed or not.

* If the immediate is typed, a ``G_CONSTANT`` is implicitly emitted
  (= a register operand is added to the instruction).
* If the immediate is untyped, a simple immediate is added
  (``MachineInstrBuilder::addImm``).

There is of course a special case for ``G_CONSTANT``. Immediates for
``G_CONSTANT`` must always be typed, and a CImm is added
(``MachineInstrBuilder::addCImm``).

.. code-block:: text
  :caption: Constant Emission Examples:

  // Example output:
  //    %0 = G_CONSTANT i32 0
  //    %dst = COPY %0
  def Foo : GICombineRule<
    (defs root:$dst),
    (match (G_FOO $dst, $src)),
    (apply (COPY $dst, (i32 0)))>;

  // Example output:
  //    %dst = COPY 0
  // Note that this would be ill-formed because COPY
  // expects a register operand!
  def Bar : GICombineRule<
    (defs root:$dst),
    (match (G_FOO $dst, $src)),
    (apply (COPY $dst, (i32 0)))>;

  // Example output:
  //    %dst = G_CONSTANT i32 0
  def Bux : GICombineRule<
    (defs root:$dst),
    (match (G_FOO $dst, $src)),
    (apply (G_CONSTANT $dst, (i32 0)))>;

GICombinePatFrag
----------------

``GICombinePatFrag`` is an equivalent of ``PatFrags`` for MIR patterns.
They have two main usecases:

* Reduce repetition by creating a ``GICombinePatFrag`` for common
  patterns (see example 1).
* Implicitly duplicate a CombineRule for multiple variants of a
  pattern (see example 2).

A ``GICombinePatFrag`` is composed of three elements:

* zero or more ``in`` (def) parameter
* zero or more ``out`` parameter
* A list of MIR patterns that can match.

  * When a ``GICombinePatFrag`` is used within a pattern, the pattern is
    cloned once for each alternative that can match.

Parameters can have the following types:

* ``gi_mo``, which is the implicit default (no type = ``gi_mo``).

  * Refers to any operand of an instruction (register, BB ref, imm, etc.).
  * Can be used in both ``in`` and ``out`` parameters.
  * Users of the PatFrag can only use an operand name for this
    parameter (e.g. ``(my_pat_frag $foo)``).

* ``root``

  * This is identical to ``gi_mo``.
  * Can only be used in ``out`` parameters to declare the root of the
    pattern.
  * Non-empty ``out`` parameter lists must always have exactly one ``root``.

* ``gi_imm``

  * Refers to an (potentially typed) immediate.
  * Can only be used in ``in`` parameters.
  * Users of the PatFrag can only use an immediate for this parameter
    (e.g. ``(my_pat_frag 0)`` or ``(my_pat_frag (i32 0))``)

``out`` operands can only be empty if the ``GICombinePatFrag`` only contains
C++ code. If the fragment contains instruction patterns, it has to have at
least one ``out`` operand of type ``root``.

``in`` operands are less restricted, but there is one important concept to
remember: you can pass "unbound" operand names, but only if the
``GICombinePatFrag`` binds it. See example 3 below.

``GICombinePatFrag`` are used just like any other instructions.
Note that the ``out`` operands are defs, so they come first in the list
of operands.

.. code-block:: text
  :caption: Example 1: Reduce Repetition

  def zext_cst : GICombinePatFrag<(outs root:$dst, $cst), (ins gi_imm:$val),
    [(pattern (G_CONSTANT $cst, $val),
              (G_ZEXT $dst, $cst))]
  >;

  def foo_to_impdef : GICombineRule<
   (defs root:$dst),
   (match (zext_cst $y, $cst, (i32 0))
          (G_FOO $dst, $y)),
   (apply (G_IMPLICIT_DEF $dst))>;

  def store_ext_zero : GICombineRule<
   (defs root:$root),
   (match (zext_cst $y, $cst, (i32 0))
          (G_STORE $y, $ptr):$root),
   (apply (G_STORE $cst, $ptr):$root)>;

.. code-block:: text
  :caption: Example 2: Generate Multiple Rules at Once

  // Fold (freeze (freeze x)) -> (freeze x).
  // Fold (fabs (fabs x)) -> (fabs x).
  // Fold (fcanonicalize (fcanonicalize x)) -> (fcanonicalize x).
  def idempotent_prop_frags : GICombinePatFrag<(outs root:$dst, $src), (ins),
    [
      (pattern (G_FREEZE $dst, $src), (G_FREEZE $src, $x)),
      (pattern (G_FABS $dst, $src), (G_FABS $src, $x)),
      (pattern (G_FCANONICALIZE $dst, $src), (G_FCANONICALIZE $src, $x))
    ]
  >;

  def idempotent_prop : GICombineRule<
    (defs root:$dst),
    (match (idempotent_prop_frags $dst, $src)),
    (apply (COPY $dst, $src))>;



.. code-block:: text
  :caption: Example 3: Unbound Operand Names

  // This fragment binds $x to an operand in all of its
  // alternative patterns.
  def always_binds : GICombinePatFrag<
    (outs root:$dst), (ins $x),
    [
      (pattern (G_FREEZE $dst, $x)),
      (pattern (G_FABS $dst, $x)),
    ]
  >;

  // This fragment does not bind $x to an operand in any
  // of its alternative patterns.
  def does_not_bind : GICombinePatFrag<
    (outs root:$dst), (ins $x),
    [
      (pattern (G_FREEZE $dst, $x)), // binds $x
      (pattern (G_FOO $dst (i32 0))), // does not bind $x
      (pattern "return myCheck(${x}.getReg())"), // does not bind $x
    ]
  >;

  // Here we pass $x, which is unbound, to always_binds.
  // This works because if $x is unbound, always_binds will bind it for us.
  def test0 : GICombineRule<
    (defs root:$dst),
    (match (always_binds $dst, $x)),
    (apply (COPY $dst, $x))>;

  // Here we pass $x, which is unbound, to does_not_bind.
  // This cannot work because $x may not have been initialized in 'apply'.
  // error: operand 'x' (for parameter 'src' of 'does_not_bind') cannot be unbound
  def test1 : GICombineRule<
    (defs root:$dst),
    (match (does_not_bind $dst, $x)),
    (apply (COPY $dst, $x))>;

  // Here we pass $x, which is bound, to does_not_bind.
  // This is fine because $x will always be bound when emitting does_not_bind
  def test2 : GICombineRule<
    (defs root:$dst),
    (match (does_not_bind $tmp, $x)
           (G_MUL $dst, $x, $tmp)),
    (apply (COPY $dst, $x))>;
