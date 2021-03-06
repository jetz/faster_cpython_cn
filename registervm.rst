.. _registervm:

+++++++++++++++++++++++++++++++++++++++++
Register-based Virtual Machine for Python
+++++++++++++++++++++++++++++++++++++++++

Intro
=====

registervm is a fork of CPython 3.3 using register-based bytecode, instead of
stack-code bytecode

More information: `REGISTERVM.txt
<http://hg.python.org/sandbox/registervm/file/tip/REGISTERVM.txt>`_

Thread on the Python-Dev mailing list: `Register-based VM for CPython
<https://mail.python.org/pipermail/python-dev/2012-November/122777.html>`_.

The project was created in November 2012.


Status
======

 * Most instructions using the stack are converted to instructions using
   registers
 * Bytecode using registers with all optimizations enable is usually 10% faster
   than bytecode using the stack, according to pybench
 * registervm generates invalid code, see TODO section below, so it's not
   possible yet to use it on the Python test suite


TODO
====

Bugs
----

 * Register allocator doesn't handle correctly conditional branches: CLEAR_REG
   is removed on the wrong branch in test_move_instr.
 * Fail to track the stack state in if/else. Bug hidden by the register
   allocator in the following example:

        def func(obj):
            obj.attr = sys.modules['warnings'] if module is None else module

 * Don't move globals out of if. Only out of loops? subprocess.py::

        if mswindows:
            if p2cwrite != -1:
                p2cwrite = msvcrt.open_osfhandle(p2cwrite.Detach(), 0)

   But do move len() out of loop for::

       def loop_move_instr():
           length = 0
           for i in range(5):
               length += len("abc") - 1
           return length

 * Don't remove duplicate LOAD_GLOBAL in
   "LOAD_GLOBAL ...; CALL_PROCEDURE ...; LOAD_GLOBAL ...":
   CALL_PROCEDURE has border effect

 * Don't remove duplicate LOAD_NAME if a function has a border effect::

        x=1
        def modify():
            global x
            x = 2
        print(x)
        modify()
        print(x)

Improvments
-----------

 * Move LOAD_CONST out of loops: it was done in a previous version, but the
   optimization was broken by the introduction of CLEAR_REG
 * Copy constants to the frame objects so constants can be used as registers
   and LOAD_CONST instructions can be simplify removed
 * Enable move_load_const by default?
 * Fix moving LOAD_ATTR_REG: only do that when calling methods.
   See test_sieve() of test_registervm: primes.append().
   ::

       result = Result()
       while 1:
           if result.done:
               break
           func(result)

 * Reenable merging duplicate LOAD_ATTR
 * Register allocation for locale_alias = {...} is very very slow
 * "while 1: ... return" generates useless SETUP_LOOP
 * Reuse locals?
 * implement register version of the following instructions:

   - DELETE_ATTR
   - try/finally
   - yield from
   - CALL_FUNCTION_VAR_KW
   - CALL_FUNCTION_VAR
   - operators: ``a | b, a & b, a ^ b, a |= b, a &= b, a ^= b``

 * DEREF:

   - add a test using free variables
   - Move LOAD_DEREF_REG out of loops

 * NAME:

   - test_list_append() of test_registervm.py
   - Move LOAD_NAME_REG out of loop

 * Handle JUMP_IF_TRUE_OR_POP: see test_getline() of test_registervm
 * Compute the number of used registers in a frame
 * Write a new test per configuration option
 * Factorize code processing arg_types, ex: disassmblers of dis and registervm modules
 * Add tests on class methods
 * Fix lnotab


Changelog
=========

2012-12-21

 * Use RegisterTracker to merge duplicated LOAD, STORE_GLOBAL/LOAD_GLOBAL
   are now also simplified

2012-12-19

 * Emit POP_REG to simplify the stack tracker

2012-12-18

 * LOAD are now only moved out of loops

2012-12-14

 * Duplicated LOAD instructions can be merged without moving them
 * Rewrite the stack tracker: PUSH_REG don't need to be moved anymore
 * Fix JUMP_IF_TRUE_OR_POP/JUMP_IF_FALSE_OR_POP to not generate invalid code
 * Don't move LOAD_ATTR_REG out of try/except block

