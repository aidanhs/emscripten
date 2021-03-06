.. Pthreads support:

==============================
Pthreads support
==============================

Low level multithreading is currently supported in Firefox Nightly release channel via an experimental extension to JavaScript Web Workers. This is a work-in-progress research project, and the prototype specification can be located `here <https://docs.google.com/document/d/1NDGA_gZJ7M7w1Bh8S0AoDyEqwDdRh4uSoTPSNn77PFk/edit?usp=sharing>`.

The proposed specification allows Emscripten applications to share the main memory heap between web workers. This along with primitives for low level atomics and futex support enables Emscripten to implement support for the Pthreads (POSIX threads) API.

Compiling with pthreads enabled
===============================

By default, support for pthreads is not enabled, since the specification is still in a prototyping stage. To enable code generation for pthreads, the following command line flags exist:

- Pass the compiler flag -s USE_PTHREADS=1 when compiling any .c/.cpp files, AND when linking to generate the final output .js file.
- Optionally, pass the linker flag -s PTHREAD_POOL_SIZE=<integer> to specify a predefined pool of web workers to populate at page preRun time before application main() is called. If -1 is passed to both PTHREAD_POOL_SIZE and PTHREAD_HINT_NUM_CORES, then a popup dialog will ask the user the size of the pool (useful for testing).
- Optionally, pass the linker flag -s PTHREAD_HINT_NUM_CORES=<integer> to choose what the function emscripten_num_logical_cores(); will return if navigator.hardwareConcurrency is not supported. If -1 is specified here, a popup dialog will be shown at startup to let the user specify the value that is returned here. This can be helpful in order to dynamically test how an application behaves with different values here. 

There should be no other changes required. In C/C++ code, the preprocessor check #ifdef __EMSCRIPTEN_PTHREADS__ can be used to detect whether Emscripten is currently targeting pthreads.

Special considerations
======================

The Emscripten implementation for the pthreads API should follow the POSIX standard closely, but some behavioral differences do exist:

- If a page is built with the -s USE_PTHREADS=1 flag, then it will not run backwards compatibly in a non-supporting browser. In order to enable backwards compatibility so that multithreaded pages can run in non-supporting browsers (except with pthread_create() disabled), pass the flag -s USE_PTHREADS=2 instead. At runtime, you can use the emscripten_has_threading_support() function to test whether the current browser does have the capability to launch pthreads with pthread_create(). If a browser does not support threads, calls to pthread_create() will fail with error code EAGAIN.

- When -s PTHREAD_POOL_SIZE=<integer> is not specified and pthread_create() is called, the new thread will not actually start to run immediately, but the main JS thread must yield execution back to browser first. This behavior is a result of `#1049079 <https://bugzilla.mozilla.org/show_bug.cgi?id=1049079>`.

- Currently several of the functions in the C runtime, such as filesystem functions like fopen(), fread(), printf(), fprintf() etc. are not multithreaded, but instead their execution is proxied over to the main application thread. Memory allocation via malloc() and free() is fully multithreaded though. This proxying can generate a deadlock in a special situation that native code running pthreads does not have. See `bug 3495 <https://github.com/kripken/emscripten/issues/3495>` for more information and how to work around this until proxying is no longer needed in Emscripten.

- The Emscripten implementation does not support `POSIX signals <http://man7.org/linux/man-pages/man7/signal.7.html>`, which are sometimes used in conjunction with pthreads. This is because it is not possible to send signals to web workers and pre-empt their execution. The only exception to this is pthread_kill() which can be used as normal to forcibly terminate a running thread.

- The Emscripten implementation does also not support multiprocessing via fork() and join().

- For web security purposes, there exists a fixed limit (by default 20) of threads that can be spawned when running in Firefox Nightly. `#1052398 <https://bugzilla.mozilla.org/show_bug.cgi?id=1052398>`. To adjust the limit, navigate to about:config and change the value of the pref "dom.workers.maxPerDomain".

- Some of the features in the pthreads specification are unsupported since the upstream musl library that Emscripten utilizes does not support them, or they are marked optional and a conformant implementation need not support them. Such unsupported features in Emscripten include prioritization of threads, and pthread_rwlock_unlock() is not performed in thread priority order. The functions pthread_mutexattr_set/getprotocol(), pthread_mutexattr_set/getprioceiling() and pthread_attr_set/getscope() are no-ops.

- One particular note to pay attention to when porting is that sometimes in existing codebases the callback function pointers to pthread_create() and pthread_cleanup_push() omit the void* argument, which strictly speaking is undefined behavior in C/C++, but works in several x86 calling conventions. Doing this in Emscripten will issue a compiler warning, and can abort at runtime when attempting to call a function pointer with incorrect signature, so in the presence of such errors, it is good to check the signatures of the thread callback functions.

Also note that when compiling code that uses pthreads, an additional JavaScript file `pthread-main.js` is generated alongside the output .js file. That file must be deployed with the rest of the generated code files.

Running code and tests
======================

Any code that is compiled with pthreads support enabled will currently only work in the Firefox Nightly channel, since the SharedArrayBuffer specification is still in an experimental research stage before standardization. There exists two test suites that can be used to verify the behavior of the pthreads API implementation in Emscripten:

- The Emscripten unit test suite contains several pthreads-specific tests in the "browser." suite. Run any of the tests named browser.test_pthread_*.

- An Emscripten-specialized version of the `Open POSIX Test Suite <http://posixtest.sourceforge.net/>` is available at `juj/posixtestsuite <https://github.com/juj/posixtestsuite>` GitHub repository. This suite contains about 300 tests for pthreads conformance. To run this suite, the pref dom.workers.maxPerDomain should first be increased to at least 50.

Please check these first in case of any issues. Bugs can be reported to the Emscripten bug tracker as usual.
