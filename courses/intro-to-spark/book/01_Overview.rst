:code-config:`run_button=False;prove_button=True;accumulate_code=False`

.. _SPARK Overview:

SPARK Overview
=====================================================================

.. role:: ada(code)
   :language: ada

This tutorial is an interactive introduction to the SPARK programming language
and formal verification tools. It does not require you to know any specific
programming language (although going over the :doc:`../../intro-to-ada/index`
first may help) or to have experience in formal verification.

For some of the code snippets presented, you will be able to compile and run
the program and/or run the formal verification tool on the program. These are
available through the buttons labelled:

- `Run`: compile the code with assertions enabled and run the executable just
  produced.
- `Examine`: perform the `flow analysis` stage of formal verification
- `Prove`: perform the `proof` stage of formal verification (which includes
  `flow analysis`)

These code snippets are editable, so you can modify the code and rerun the
tools to see the effect on compilation or analysis. Use the button `Reset` to
restore the initial version of the example.


What is it?
---------------------------------------------------------------------

SPARK refers to two different things:

- a programming language targeted at functional specification and static
  verification, plus
- a set of development and verification tools.

The SPARK language is based on a subset of the Ada language, which is
particularly suited to formal verification, as it is designed for critical
software development.

.. image:: 01_spark_ada.png
   :align: center

Version 2012 of Ada introduced the use of aspects that can be used for
subprogram contracts, and version 2014 of SPARK added its own aspects in order
to aid static analysis.


What do the tools do?
---------------------------------------------------------------------

We shall start by reviewing static verification of your programs, meaning the
verification of the source code directly without compiling or executing
it. This verification involves tools that perform static analysis and this, in
general, can take various forms. It includes, for example, type checking and
visibility rules enforcement, as done by the compiler, as well as more complex
reasoning, such as abstract interpretation, as done by a tool like `CodePeer
<https://www.adacore.com/codepeer>`_ from AdaCore. The tools that come with
SPARK perform two different forms of static analysis:

- The first one is called `flow analysis` and, in general, it is the fastest
  form of analysis. It checks in particular initialization of variables and
  data dependencies between inputs and outputs of subprograms. It can also find
  unused assignments and unmodified variables.

- The second one is called `proof` and it checks in particular absence
  of runtime errors in program execution as well as conformance with
  specifications.


Key Tools
---------------------------------------------------------------------

The tool for formal verification of the SPARK language is called GNATprove. It
checks for conformance with the SPARK subset and performs flow analysis and
proof of the source code. The SPARK language is also supported by several other
tools. In particular, it is fully supported by the `GNAT compiler
<https://www.adacore.com/gnatpro>`_ and by the `GPS integrated development
environment <https://www.adacore.com/gnatpro/toolsuite/gps>`_.


A trivial example
---------------------------------------------------------------------

We will now look at a simple example of subprogram in Ada that also uses
SPARK aspects to specify a verifiable subprogram contract. The subprogram
called ``Increment`` adds 1 to the value of its parameter ``X``:

.. code:: ada spark-report-all

   procedure Increment
     (X : in out Integer)
   with
     Global  => null,
     Depends => (X => X),
     Pre     => X < Integer'Last,
     Post    => X = X'Old + 1;

   procedure Increment
     (X : in out Integer)
   is
   begin
     X := X + 1;
   end Increment;

Several properties are specified on this subprogram using the contracts
shown:

- The SPARK Global aspect specifies that ``Increment`` does not read
  and does not write any global variable.

- The SPARK Depend aspect is especially interesting for security, as it
  specifies that the value of the parameter ``X`` after the call only depends
  on the value of ``X`` before the call.

- Functional properties of Increment are specified using the :ada:`Pre` and
  :ada:`Post` aspects of Ada.

   - Increment can only be called if the value of ``X`` before the call is
     smaller that :ada:`Integer'Last`. It is necessary to ensure that the
     addition operation performed in the subprogram body will also not
     overflow.

   - Finally, we specify that ``Increment`` does indeed perform an increment of
     ``X``, that is, the value of ``X`` after a call is one more than its value
     before the call.

