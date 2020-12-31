PEP: 9999
Title: Maintaining the Stable ABI
Author: Petr Viktorin <encukou@gmail.com>
Discussions-To: 
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 08-Dec-2020


Abstract
========

[A short (~200 word) description of the technical issue being addressed.]

XXX: Abstract should be written last


Motivation
==========

:pep:`384` defined a limited API and stable ABI, which allows extenders and
embedders of CPython to compile extension modules that are binary-compatible
with any subsequent version of 3.x.
In theory, this brings many advantages:

* A module can be built only once per platform and support multiple versions
  of Python, reducing time, power and maintainer attention needed for builds.
* Binary wheels using the stable ABI work with new versions of CPython
  throughout the pre-release period, and can be tested in environments where
  building from source is not practical.

As a welcome side effect of the stable ABI's hiding of implementation details
is that it is becoming a viable target for alternate Python implementations
that need to implement (parts of) the C API.

However, in hindsignt, PEP 384 and its implementation has several issues:

* There is no process keep the ABI up to date.
* Contents of the limited API are not listed explicitly, making it unclear
  if a particular member (e.g. function, structure) is a part of it.
* There is no way to deprecate parts of the limited API.

This PEP defines the limited API more clearly and introducess process
designed to make the API more useful.

Additionally, PEP 384 defines a *limited API* as a way to build against the
stable ABI.
This PEP defines the limited API more robustly.


Rationale
=========

This PEP contains a lot of clarifications and definitions, but just one big
technical change: the stable ABI will be explicitly listed in
a human-maintained “manifest” file.

There have been efforts to collect such lists automatically, e.g. by scanning
the symbols exported from Python.
This might seem to be easier to maintain by our volunteer team.

However, designing a future-proof API is not a trivial task.
The cost of updating an explicit manifest is small compared
to the overall work that should go into changing API that will need to
be suppported forever (or until Python 3 reaches
end of life, if that comes sooner).

This PEP proposes automatically generating things *from* the manifest:
initially documentation and DLL contents, with later possibilities
for also automating tests.


Stable ABI vs. Limited API
==========================

:pep:`384` and this document deal with the *Limited API* and the *Stable ABI*,
two related but distinct concepts.
This section clarifies what they mean and defines some of their semantics
(either pre-existing or newly proposed here).

The word “Extensions” is used as a shorthand for all code that uses the
Python API, e.g. extension modules or software that embeds Python.


Stable ABI
----------

The CPython *Stable ABI* is a promise that extensions built with a specific
Cpython version will be usable with any newer interpreter of the same major
version, on the same platform and with the same compiler & settings.
For example, a extension built with CPython 3.10 Stable ABI will be usable with
CPython 3.11, 3.12, and so on, but not necessarily with 4.0.

The Stable ABI is not generally forward-compatible: an extension built and
tested with CPython 3.10 will not generally be compatible with CPython 3.9.

.. note::
   For example, starting in Python 3.10, the `Py_tp_doc` slot may be set to
   `NULL`, while in older versions, a `NULL` value will likely crash the 
   interpreter.

The Stable ABI trades performance for its stability.
For example, many functions in the stable ABI are available as faster macros
to extensions that are built for a specific CPython version.

Future Python sversions may deprecate some members of the Stable ABI.
Such members will still work, but may suffer from issues like reduced
performance or, in the most extreme cases, memory/resource leaks.


Limited API
-----------

Stable ABI guarantee holds for extensions compiled from code that restricts
itself to the *Limited API*, a subset of CPython's C API.

The limited API is used when preprocessor macro `Py_LIMITED_API` is defined
to either `3` or the current `PYTHON_API_VERSION`.

The Limited API is not guaranteed to be *stable*.
In the future, parts of the limited API may be deprecated.
They may even be removed, as long as the *stable ABI* is kept
stable and Python's general backwards compatibility policy, :pep:`387`,
is followed.

.. note::

   For example, a function declaration might be removed from public header
   files but kept in the library.
   This is currently a possibility for the future; this PEP does not to propose
   a concrete process for deprecations and removals.

The goal is for the limited API to cover everything needed to interact
with the interpreter.
There main reasons to not include a public API in the limited subset
should be that it needs implementation details that change between CPython
versions, like struct memory layouts, for performance reasons.

The limited API is not limited to CPython; other implementations are
encouraged to implement it and help drive its design.


Specification
=============

To make the stable ABI more useful and stable, the following changes
are proposed.


Stable ABI Manifest
-------------------

All members of the stable ABI – functions, typedefs, structs, struct fields,
data values etc. – will be explicitly listed in a single "manifest" file,
along with the Limited API version they were added in.
Struct fields that users of the stable ABI are allowed to access will be
listed explicitly.
Members that are not part of the Limited API, but are part of the Stable ABI
(e.g. ``PyObject.ob_type``, which is accessible by the ``Py_TYPE`` macro),
will be annotated as such.

Notes saying “Part of the stable ABI” will be added to Python's documentation
automatically, in a way similar to the notes on functions that return borrowed 
references.

Source for the Windows shared library `python3.dll` will be generated from the
stable ABI definition.

The format of the manifest will be subject to change whenever needed.
It should be consumed only by scripts in the CPython repository.
If a more public list is needed, a script can be added to generate it.


Contents of the Stable ABI
--------------------------

The initial stable ABI manifest will include:

* The Stable ABI specified in :pep:`384`.
* All functions listed in ``PC/python3dll.c``.
* All structs (struct typedefs) which these functions return or take as
  arguments. (Fields of such structs will not necessarily be added.)
