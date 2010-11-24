.. _Object-oriented-approach:

Object-oriented approach
========================

Often libraries provide some context with object-oriented like API.
For example basic IO functions **open()**, **close()**, etc. manipulate
with a file descriptor. In object-oriented languages it is more convenient
to represent descriptor as object, constructed with **open()** with methods
**read()**, **write()**, etc.

**DLFFI** provides a special class **Dlffi**, which can be used for such
libraries.

Objects can be created with **Dlffi:new()** constructor:

.. code-block:: lua

	function Dlffi:new(
		{
			api_1,	-- table
			api_2,	-- table
			...
		},
		context,	-- any object
		destructor,	-- function or anything with __call metamethod
		constructors	-- table
	)

* **api_n** -- roughly say, it is an **__index** table for the newly created
  object; specified tables will be looked through consequently; why several
  tables are allowed here, will be discussed later;
* **context** -- a context, which will be passed to every object's method on
  invocation as the first argument
* **destructor** -- a function (or anything callable) which will be invoked
  by garbage collector; can be *nil*
* **constructors** -- if any methods create new context,
  put them in a hash-table either along with a destructor
  or as a boolean *true* value when destructor is unwanted;
  can be *nil*

Let's return to the mentioned IO API. Assuming necessary symbols are loaded,
put them together into a table:

.. code-block:: lua

	libc = {
		["close"]	= libc_close,
		["read"]	= libc_read,
		["write"]	= libc_write,
		["lseek"]	= libc_lseek,
	}

Then, open the file:

.. code-block:: lua

	local fd = libc_open(...smth here...);
	assert(fd > 0);

And create the object:

.. code-block:: lua

	local file, err = dlffi.Dlffi:new(
		{ libc },	-- put the API table here
		fd,		-- put the context here
		libc_close,	-- close() in role of destructor
		nil		-- nothing else
	)
	assert(file ~= nil, err)

Since now the IO API, loaded from C library, can be accessed in
object-oriented fashion:

.. code-block:: lua

	file:write(...smth here...);
	file:lseek(...smth here...);
	file:read(...smth here...);
	file:close();

Heap management
---------------

In order to use **read()**, a buffer must be allocated. It is simple
with calling **malloc()** in C, and one can load this symbol from
*libc*, but there's a special technique for heap management:

.. code-block:: lua

	local buffer = dlffi.dlffi_Pointer(count, gc);

* **count** (integer) -- size of the allocating memory
* **gc** (boolean) -- if the buffer should be managed by garbage collector

If then a string will be stored into the *buffer*, it can be easily fetched:

.. code-block:: lua

	local str = buffer:tostring(str_length);

where **str_len** is a length of the string;
optional when the string is null-terminated

And here is a complete example of the object-oriented approach:

.. code-block:: lua

	local dlffi = require("dlffi");

	-- {{{ library header
	local LIBC = "libc.so.6";
	local libc = {
	{
		"open",
		dlffi.ffi_type_sint,		-- file descriptor
		{
			dlffi.ffi_type_pointer,	-- path
			dlffi.ffi_type_sint,	-- flags
			dlffi.ffi_type_sint,	-- mode
		},
	},
	{
		"close",
		dlffi.ffi_type_sint,		-- return code
		{
			dlffi.ffi_type_sint,	-- file descriptor
		},
	},
	{
		"write",
		dlffi.ffi_type_size_t,		-- number of written bytes
		{
			dlffi.ffi_type_sint,	-- file descriptor
			dlffi.ffi_type_pointer,	-- buffer
			dlffi.ffi_type_size_t,	-- count
		},
	},
	{
		"read",
		dlffi.ffi_type_size_t,		-- number of read bytes
		{
			dlffi.ffi_type_sint,	-- file descriptor
			dlffi.ffi_type_pointer,	-- buffer
			dlffi.ffi_type_size_t,	-- count
		},
	},
	{
		"lseek",
		dlffi.ffi_type_size_t,		-- resulting offest
		{
			dlffi.ffi_type_sint,	-- file descriptor
			dlffi.ffi_type_size_t,	-- offest
			dlffi.ffi_type_sint,	-- whence
		},
	},
	-- {{{ usefull constants
	["O_RDWR"]	= 2,
	["O_CREAT"]	= 64,
	["O_TRUNC"]	= 512,
	["S_IRWXU"]	= 448,
	["SEEK_SET"]	= 0,
	-- }}} usefull constants
	};

	-- actually load symbols here
	for i = 1, #libc, 1 do
		local profile = libc[i];
		-- load the symbol
		local symbol, e = dlffi.load(LIBC, unpack(profile));
		-- check errors
		assert(symbol, e);
		-- store the symbol into the table
		libc[profile[1]] = symbol;
	end;
	-- }}} library header

	-- initialize the file descriptor
	local file = libc.open(
		"./file_for_test",				-- path
		libc.O_RDWR + libc.O_CREAT + libc.O_TRUNC,	-- flags
		libc.S_IRWXU					-- mode
	);
	assert(file > 0, "Unable to open the file");

	-- object constructor
	local file, e = dlffi.Dlffi:new(
		{ libc },	-- hash table of functions
		file,		-- the object itself
		libc.close	-- destructor
	);
	assert(file, e);

	-- write the sample string
	local sample = "dlffi:: sample string";
	file:write(sample, #sample);

	-- reset the file's offset
	file:lseek(0, libc.SEEK_SET);

	-- create a buffer
	local buffer, e = dlffi.dlffi_Pointer(#sample, true);
	assert(buffer, e);

	-- read the file content
	local bytes = file:read(buffer, #sample);
	assert(bytes == #sample, "File reading failed");

	-- output the result
	print("read:", buffer:tostring(#sample));
