# Python Stable ABI improvement

This repo is for tracking the effort of improving `abi3` and a place to keep any tooling.

[wiki](https://github.com/encukou/abi3/wiki/) | [projects](https://github.com/encukou/abi3/projects)

## Project summary

Python's [stable ABI](https://www.python.org/dev/peps/pep-0384/) has its issues.

* It is ill-defined. According to PEP 384, functions are *opt-out*: all functions not specially marked are part of the stable ABI. But in practice, for Windows there's [a list] that's *opt-in*. For users there is a `#define` that should make only the stable ABI available, but it's not kept up-to date. Neither is the documentation.
* It is not tested. Therefore, it tends to break. For example, changing a function to a macro can break the stable ABI as the function symbol is removed.
* It is incomplete. Some operations are not available in the stable ABI, with little reason except "we forgot".

[a list]: https://github.com/python/cpython/blob/master/PC/python3dll.c

I would like to make the stable API live up to its promise. For example:

* Projects that can sacrifice some speed (of Python bindings) to simplify their build/release infrastructure – one wheel per platform per release.
* If generators like Cython support the stable ABI, some projects could offer stable-ABI wheels *in addition* to specific-ABI ones, in order to support future (alpha/beta/early release) interpreters (with a possible speed penalty).
* Experiments in C-API evolution (such as HPy) and alternative Python implementations would have a useful subset of the full C-API to target, at least for their initial stages.

## Solutions

* Only promise what we can test
* Resolve existing bugs/blockers

## Constraints / Requirements

* Do not break the existing stable ABI! This is not a rewrite!
* The stable ABI should generally strive to be a "better C API", so that we can collaborate/align with other efforts:
  * Borrowed references are evil
  * Individual [interpreters](https://github.com/ericsnowcurrently/multi-core-python) and [modules](https://www.python.org/dev/peps/pep-0630/) should be isolated
  * We might want to change things -- make as few assumptions as possible about:
    * The GIL
    * Garbage collection
    * Layout of PyObject and other structs
  * Victor has [extensive notes](https://pythoncapi.readthedocs.io/bad_api.html) on good C API

## Status

* Idea/issue collection

### Projects using the stable ABI (to various extent):

* [PySide2](https://pypi.org/project/PySide2/) (Qt for Python)
* [cryptography](https://pypi.org/project/cryptography)
* [PyO3](https://pyo3.rs/v0.10.1/) (Rust/Python interop)

## Key Collaborators

* @encukou
* *Add yourself if you wish!*

## Contributing

There are many ways to contribute to this project.
Aside from the main technical work in the CPython runtime and implementation in various projects, there are also a number of jobs that need to be done that are less expert-related and even (somewhat) non-technical.
All of it is important and anyone interested in helping is welcome!

Keep in mind that this project/repo is actually just a tool to organize the effort.
The actual work is done on [bugs.python.org](https://bugs.python.org/), [github.com/python/cpython](https://github.com/python/cpython/), and the [Core Dev](https://discuss.python.org/c/core-dev/23) Discourse.

Also note that contacting Petr ([@encukou](https://github.com/encukou)) directly is fine, but you might get a faster response through the issue tracker here, or through those other channels. :)

For more information see the [wiki page](https://github.com/ericsnowcurrently/multi-core-python/wiki/9-%22How-Can-I-Help%3F%22).

## Why a separate place

Discussions in this repository assume that you agree with the general direction of the effort.
We don't need to justify small decisions or explain stuff to the general audience, as we would in the general Python spaces.

Also, the issue tracker is easier to manage/search than all Python issues.
