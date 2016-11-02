# Python style printf for C++

[![Build Status](https://travis-ci.org/tfc/pprintpp.svg?branch=master)](https://travis-ci.org/tfc/pprintpp) (GCC and Clang)

## What is `pprintpp`?

The acronym stands for "Python style print for C plus plus".

`pprintpp` is a **header-only** C++ library, which aims to make `printf` use safe and easy.
It is a *pure compile time library*, and will add **no overhead** to the runtime of your programs.

This library is for everyone, who uses C++, but sticks to printf-like functions (like `printf`, `fprintf`, `sprintf`, `snprintf`, etc...).
`pprintpp` adds a typesafe adapter on top of those functions by preprocessing strings to the format printf and its friends are excepting.
Apart from the preformatted string, *no other symbols are added to the resulting binary*.
This means that this library produces **no runtime code** at all, which distinguishes it from libraries like [fmtlib](https://github.com/fmtlib/fmt).

## Dependencies

The library does only depend on **C++11** (or higher) and the **STL** (`<tuple>` and `<type_traits>`).

The STL dependency can easily get rid of by reimplementing some type traits. This way `pprintpp` can be used in hardcore baremetal environments (where it already has been in use actually).

## Example

When using `printf`, the programmer has to choose the right types in the format string.
``` C++
printf("An int %d, a float %f, a string %s\n", 123, 7.89, "abc");
```
If this format string is wrong, the programm will misformat something in the best case.
In the worst case, the program might even crash.

The python style print library allows for the following:
``` C++
pprintf("An int {}, a float {}, a string {s}\n", 123, 7.89, "abc");
```
The types are chosen *automatically* **at compile time**.
This is both safe *and* convenient.

> Note the `{s}` in the format string: It is a safety detail of the library to require an additional "s" in the format string to print `char*` types really as strings. Otherwise, every `char*` would be printed as string, although it might be some non-null-terminated buffer.


The assembly generated from the simple program...
``` c++
int main()
{
    pprintf("{} hello {s}! {}\n", 1, "world", 2);
}
```

...shows, that this library comes with *no* runtime overhead:
```
bash $ objdump -d example
...
0000000000400450 <main>:
  400450:       48 83 ec 08             sub    $0x8,%rsp
  400454:       41 b8 02 00 00 00       mov    $0x2,%r8d
  40045a:       b9 04 06 40 00          mov    $0x400604,%ecx # <-- "world"
  40045f:       ba 01 00 00 00          mov    $0x1,%edx
  400464:       be 10 06 40 00          mov    $0x400610,%esi # <-- "%d hello world %s!..."
  400469:       bf 01 00 00 00          mov    $0x1,%edi
  40046e:       31 c0                   xor    %eax,%eax
  400470:       e8 bb ff ff ff          callq  400430 <__printf_chk@plt>
  400475:       31 c0                   xor    %eax,%eax
  400477:       48 83 c4 08             add    $0x8,%rsp
  40047b:       c3                      retq
...
```

Dumping the read-only data section of the binary shows the `printf` format string.
It looks as if the programmer had directly written the printf line without ever having used `pprintpp`:
```
bash $ objdump -s -j .rodata example
...
Contents of section .rodata:
 400600 01000200 776f726c 64000000 00000000  ....world.......
 400610 25642068 656c6c6f 20257321 2025640a  %d hello %s! %d.
 400620 00                                   .
```

## Printf Compatibility

`pprintpp` will transform a tuple `(format_str, [type list])` to `printf_compatible_format_string`.

That means, that it can be used with any `printf`-like function. You just need to define a macro, like for example these ones for `printf` and `snprintf`:

``` c++
#define pprintf(fmtstr, ...) printf(AUTOFORMAT(fmtstr, ## __VA_ARGS), ## __VA_ARGS__)
#define psnprintf(outbuf, len, fmtstr, ...) \
    snprintf(outbuf, len, AUTOFORMAT(fmtstr, ## __VA_ARGS__), ## __VA_ARGS__)
```

> Unfortunately, it is not possible to express this detail without macros in C++. However, the rest of the library was designed without macros at all.

Embedded projects, which introduce their own logging/tracing functions, which accept `printf`-style format string, will also profit from this library.

## FAQ

### Why `printf`, when there is stream style printing in C++?

Yes, stream style printing is type safe, and from a features perspective clearly superior to `printf`.
However, in some projects, C++ is used without streams, sometimes even without the STL.
This library was designed to help out developers of such projects with some type safety and comfort.

> The current version of this library is a reimplementation of what i once wrote some time ago.
> This version uses STL stuff from `<tuple>` and `<type_traits>`
> These can easily be reimplemented without the STL, if someone wishes to use this on some hardcore baremetal project where no STL is available. 
> It's just that no one asked for that reimplementation, yet.

### I am pretty happy with `fmtlib`. Why `pprintpp`?

Those two libs have similar purposes, but put different weights on some objectives:

- `fmtlib` is e.g. able to do reorder parameters like this: `fmt::format("{0}{1}{0}", "abra", "cad");`. `pprintpp` will never be able to do that, because `fmtlib` does the actual job of **formatting** at this point. `pprintpp` is just a preprocessor for `printf` strings. As `printf` can't reorder arguments, `pprintpp` will not be able to provide this functionality, either.
- `fmtlib` can also be used to put the formatted result into `std::string`, or streams.

Base line: `fmtlib` is actually a *formatting library*. `pprintpp` is only a *preprocessor* which enables to automatically composing `printf`-compatible format strings. With other words: `pprintpp` is a `printf` *frontend*.

You will profit from `pprintpp` over `fmtlib` if:
- you don't need more formatting features than `printf` and friends provide
- You want to add type safety and comfort to your printing **without adding runtime code**.

### I don't see how this helps printing my own types/classes?

You are right. It doesn't. Printing for example a custom vector type with `pprintpp` will always look like this:
``` c++
pprintf("My vector: ({}, {}, {})\n", vec[0], vec[1], vec[2]);
```

Due to the nature of `pprintpp` just being a *format string preprocessor* which will call a `printf`-like function in the end, it will not be possible to print custom types.

If it is not possible to express something with printf directly (`printf(" %FOOBARXZY ", my_custom_type_instance);`), then `pprintpp` will not help out here.

However, if you use some kind of `my_own_printf` implementation, which actually accepts a format specifier like `%v` for `struct my_vector`, and is able to pretty print that type at runtime, then it is easy to extend `pprintpp` with knowledge of this type, yes.
*Then* it is possible to do `pprintf("my own vector type: {}\n", my_own_vector);`.

Baseline: If you are asking for actual *formatting features*, you will need to add these features to the formatting function, not to `pprintpp`.

### Isn't that *no runtime overhead* feature just a result of compiler optimization?

No. 
The whole format string is preprocessed at compile time, this is guaranteed.
