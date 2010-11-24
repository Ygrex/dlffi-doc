Basic API
=========

Subject
-------

**Basic API** stands for a binding for native functions
of **libdl** and **libffi**. This API description, without any doubt,
relies on the assumption that the reader somehow familiar with these
two libraries.

Online version of **libffi**'s manual is not present on the project's homepage
and can be found in any source directory or installed with development
library package.

The 3rd part manual is published here:
`libffi manual <http://www.sfr-fresh.com/unix/misc/Pike-v7.8.116.tar.gz:a/Pike-v7.8.116/bundles/libffi-3.0.4/doc/libffi.texi>`_

Of course, there are no guarantee, that the link is not broken and follows
to an actual version.

Manual page for **libdl** can be easily retreived online at
`kernel.org <http://kernel.org/doc/man-pages/online/pages/man3/dlopen.3.html>`_
or locally::

	$ man 3 dlopen

The later requires development manpages to be installed on your system.

.. _common-conventions:

Common conventions
------------------

Any functions return single value or nothing.
In case of error, descriptive message follows then.

No functions ever raise an error, except while arguments verification.

Any example code assume the **dlffi** module is loaded:

.. code-block:: lua

	local dlffi = require("dlffi");
	assert(type(dlffi) == "table", dlffi);

liblua_dlffi
============

dlffi_Function
--------------

.. class:: dlffi_Function metatable

	Implements all FFI functions as Lua userdata objects
	Instances can be constructed with :func:`load()`.

.. classmethod:: dlffi_Function.__gc(): C function

	1. clear reference from registry (for closures),

	2. decrement the dynamic library reference counter,

	3. free all allocated memory.

	:return: nothing

.. classmethod:: dlffi_Function.__call(): C function

	1. initialize array of given arguments

	2. call function with *ffi_call()*

	3. free memory allocated for arguments

	4. push the returned value

	:return: the value, returned by the referent C function
		(**nil**, if void function)

dlffi_Pointer
-------------

.. class:: dlffi_Pointer

	Implements all C values and types through Lua userdata.
	Instances can be constructed with :func:`dlffi_Pointer()`.

.. classmethod:: dlffi_Pointer.__index

	Referes to the :class:`dlffi_Pointer` itself.

	:rtype: userdata

	Example:

	.. code-block:: lua

		-- dlffi_Pointer instance cannot be iterated, but metatable
		local a = getmetatable(dlffi.dlffi_Pointer());
		-- output methods and metamethods
		for k in pairs(a) do print(k) end;

.. classmethod:: dlffi_Pointer.__eq(PointerL, PointerR)

	Compares :class:`dlffi_Pointer` objects,
	if they points to the same address.

	:rtype: boolean

	Example is here :meth:`dlffi_Pointer.copy`.

.. classmethod:: dlffi_Pointer.__sub(PointerL, PointerR)

	Calculates C pointers distance. Be sure, that PointerL >= PointerR,
	for **size_t** is unsigned and wrong result will be otherwise.

	:rtype: integer

	Example:

	.. code-block:: lua

		-- get 2 random buffers
		local a = dlffi.dlffi_Pointer(1024);
		local b = dlffi.dlffi_Pointer(1024);
		-- compare them
		print(string.format(
			"%s - %s = 0x%x",
			tostring(b),
			tostring(a),
			b - a
		));

.. classmethod:: dlffi_Pointer.__gc(Pointer)

	1. execute and dereference GC handler,
		if it was manually set with :meth:`dlffi_Pointer.set_gc`

	2. free the memory block the Pointer references to,
		if GC has not been disabled

	:return: nothing

	Example is here :meth:`dlffi_Pointer.set_gc`.

.. classmethod:: dlffi_Pointer.index(Pointer, index[, ffi_type])

	Method to fetch individual elements of the array.
	Arrays in C contain elements of equal types.
	Use **type_element()** for structures.

	:arg Pointer:	**self**
	:arg index:	index of the element;
			index is not equal to offset and starts from 1
	:arg ffi_type:	if array is two-dimensional, then **ffi_type**
			may be omitted and new **Pointer** value will
			be read from **index**-th element; otherwise,
			**index**-th element of the specified type will
			be read and its value returned;

	:return:

		1. when no **ffi_type** specified, then, in contrast to
			common convention, 2 values returned:
			new :meth:`dlffi_Pointer` instance with GC disabled,
			and **lightuserdata**; their value is read from
			**index**-th element;

		2. casted Lua value, read from **index**-th, element;

		3. nothing, if something bad occured

	example with 1D array:

	.. code-block:: lua

		-- load string generator
		local strdup = dlffi.load(
			"libc.so.6",
			"strdup",
			dlffi.ffi_type_pointer,
			{
				dlffi.ffi_type_pointer,
			}
		);
		-- initialize array of chars
		local charray = dlffi.dlffi_Pointer(strdup("abcdefgh"));
		-- get the 3rd element
		local byte = charray:index(3, dlffi.ffi_type_schar);
		-- here byte == 99, transform it to ASCII character
		print(string.char(byte)); -- print 'c'

	example with 2D array:

	.. code-block:: lua

		-- load 2D array generator
		local td_symbol_list, err = dlffi.load(
			"libthread_db.so",
			"td_symbol_list",
			dlffi.ffi_type_pointer,
			{ }
		);
		assert(td_symbol_list, err);
		
		-- function returns lightuserdata (char **)
		-- wrap it with dlffi_Pointer for to call any methods
		local array = dlffi.dlffi_Pointer(td_symbol_list());
		
		-- read maximum 128 first elements
		for i = 1, 128, 1 do
			-- retreive the i-th string from array
			local first, light = array:index(i);
			-- array is NULL-terminated
			if light == dlffi.NULL then break end;
			-- `first` is a dlffi_Pointer and can be stringified
			print(i, first:tostring());
		end;