GNATprove can verify all of these contracts. It additionally makes sure that no
error may be raised at runtime when executing ``Increment``'s body.


The Programming Language
---------------------------------------------------------------------

At this point it helps to understand the rationale behind the differences
between the SPARK and Ada languages. The aim while designing the SPARK
subset of Ada was to create the biggest possible subset still amenable to
easy specification and sound verification.

The most notable exclusions include access types and allocators, as well as
handling of exceptions, which are both known to increase considerably the
amount of required user-written annotations. Goto statements and
controlled types are also not supported as they introduce non-trivial
control flow. The two remaining restrictions are side-effects in
expressions and aliasing of names, which we will now look at in more
detail.


Limitations
---------------------------------------------------------------------

No side-effects in expressions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The SPARK language does not support side-effects in expressions, that is,
evaluating a SPARK expression cannot update any object. This limitation is
necessary to avoid unpredictable behavior depending on order of
evaluation, parameter passing mechanism, or compiler optimizations. The
expression below for ``G`` is non-deterministic due to the order in which
the two calls to F are evaluated, and is therefore not legal SPARK.

.. code:: ada spark-flow run_button
    :class: ada-expect-compile-error

    procedure Show_Illegal_Ada_Code is

       function F (X : in out Integer) return Integer is
          Tmp : constant Integer := X;
       begin
          X := X + 1;
          return Tmp;
       end F;

       G : Integer := 0;

    begin
       G := F (G) - F (G); -- ??
    end Show_Illegal_Ada_Code;

In fact, the code above is not even legal Ada, so the same error is generated
by the GNAT compiler. But SPARK goes further and GNATprove issues also an error
on the following equivalent code that is accepted by the compiler:

.. code:: ada spark-flow run_button

    procedure Show_Illegal_SPARK_Code is

       G : Integer := 0;

       function F return Integer is
          Tmp : constant Integer := G;
       begin
          G := G + 1;
          return Tmp;
       end F;

    begin
       G := F - F; -- ??
    end Show_Illegal_SPARK_Code;

This is enforced in SPARK by forbidding side-effects in functions, which
include updates to both parameters and global variables. As a consequence,
SPARK forbids functions with :ada:`out` or :ada:`in out` parameters, as well as
functions updating a global variable. Thus function ``F`` below is illegal in
SPARK, while function ``Incr`` might be legal if it does not update any global
variables, and function ``Incr_And_Log`` might be illegal if it updates global
variables for logging.

.. code-block:: ada

    function F (X : in out Integer) return Integer;     -- Illegal

    function Incr (X : Integer) return Integer;         -- OK?

    function Incr_And_Log (X : Integer) return Integer; -- OK?

In most cases, these functions can easily be replaced by procedures with an
:ada:`out` parameter for returning the computed value.

When it has access to their body, GNATprove verifies which functions are indeed
free from side-effects. Here for example, the two functions ``Incr`` and
``Incr_And_Log`` have the same signature, but only ``Incr`` is legal in SPARK
while ``Incr_And_Log`` is not as it attempts to update the global variable
``Call_Count``.

.. code:: ada spark-flow

    package Side_Effects is

       function Incr (X : Integer) return Integer;         -- OK?

       function Incr_And_Log (X : Integer) return Integer; -- OK?

    end Side_Effects;

    package body Side_Effects is

       function Incr (X : Integer) return Integer
       is (X + 1); -- OK

       Call_Count : Natural := 0;

       function Incr_And_Log (X : Integer) return Integer is
       begin
          Call_Count := Call_Count + 1; -- Illegal
          return X + 1;
       end Incr_And_Log;

    end Side_Effects;

No aliasing of names
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another restriction imposed in the SPARK subset concerns `aliasing
<https://en.wikipedia.org/wiki/Aliasing_(computing)>`_. We say that two names
are aliased if they refer to the same object. There are two reasons to forbid
aliasing in SPARK:

