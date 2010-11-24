Wrapping module
===============

Subject
-------

The module provides wrappers for some library functions and adds proxy
objects to allow developing in Lua style rather than C.

Common conventions
------------------

Same for the basic API :ref:`common-conventions`.

FFI type management
-------------------

.. class:: Dlffi_t

	Class for FFI structure management.

	FFI structure normally must be created with :func:`type_init()`
	and freed after anything done with :func:`type_free()`.

	Structure destruction can be queued to collectgarbage. That is
	what :class:`Dlffi_t` does.

	Instances are containers of actual FFI structures. Dynamic creation
	of the types looks like:

	.. code-block:: lua

		instance["complex_structure"] = {
			dlffi.ffi_type_sint,
			dlffi.ffi_type_pointer,
		};

		assert( type(instance["complex_structure"]) == "userdata" );

.. classmethod:: Dlffi_t.new( [struct, fields] )

	Create container for FFI structures.

	**struct** and **fields** can be specified to create the FFI structure
	immediately after instance construction. Though it can be done later
	like:

	.. code-block:: lua

		instance[structure] = fields

	:return:	object which implements container
			for FFI structure types

Dlffi proxy object
------------------

.. class:: Dlffi

	Provides a proxy object class to use in C libraries with object-like
	context.

.. classmethod:: Dlffi.new(self, api, init, gc, spec)

	Constructor for proxy objects.
	Returned instance does not inherit :class:`Dlffi`.

	:arg api:	table of tables with inheritted methods; only indexed
			tables' parts are traversed and keys are probed with
			**rawget()**;
	:arg init:	initialized library context;
	:arg gc:	context destructor, if needed; desctrutor is queued to
			garbagecollector, so must obey adjacent restrictions;
	:arg spec:	table of methods, which are constructors too;
			the returned value will be wrapped as well;

	:rtype:		Lua table

	:return:	object with provied with **api** methods
			and properties;

			the object can be passed to any functions directly
			as well; it will be substituted with appropriate
			C value when passed to some C function (in case
			the function loaded with proper **cast_table**
			function, i.e. with :func:`load()`).

	See :ref:`Object-oriented-approach` for usage example.

Module functions
----------------

During initialization the module overrides :func:`load` renaming
initial one to :func:`rawload`.

.. function:: load(...)

	Proxy function for :func:`rawload()`.
	It adds default :func:`cast_table()` if no 5th argument given.

.. function:: cast_table(func, tbl)

	Function to cast :class:`Dlffi` objects into appropriate
	C typed value.

	:arg func:	C function, that requires context
	:arg tbl:	initialized instance of :class:`Dlffi`

	:return:	C typed value, that has been assigned when
			:meth:`Dlffi.new()`.