.. classmethod:: dlffi_Pointer.tostring(Pointer, [length])

	Method make assumption, that **Pointer** is a char array and
	returns string duplicate.

	:arg Pointer:	**self**
	:arg length:	length of the string without **\0**;
			string is considered to be NUL-terminated if this
			argument is not specified

	.. code-block:: lua

		-- load sprintf from glibc
		local sprintf = dlffi.load(
			"libc.so.6",
			"sprintf",
			dlffi.ffi_type_sint,
			{
				dlffi.ffi_type_pointer, -- str
				dlffi.ffi_type_pointer, -- format
				dlffi.ffi_type_pointer, -- arg1
			}
		);
		-- allocate buffer
		local p = dlffi.dlffi_Pointer(4096, true);
		-- get formatted string
		sprintf(p, "dlffi loaded: %s\n", tostring(dlffi));
		-- display string
		print(p:tostring());

.. classmethod:: dlffi_Pointer.set_gc(Pointer, function)

	Store function's reference in registry and call it later
	from :meth:`dlffi_Pointer.__gc`.

	:arg Pointer:	**self**
	:arg function:	any callable object;
			the function will do nothing if **nil** specified

	:return: nothing

	.. code-block:: lua

		do
			local a = dlffi.dlffi_Pointer(4, true);
			local gc = function (p)
				-- execution indicator
				print("__gc of " .. tostring(p))
			end;
			a:set_gc(gc);
		end;
		collectgarbage("collect");

.. classmethod:: dlffi_Pointer.copy(Pointer)

	Duplicates **Pointer** with GC disabled.

	:arg Pointer:	**self**

	:return: new :class:`dlffi_Pointer` instance

	.. code-block:: lua

		local a = dlffi.dlffi_Pointer(4, true);
		local b = a:copy();
		-- instances are different
		print("different instances: " ..
			tostring(a) ..
			", " ..
			tostring(b)
		);
		-- but point to the same address
		print("equal pointers: ", a == b);

Library functions
-----------------
.. function:: dlffi_Pointer([Pointer | lightuserdata | size[, gc]])

	Create new instance of :class:`dlffi_Pointer`

	:arg size:	if 1st parameter is integer, then **size** bytes will
			be allocated with **malloc()** and constructed new
			**Pointer** to the buffer

	:arg lightuserdata:	if 1st parameter is **lightuserdata** then
				newly constructed **Pointer** will have this
				value

	:arg Pointer:	if 1st parameter is **dlffi_Pointer** then newly
			constructed **Pointer** will point to the specified
			pointer

	:arg gc:	if target buffer should be freed when
			:meth:`dlffi_Pointer.__gc`

	:rtype: :class:`dlffi_Pointer`

	:return: without arguments **Pointer** will target to NULL

	.. code-block:: lua

		-- initialize buffer
		local buffer = dlffi.dlffi_Pointer(1024, true);
		-- make a pointer to a pointer to the buffer
		local pointer = dlffi.dlffi_Pointer(buffer);
		-- they are different
		assert(buffer ~= pointer);
		-- make a NULL pointer (dlffi.NULL is a lightuserdata)
		pointer = dlffi.dlffi_Pointer(dlffi.NULL);
		-- make an empty pointer
		empty = dlffi.dlffi_Pointer();
		-- they are equal
		assert(pointer == empty);