- First, it makes verification more difficult as it requires taking into
  account the fact that updates to two variables with different names may in
  fact update the same object.

- Then, results may seem unexpected from a user point of view. Indeed, when its
  parameters are aliased, the results of a subprogram call may depend on
  compiler specific treatment, like parameter passing mechanisms.

Since access types (`pointers
<https://en.m.wikipedia.org/wiki/Pointer_(computer_programming)>`_ in Ada) are
not allowed in SPARK, aliasing can only occur as part of the parameter passing
in a subprogram call. As functions have no side-effects in SPARK, aliasing of
parameters in function calls is not problematic, so we only need to consider
procedure calls. When a procedure is called, SPARK makes sure that no
:ada:`out` or :ada:`in out` parameter is aliased with either another parameter
of the procedure or a global variable updated in the procedure's body.

Procedure ``Move_To_Total`` shows an example where the possibility of aliasing
was not taken into account by the programmer:

.. code:: ada run_button spark-flow
    :class: ada-run-expect-failure

    procedure No_Aliasing is

       Total : Natural := 0;

       procedure Move_To_Total (Source : in out Natural)
         with Post => Total = Total'Old + Source'Old and Source = 0
       is
       begin
          Total  := Total + Source;
          Source := 0;
       end Move_To_Total;

       X : Natural := 3;

    begin
       Move_To_Total (X);         -- OK
       pragma Assert (Total = 3); -- OK
       Move_To_Total (Total);     -- flow analysis error
       pragma Assert (Total = 6); -- runtime error
    end No_Aliasing;

The example subprogram ``Move_To_Total`` shown here increases the global
variable ``Total`` of the value of its input parameter ``Source``. It then
resets ``Source`` to 0. Here obviously, the programmer has not taken into
account the possibility of an aliasing between ``Total`` and ``Source``.  This
is common practice. This subprogram is valid SPARK, and, for its verification,
GNATprove assumes, like the programmer, non-aliasing between ``Total`` and
``Source``. To ensure that this assumption is correct, GNATprove will then
check for non-aliasing on every call to ``Move_To_Total``. The final call to
``Move_To_Total`` in procedure ``No_Aliasing`` violates this property, which
leads to both a message from GNATprove and a runtime error (assertion violation
corresponding to the expected increase in ``Total`` from calling
``Move_To_Total``) when compiling and running. Note that the postcondition of
``Move_To_Total`` is not violated on this second call, as integer parameters
are passes by copy, and the postcondition is checked here before the copy-back
from formal parameters to actual arguments.

Identifying SPARK Code
---------------------------------------------------------------------

The SPARK language has been restricted to only allow easily specifiable and
verifiable constructs. However, sometimes, a user cannot or does not want to
abide by these limitations on all her code base. Therefore, the SPARK tools
only check conformance to the SPARK subset on code which is identified as being
in SPARK.

This can be done using an aspect named :ada:`SPARK_Mode`. If not explicitly
specified, :ada:`SPARK_Mode` is `Off`, which means that the code can use the
complete set of features from Ada and as a result should not be analyzed by
GNATprove. This default can be changed either selectively on some units or some
subprograms/packages inside units, or globally using a configuration pragma
like we're doing in this tutorial. To allow easy reuse of existing Ada
libraries, entities declared in imported units with no explicit
:ada:`SPARK_Mode` can still be used from SPARK code. The tool will only check
for SPARK conformance on the declaration of those entities which are
effectively used within the SPARK code.

Here is a common case of use of the :ada:`SPARK_Mode` aspect:

.. code-block:: ada

    package P
      with SPARK_Mode => On
    is
       -- package spec is IN SPARK, so can be used by SPARK clients
    end P;

    package body P
      with SPARK_Mode => Off
    is
       -- body is NOT IN SPARK, so ignored by GNATprove
    end P;

