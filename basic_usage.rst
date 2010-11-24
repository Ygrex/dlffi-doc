Basic usage
===========

The simplest case is when few functions are required from some C library.

These functions can be used strictly, once they are loaded.

.. highlight:: lua

Let's see an example::

	-- include the module
	local dlffi = require("dlffi");
	assert(type("dlffi") == "table");

	-- int puts(const char *s)
	local puts, e = dlffi.load(
		"libc.so.6",
		"puts",
		dlffi.ffi_type_sint,
		{ dlffi.ffi_type_pointer }
	);
	-- make sure there's no error
	assert(puts, e);
	-- invoke the loaded function
	local r, e = puts("puts() from libc.so.6");
	-- output the return value and error if any
	print(r, e);

the output::

	puts() from libc.so.6
	22	nil

Notes
-----

**require("dlffi")** must return the module's object (table)
if no errors occured.

**dlffi.load()** returns the loaded symbol handler.
In case of error it returns *nil* and error description (string)
as the second value. If there's a memory allocation error (ENOMEM),
the second value will be *nil* too.

Calling the loaded symbol normally returns the value according to the
specified profile, as expected, e.g. **puts()** returns *int*.
The *nil* will be returned for *void* functions.

If error occured, the first value will be nil and the error description
will be the second returned value. The later will be *nil* as well in case
of heavy memory limitations.