2012-12-11

 * Split instructions into linked-blocks

2012-11-26

 * Add a stack tracker

2012-11-20

 * Remove useless jumps
 * CALL_FUNCTION_REG and CALL_PROCEDURE_REG are fully implemented

2012-10-29

 * Remove "if (HAS_ARG(op))" check in PyEval_EvalFrameEx()

2012-10-27

 * Duplicated LOAD_CONST and LOAD_GLOBAL are merged (optimization disabled on
   LOAD_GLOBAL because it is buggy)

2012-10-23

 * initial commit, 0f7f49b7083c



CPython 3.3 bytecode is inefficient
===================================

 * Useless jump: JUMP_ABSOLUTE <offset+0>
 * Generate dead code: RETURN_VALUE; RETURN_VALUE (the second instruction is unreachable)
 * Duplicate constants: see TupleSlicing of pybench
 * Constant folding: see astoptimizer project
 * STORE_NAME 'f'; LOAD_NAME 'f'
 * STORE_GLOBAL 'x'; LOAD_GLOBAL 'x'


Rationale
=========

The performance of the loop evaluating bytecode is critical in Python. For
Python example, using computed-goto instead of switch to dispatch bytecode
improved performances by 20%. Related issues:

 * `use computed goto's in ceval loop <http://bugs.python.org/issue1408710>`_
 * `Faster opcode dispatch on gcc <http://bugs.python.org/issue4753>`_
 * `Computed-goto patch for RE engine <http://bugs.python.org/issue7593>`_

Using registers of a stack reduce the number of operations, but increase the
size of the code. I expect an significant speedup when all operations will use
registers.


Optimizations
=============

Optimizations:

 * Remove useless LOAD_NAME and LOAD_GLOBAL.
   For example: "STORE_NAME var; LOAD_NAME var"
 * Merge duplicate loads (LOAD_CONST, LOAD_GLOBAL_REG, LOAD_ATTR).
   For example, "lst.append(1); lst.append(1)" only gets constant "1" and the
   "lst.append" attribute once.

Misc:

 * Automatically detect inplace operations. For example, "x = x + y" is
   compiled to "BINARY_ADD_REG 'x', 'x', 'y'" which calls
   PyNumber_InPlaceAdd(), instead of PyNumber_Add().
 * Move constant, global and attribute loads out of loops (to the beginning)
 * Remove useless jumps (ex: JUMP_FORWARD <relative jump to 103 (+0)>)


Algorithm
=========

The current implementation rewrites the stack-based operations to use
register-based operations instead. For example, "LOAD_GLOBAL range" is replaced
with "LOAD_GLOBAL_REG R0, range; PUSH_REG R0". This first step is inefficient
because it increases the number of operations.

Then, operations are reordered: PUSH_REG and POP_REG to the end. So we can
replace "PUSH_REG R0; PUSH_REG R1; STACK_OPERATION; POP_REG R2" with a single
operatiton: "REGISTER_OPERATION R2, R0, R1".

Move invariant out of the loop: it is possible to move constants out of the loop.
For example, LOAD_CONST_REG are moved to the beginning. We might also move
LOAD_GLOBAL_REG and LOAD_ATTR_REG to the beginning.

Later, a new AST to bytecote compiler can be implemented to emit directly
operations using registers.


Example
=======

Simple function computing the factorial of n::

    def fact_iter(n):
        f = 1
        for i in range(2, n+1):
            f *= i
        return f