The package ``P`` only defines entities whose specifications are in the
SPARK subset. However, it can use all Ada features in its body which,
therefore, should not be analyzed and have the :ada:`SPARK_Mode` aspect
set to `Off`.

:ada:`SPARK_Mode` can be specified in a fine-grained manner on a per-unit
basis. More precisely, a package has four different parts: the visible and
private parts of its specification, as well as the declarative and
statement part of its body. On each of these parts, :ada:`SPARK_Mode` can
be specified to be either `On` or `Off`. In the same way, a subprogram has
two parts: its specification and its body.

A general rule in SPARK is that when :ada:`SPARK_Mode` has been set to
`Off`, it can never be switched to `On` again. This prevents both setting
:ada:`SPARK_Mode` to `On` on subunits of a unit with :ada:`SPARK_Mode`
`Off` and switching back to :ada:`SPARK_Mode` `On` on a part of a given
unit when a previous part had been set to `Off`.


Code Examples / Pitfalls
---------------------------------------------------------------------

Example #1
~~~~~~~~~~

Here is a package defining an abstract stack type (defined as a private type in
SPARK) of ``Element`` objects along with some subprograms providing the usual
functionalities over stacks. It is marked to be in the SPARK subset.

.. code:: ada spark-flow

    package Stack_Package
      with SPARK_Mode => On
    is
       type Element is new Natural;
       type Stack is private;

       function Empty return Stack;
       procedure Push (S : in out Stack; E : Element);
       function Pop (S : in out Stack) return Element;

    private
       type Stack is record
          Top : Integer;
          --  ...
       end record;

    end Stack_Package;

Side-effects in expressions are not allowed in SPARK. Therefore, ``Pop``
is not allowed to modify its parameter ``S``.


Example #2
~~~~~~~~~~

Let's turn to an abstract state machine version of stack, where the unit
provides a single instance of stack. The content of the stack (global variables
``Content`` and ``Top``) is not directly visible to clients. On this stripped
down version, only the function ``Pop`` is available to clients. The unit spec
and body are marked to be in the SPARK subset.

.. code:: ada spark-flow

    package Global_Stack
      with SPARK_Mode => On
    is
       type Element is new Integer;

       function Pop return Element;

    end Global_Stack;

    package body Global_Stack
      with SPARK_Mode => On
    is
       Max : constant Natural := 100;
       type Element_Array is array (1 .. Max) of Element;

       Content : Element_Array;
       Top     : Natural;

       function Pop return Element is
          E : constant Element := Content (Top);
       begin
          Top := Top - 1;
          return E;
       end Pop;

    end Global_Stack;

Like previously, functions should be free from side-effects. Here, ``Pop``
updates the global variable ``Top``, which is not allowed in SPARK.


Example #3
~~~~~~~~~~

We now consider two procedures ``Permute`` and ``Swap``. ``Permute``
applies a circular permutation to the value of its three parameters.
``Swap`` then uses ``Permute`` to swap the value of ``X`` and ``Y``.

.. code:: ada run_button spark-flow
    :class: ada-expect-compile-error

    package P
      with SPARK_Mode => On
    is
       procedure Permute (X, Y, Z : in out Positive);
       procedure Swap (X, Y : in out Positive);
    end P;

    package body P
      with SPARK_Mode => On
    is
       procedure Permute (X, Y, Z : in out Positive) is
          Tmp : constant Positive := X;
       begin
          X := Y;
          Y := Z;
          Z := Tmp;
       end Permute;

       procedure Swap (X, Y : in out Positive) is
       begin
          Permute (X, Y, Y);
       end Swap;
    end P;

    with P; use P;

    procedure Test_Swap
      with SPARK_Mode => On
    is
       A : Integer := 1;
       B : Integer := 2;
    begin
       Swap (A, B);
    end Test_Swap;

Here, in the call to ``Permute``, actual values for parameters ``Y`` and ``Z``
are aliased, which is not allowed in SPARK. In fact, in this particular case,
this is even a violation of Ada rules so the same error is issued by the
compiler.

