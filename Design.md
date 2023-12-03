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

#### Existing Phobos V2 roots remain reserved indefinitely

The existing roots of `core` for the Runtime, `etc` for C Interfaces, and `std` for the V2 Standard Library are to remain reserved for the foreseeable future. Both V2 and V3 will share usage of the `core` and `etc` roots. The `std` root is to be maintained for compatibility but no new features will be added.

#### Use multiple package prefixes/roots

Phobos V3 will use multiple package roots. This allows us to keep the old `std` package root while splitting up the new library into smaller, more manageable, components. Splitting the library into multiple roots provides the following benefits:

1. Only pay for what you use. Different roots can be compiled as separate static/shared libraries and linked as needed, reducing the overall weight of executables. This offers some relief to the long standing community request to break Phobos into individual packages. While we do not agree with atomizing Phobos to the extreme degree of one package per module, multiple package roots allow us to achieve some of that goal in a logical manner.
2. Multiple roots allow for layering components. The core roots form a foundation upon which to build higher-level packages.
3. Multiple roots allow for an expanded feature set without adding to package depth.
4. Not all roots need to be implemented for a given platform to be considered "supported". If only the core roots are required for a platform to be considered supported it becomes significantly less complicated to port D to new platforms.
5. Rules can be applied differently to core vs. non-core roots. For example, a possible rule could be that core roots may not throw exceptions under any circumstance, but non-core roots can. This is not intended to imply that throwing exceptions is encouraged, merely made available to allow more implementation flexibility for more complex constructs.
6. The old `std` root can continue to be maintained and built independently of the new Phobos roots.

Currently the core roots for Phobos V3 are `core`, `etc`, and `sys`. As a rule, Core roots *MUST NOT* import from non-core roots.

Proposed Package Structure (Existing Modules Only):
```
core.*
etc.*
std.*
sys
| algorithm
  | comparison
  | iteration
  | mutation
  | searching
  | setops
  | sorting
| array
| bigint
| bitmanip
| checkedint
| compiler
| complex
| conv
| datetime
  | date
  | interval
  | stopwatch
  | systime
  | timezone
| demangle
| exception
| functional
| getopt
| int128
| io (stdio)
  | file
  | mmfile
  | path
| math
  | algebraic
  | constants
  | exponential
  | hardware
  | operations
  | remainder
  | rounding
  | traits
  | trigonometry
  | special
| meta
| numeric
| outbuffer
| process
| random
| range
| signals
| stdint
| string
| sumtype
| system
| traits
| typecons
| uuid
| variant
data 
| base64
| csv
| json
| zip
text
| ascii
| encoding
| uni
| utf
net
| socket
| uri
```

**Open Questions**

1. `std.container`: Does it belong in `sys` or it's own root? I suspect this answer hinges on what rules are applied to it.
2. `std.concurrency`: Not all platforms support Concurrency mechanics. Should `std.concurrency` be required in the core roots or moved to a non-core root?
3. `std.parallelism`: Not all platforms support Task Parallelism mechanics. Should `std.parallelism` be required in the core roots or moved to a non-core root?
4. `std.digest`: Cryptography routines have unique requirements that would be best served by providing them in their own root. But in all cases providing bespoke implementations of cryptography routines is never recommended. Third-party trusted implementations should be used instead. This could be from OpenSSL/LibreSSL (POSIX), SChannel/BCrypt (Windows), or CryptoKit (MacOS/IOS). It is acceptable to provide implementations of Non-Cryptographic routines such as CRC and Murmur. This would entail removing all the digests except CRC and Murmur from this package and building a separate cryptography package.
5. Which, if any, modules should be removed? This could be either a permanent removal or pending replacement with ground-up rewrites.

## Overarching Technical Goals

### Strings

#### No Autodecoding

Autodecoding turned out to be a mistake because it is pervasive and impractical to disable. The user will need to specifically ask for decoding using a filter such as `utf.byDchar`.

#### No Support For wchar And dchar

Since v2 was designed, the programming world has more or less standardized on UTF-8. The internals of algorithms, ranges, and functions will only work with UTF-8. Support for `wchar` and `dchar` will come in the form of algorithms `utf.byChar`, `utf.byWchar` and `utf.byDchar`.

#### Invalid Unicode

v2 throws an Exception when encountering invalid Unicode. Throwing an Exception entails using the GC, meaning string code cannot be `@nogc` nor `nothrow`.
Besides, common processing of strings means being tolerant of invalid Unicode rather than failing. For example, invalid Unicode is commonplace in web pages, and throwing an Exception when rendering such pages is unacceptable.
Hence, invalid Unicode will be ignored when practical, and if not practical the Unicode replacement character code point will be substituted for the invalid Unicode.

If the user chooses, a separate adapter can be used to detect invalid Unicode and handle it as desired.

### No Memory Allocation

Enormous troubles and inefficiencies stem from general purpose library code allocating memory as the library sees fit. This makes the library code far less usable. Memory allocation strategies should be decided upon by the user of the library, not the library.

The easiest way to achieve this is to design the library to not use memory allocation at all. For example, std.path assembles paths from parts without doing allocations - it returns Voldemort ranges that the user can then use to emit the result into a buffer of the user's choosing.

Library routines may allocate memory internally, but not in a way that affects the user.

### No Exceptions

Exceptions are inefficient and use the GC. Of course, they cannot be used in `nothrow` code. Examine each use of an Exception to see if it can be designed out of existence, like the Replacement Character method above. Design the return value such that an error is not necessary - for example, a string search function can return an empty string if not found rather than throw an Exception.
Investigate the use of Option types for error returns.

### Header Only

By making the library header only, it becomes inured against variations in compiler flags. Whether it's a static or dynamic library becomes irrelevant, and there won't be impedance mismatches.

## Specific Issues

### std.stdio

This module merges file I/O with formatting. Those should be split apart. File I/O should be done with ranges, and the formatting should work with any ranges.

stdin, stdout, and stderr match the stdin, stdout and stderr of core.stdc.stdio.
This causes terrible confusion. Should be stdIn, stdOut, and stdErr.

#### std.stdio.File

Does way, way too much. It's incomprehensible. Should be redesigned using building blocks.

### isXXXX templates

Are generally very hard to figure out what they do, such as `isSomeChar`. What the heck does that mean? The string ones are even worse.
