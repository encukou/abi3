# Isolating modules in the standard library

This document serves as a checklist for how to isolate a module in the standard
library.

As of Python 3.11, a lot of modules have been adapted to multi-phase
initialisation, and a lot of standard library types have been converted from
static to heap types. This process has not been straight-forward, and most of
the issues relate to how heap types differ from static types. Some performance
issues have also been encountered when converting global state to module state.

Before undertaking this, you should familiarise yourself with the following
PEPs:

- [PEP 384](https://www.python.org/dev/peps/pep-0384/)
- [PEP 489](https://www.python.org/dev/peps/pep-0489/)
- [PEP 573](https://www.python.org/dev/peps/pep-0573/)
- [PEP 630](https://www.python.org/dev/peps/pep-0630/)


## Part 1: Preparation

1. Open a discussion, either on the bug tracker or on Discourse. Involve the
   module maintainer and/or code owner. Explain the reason and rationale for
   the changes.
2. Identify global state performance bottlenecks, if there are such. Create a
   proof-of-concept implementation and measure the performance impact. `pyperf`
   is a good tool for benchmarking.
3. Create an implementation plan. For small modules with few types, a single PR
   may do the job. For larger modules with lots of types, and possibly also
   external library callbacks, multiple PR's will be needed.


## Part 2: Implementation

Note: this is a suggested implementation plan, based on lessons learned with
other modules.

1. Add Argument Clinic where possible; it enables you to easily use the
   defining class to fetch module state from type methods.
2. Prepare for module state; establish a module state `struct`, add an instance
   as a static global variable, and create helper stubs for fetching the module
   state.
3. Add relevant global variables to the module state `struct` and modify code
   that accesses the global state to use the module state helpers instead. This
   step may be broken into several PR's.
4. Convert heap types to static types.
5. Convert the global module state struct to true module state.
6. Implement multi-phase initialisation.

Preferably, steps 4 through 6 should all land early in a single alpha
development phase.


## Got'chas

- All heap types **must fully implement the GC protocol**. See
  [bpo-42972](https://bugs.python.org/issue42972).
- All standard library types should remain immutable. Heap types are mutable by
  default, static types are not. Use `Py_TPFLAGS_IMMUTABLE_TYPE` to retain
  immutability. See [bpo-43908](https://bugs.python.org/issue43908).
- Static type with tp_new = NULL does not have public constructor, but heap
  type inherits constructor from base class. Make sure types that previously
  were impossible to instantiate, retain that feature; use
  `Py_TPFLAGS_DISALLOW_INSTANTIATION`. Add tests using
  `test.support.check_disallow_instantiation()`. See
  [bpo-43916](https://bugs.python.org/issue43916).
- Use strong "back-refs" to the module object to ensure the module state
  pointer never outlives objects that access module state. Keep this in mind
  for external library callbacks that access module state.