Stack-based bytecode (20 instructions)::

          0 LOAD_CONST           1 (const#1)
          3 STORE_FAST           'f'
          6 SETUP_LOOP           <relative jump to 46 (+37)>
          9 LOAD_GLOBAL          0 (range)
         12 LOAD_CONST           2 (const#2)
         15 LOAD_FAST            'n'
         18 LOAD_CONST           1 (const#1)
         21 BINARY_ADD
         22 CALL_FUNCTION        2 (2 positional, 0 keyword pair)
         25 GET_ITER
    >>   26 FOR_ITER             <relative jump to 45 (+16)>
         29 STORE_FAST           'i'
         32 LOAD_FAST            'f'
         35 LOAD_FAST            'i'
         38 INPLACE_MULTIPLY
         39 STORE_FAST           'f'
         42 JUMP_ABSOLUTE        <jump to 26>
    >>   45 POP_BLOCK
    >>   46 LOAD_FAST            'f'
         49 RETURN_VALUE

Register-based bytecode (13 instructions)::


          0 LOAD_CONST_REG       'f', 1 (const#1)
          5 LOAD_CONST_REG       R0, 2 (const#2)
         10 LOAD_GLOBAL_REG      R1, 'range' (name#0)
         15 SETUP_LOOP           <relative jump to 57 (+39)>
         18 BINARY_ADD_REG       R2, 'n', 'f'
         25 CALL_FUNCTION_REG    4, R1, R1, R0, R2
         36 GET_ITER_REG         R1, R1
    >>   41 FOR_ITER_REG         'i', R1, <relative jump to 56 (+8)>
         48 INPLACE_MULTIPLY_REG 'f', 'i'
         53 JUMP_ABSOLUTE        <jump to 41>
    >>   56 POP_BLOCK
    >>   57 RETURN_VALUE_REG     'f'

The body of the main loop of this function is composed of 1 instructions
instead of 5.


Comparative table
=================

::

    Example     |S|r|R|            Stack                 |         Register
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    append(2)   |4|1|2| LOAD_FAST 'append'               | LOAD_CONST_REG R1, 2 (const#2)
                | | | | LOAD_CONST 2 (const#2)           | ...
                | | | | CALL_FUNCTION (1 positional)     | ...
                | | | | POP_TOP                          | CALL_PROCEDURE_REG 'append', (1 positional), R1
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    l[0] = 3    |4|1|2| LOAD_CONST 3 (const#1)           | LOAD_CONST_REG R0, 3 (const#1)
                | | | | LOAD_FAST 'l'                    | LOAD_CONST_REG R3, 0 (const#4)
                | | | | LOAD_CONST 0 (const#4)           | ...
                | | | | STORE_SUBSCR                     | STORE_SUBSCR_REG 'l', R3, R0
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    x = l[0]    |4|1|2| LOAD_FAST 'l'                    | LOAD_CONST_REG R3, 0 (const#4)
                | | | | LOAD_CONST 0 (const#4)           | ...
                | | | | BINARY_SUBSCR                    | ...
                | | | | STORE_FAST 'x'                   | BINARY_SUBSCR_REG 'x', 'l', R3
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    s.isalnum() |4|1|2| LOAD_FAST 's'                    | LOAD_ATTR_REG R5, 's', 'isalnum' (name#3)
                | | | | LOAD_ATTR 'isalnum' (name#3)     | ...
                | | | | CALL_FUNCTION (0 positional)     | ...
                | | | | POP_TOP                          | CALL_PROCEDURE_REG R5, (0 positional)
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    o.a = 2     |3|1|2| LOAD_CONST 2 (const#3)           | LOAD_CONST_REG R2, 2 (const#3)
                | | | | LOAD_FAST 'o'                    | ...
                | | | | STORE_ATTR 'a' (name#2)          | STORE_ATTR_REG 'o', 'a' (name#2), R2
    ------------+-+-+-+----------------------------------+----------------------------------------------------
    x = o.a     |3|1|1| LOAD_FAST 'o'                    | LOAD_ATTR_REG 'x', 'o', 'a' (name#2)
                | | | | LOAD_ATTR 'a' (name#2)           |
                | | | | STORE_FAST 'x'                   |
    ------------+-+-+-+----------------------------------+----------------------------------------------------

Columns:

 * "S": Number of stack-based instructions
 * "r": Number of stack-based instructions exclusing instructions moved
   out of loops (ex: LOAD_CONST_REG)
 * "R": Total number of stack-based instructions (including instructions moved
   out of loops)