* New type slots, such as ``Py_am_aiter``.
* The type flags  ``Py_TPFLAGS_DEFAULT``, ``Py_TPFLAGS_BASETYPE``,
  ``Py_TPFLAGS_HAVE_GC``, ``Py_TPFLAGS_METHOD_DESCRIPTOR``.
* The calling conventions ``METH_*`` (except deprecated ones).
* All API needed by macros is the stable ABI (usually annotated as not being
  part of the limited API).

Additional items may be aded to the initial manifest according to
the checklist below.


Testing the Stable ABI
----------------------

An automatically generated test module will be added to ensure that all members
of the stable ABI are available at compile time

For each function in the stable ABI, a test will be added that calls the
function using `ctypes`. (Where calling is not practical, such as with
functions related to intepreter initialization and shutdown, the test will
only look the function up.)
This should prevent regressions when a function is converted to a macro,
which keeps the same API but breaks the ABI.
An check will be added to ensure all functions in the stable ABI are tested
this way.


Changing the Limited API
------------------------

A checklist for changing the limited API, including new members (structs,
functions or values), will be added to the `Devguide`_.
The checklist will 1) mention best practices and common pitfalls in Python
C API design and 2) guide the developer around the files that need changing and
scripts that need running when the limited API is changed.

Below is the initial proposal for the checklist. After the PEP is accepted,
see the Devguide for the current version.

Note that the checklist applies to new additions; not the existing limited API.

Design considerations:

* Make sure the change does not break the Stable ABI of any version of Python
  since 3.5.
* Make sure no exposed names are private (i.e. begin with an underscore).
* Make sure the new API is well documented.
* Make sure the types of all parameters and return values of the added
  function(s) and all fields of the added struct(s) are be part of the
  limited API (or standard C).
* Make sure the new API and its intended use follows standard C, not just
  features of currently suppoerted platforms.

  * Do not cast a function pointer to ``void*`` (a data pointer) or vice versa.

* Make sure the new API follows reference counting conventions. (Following them
  makes the API easier to reason about, and easier use in other Python
  implementations.)

  * Do not return borrowed references from functions.
  * Do not steal references to function arguments.

* Make sure the ownership rules and lifetimes of all applicable struct fields,
  arguments and return values are well defined.
* Think about ease of use for the user. (In C, ease of use itself is not very 
  important; what *is* important is reducing boilerplate code needed to use the
  API. Bugs like to hide in boiler plates.)

  * If a function will be often called with specific value for an argument,
    consider making it default (assumed when ``NULL`` is passed in).

* Think about future extensions: for example, if it's possible that future
  Python versions will need  to add a new field to your struct,
  how will that be done?

* Make as few assumptions as possible about details that might change in
  future CPython versions or differ across C API implementations:

    * The GIL
    * Garbage collection
    * Layout of PyObject and other structs

If following these guidelines would hurt performance, add a fast function
(or macro) to the non-limited API and a stable equivalent to the limited API.

If anything is unclear, or you have a good reason to break the guidelines,
consider discussing the change at the `capi-sig`_ mailing list.

.. _capi-sig: https://mail.python.org/mailman3/lists/capi-sig.python.org/

Procedure:

* Move the declaration to a header file directly under ``Include/`` and
  ``#if !defined(Py_LIMITED_API) || Py_LIMITED_API+0 >= 0x03yy0000``
  (with the ``yy`` corresponding to Python version).
* Make an entry in the stable ABI list. (XXX: mention filename)
* For functions, add a test that calls the function using ctypes
  (XXX: mention filename).
* Regenerate the autogenerated files. (XXX: specific instructions)


Advice for Extenders and Embedders
----------------------------------

The following notes will be added to documentation.

Extension authors should test with all Python versions they support,
and preferably build with the lowest such version.

Compiling with ``Py_LIMITED_API`` defined is *not* a guarantee that your code
conforms to the limited API or the stable ABI.
It only covers definitions, but an API also includes other issues,
such as expected semantics.

Examples of issues that ``Py_LIMITED_API`` does not guard against are:

* Calling a function with invalid arguments 
* An function that started accepting ``NULL`` values for an argument
  in Python 3.9 will fail if ``NULL`` is passed to it under Python 3.8.
  Only testing with 3.8 (or lower versions) will uncover this issue.
* Some structs include a few fields that are part of the stable ABI and other
  fields that aren't.
  ``Py_LIMITED_API`` does not filter out such “private” fields.
* Using something that is not documented as part of the stable ABI,
  but exposed even with ``Py_LIMITED_API`` defined.
  Despite the team's best efforts, such issues may happen.


Backwards Compatibility
=======================

The PEP aims at full compatibility with the existing stable ABI and limited
API, but defines them terms more explicitly.
It might not be consistent with some interpretations of what the existing
stable ABI/limited API is.


Security Implications
=====================

None known.


How to Teach This
=================

Technical documentation will be provided.
It will be aimed at experienced users familiar with C.


Reference Implementation
========================

None so far.


Rejected Ideas
==============

While this PEP acknowledges that parts of the limited API might be deprecated
or removed in the future, a process to do this is not in scope, and is left
to a possible future PEP.


Open Issues
===========

None so far.


References
==========


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.


.. _Devguide: https://devguide.python.org/

..
    Local Variables:
    mode: indented-text
    indent-tabs-mode: nil
    sentence-end-double-space: t
    fill-column: 70
    coding: utf-8
    End:
