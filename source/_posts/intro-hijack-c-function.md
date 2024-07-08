---
title: Introduction of Function Hijacking in C
date: 2024-07-08 19:28:00
categories: [tips]
tags: [c, testing, development]
---

Thanks to the symbol-lazy-loading ability in Unix environment,
we can do many interesting thing on functions from some shared library
when executing some executables.

All we need to do are

- Implement a shared library that contains the functions we want to hijack.
- Run the executable with our magic library inserted.

## Make a Shared Library

If we want to replace some function with stub / fake implementation.
we can just implement a function with the same name and the same signature.

For example, if we want to fixed the clock during unit test ...

```c
// in hijack.c

#include <time.h>

// a "time" function which always return the timestamp of the epoch.
time_t time(time_t* arg) {
    if (arg)
        *arg = 0;

    return 0;
}
```

If we want do observation about some function call, but still delegate the
call to the original function, we can implement a function that load
corresponding function at runtime and pass the function call.

For example, if we want to monitor the call sequence of file open action.

```c
// in hijack.c

#include <dlfcn.h>
#include <stdio.h>
#include <time.h>

// write "open" action to standard error before open the file
FILE* fopen(const char* restrict path, const char* restrict mode) {
    static int used_count = 0;
    used_count += 1;
    fprintf(stderr, "open file [%d]: \"%s\"\n", used_count, path);

    typedef FILE* (*wrapped_type)(const char* restrict path, const char* restrict mode);
    // no dlopen, just search the function in magic handle RTLD_NEXT
    wrapped_type wrapped = dlsym(RTLD_NEXT, "fopen");
    return wrapped(path, mode);
}
```

After finish our implementation, compile them as a shared library,
called `hijack.so` here.

```shell
cc -fPIC -shared -o hijack.so hijack.c
```

## Hijack during Actual Execution

We can use `LD_PRELOAD` environment variable to do insert our special
shared library for any executable during execution.

```shell
LD_PRELOAD="path-to-shared-lib" executable
```

For example, if we want to use the implementations in last section in
our executable, called `app` here.

```c
// app.c

#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main() {
    FILE* handle = fopen("output.txt", "a");
    assert(handle);
    fclose(handle);

    time_t current = time(NULL);
    printf("now: %s", asctime(gmtime(&current)));

    return EXIT_SUCCESS;
}
```

(Compile and) run the executable

```shell
cc -o app app.c
LD_PRELOAD=./hijack.so ./app
```

Output

```
open file [1]: "output.txt"
now: Thu Jan  1 00:00:00 1970
```

The open-file action is traced, and the time is fixed to the epoch.

If we need to overwrite functions with more than one shared library,
just use `:` to separate them in `LD_PRELOAD`.

## Conclusion

It's a powerful feature, it allows we to do observation, to replace with
mock/fake implementation, or sometime even to apply emergency patch.

And all make this possible are the dynamic linking mechanism, and
the one-to-one mapping from symbol from sources to libraries/binaries.

Although development in C is somehow inconvenient, but it's still
a interesting experience when seeing this kind of usage. :D