.. function:: rawload([library, ]function, rtype, arg_types[, cast_table])

	Make FFI function, taht can be either loaded from dynamic library, or
	constructed as closure for Lua function.

	This function named **load** initially but overriden in **dlffi**
	module, preserving original function as **rawload**.

	Probably :func:`load()` is what must be always used instead.

	:arg library:	library name, if you load function from library;
			it is exactly 1st argument for **dlopen()**

	:arg function:	if you load function from library,
			this is a function name
			and exactly 2nd argument for **dlsym()**;

			if **function** is a Lua function and given at 1st
			position, then FFI closure will be created; the
			reference will be stored in registry to prevent
			**function** from being GC'ed

	:arg rtype:	FFI type of function's return value

	:arg arg_types:	continuation table of FFI types
			of positional arguments; variable argument
			lists are not supported by libffi (load them with
			a static argument list, that you intend to use, or
			make a wrapper in Lua world)

	:arg cast_table:	Lua function to cast tables; it will be used
				whenever a Lua table passed to the loaded
				function;

				this function will receive two
				parameters: called C function
				(**lightuserdata**) and the Lua table;

				the return value will be used instead the
				table (returned table will be casted again
				and so on, beware recursions);

	:return:	:meth:`dlffi_Function` instance; it can be called like
			a regular Lua function or passed as a callback in
			another C function; closures cannot be called directly

	Example:

	.. code-block:: lua

		-- load pthread_create symbol
		local pthread_create, err = dlffi.rawload(
			"libpthread.so.0",
			"pthread_create",
			dlffi.ffi_type_sint,
			{
				dlffi.ffi_type_pointer,	-- thread
				dlffi.ffi_type_pointer,	-- attr
				dlffi.ffi_type_pointer,	-- start_routine
				dlffi.ffi_type_pointer,	-- arg
			}
		);
		assert(pthread_create, err);
		
		-- Lua function
		local child = function(arg)
			-- execution indicator
			print("child process received", arg);
		end;
		-- initialize a closure
		local closure = dlffi.rawload(child,
			dlffi.ffi_type_void,
			{ dlffi.ffi_type_pointer }
		);
		-- allocate definitly enough memory for pthread_t structure
		local thread = dlffi.dlffi_Pointer(1024);
		-- create child process
		-- and even pass it some pointer (`thread` here)
		local r = pthread_create(thread, dlffi.NULL, closure, thread)
		-- threads in Lua is another subject, just wait alittle here
		os.execute("sleep 0.1");
		print("parent process");

.. function:: sizeof(type)

	Find size in bytes of memory buffer to allocate
	value of the given FFI type

	:arg type: any FFI type (lightuserdata or :class:`dlffi_Pointer`)

	:rtype: integer

	Example:

	.. code-block:: lua

		-- C equivalent for sizeof(size_t)
		dlffi.sizeof(dlffi.ffi_type_size_t);

.. function:: type_element(Pointer, type, index[, value])

	Return or set index-th element in the given structure.

	:arg Pointer:	instance of :class:`dlffi_Pointer`
	:arg type:	FFI structure type;
			function do nothing with non structures
	:arg index:	index of the element in the structure;
	:arg value:	Lua value to set to the index-th element;
			the element will not be modified,
			if **nil** is given;

			if Lua string is given, it'll be duplicated
			and pointer will be written; definitly, you'll
			need to free it later

	:rtype:		casted value of the index-th element or
			boolean

	:return:	if **value** is given the function returns
			**true**/**false** on success/failure;
			otherwise return value is read from the
			requested element and casted to Lua value

	See :func:`type_init` for usage example.

.. function:: type_free(structure)

	Free FFI structure.

	:arg structure:	**lightuserdata** corresponding to FFI structure,
			returned by :func:`type_init()` or
			**ffi_prep_cif()**

	:return: nothing

	See :func:`type_init` for usage example.

.. function:: type_init(fields)

	Create new FFI structure. It must be freed after you done with it.

	:arg fields: Lua table of FFI types in appropriate order

	:rtype: **lightuserdata**

	:return: new FFI structure

	Example:

	.. code-block:: lua

		-- create FFI structure with 2 elements
		local struct = {
			dlffi.ffi_type_uchar,
			dlffi.ffi_type_pointer,
		};
		struct = dlffi.type_init(struct);
		assert(struct);
		
		-- initialize buffer to store the structure
		local var = dlffi.dlffi_Pointer(
			dlffi.sizeof(struct), true
		);
		assert(var);
		
		-- initialize elements
		local chk;
		-- put 3 at 1st element
		chk = dlffi.type_element(var, struct, 1, 3);
		assert(chk);
		-- duplicate string and put its pointer
		-- allocated string must be freed later
		chk = dlffi.type_element(var, struct, 2, "sample string");
		assert(chk);
		
		-- read elements
		local fir = dlffi.type_element(var, struct, 1);
		local sec = dlffi.type_element(var, struct, 2);
		
		-- note: GC will free string duplicate
		sec = dlffi.dlffi_Pointer(sec, true);
		
		assert(fir == 3);
		assert(sec:tostring() == "sample string");
		print("read:", fir, sec:tostring());
		
		-- free FFI structure
		dlffi.type_free(struct);

.. function:: type_offset(structure, index)

	Find offest of the **index**-th element inside the **structure**

	:arg structure:	**lightuserdata** corresponding to FFI structure type
	:arg index:	index of the element inside the **structure**

	:rtype: integer

	Example:

	.. code-block:: lua

		-- create structure with implicit alignment
		local struct = dlffi.type_init {
			dlffi.ffi_type_uint64,
			dlffi.ffi_type_uint8,
			dlffi.ffi_type_uint64,
		};
		assert(struct);
		
		-- display offset of the 3rd element
		print(dlffi.type_offset(struct, 3));
		
		-- free initialized structure
		dlffi.type_free(struct);

