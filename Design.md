# Phobos Version 3

## Rationale

Phobos v2 is well over a decade old. As time goes by, best practices have evolved, mistakes have become apparent, needs have changed, and better ideas have appeared.

Phobos must also evolve.

## History

* Phobos v1 was for D1
* Phobos v2 was for D2

## Ground Rules

### Phobos v2

#### Must continue to be supported

When Phobos v2 was developed, Phobos v1 was obsoleted. This broke a lot of existing code, and was a wrenching change. Old code needed rewrites to use Phobos v2. Some of the discarded v1 modules were restored in the unDead library for the convenience of older code, but the damage had been done.

Therefore, v2 will continue to be supported. A user must be able to mix and match v3 code with v2 code.

#### Will receive bug fixes and enhancements to adapt to new operating systems and new D Compiler Editions

#### Will not focus on new functionality

### Phobos v3

#### Will use a different package prefix for it, `std3`, thus avoiding any name conflicts with v2

## Overarching Technical Goals

### Strings

#### No Autodecoding

Autodecoding turned out to be a mistake because it is pervasive and impractical to disable. The user will need to specifically ask for decoding using a filter such as `utf.byDchar`.

#### No Support For wchar And dchar

Since v2 was designed, the programming world has more or less standardized on UTF-8. The internals of algorithms, ranges, and functions will only work with UTF-8. Support for `wchar` and `dchar` will come in the form of algorithms `utf.byChar`, `utf.byWchar` and `utf.byDchar`.

#### Invalid Unicode

v2 throws an Exception when encountering invalid Unicode. Throwing an Exception entails using the GC, meaning string code cannot be `@nogc` nor `nothrow`.
Besides, common processing of strings means being tolerant of invalid Unicode rather than failing. For example, invalid Unicode is commonplace in web pages, and throwing an Exception when rendering such pages is unacceptable.

Removal of autodecoding will in itself address most of the problem. When decoding code units into code points is needed, APIs should allow callers to specify the desired behavior, such as returning an "error" result, or replacing invalid sequences with the Unicode substitution character.
Applications which need to handle untrusted data should be encouraged to use functions such as `std.utf.validate` (which return a `string` from `ubyte[]` only when it is valid UTF-8), or by-code-point decoding which reports errors for individual decoding operations.

### Minimize Memory Allocation

Enormous troubles and inefficiencies stem from general purpose library code allocating memory as the library sees fit. This makes the library code far less usable. Memory allocation strategies should be decided upon by the user of the library, not the library.

The easiest way to achieve this is to design the library to not use memory allocation at all. For example, std.path assembles paths from parts without doing allocations - it returns Voldemort ranges that the user can then use to emit the result into a buffer of the user's choosing.

Library routines may allocate memory internally, but not in a way that affects the user.

### Minimize Exceptions

Exceptions are inefficient and use the GC. Of course, they cannot be used in `nothrow` code. Examine each use of an Exception to see if it can be designed out of existence, like the Replacement Character method above. Design the return value such that an error is not necessary - for example, a string search function can return an empty string if not found rather than throw an Exception.
Investigate the use of Option/Sum types for error returns.

### Source Only

Currently, Phobos is distributed as a separately compiled library with "header" files that contain only the necessary Template implementations. The net result is that Phobos is primarily a source-only library in practice. By formalizing the library as a source only library, it becomes inured against variations in compiler flags. Whether it's a static or dynamic library becomes irrelevant, and there won't be impedance mismatches. DRuntime will remain a separately compiled library.

### Naming and Style Guidelines

Names should follow the existing naming guidelines here: [D Style Guide](https://dlang.org/dstyle.html)

When selecting a name for a type or method, a quick survey of how other popular languages name the equivalent type/method should be performed. For example, in .NET and Java, the `currTime()` method would be named `now()`. Using the same names as popular languages reduces the friction experienced by the engineer when migrating to D. Be prepared to provide examples from your survey in the Pull Request. In cases where there is no clear agreement or two examples are equally represented and alias *may* be appropriate for the purposes of moving past the block. 

Prefer whole words over abbreviations and dropped letters. For example, prefer `writeLine` over `writeln`. Choose the shortest name that accurately describes the feature. Abbreviations are acceptable where the abbreviation is in common usage and/or would result in a cumbersome name.

Phobos will use a 100 soft and 120 hard character column limit. This will be enforced via `dfmt` and `.editorconfig` files provided with the distribution.

### DUB Only Builds

The current Phobos build process is convoluted and relies on old or niche tools. This makes the process of building Phobos tedious and discourages active community participation. DUB will be the only tool needed to build and test Phobos V3. Building and testing Phobos V3 should be as simple as issuing a `dub test` command. The complete CI process for Phobos3 will be a single build script that the developer can comprehend and execute easily.

### Versioning and Release Schedule

Phobos3 versions will be versioned and distributed on the same schedule as the corresponding Compiler Edition. This allows Phobos to adopt the latest features from the in-development edition during the development of that edition. However, while Phobos will follow the same release schedule as the compiler editions, Phobos itself will not use the 'edition' terminology and will instead retain the use of the term 'version'.

Phobos will use a slightly modified SemVer. The major version will increment with each Phobos release that is tied to a compiler edition, the minor version will increment with the monthly compiler releases, and the patch version will be incremented on any out-of-band bugfix releases that occur.

## Specific Issues

### std.stdio

This module merges file I/O with formatting. Those should be split apart. File I/O should be done with ranges, and the formatting should work with any ranges.

stdin, stdout, and stderr match the stdin, stdout and stderr of core.stdc.stdio.
This causes terrible confusion. Should be stdIn, stdOut, and stdErr.

#### std.stdio.File

Does way, way too much. It's incomprehensible. Should be redesigned using building blocks.

### isXXXX templates

Are generally very hard to figure out what they do, such as `isSomeChar`. What the heck does that mean? The string ones are even worse.