On this example, we see the reason why aliasing is not allowed in
SPARK. Indeed, since ``Y`` and ``Z`` are :ada:`Positive`, they are passed by
copy, and the result of the call to ``Permute`` therefore depends on the order
in which they are copied back after the call.


Example #4
~~~~~~~~~~

Here, the ``Swap`` procedure is used to swap the value of the two record
components of ``R``.

.. code:: ada spark-flow

    package P
      with SPARK_Mode => On
    is
       type Rec is record
          F1 : Positive;
          F2 : Positive;
       end record;

       procedure Swap_Fields (R : in out Rec);
       procedure Swap (X, Y : in out Positive);
    end P;

    package body P
      with SPARK_Mode => On
    is
       procedure Swap (X, Y : in out Positive) is
          Tmp : constant Positive := X;
       begin
          X := Y;
          Y := Tmp;
       end Swap;

       procedure Swap_Fields (R : in out Rec) is
       begin
          Swap (R.F1, R.F2);
       end Swap_Fields;

    end P;

This code is correct. The call to ``Swap`` is safe, as two different
components of the same record object cannot refer to the same object.


Example #5
~~~~~~~~~~

Here is a slight modification of the previous example using an array
instead of a record. ``Swap_Indexes`` uses ``Swap`` on values stored in
the array ``A``.

.. code:: ada spark-flow

    package P
      with SPARK_Mode => On
    is
       type P_Array is array (Natural range <>) of Positive;

       procedure Swap_Indexes (A : in out P_Array; I, J : Natural);
       procedure Swap (X, Y : in out Positive);
    end P;

    package body P
      with SPARK_Mode => On
    is
       procedure Swap (X, Y : in out Positive) is
          Tmp : constant Positive := X;
       begin
          X := Y;
          Y := Tmp;
       end Swap;

       procedure Swap_Indexes (A : in out P_Array; I, J : Natural) is
       begin
          Swap (A (I), A (J));
       end Swap_Indexes;

    end P;

GNATprove detects a possible aliasing. Unlike the previous example, we have no
way here to know that the two elements ``A (I)`` and ``A (J)`` really are
distinct when we call ``Swap``. Note that GNATprove issues a check message here
instead of an error, so the user has the possibility to justify the message
after review.


Example #6
~~~~~~~~~~

We now consider a package declaring a type ``Dictionary``, which is an array
containing a word per letter. The procedure ``Store`` allows to insert a word
at the correct index in a dictionary.

