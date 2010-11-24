General information
===================

**DLFFI** -- Lua wrapper for
`DL <http://kernel.org/doc/man-pages/online/pages/man3/dlopen.3.html>`_
and `FFI <http://sourceware.org/libffi/>`_
libraries API.

What is the project started for
-------------------------------

The project's aim is to allow C-libraries API usage from Lua programs without
need to write and compile glue-code.

Usage outline
-------------

The common C API usage in Lua consists of the following steps:

#. Create a glue-code in C for every necessary function
#. Compile the code
#. Compose a high-level interface in Lua according to one's specific needs
#. Include the resulting module to the main project

**DLFFI** allows to substitute first two steps with (re)writting the header of
library, that one intend to use. The header explicitly describes the API in
terms of the FFI library.

.. highlight:: c

This header is quite similar to regular C headers. E.g. if *open()* from *libc*
described in *fcntl.h* something like::

	int open(const char *pathname, int flags, mode_t mode);

.. highlight:: lua

then it'll be loaded thus::

	load(
		"libc.so.6",
		"open",
		ffi_type_sint,
		{ ffi_type_pointer, ffi_type_sint, ffi_type_sint }
	);

How it works
------------

**DLFFI** consequently calls following functions for every loading symbol:

#. **dlopen()** -- load the requested library
#. **dlsym()** -- define the handler for the requested symbol
#. **ffi_prep_cif()** -- learn the ABI of the symbol

The function can be invoked then through **ffi_call()**.

Closure functions (wrappers) can be constructed as well (without *libdl*):

#. **ffi_prep_cif()** -- learn the function's profile
#. **ffi_closure_alloc()** -- create the handler
#. **ffi_prep_closure_loc()** -- initialize the handler

Closure functions invocation is not implemented, for the original function are
always available.
