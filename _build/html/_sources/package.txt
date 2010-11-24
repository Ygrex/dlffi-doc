Package
=======

**DLFFI** project provides two modules:

* *liblua_dlffi.so* -- core C library, do not use it directly
* *dlffi.lua* -- Lua module, use it instead
* *dlfcn.supp* -- suppressions file for `valgrind <http://valgrind.org/>`_, use it to suppress "still reachable" memory blocks, allocated by *libdl*; they are not memory leaks and you can do nothing to free them