.. code:: ada spark-flow

    package P
      with SPARK_Mode => On
    is
       subtype Letter is Character range 'a' .. 'z';
       type String_Access is access String;
       type Dictionary is array (Letter) of String_Access;

       procedure Store (D : in out Dictionary; W : String);
    end P;

    package body P
      with SPARK_Mode => On
    is
       procedure Store (D : in out Dictionary; W : String) is
          First_Letter : constant Letter := W (W'First);
       begin
          D (First_Letter) := new String'(W);
       end Store;
    end P;

This code is not correct, as access types are not part of the SPARK
subset. In this case, they are really useful though, as, without them, we
cannot store arbitrarily long strings into an array. The solution here is
to use :ada:`SPARK_Mode` to separate parts of the access type from the
rest of the code in a fine grained manner.


Example #7
~~~~~~~~~~

Here is a modified version of the previous example. It has been adapted to hide
the access type inside the private part of package ``P``, using ``pragma
SPARK_Mode (Off)`` at the start of the private part.

.. code:: ada spark-flow

    package P
      with SPARK_Mode => On
    is
       subtype Letter is Character range 'a' .. 'z';
       type String_Access is private;
       type Dictionary is array (Letter) of String_Access;

       function New_String_Access (W : String) return String_Access;

       procedure Store (D : in out Dictionary; W : String);

    private
       pragma SPARK_Mode (Off);

       type String_Access is access String;

       function New_String_Access (W : String) return String_Access is
         (new String'(W));
    end P;

As the access type is defined and used inside of a part of the code ignored by
GNATprove, this code is correct.


Example #8
~~~~~~~~~~

Let's put together the new spec for package ``P`` with the body of ``P`` seen
previously.

.. code:: ada spark-flow
    :class: ada-expect-compile-error

    package P
      with SPARK_Mode => On
    is
       subtype Letter is Character range 'a' .. 'z';
       type String_Access is private;
       type Dictionary is array (Letter) of String_Access;

       function New_String_Access (W : String) return String_Access;

       procedure Store (D : in out Dictionary; W : String);

    private
       pragma SPARK_Mode (Off);

       type String_Access is access String;

       function New_String_Access (W : String) return String_Access is
         (new String'(W));
    end P;

    package body P
      with SPARK_Mode => On
    is
       procedure Store (D : in out Dictionary; W : String) is
          First_Letter : constant Letter := W (W'First);
       begin
          D (First_Letter) := New_String_Access (W);
       end Store;
    end P;

Although the body of ``Store`` really uses no construct that are out of the
SPARK subset, it is not possible to set :ada:`SPARK_Mode` to ``On`` on ``P``'s
body. Indeed, even if we don't use it, we have the visibility here on ``P``'s
private part which is not in SPARK.


Example #9
~~~~~~~~~~

Here, we have moved the declaration and the body of the procedure ``Store`` to
another package named ``Q``.

.. code:: ada spark-flow

    package P
      with SPARK_Mode => On
    is
       subtype Letter is Character range 'a' .. 'z';
       type String_Access is private;
       type Dictionary is array (Letter) of String_Access;

       function New_String_Access (W : String) return String_Access;

    private
       pragma SPARK_Mode (Off);

       type String_Access is access String;

       function New_String_Access (W : String) return String_Access is
         (new String'(W));
    end P;

    with P; use P;
    package Q
      with SPARK_Mode => On
    is
       procedure Store (D : in out Dictionary; W : String);
    end Q;

    package body Q
      with SPARK_Mode => On
    is
       procedure Store (D : in out Dictionary; W : String)  is
          First_Letter : constant Letter := W (W'First);
       begin
          D (First_Letter) := New_String_Access (W);
       end Store;
    end Q;

Here everything is fine. We have managed to retain the use of the access
type while having most of our code in the SPARK subset, so that GNATprove
will be able to analyze it.


Example #10
~~~~~~~~~~~

We now consider two functions searching for the value 0 inside an array
``A``. The first one raises an exception if 0 is not found in ``A`` while the
other simply returns 0 in that case.

.. code:: ada

    package P
      with SPARK_Mode => On
    is
       type N_Array is array (Positive range <>) of Natural;
       Not_Found : exception;

       function Search_Zero_P (A : N_Array) return Positive;

       function Search_Zero_N (A : N_Array) return Natural;
    end P;

    package body P
      with SPARK_Mode => On
    is
       function Search_Zero_P (A : N_Array) return Positive is
       begin
          for I in A'Range loop
             if A (I) = 0 then
                return I;
             end if;
          end loop;
          raise Not_Found;
       end Search_Zero_P;

       function Search_Zero_N (A : N_Array) return Natural
         with SPARK_Mode => Off is
       begin
          return Search_Zero_P (A);
       exception
          when Not_Found => return 0;
       end Search_Zero_N;
    end P;

This code is perfectly correct, despite the use of exception handling. Indeed,
this non-SPARK feature is carefully isolated in a function body marked with a
``SPARK_Mode`` of ``Off``, so that it is ignored by GNATprove. Remark that
GNATprove will try to demonstrate that ``Not_Found`` will never be raised in
``Search_Zero_P``, leading to a message about a possible exception being
raised. Looking at ``Search_Zero_N``, it is indeed likely that an exception is
meant to be raised in some cases, which means that the user needs to verify
that ``Not_Found`` is only raised when appropriate by other means like review
or testing.
