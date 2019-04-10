
[TOC]

# Preface
## Overview
Hyperscan is a regular expression engine designed to offer high performance, the ability to match multiple expressions simultaneously and flexibility in scanning operation.

Patterns are provided to a compilation interface which generates an immutable pattern database. The scan interface then can be used to scan a target data buffer for the given patterns, returning any matching results from that data buffer. Hyperscan also provides a streaming mode, in which matches that span several blocks in a stream are detected.

This document is designed to facilitate code-level integration of the Hyperscan library with existing or new applications.

Introduction is a short overview of the Hyperscan library, with more detail on the Hyperscan API provided in the subsequent sections: Compiling Patterns and Scanning for Patterns.

Performance Considerations provides details on various factors which may impact the performance of a Hyperscan integration.

API Reference: Constants and API Reference: Files provides a detailed summary of the Hyperscan Application Programming Interface (API).

## Audience
This guide is aimed at developers interested in integrating Hyperscan into an application. For information on building the Hyperscan library, see the Quick Start Guide.

## Conventions
- Text in a fixed-width font refers to a code element, e.g. type name; function or method name.
- Text in a coloured fixed-width font refers to a regular expression or a part of a regular expression.

# Introduction
Hyperscan is a software regular expression matching engine designed with high performance and flexibility in mind. It is implemented as a library that exposes a straightforward C API.

The Hyperscan API itself is composed of two major components:

## Compilation
These functions take a group of regular expressions, along with identifiers and option flags, and compile them into an immutable database that can be used by the Hyperscan scanning API. This compilation process performs considerable analysis and optimization work in order to build a database that will match the given expressions efficiently.

If a pattern cannot be built into a database for any reason (such as the use of an unsupported expression construct, or the overflowing of a resource limit), an error will be returned by the pattern compiler.

Compiled databases can be serialized and relocated, so that they can be stored to disk or moved between hosts. They can also be targeted to particular platform features (for example, the use of Intel® Advanced Vector Extensions 2 (Intel® AVX2) instructions).

See Compiling Patterns for more detail.

## Scanning
Once a Hyperscan database has been created, it can be used to scan data in memory. Hyperscan provides several scanning modes, depending on whether the data to be scanned is available as a single contiguous block, whether it is distributed amongst several blocks in memory at the same time, or whether it is to be scanned as a sequence of blocks in a stream.

Matches are delivered to the application via a user-supplied callback function that is called synchronously for each match.

For a given database, Hyperscan provides several guarantees:

- No memory allocations occur at runtime with the exception of two fixed-size allocations, both of which should be done ahead of time for performance-critical applications:
- Scratch space: temporary memory used for internal data at scan time. Structures in scratch space do not persist beyond the end of a single scan call.
- Stream state: in streaming mode only, some state space is required to store data that persists between scan calls for each stream. This allows Hyperscan to track matches that span multiple blocks of data.
- The sizes of the scratch space and stream state (in streaming mode) required for a given database are fixed and determined at database compile time. This means that the memory requirements of the application are known ahead of time, and these structures can be pre-allocated if required for performance reasons.
- Any pattern that has successfully been compiled by the Hyperscan compiler can be scanned against any input. There are no internal resource limits or other limitations at runtime that could cause a scan call to return an error.
See Scanning for Patterns for more detail.

## Tools
Some utilities for testing and benchmarking Hyperscan are included with the library. See Tools for more information.

## Example Code
Some simple example code demonstrating the use of the Hyperscan API is available in the examples/ subdirectory of the Hyperscan distribution.

# Getting Started
## Very Quick Start
1. Clone Hyperscan

    	cd <where-you-want-hyperscan-source>
    	git clone git://github.com/intel/hyperscan
2. Configure Hyperscan

	Ensure that you have the correct dependencies present, and then:

        cd <where-you-want-to-build-hyperscan>
        mkdir <build-dir>
        cd <build-dir>
        cmake [-G <generator>] [options] <hyperscan-source-path>
	Known working generators:
 		Unix Makefiles — make-compatible makefiles (default on Linux/FreeBSD/Mac OS X)
		Ninja — Ninja build files.
		Visual Studio 15 2017 — Visual Studio projects
	Generators that might work include:
 		Xcode — OS X Xcode projects.

3. Build Hyperscan

	Depending on the generator used:
 		cmake --build . — will build everything
		make -j<jobs> — use makefiles in parallel
 		ninja — use Ninja build
		MsBuild.exe — use Visual Studio MsBuild
 		etc.

4. Check Hyperscan

	Run the Hyperscan unit tests:

		bin/unit-hyperscan

## Requirements
### Hardware
Hyperscan will run on x86 processors in 64-bit (Intel® 64 Architecture) and 32-bit (IA-32 Architecture) modes.

Hyperscan is a high performance software library that takes advantage of recent Intel architecture advances. At a minimum, support for Supplemental Streaming SIMD Extensions 3 (SSSE3) is required, which should be available on any modern x86 processor.

Additionally, Hyperscan can make use of:

- Intel Streaming SIMD Extensions 4.2 (SSE4.2)
- the POPCNT instruction
- Bit Manipulation Instructions (BMI, BMI2)
- Intel Advanced Vector Extensions 2 (Intel AVX2)

if present.

These can be determined at library compile time, see Target Architecture.

### Software
As a software library, Hyperscan doesn’t impose any particular runtime software requirements, however to build the Hyperscan library we require a modern C and C++ compiler – in particular, Hyperscan requires C99 and C++11 compiler support. The supported compilers are:

- GCC, v4.8.1 or higher
- Clang, v3.4 or higher (with libstdc++ or libc++)
- Intel C++ Compiler v15 or higher
- Visual C++ 2017 Build Tools

Examples of operating systems that Hyperscan is known to work on include:

Linux:

- Ubuntu 14.04 LTS or newer
- RedHat/CentOS 7 or newer

FreeBSD:

- 10.0 or newer

Windows:

- 8 or newer

Mac OS X:

- 10.8 or newer, using XCode/Clang

Hyperscan may compile and run on other platforms, but there is no guarantee. We currently have experimental support for Windows using Intel C++ Compiler or Visual Studio 2017.

In addition, the following software is required for compiling the Hyperscan library:

| Dependency | Version | Notes |
| :-: | :-: | :-: |
|CMake|	>=2.8.11| |
|Ragel|	6.9	 ||
|Python	|2.7||
|Boost	|>=1.57	|Boost headers required|
|Pcap	|>=0.8	|Optional: needed for example code only|

Most of these dependencies can be provided by the package manager on the build system (e.g. Debian/Ubuntu/RedHat packages, FreeBSD ports, etc). However, ensure that the correct version is present. As for Windows, in order to have Ragel, you may use Cygwin to build it from source.

### Boost Headers
Compiling Hyperscan depends on a recent version of the Boost C++ header library. If the Boost libraries are installed on the build machine in the usual paths, CMake will find them. If the Boost libraries are not installed, the location of the Boost source tree can be specified during the CMake configuration step using the BOOST_ROOT variable (described below).

Another alternative is to put a copy of (or a symlink to) the boost subdirectory in <hyperscan-source-path>/include/boost.

For example: for the Boost-1.59.0 release:

	ln -s boost_1_59_0/boost <hyperscan-source-path>/include/boost
As Hyperscan uses the header-only parts of Boost, it is not necessary to compile the Boost libraries.

### CMake Configuration
When CMake is invoked, it generates build files using the given options. Options are passed to CMake in the form -D<variable name>=<value>. Common options for CMake include:

|Variable | Description |
| :-: | :-: |
|CMAKE_C_COMPILER	|C compiler to use. Default is /usr/bin/cc.|
|CMAKE_CXX_COMPILER	|C++ compiler to use. Default is /usr/bin/c++.|
|CMAKE_INSTALL_PREFIX	|Install directory for install target|
|CMAKE_BUILD_TYPE	|Define which kind of build to generate. Valid options are Debug, Release, RelWithDebInfo, and MinSizeRel. Default is RelWithDebInfo.|
|BUILD_SHARED_LIBS	|Build Hyperscan as a shared library instead of the default static library.|
|BUILD_STATIC_AND_SHARED	|Build both static and shared Hyperscan libs. Default off.|
|BOOST_ROOT	|Location of Boost source tree.|
|DEBUG_OUTPUT	|Enable very verbose debug output. Default off.|
|FAT_RUNTIME	|Build the fat runtime. Default true on Linux, not available elsewhere.|

For example, to generate a Debug build:

    cd <build-dir>
    cmake -DCMAKE_BUILD_TYPE=Debug <hyperscan-source-path>

### Build Type
CMake determines a number of features for a build based on the Build Type. Hyperscan defaults to RelWithDebInfo, i.e. “release with debugging information”. This is a performance optimized build without runtime assertions but with debug symbols enabled.

The other types of builds are:

- Release: as above, but without debug symbols
- MinSizeRel: a stripped release build
- Debug: used when developing Hyperscan. Includes runtime assertions (which has a large impact on runtime performance), and will also enable some other build features like building internal unit tests.

### Target Architecture
Unless using the fat runtime, by default Hyperscan will be compiled to target the instruction set of the processor of the machine that being used for compilation. This is done via the use of -march=native. The result of this means that a library built on one machine may not work on a different machine if they differ in supported instruction subsets.

To override the use of -march=native, set appropriate flags for the compiler in CFLAGS and CXXFLAGS environment variables before invoking CMake, or CMAKE_C_FLAGS and CMAKE_CXX_FLAGS on the CMake command line. For example, to set the instruction subsets up to SSE4.2 using GCC 4.8:

    cmake -DCMAKE_C_FLAGS="-march=corei7" \
      -DCMAKE_CXX_FLAGS="-march=corei7" <hyperscan-source-path>
For more information, refer to Instruction Set Specialization.

### Fat Runtime
A feature introduced in Hyperscan v4.4 is the ability for the Hyperscan library to dispatch the most appropriate runtime code for the host processor. This feature is called the “fat runtime”, as a single Hyperscan library contains multiple copies of the runtime code for different instruction sets.

    Note
    The fat runtime feature is only available on Linux. Release builds of Hyperscan will default to having the fat runtime enabled where supported.

When building the library with the fat runtime, the Hyperscan runtime code will be compiled multiple times for these different instruction sets, and these compiled objects are combined into one library. There are no changes to how user applications are built against this library.

When applications are executed, the correct version of the runtime is selected for the machine that it is running on. This is done using a CPUID check for the presence of the instruction set, and then an indirect function is resolved so that the right version of each API function is used. There is no impact on function call performance, as this check and resolution is performed by the ELF loader once when the binary is loaded.

If the Hyperscan library is used on x86 systems without SSSE3, the runtime API functions will resolve to functions that return HS_ARCH_ERROR instead of potentially executing illegal instructions. The API function hs_valid_platform() can be used by application writers to determine if the current platform is supported by Hyperscan.

At of this release, the variants of the runtime that are built, and the CPU capability that is required, are the following:

|Variant	|CPU Feature Flag(s) Required	|gcc arch flag|
|:-:|:-:|:-:|
|Core 2	|SSSE3	|-march=core2|
|Core i7	|SSE4_2 and POPCNT	|-march=corei7|
|AVX 2	|AVX2	|-march=core-avx2|
|AVX 512|	AVX512BW (see note below)|	-march=skylake-avx512|

    Note
    Hyperscan v4.5 adds support for AVX-512 instructions - in particular the AVX-512BW instruction set that was introduced on Intel “Skylake” Xeon processors - however the AVX-512 runtime variant is not enabled by default in fat runtime builds as not all toolchains support AVX-512 instruction sets. To build an AVX-512 runtime, the CMake variable BUILD_AVX512 must be enabled manually during configuration. For example:

    cmake -DBUILD_AVX512=on <...>

As the fat runtime requires compiler, libc, and binutils support, at this time it will only be enabled for Linux builds where the compiler supports the indirect function “ifunc” function attribute.

This attribute should be available on all supported versions of GCC, and recent versions of Clang and ICC. There is currently no operating system support for this feature on non-Linux systems.

# Compiling Patterns
## Building a Database

The Hyperscan compiler API accepts regular expressions and converts them into a compiled pattern database that can then be used to scan data.

The API provides three functions that compile regular expressions into databases:

1. hs_compile(): compiles a single expression into a pattern database.
2. hs_compile_multi(): compiles an array of expressions into a pattern database. All of the supplied patterns will be scanned for concurrently at scan time, with user-supplied identifiers returned when they match.
3. hs_compile_ext_multi(): compiles an array of expressions as above, but allows Extended Parameters to be specified for each expression.

Compilation allows the Hyperscan library to analyze the given pattern(s) and pre-determine how to scan for these patterns in an optimized fashion that would be far too expensive to compute at run-time.

When compiling expressions, a decision needs to be made whether the resulting compiled patterns are to be used in a streaming, block or vectored mode:

- Streaming mode: the target data to be scanned is a continuous stream, not all of which is available at once; blocks of data are scanned in sequence and matches may span multiple blocks in a stream. In streaming mode, each stream requires a block of memory to store its state between scan calls.
- Block mode: the target data is a discrete, contiguous block which can be scanned in one call and does not require state to be retained.
- Vectored mode: the target data consists of a list of non-contiguous blocks that are available all at once. As for block mode, no retention of state is required.

To compile patterns to be used in streaming mode, the mode parameter of hs_compile() must be set to HS_MODE_STREAM; similarly, block mode requires the use of HS_MODE_BLOCK and vectored mode requires the use of HS_MODE_VECTORED. A pattern database compiled for one mode (streaming, block or vectored) can only be used in that mode. The version of Hyperscan used to produce a compiled pattern database must match the version of Hyperscan used to scan with it.

Hyperscan provides support for targeting a database at a particular CPU platform; see Instruction Set Specialization for details.

## Pattern Support
Hyperscan supports the pattern syntax used by the PCRE library (“libpcre”), described at <http://www.pcre.org/>. However, not all constructs available in libpcre are supported. The use of unsupported constructs will result in compilation errors.

The version of PCRE used to validate Hyperscan’s interpretation of this syntax is 8.41 or above.

### Supported Constructs
The following regex constructs are supported by Hyperscan:

- Literal characters and strings, with all libpcre quoting and character escapes.
- Character classes such as . (dot), [abc], and [^abc], as well as the predefined character classes \s, \d, \w, \v, and \h and their negated counterparts (\S, \D, \W, \V, and \H).
- The POSIX named character classes [[:xxx:]] and negated named character classes [[:^xxx:]].
- Unicode character properties, such as \p{L}, \P{Sc}, \p{Greek}.
- Quantifiers:
- Quantifiers such as ?, * and + are supported when applied to arbitrary supported sub-expressions.
- Bounded repeat qualifiers such as {n}, {m,n}, {n,} are supported with limitations.
- For arbitrary repeated sub-patterns: n and m should be either small or infinite, e.g. (a|b}{4}, (ab?c?d){4,10} or (ab(cd)*){6,}.
- For single-character width sub-patterns such as [^\a] or . or x, nearly all repeat counts are supported, except where repeats are extremely large (maximum bound greater than 32767). Stream states may be very large for large bounded repeats, e.g. a.{2000}b. Note: such sub-patterns may be considerably cheaper if at the beginning or end of patterns and especially if the HS_FLAG_SINGLEMATCH flag is on for that pattern.
- Lazy modifiers (? appended to another quantifier, e.g. \w+?) are supported but ignored (as Hyperscan reports all matches).
- Parenthesization, including the named and unnamed capturing and non-capturing forms. However, capturing is ignored.
- Alternation with the | symbol, as in foo|bar.
- The anchors ^, $, \A, \Z and \z.

- Option modifiers:

	These allow behaviour to be switched on (with (?<option>)) and off (with (?-<option>)) for a sub-pattern. The supported options are:

        i: Case-insensitive matching, as per HS_FLAG_CASELESS.
        m: Multi-line matching, as per HS_FLAG_MULTILINE.
        s: Interpret . as “any character”, as per HS_FLAG_DOTALL.
        x: Extended syntax, which will ignore most whitespace in the pattern for compatibility with libpcre’s PCRE_EXTENDED option.

	For example, the expression foo(?i)bar(?-i)baz will switch on case-insensitive matching only for the bar portion of the match.

- The \b and \B zero-width assertions (word boundary and ‘not word boundary’, respectively).

- Comments in (?# comment) syntax.

- The (*UTF8) and (*UCP) control verbs at the beginning of a pattern, used to enable UTF-8 and UCP mode.

        Note
        Bounded-repeat quantifiers with large repeat counts of arbitrary expressions (e.g. ([a-z]|bc*d|xy?z){1000,5000}) will result in a “Pattern too large” error at pattern compile time.

        Note
        At this time, not all patterns can be successfully compiled with the HS_FLAG_SOM_LEFTMOST flag, which enables per-pattern support for Start of Match. The patterns that support this flag are a subset of patterns that can be successfully compiled with Hyperscan; notably, many bounded repeat forms that can be compiled with Hyperscan without the Start of Match flag enabled cannot be compiled with the flag enabled.

### Unsupported Constructs
The following regex constructs are not supported by Hyperscan:

- Backreferences and capturing sub-expressions.
- Arbitrary zero-width assertions.
- Subroutine references and recursive patterns.
- Conditional patterns.
- Backtracking control verbs.
- The \C “single-byte” directive (which breaks UTF-8 sequences).
- The \R newline match.
- The \K start of match reset directive.
- Callouts and embedded code.
- Atomic grouping and possessive quantifiers.

## Semantics
While Hyperscan follows libpcre syntax, it provides different semantics. The major departures from libpcre semantics are motivated by the requirements of streaming and multiple simultaneous pattern matching.

The major departures from libpcre semantics are:

1. Multiple pattern matching: Hyperscan allows matches to be reported for several patterns simultaneously. This is not equivalent to separating the patterns by | in libpcre, which evaluates alternations left-to-right.
2. Lack of ordering: the multiple matches that Hyperscan produces are not guaranteed to be ordered, although they will always fall within the bounds of the current scan.
3. End offsets only: Hyperscan’s default behaviour is only to report the end offset of a match. Reporting of the start offset can be enabled with per-expression flags at pattern compile time. See Start of Match for details.
4. “All matches” reported: scanning /foo.*bar/ against fooxyzbarbar will return two matches from Hyperscan – at the points corresponding to the ends of fooxyzbar and fooxyzbarbar. In contrast, libpcre semantics by default would report only one match at fooxyzbarbar (greedy semantics) or, if non-greedy semantics were switched on, one match at fooxyzbar. This means that switching between greedy and non-greedy semantics is a no-op in Hyperscan.

To support libpcre quantifier semantics while accurately reporting streaming matches at the time they occur is impossible. For example, consider the pattern above, /foo.*bar/, in streaming mode, against the following stream (three blocks scanned in sequence):

|block 1	|block 2	|block 3|
|:-:|:-:|
|fooxyzbar	|baz	|qbar|
Since the .* repeat in the pattern is a greedy repeat in libpcre, it must match as much as possible without causing the rest of the pattern to fail. However, in streaming mode, this would require knowledge of data in the stream beyond the current block being scanned.

In this example, the match at offset 9 in the first block is only the correct match (under libpcre semantics) if there is no bar in a subsequent block – as in block 3 – which would constitute a better match for the pattern.

### Start of Match
In standard operation, Hyperscan will only provide the end offset of a match when the match callback is called. If the HS_FLAG_SOM_LEFTMOST flag is specified for a particular pattern, then the same set of matches is returned, but each match will also provide the leftmost possible start offset corresponding to its end offset.

Using the SOM flag entails a number of trade-offs and limitations:

- Reduced pattern support: For many patterns, tracking SOM is complex and can result in Hyperscan failing to compile a pattern with a “Pattern too large” error, even if the pattern is supported in normal operation.
- - Increased stream state: At scan time, state space is required to track potential SOM offsets, and this must be stored in persistent stream state in streaming mode. Accordingly, SOM will generally increase the stream state required to match a pattern.
- Performance overhead: Similarly, there is generally a performance cost associated with tracking SOM.
Incompatible features: Some other Hyperscan pattern flags (such as HS_FLAG_SINGLEMATCH and HS_FLAG_PREFILTER) can not be used in combination with SOM. Specifying them together with HS_FLAG_SOM_LEFTMOST will result in a compilation error.
- In streaming mode, the amount of precision delivered by SOM can be controlled with the SOM horizon flags. These instruct Hyperscan to deliver accurate SOM information within a certain distance of the end offset, and return a special start offset of HS_OFFSET_PAST_HORIZON otherwise. Specifying a small or medium SOM horizon will usually reduce the stream state required for a given database.

        Note
        In streaming mode, the start offset returned for a match may refer to a point in the stream before the current block being scanned. Hyperscan provides no facility for accessing earlier blocks; if the calling application needs to inspect historical data, then it must store it itself.

### Extended Parameters
In some circumstances, more control over the matching behaviour of a pattern is required than can be specified easily using regular expression syntax. For these scenarios, Hyperscan provides the hs_compile_ext_multi() function that allows a set of “extended parameters” to be set on a per-pattern basis.

Extended parameters are specified using an hs_expr_ext_t structure, which provides the following fields:

- flags: Flags governing which of the other fields in the structure are used.
- min_offset: The minimum end offset in the data stream at which this expression should match successfully.
- max_offset: The maximum end offset in the data stream at which this expression should match successfully.
- min_length: The minimum match length (from start to end) required to successfully match this expression.
- edit_distance: Match this expression within a given Levenshtein distance.
- hamming_distance: Match this expression within a given Hamming distance.
These parameters either allow the set of matches produced by a pattern to be constrained at compile time (rather than relying on the application to process unwanted matches at runtime), or allow matching a pattern approximately (within a given edit distance) to produce more matches.

For example, the pattern /foo.*bar/ when given a min_offset of 10 and a max_offset of 15 will not produce matches when scanned against foobar or foo0123456789bar but will produce a match against the data streams foo0123bar or foo0123456bar.

Similarly, the pattern /foobar/ when given an edit_distance of 2 will produce matches when scanned against foobar, f00bar, fooba, fobr, fo_baz, foooobar, and anything else that lies within edit distance of 2 (as defined by Levenshtein distance).

When the same pattern /foobar/ is given a hamming_distance of 2, it will produce matches when scanned against foobar, boofar, f00bar, and anything else with at most two characters substituted from the original pattern. For more details, see the Approximate matching section.

### Prefiltering Mode
Hyperscan provides a per-pattern flag, HS_FLAG_PREFILTER, which can be used to implement a prefilter for a pattern than Hyperscan would not ordinarily support.

This flag instructs Hyperscan to compile an “approximate” version of this pattern for use in a prefiltering application, even if Hyperscan does not support the pattern in normal operation.

The set of matches returned when this flag is used is guaranteed to be a superset of the matches specified by the non-prefiltering expression.

If the pattern contains pattern constructs not supported by Hyperscan (such as zero-width assertions, back-references or conditional references) these constructs will be replaced internally with broader constructs that may match more often.

For example, the pattern /(\w+) again \1/ contains the back-reference \1. In prefiltering mode, this pattern might be approximated by having its back-reference replaced with its referent, forming /\w+ again \w+/.

Furthermore, in prefiltering mode Hyperscan may simplify a pattern that would otherwise return a “Pattern too large” error at compile time, or for performance reasons (subject to the matching guarantee above).

It is generally expected that the application will subsequently confirm prefilter matches with another regular expression matcher that can provide exact matches for the pattern.

        Note
        The use of this flag in combination with Start of Match mode (using the HS_FLAG_SOM_LEFTMOST flag) is not currently supported and will result in a pattern compilation error.

## Instruction Set Specialization
Hyperscan is able to make use of several modern instruction set features found on x86 processors to provide improvements in scanning performance.

Some of these features are selected when the library is built; for example, Hyperscan will use the native POPCNT instruction on processors where it is available and the library has been optimized for the host architecture.

        Note
        By default, the Hyperscan runtime is built with the -march=native compiler flag and (where possible) will make use of all instructions known by the host’s C compiler.

To use some instruction set features, however, Hyperscan must build a specialized database to support them. This means that the target platform must be specified at pattern compile time.

The Hyperscan compiler API functions all accept an optional hs_platform_info_t argument, which describes the target platform for the database to be built. If this argument is NULL, the database will be targeted at the current host platform.

The hs_platform_info_t structure has two fields:

1. tune: This allows the application to specify information about the target platform which may be used to guide the optimisation process of the compile. Use of this field does not limit the processors that the resulting database can run on, but may impact the performance of the resulting database.
2. cpu_features: This allows the application to specify a mask of CPU features that may be used on the target platform. For example, HS_CPU_FEATURES_AVX2 can be specified for Intel® Advanced Vector Extensions 2 (Intel® AVX2) instruction set support. If a flag for a particular CPU feature is specified, the database will not be usable on a CPU without that feature.
An hs_platform_info_t structure targeted at the current host can be built with the hs_populate_platform() function.

See API Reference: Constants for the full list of CPU tuning and feature flags.

## Approximate matching
Hyperscan provides an experimental approximate matching mode, which will match patterns within a given edit distance. The exact matching behavior is defined as follows:

1. Edit distance is defined as Levenshtein distance. That is, there are three possible edit types considered: insertion, removal and substitution. A more formal description can be found on Wikipedia.
2. Hamming distance is the number of positions by which two strings of equal length differ. That is, it is the number of substitutions required to convert one string to the other. There are no insertions or removals when approximate matching using a Hamming distance. A more formal description can be found on Wikipedia.
3. Approximate matching will match all corpora within a given edit or Hamming distance. That is, given a pattern, approximate matching will match anything that can be edited to arrive at a corpus that exactly matches the original pattern.
4. Matching semantics are exactly the same as described in Semantics.
Here are a few examples of approximate matching:

- Pattern /foo/ can match foo when using regular Hyperscan matching behavior. With approximate matching within edit distance 2, the pattern will produce matches when scanned against foo, foooo, f00, f, and anything else that lies within edit distance 2 of matching corpora for the original pattern (foo in this case).
- Pattern /foo(bar)+/ with edit distance 1 will match foobarbar, foobarb0r, fooarbar, foobarba, f0obarbar, fobarbar and anything else that lies within edit distance 1 of matching corpora for the original pattern (foobarbar in this case).
- Pattern /foob?ar/ with edit distance 2 will match fooar, foo, fabar, oar and anything else that lies within edit distance 2 of matching corpora for the original pattern (fooar in this case).
Currently, there are trade-offs and limitations that come with approximate matching support. Here they are, in a nutshell:

- Reduced pattern support:
- For many patterns, approximate matching is complex and can result in Hyperscan failing to compile a pattern with a “Pattern too large” error, even if the pattern is supported in normal operation.
- Additionally, some patterns cannot be approximately matched because they reduce to so-called “vacuous” patterns (patterns that match everything). For example, pattern /foo/ with edit distance 3, if implemented, would reduce to matching zero-length buffers. Such patterns will result in a “Pattern cannot be approximately matched” compile error. Approximate matching within a Hamming distance does not remove symbols, so will not reduce to a vacuous pattern.
- Finally, due to the inherent complexities of defining matching behavior, approximate matching implements a reduced subset of regular expression syntax. Approximate matching does not support UTF-8 (and other multibyte character encodings), and word boundaries (that is, \b, \B and other equivalent constructs). Patterns containing unsupported constructs will result in “Pattern cannot be approximately matched” compile error.
- When using approximate matching in conjunction with SOM, all of the restrictions of SOM also apply. See Start of Match for more details.
- Increased stream state/byte code size requirements: due to approximate matching byte code being inherently larger and more complex than exact matching, the corresponding requirements also increase.
- Performance overhead: similarly, there is generally a performance cost associated with approximate matching, both due to increased matching complexity, and due to the fact that it will produce more matches.
Approximate matching is always disabled by default, and can be enabled on a per-pattern basis by using an extended parameter described in Extended Parameters.

## Logical Combinations
For situations when a user requires behaviour that depends on the presence or absence of matches from groups of patterns, Hyperscan provides support for the logical combination of patterns in a given pattern set, with three operators: NOT, AND and OR.

The logical value of such a combination is based on each expression’s matching status at a given offset. The matching status of any expression has a boolean value: false if the expression has not yet matched or true if the expression has already matched. In particular, the value of a NOT operation at a given offset is true if the expression it refers to is false at this offset.

For example, NOT 101 means that expression 101 has not yet matched at this offset.

A logical combination is passed to Hyperscan at compile time as an expression. This combination expression will raise matches at every offset where one of its sub-expressions matches and the logical value of the whole expression is true.

To illustrate, here is an example combination expression:

		((301 OR 302) AND 303) AND (304 OR NOT 305)
If expression 301 matches at offset 10, the logical value of 301 is true while the other patterns’ values are false. Hence, the whole combination’s value is false.

Then expression 303 matches at offset 20. Now the values of 301 and 303 are true while the other patterns’ values are still false. In this case, the combination’s value is true, so the combination expression raises a match at offset 20.

Finally, expression 305 has matches at offset 30. Now the values of 301, 303 and 305 are true while the other patterns’ values are still false. In this case, the combination’s value is false and no match is raised.

**Using Logical Combinations**

In logical combination syntax, an expression is written as infix notation, it consists of operands, operators and parentheses. The operands are expression IDs, and operators are ! (NOT), & (AND) or | (OR). For example, the combination described in the previous section would be written as:

		((301 | 302) & 303) & (304 | !305)
In a logical combination expression:

- The priority of operators are ! > & > |. For example:
        A&B|C is treated as (A&B)|C,
        A|B&C is treated as A|(B&C),
        A&!B is treated as A&(!B).
- Extra parentheses are allowed. For example:
        (A)&!(B) is the same as A&!B,
        (A&B)|C is the same as A&B|C.
- Whitespace is ignored.

To use a logical combination expression, it must be passed to one of the Hyperscan compile functions (hs_compile_multi(), hs_compile_ext_multi()) along with the HS_FLAG_COMBINATION flag, which identifies the pattern as a logical combination expression. The patterns referred to in the logical combination expression must be compiled together in the same pattern set as the combination expression.

When an expression has the HS_FLAG_COMBINATION flag set, it ignores all other flags except the HS_FLAG_SINGLEMATCH flag and the HS_FLAG_QUIET flag.

Hyperscan will reject logical combination expressions at compile time that evaluate to true when no patterns have matched; for example:

        !101
        !101|102
        !101&!102
        !(101&102)
Patterns that are referred to as operands within a logical combination (for example, 301 through 305 in the examples above) may also use the HS_FLAG_QUIET flag to silence the reporting of individual matches for those patterns. In the absence of this flag, all matches (for both individual patterns and their logical combinations) will be reported.

When an expression has both the HS_FLAG_COMBINATION flag and the HS_FLAG_QUIET flag set, no matches for this logical combination will be reported.

-------------------

# Scanning for Patterns
Hyperscan provides three different scanning modes, each with its own scan function beginning with hs_scan. In addition, streaming mode has a number of other API functions for managing stream state.

## Handling Matches
All of these functions will call a user-supplied callback function when a match is found. This function has the following signature:

	typedef ( * match_event_handler)(unsigned int id, unsigned long long from, unsigned long long to, unsigned int flags, void *context)
The id argument will be set to the identifier for the matching expression provided at compile time, and the to argument will be set to the end-offset of the match. If SOM was requested for the pattern (see Start of Match), the from argument will be set to the leftmost possible start-offset for the match.

The match callback function has the capability to halt scanning by returning a non-zero value.

See match_event_handler for more information.

## Streaming Mode
The core of the Hyperscan streaming runtime API consists of functions to open, scan, and close Hyperscan data streams:

- hs_open_stream(): allocates and initializes a new stream for scanning.
- hs_scan_stream(): scans a block of data in a given stream, raising matches as they are detected.
- hs_close_stream(): completes scanning of a given stream (raising any matches that occur at the end of the stream) and frees the stream state. After a call to hs_close_stream(), the stream handle is invalid and should not be used again for any purpose.

Any matches detected in the data as it is scanned are returned to the calling application via a function pointer callback.

The match callback function has the capability to halt scanning of the current data stream by returning a non-zero value. In streaming mode, the result of this is that the stream is then left in a state where no more data can be scanned, and any subsequent calls to hs_scan_stream() for that stream will return immediately with HS_SCAN_TERMINATED. The caller must still call hs_close_stream() to complete the clean-up process for that stream.

Streams exist in the Hyperscan library so that pattern matching state can be maintained across multiple blocks of target data – without maintaining this state, it would not be possible to detect patterns that span these blocks of data. This, however, does come at the cost of requiring an amount of storage per-stream (the size of this storage is fixed at compile time), and a slight performance penalty in some cases to manage the state.

While Hyperscan does always support a strict ordering of multiple matches, streaming matches will not be delivered at offsets before the current stream write, with the exception of zero-width asserts, where constructs such as \b and $ can cause a match on the final character of a stream write to be delayed until the next stream write or stream close operation.

### Stream Management
In addition to hs_open_stream(), hs_scan_stream(), and hs_close_stream(), the Hyperscan API provides a number of other functions for the management of streams:

- **hs_reset_stream()**: resets a stream to its initial state; this is equivalent to calling hs_close_stream() but will not free the memory used for stream state.
- **hs_copy_stream()**: constructs a (newly allocated) duplicate of a stream.
- **hs_reset_and_copy_stream()**: constructs a duplicate of a stream into another, resetting the destination stream first. This call avoids the allocation done by hs_copy_stream().

### Stream Compression
A stream object is allocated as a fixed size region of memory which has been sized to ensure that no memory allocations are required during scan operations. When the system is under memory pressure, it may be useful to reduce the memory consumed by streams that are not expected to be used soon. The Hyperscan API provides calls for translating a stream to and from a compressed representation for this purpose. The compressed representation differs from the full stream object as it does not reserve space for components which are not required given the current stream state. The Hyperscan API functions for this functionality are:

- **hs_compress_stream()**: fills the provided buffer with a compressed representation of the stream and returns the number of bytes consumed by the compressed representation. If the buffer is not large enough to hold the compressed representation, HS_INSUFFICIENT_SPACE is returned along with the required size. This call does not modify the original stream in any way: it may still be written to with hs_scan_stream(), used as part of the various reset calls to reinitialise its state, or hs_close_stream() may be called to free its resources.
- **hs_expand_stream()**: creates a new stream based on a buffer containing a compressed representation.
- **hs_reset_and_expand_stream()**: constructs a stream based on a buffer containing a compressed representation on top of an existing stream, resetting the existing stream first. This call avoids the allocation done by hs_expand_stream().
Note: it is not recommended to use stream compression between every call to scan for performance reasons as it takes time to convert between the compressed representation and a standard stream.

## Block Mode
The block mode runtime API consists of a single function: hs_scan(). Using the compiled patterns this function identifies matches in the target data, using a function pointer callback to communicate with the application.

This single hs_scan() function is essentially equivalent to calling hs_open_stream(), making a single call to hs_scan_stream(), and then hs_close_stream(), except that block mode operation does not incur all the stream related overhead.

## Vectored Mode
The vectored mode runtime API, like the block mode API, consists of a single function: hs_scan_vector(). This function accepts an array of data pointers and lengths, facilitating the scanning in sequence of a set of data blocks that are not contiguous in memory.

From the caller’s perspective, this mode will produce the same matches as if the set of data blocks were (a) scanned in sequence with a series of streaming mode scans, or (b) copied in sequence into a single block of memory and then scanned in block mode.

## Scratch Space
While scanning data, Hyperscan needs a small amount of temporary memory to store on-the-fly internal data. This amount is unfortunately too large to fit on the stack, particularly for embedded applications, and allocating memory dynamically is too expensive, so a pre-allocated “scratch” space must be provided to the scanning functions.

The function hs_alloc_scratch() allocates a large enough region of scratch space to support a given database. If the application uses multiple databases, only a single scratch region is necessary: in this case, calling hs_alloc_scratch() on each database (with the same scratch pointer) will ensure that the scratch space is large enough to support scanning against any of the given databases.

While the Hyperscan library is re-entrant, the use of scratch spaces is not. For example, if by design it is deemed necessary to run recursive or nested scanning (say, from the match callback function), then an additional scratch space is required for that context.

In the absence of recursive scanning, only one such space is required per thread and can (and indeed should) be allocated before data scanning is to commence.

In a scenario where a set of expressions are compiled by a single “master” thread and data will be scanned by multiple “worker” threads, the convenience function hs_clone_scratch() allows multiple copies of an existing scratch space to be made for each thread (rather than forcing the caller to pass all the compiled databases through hs_alloc_scratch() multiple times).

For example:

    hs_error_t err;
    hs_scratch_t *scratch_prototype = NULL;
    err = hs_alloc_scratch(db, &scratch_prototype);
    if (err != HS_SUCCESS) {
        printf("hs_alloc_scratch failed!");
        exit(1);
    }

    hs_scratch_t *scratch_thread1 = NULL;
    hs_scratch_t *scratch_thread2 = NULL;

    err = hs_clone_scratch(scratch_prototype, &scratch_thread1);
    if (err != HS_SUCCESS) {
        printf("hs_clone_scratch failed!");
        exit(1);
    }
    err = hs_clone_scratch(scratch_prototype, &scratch_thread2);
    if (err != HS_SUCCESS) {
        printf("hs_clone_scratch failed!");
        exit(1);
    }

    hs_free_scratch(scratch_prototype);

    /* Now two threads can both scan against database db,
       each with its own scratch space. */

## Custom Allocators
By default, structures used by Hyperscan at runtime (scratch space, stream state, etc) are allocated with the default system allocators, usually malloc() and free().

The Hyperscan API provides a facility for changing this behaviour to support applications that use custom memory allocators.

These functions are:

- **hs_set_database_allocator()**, which sets the allocate and free functions used for compiled pattern databases.
- **hs_set_scratch_allocator()**, which sets the allocate and free functions used for scratch space.
- **hs_set_stream_allocator()**, which sets the allocate and free functions used for stream state in streaming mode.
- **hs_set_misc_allocator()**, which sets the allocate and free functions used for miscellaneous data, such as compile error structures and informational strings.
The hs_set_allocator() function can be used to set all of the custom allocators to the same allocate/free pair.

----------------

# Serialization
For some applications, compiling Hyperscan pattern databases immediately prior to use is not an appropriate design. Some users may wish to:

- Compile pattern databases on a different host;
- Persist compiled databases to storage and only re-compile pattern databases when the patterns change;
- Control the region of memory in which the compiled database is located.

Hyperscan pattern databases are not completely flat in memory: they contain pointers and have specific alignment requirements. Therefore, they cannot be copied (or otherwise relocated) directly. To enable these use cases, Hyperscan provides functionality for serializing and deserializing compiled pattern databases.

The API provides the following functions:

1. **hs_serialize_database()**: serializes a pattern database into a flat relocatable buffer of bytes.
2. **hs_deserialize_database()**: reconstructs a newly allocated pattern database from the output of hs_serialize_database().
3. **hs_deserialize_database_at()**: reconstructs a pattern database at a given memory location from the output of hs_serialize_database().
4. **hs_serialized_database_size()**: given a serialized pattern database, returns the size of the memory block required by the database when deserialized.
5. **hs_serialized_database_info()**: given a serialized pattern database, returns a string containing information about the database. This call is analogous to hs_database_info().
        Note
        Hyperscan performs both version and platform compatibility checks upon deserialization. The hs_deserialize_database() and hs_deserialize_database_at() functions will only permit the deserialization of databases compiled with (a) the same version of Hyperscan and (b) platform features supported by the current host platform. See Instruction Set Specialization for more information on platform specialization.

## The Runtime Library
The main Hyperscan library (libhs) contains both the compiler and runtime portions of the library. This means that in order to support the Hyperscan compiler, which is written in C++, it requires C++ linkage and has a dependency on the C++ standard library.

Many embedded applications require only the scanning (“runtime”) portion of the Hyperscan library. In these cases, pattern compilation generally takes place on another host, and serialized pattern databases are delivered to the application for use.

To support these applications without requiring the C++ dependency, a runtime-only version of the Hyperscan library, called libhs_runtime, is also distributed. This library does not depend on the C++ standard library and provides all Hyperscan functions other that those used to compile databases.

-----------------

# Performance Considerations
Hyperscan supports a wide range of patterns in all three scanning modes. It is capable of extremely high levels of performance, but certain patterns can reduce performance markedly.

The following guidelines will help construct patterns and pattern sets that will perform better:

## Regular expression constructs
    Tip
    Do not hand-optimize regular expression constructs.

Quite a large number of regular expressions can be written in multiple ways. For example, caseless matching of /abc/ can be written as:

- /[Aa][Bb][Cc]/
- /(A|a)(B|b)(C|c)/
- /(?i)abc(?-i)/
- /abc/i

Hyperscan is capable of handling all these constructs. Unless there is a specific reason otherwise, do not rewrite patterns from one form to another.

As another example, matching of /foo(bar|baz)(frotz)?/ can be equivalently written as:

- /foobarfrotz|foobazfrotz|foobar|foobaz/

This change will not improve performance or reduce overheads.

## Library usage
    Tip
    Do not hand-optimize library usage.

The Hyperscan library is capable of dealing with small writes, unusually large and small pattern sets, etc. Unless there is a specific performance problem with some usage of the library, it is best to use Hyperscan in a simple and direct fashion. For example, it is unlikely for there to be much benefit in buffering input to the library into larger blocks unless streaming writes are tiny (say, 1-2 bytes at a time).

Unlike many other pattern matching products, Hyperscan will run faster with small numbers of patterns and slower with large numbers of patterns in a smooth fashion (as opposed to, typically, running at a moderate speed up to some fixed limit then either breaking or running half as fast).

Hyperscan also provides high-throughput matching with a single thread of control per core; if a database runs at 3.0 Gbps in Hyperscan it means that a 3000-bit block of data will be scanned in 1 microsecond in a single thread of control, not that it is required to scan 22 3000-bit blocks of data in 22 microseconds. Thus, it is not usually necessary to buffer data to supply Hyperscan with available parallelism.

## Block-based matching
    Tip
    Prefer block-based matching to streaming matching where possible.

Whenever input data appears in discrete records, or already requires some sort of transformation (e.g. URI normalization) that requires all the data to be accumulated before processing, it should be scanned in block rather than in streaming mode.

Unnecessary use of streaming mode reduces the number of optimizations that can be applied in Hyperscan and may make some patterns run slower.

If there is a mixture of ‘block’ and ‘streaming’ mode patterns, these should be scanned in separate databases except in the case that the streaming patterns vastly outnumber the block mode patterns.

## Unnecessary databases
    Tip
    Avoid unnecessary ‘union’ databases.

If there are 5 different types of network traffic T1 through T5 that must be scanned against 5 different signature sets, it will be far more efficient to construct 5 separate databases and scan traffic against the appropriate one than it will be to merge all 5 signature sets and remove inappropriate matches after the fact.

This will be true even in the case where there is substantial overlap among the signatures. Only if the common subset of the signatures is overwhelmingly large (say, 90% of the signatures appear in all 5 traffic types) should a database that merges all 5 signature sets be considered, and only then if there are no performance issues with specific patterns that appear outside the common subset.

## Allocate scratch ahead of time
    Tip
    Do not allocate scratch space for your pattern database just before calling a scan function. Instead, do it just after the pattern database is compiled or deserialized.

Scratch allocation is not necessarily a cheap operation. Since it is the first time (after compilation or deserialization) that a pattern database is used, Hyperscan performs some validation checks inside hs_alloc_scratch() and must also allocate memory.

Therefore, it is important to ensure that hs_alloc_scratch() is not called in the application’s scanning path just before hs_scan() (for example).

Instead, scratch should be allocated immediately after a pattern database is compiled or deserialized, then retained for later scanning operations.

## Allocate one scratch space per scanning context
    Tip
    A scratch space can be allocated so that it can be used with any one of a number of databases. Each concurrent scan operation (such as a thread) needs its own scratch space.

The hs_alloc_scratch() function can accept an existing scratch space and “grow” it to support scanning with another pattern database. This means that instead of allocating one scratch space for every database used by an application, one can call hs_alloc_scratch() with a pointer to the same hs_scratch_t and it will be sized appropriately for use with any of the given databases. For example:

    hs_database_t *db1 = buildDatabaseOne();
    hs_database_t *db2 = buildDatabaseTwo();
    hs_database_t *db3 = buildDatabaseThree();

    hs_error_t err;
    hs_scratch_t *scratch = NULL;
    err = hs_alloc_scratch(db1, &scratch);
    if (err != HS_SUCCESS) {
        printf("hs_alloc_scratch failed!");
        exit(1);
    }
    err = hs_alloc_scratch(db2, &scratch);
    if (err != HS_SUCCESS) {
        printf("hs_alloc_scratch failed!");
        exit(1);
    }
    err = hs_alloc_scratch(db3, &scratch);
    if (err != HS_SUCCESS) {
        printf("hs_alloc_scratch failed!");
        exit(1);
    }

    /* scratch may now be used to scan against any of
       the databases db1, db2, db3. */

## Anchored patterns
    Tip
    If a pattern is meant to appear at the start of data, be sure to anchor it.

Anchored patterns (/^.../) are far simpler to match than other patterns, especially patterns anchored to the start of the buffer (or stream, in streaming mode). Anchoring patterns to the end of the buffer results in less of a performance gain, especially in streaming mode.

There are a variety of ways to anchor a pattern to a particular offset:

- The ^ and \A constructs anchor the pattern to the start of the buffer. For example, /^foo/ can only match at offset 3.
- The $, \z and \Z constructs anchor the pattern to the end of the buffer. For example, /foo\z/ can only match when the data buffer being scanned ends in foo. (It should be noted that $ and \Z will also match before a newline at the end of the buffer, so /foo\z/ would match against either abc foo or abc foo\n.)
- The min_offset and max_offset extended parameters may also be used to constrain where a pattern could match. For example, the pattern /foo/ with a max_offset of 10 will only match at offsets less than or equal to 10 in the buffer. (This pattern could also be written as /^.{0,7}foo/, compiled with the HS_FLAG_DOTALL flag).

## Matching everywhere
    Tip
    Avoid patterns that match everywhere, and remember that our semantics are ‘match everywhere, end of match only’.

Pattern that match everywhere will run slowly due to the sheer number of matches that they return.

Patterns like /.*/ in an automata-based matcher will match before and after every single character position, so a buffer with 100 characters will return 101 matches. Greedy pattern matchers such as libpcre will return a single match in this case, but our semantics is to return all matches. This is likely to be very expensive for our code and for the client code of the library.

Another result of our semantics (“match everywhere”) is that patterns that have optional start or ending sections – for example /x?abcd*/ – may not perform as expected.

Firstly, the x? portion of the pattern is unnecessary, as it will not affect the match results.

Secondly, the above pattern will match ‘more’ than /abc/ but /abc/ will always detect any input data that will be matched by /x?abcd*/ – it will just produce fewer matches.

For example, input data 0123abcdddd will match /abc/ once but /abcd*/ five times (at abc, abcd, abcdd, abcddd, and abcdddd).

## Bounded repeats in streaming mode
    Tip
    Bounded repeats are expensive in streaming mode.

A bounded repeat construction such as /X.{1000,1001}abcd/ is extremely expensive in streaming mode, of necessity. It requires us to take action on each X character (itself expensive, relative to searching for longer strings) and potentially record a history of hundreds of offsets where X occurred in case the X and abcd characters are separated by a stream boundary.

Heavy and unnecessary use of bounded repeats should be avoided, especially where other parts of a signature are quite specific. For example, a virus signature that matches a virus payload may be sufficient without including a prefix that includes, for example, a 2-character Windows executable prefix and a bounded repeat beforehand.

## Prefer literals
    Tip
    Where possible, prefer patterns which ‘require’ literals, especially longer literals, and in streaming mode, prefer signatures that ‘require’ literals earlier in the pattern.

Patterns which must match on a literal will run faster than patterns that do not. For example:

- /\wab\d*\w\w\w/ will run faster than
- /\w\w\d*\w\w/, or, for that matter
- /\w(abc)?\d*\w\w\w/ (this contains a literal but it need not appear in the input).

Even implicit literals are better than none: /[0-2][3-5].*\w\w/ still effectively contains 9 2-character literals. No hand-optimization of this case is required; this pattern will not run faster if rewritten as: /(03|04|05|13|14|15|23|24|25).*\w\w/.

Under all circumstances it is better to use longer literals than shorter ones. A database consisting of 100 14-character literals will scan considerably faster than one consisting of 100 4-character literals and return fewer positives.

Additionally, in streaming mode, a signature that contains a longer literal early in the pattern is preferred to one that does not.

For example: /b\w*foobar/ is not as good a pattern as /blah\w*foobar/.

The disparity between these patterns is much smaller in block mode.

Longer literals anywhere in the pattern are still preferred in streaming mode. For example, both of the above patterns are stronger and will scan faster than /b\w*fo/ even in streaming mode.

## “Dot all” mode
    Tip
    Use “dot all” mode where possible.

Not using the HS_FLAG_DOTALL pattern flag can be expensive, as implicitly, it means that patterns of the form /A.*B/ become /A[^\n]*B/.

It is likely that scanning tasks without the DOTALL flag are better done ‘line at a time’, with the newline sequences marking the beginning and end of each block.

This will be true in most use-cases (an exception being where the DOTALL flag is off but the pattern contains either explicit newlines or constructs such as \s that implicitly match a newline character).

## Single-match flag
    Tip
    Consider using the single-match flag to limit matches to one match per pattern only if possible.

If only one match per pattern is required, use the flag provided to indicate this (HS_FLAG_SINGLEMATCH). This flag can allow a number of optimizations to be applied, allowing both performance improvements and state space reductions when streaming.

However, there is some overhead associated with tracking whether each pattern in the pattern set has matched, and some applications with infrequent matches may see reduced performance when the single-match flag is used.

## Start of Match flag
    Tip
    Do not request Start of Match information if it is not not needed.

Start of Match (SOM) information can be expensive to gather and can require large amounts of stream state to store in streaming mode. As such, SOM information should only be requested with the HS_FLAG_SOM_LEFTMOST flag for patterns that require it.

SOM information is not generally expected to be cheaper (in either performance terms or in stream state overhead) than the use of bounded repeats. Consequently, /foo.*bar/L with a check on start of match values after the callback is considerably more expensive and general than /foo.{300}bar/.

Similarly, the hs_expr_ext::min_length extended parameter can be used to specify a lower bound on the length of the matches for a pattern. Using this facility may be more lightweight in some circumstances than using the SOM flag and post-confirming match length in the calling application.

## Approximate matching
    Tip
    Approximate matching is an experimental feature.

There is generally a performance impact associated with approximate matching due to the reduced specificity of the matches. This impact may vary significantly depending on the pattern and edit distance.

---------------------

# Tools
This section describes the set of utilities included with the Hyperscan library.

## Quick Check: hscheck
The hscheck tool allows the user to quickly check whether Hyperscan supports a group of patterns. If a pattern is rejected by Hyperscan’s compiler, the compile error is provided on standard output.

For example, given the following three patterns (the last of which contains a syntax error) in a file called /tmp/test:

    1:/foo.*bar/
    2:/abc|def|ghi/
    3:/((foo|bar)/
... the hscheck tool will produce the following output:

    $ bin/hscheck -e /tmp/test

    OK: 1:/foo.*bar/
    OK: 2:/abc|def|ghi/
    FAIL (compile): 3:/((foo|bar)/: Missing close parenthesis for group started at index 0.
    SUMMARY: 1 of 3 failed.
## Benchmarker: hsbench
The hsbench tool provides an easy way to measure Hyperscan’s performance for a particular set of patterns and corpus of data to be scanned.

Patterns are supplied in the format described below in Pattern Format, while the corpus must be provided in the form of a corpus database: this is a simple SQLite database format intended to allow for easy control of how a corpus is broken into blocks and streams.

    Note
    A group of Python scripts for constructing corpora databases from various input types, such as PCAP network traffic captures or text files, can be found in the Hyperscan source tree in tools/hsbench/scripts.

Running hsbench
Given a file full of patterns specified with -e and a corpus database specified with -c, hsbench will perform a single-threaded benchmark and produce output like this:

    $ hsbench -e /tmp/patterns -c /tmp/corpus.db

    Signatures:        /tmp/patterns
    Hyperscan info:    Version: 4.3.1 Features:  AVX2 Mode: STREAM
    Expression count:  200
    Bytecode size:     342,540 bytes
    Database CRC:      0x6cd6b67c
    Stream state size: 252 bytes
    Scratch size:      18,406 bytes
    Compile time:      0.153 seconds
    Peak heap usage:   78,073,856 bytes

    Time spent scanning:     0.600 seconds
    Corpus size:             72,138,183 bytes (63,946 blocks in 8,891 streams)
    Scan matches:            81 (0.001 matches/kilobyte)
    Overall block rate:      2,132,004.45 blocks/sec
    Overall throughput:      19,241.10 Mbit/sec

By default, the corpus is scanned twenty times, and the overall performance reported is computed based the total number of bytes scanned in the time it takes to perform all twenty scans. The number of repeats can be changed with the -n argument, and the results of each scan will be displayed if the --per-scan argument is specified.

To benchmark Hyperscan on more than one core, you can supply a list of cores with the -T argument, which will instruct hsbench to start one benchmark thread per core given and compute the throughput from the time taken to complete all of them.

    Tip
    For single-threaded benchmarks on multi-processor systems, we recommend using a utility like taskset to lock the hsbench process to one core and minimize jitter due to the operating system’s scheduler.

## Correctness Testing: hscollider
The hscollider tool, or Pattern Collider, provides a way to verify Hyperscan’s matching behaviour. It does this by compiling and scanning patterns (either singly or in groups) against known corpora and comparing the results against another engine (the “ground truth”). Two sources of ground truth for comparison are available:

- The PCRE library (http://pcre.org/).
- An NFA simulation run on Hyperscan’s compile-time graph representation. This is used if PCRE cannot support the pattern or if PCRE execution fails due to a resource limit.

Much of Hyperscan’s testing infrastructure is built on hscollider, and the tool is designed to take advantage of multiple cores and provide considerable flexibility in controlling the test. These options are described in the help (hscollider -h) and include:

- Testing in streaming, block or vectored mode.
- Testing corpora at different alignments in memory.
- Testing patterns in groups of varying size.
- Manipulating stream state or scratch space between tests.
- Cross-compilation and serialization/deserialization of databases.
- Synthetic generation of corpora given a pattern set.

### Using hscollider to debug a pattern
One common use-case for hscollider is to determine whether Hyperscan will match a pattern in the expected location, and whether this accords with PCRE’s behaviour for the same case.

Here is an example. We put our pattern in a file in Hyperscan’s pattern format:

    $ cat /tmp/pat
    1:/hatstand.*badgerbrush/
We put the corpus to be scanned in another file, with the same numeric identifier at the start to indicate that it should match pattern 1:

    $ cat /tmp/corpus
    1:__hatstand__hatstand__badgerbrush_badgerbrush
Then we can run hscollider with its verbosity turned up (-vv) so that individual matches are displayed in the output:

    $ bin/ue2collider -e /tmp/pat -c /tmp/corpus -Z 0 -T 1 -vv
    ue2collider: The Pattern Collider Mark II

    Number of threads:  1 (1 scanner, 1 generator)
    Expression path:    /tmp/pat
    Signature files:    none
    Mode of operation:  block mode
    UE2 scan alignment: 0
    Corpora read from file: /tmp/corpus

    Running single-pattern/single-compile test for 1 expressions.

    PCRE Match @ (2,45)
    PCRE Match @ (2,33)
    PCRE Match @ (12,45)
    PCRE Match @ (12,33)
    UE2 Match @ (0,33) for 1
    UE2 Match @ (0,45) for 1
    Scan call returned 0
    PASSED: id 1, alignment 0, corpus 0 (matched pcre:2, ue2:2)
    Thread 0 processed 1 units.

    Summary:
    Mode:                           Single/Block
    =========
    Expressions processed:          1
    Corpora processed:              1
    Expressions with failures:      0
      Corpora generation failures:  0
      Compilation failures:         pcre:0, ng:0, ue2:0
      Matching failures:            pcre:0, ng:0, ue2:0
      Match differences:            0
      No ground truth:              0
    Total match differences:        0

    Total elapsed time: 0.00522815 secs.
We can see from this output that both PCRE and Hyperscan find matches ending at offset 33 and 45, and so hscollider considers this test case to have passed.

(In the example command line above, -Z 0 instructs us to only test at corpus alignment 0, and -T 1 instructs us to only use one thread.)

    Note
    In default operation, PCRE produces only one match for a scan, unlike Hyperscan’s automata semantics. The hscollider tool uses libpcre’s “callout” functionality to match Hyperscan’s semantics.

### Running a larger scan test
A set of patterns for testing purposes are distributed with Hyperscan, and these can be tested via hscollider on an in-tree build. Two CMake targets are provided to do this easily:

|Make Target	|Description|
|:-:|:-:|
|make collide_quick_test	|Tests all patterns in streaming mode.|
|make collide_quick_test_block	|Tests all patterns in block mode.|

## Debugging: hsdump
When built in debug mode (using the CMake directive CMAKE_BUILD_TYPE set to Debug), Hyperscan includes support for dumping information about its internals during pattern compilation with the hsdump tool.

This information is mostly of use to Hyperscan developers familiar with the library’s internal structure, but can be used to diagnose issues with patterns and provide more information in bug reports.

## Pattern Format
All of the Hyperscan tools accept patterns in the same format, read from plain text files with one pattern per line. Each line looks like this:

- &lt;integer id&gt;:/&lt;regex&gt;/&lt;flags&gt;

For example:

    1:/hatstand.*teakettle/s
    2:/(hatstand|teakettle)/iH
    3:/^.{10,20}hatstand/m
The integer ID is the value that will be reported when a match is found by Hyperscan and must be unique.

The pattern itself is a regular expression in PCRE syntax; see Compiling Patterns for more information on supported features.

The flags are single characters that map to Hyperscan flags as follows:

|Character	|API Flag	|Description|
|:-:|:-:|:-:|
|i	|HS_FLAG_CASELESS	|Case-insensitive matching|
|s	|HS_FLAG_DOTALL	|Dot (.) will match newlines|
|m	|HS_FLAG_MULTILINE	|Multi-line anchoring|
|H	|HS_FLAG_SINGLEMATCH	|Report match ID at most once|
|V	|HS_FLAG_ALLOWEMPTY	|Allow patterns that can match against empty buffers|
|8	|HS_FLAG_UTF8	|UTF-8 mode|
|W	|HS_FLAG_UCP	|Unicode property support|
|P	|HS_FLAG_PREFILTER	|Prefiltering mode|
|L	|HS_FLAG_SOM_LEFTMOST	|Leftmost start of match reporting|
|C	|HS_FLAG_COMBINATION	|Logical combination of patterns|
|Q	|HS_FLAG_QUIET	|Quiet at matching|
In addition to the set of flags above, Extended Parameters can be supplied for each pattern. These are supplied after the flags as key=value pairs between braces, separated by commas. For example:

	1:/hatstand.*teakettle/s{min_offset=50,max_offset=100}
All Hyperscan tools will accept a pattern file (or a directory containing pattern files) with the -e argument. If no further arguments constraining the pattern set are given, all patterns in those files are used.

To select a subset of the patterns, a single ID can be supplied with the -z argument, or a file containing a set of IDs can be supplied with the -s argument.

---------------

# API Reference: Constants
## Error Codes
**HS_SUCCESS**
The engine completed normally.

**HS_INVALID**
A parameter passed to this function was invalid.

This error is only returned in cases where the function can detect an invalid parameter it cannot be relied upon to detect (for example) pointers to freed memory or other invalid data.

**HS_NOMEM**
A memory allocation failed.

**HS_SCAN_TERMINATED**
The engine was terminated by callback.

This return value indicates that the target buffer was partially scanned, but that the callback function requested that scanning cease after a match was located.

**HS_COMPILER_ERROR**
The pattern compiler failed, and the hs_compile_error_t should be inspected for more detail.

**HS_DB_VERSION_ERROR**
The given database was built for a different version of Hyperscan.

**HS_DB_PLATFORM_ERROR**
The given database was built for a different platform (i.e., CPU type).

**HS_DB_MODE_ERROR**
The given database was built for a different mode of operation. This error is returned when streaming calls are used with a block or vectored database and vice versa.

**HS_BAD_ALIGN**
A parameter passed to this function was not correctly aligned.

**HS_BAD_ALLOC**
The memory allocator (either malloc() or the allocator set with hs_set_allocator()) did not correctly return memory suitably aligned for the largest representable data type on this platform.

**HS_SCRATCH_IN_USE**
The scratch region was already in use.

This error is returned when Hyperscan is able to detect that the scratch region given is already in use by another Hyperscan API call.

A separate scratch region, allocated with hs_alloc_scratch() or hs_clone_scratch(), is required for every concurrent caller of the Hyperscan API.

For example, this error might be returned when hs_scan() has been called inside a callback delivered by a currently-executing hs_scan() call using the same scratch region.

Note: Not all concurrent uses of scratch regions may be detected. This error is intended as a best-effort debugging tool, not a guarantee.

**HS_ARCH_ERROR**
Unsupported CPU architecture.

This error is returned when Hyperscan is able to detect that the current system does not support the required instruction set.

At a minimum, Hyperscan requires Supplemental Streaming SIMD Extensions 3 (SSSE3).

**HS_INSUFFICIENT_SPACE**
Provided buffer was too small.

This error indicates that there was insufficient space in the buffer. The call should be repeated with a larger provided buffer.

Note: in this situation, it is normal for the amount of space required to be returned in the same manner as the used space would have been returned if the call was successful.

## hs_expr_ext flags
**HS_EXT_FLAG_MIN_OFFSET**
Flag indicating that the hs_expr_ext::min_offset field is used.

**HS_EXT_FLAG_MAX_OFFSET**
Flag indicating that the hs_expr_ext::max_offset field is used.

**HS_EXT_FLAG_MIN_LENGTH**
Flag indicating that the hs_expr_ext::min_length field is used.

**HS_EXT_FLAG_EDIT_DISTANCE**
Flag indicating that the hs_expr_ext::edit_distance field is used.

**HS_EXT_FLAG_HAMMING_DISTANCE**
Flag indicating that the hs_expr_ext::hamming_distance field is used.

## Pattern flags
**HS_FLAG_CASELESS**
Compile flag: Set case-insensitive matching.

This flag sets the expression to be matched case-insensitively by default. The expression may still use PCRE tokens (notably (?i) and (?-i)) to switch case-insensitive matching on and off.

**HS_FLAG_DOTALL**
Compile flag: Matching a . will not exclude newlines.

This flag sets any instances of the . token to match newline characters as well as all other characters. The PCRE specification states that the . token does not match newline characters by default, so without this flag the . token will not cross line boundaries.

**HS_FLAG_MULTILINE**
Compile flag: Set multi-line anchoring.

This flag instructs the expression to make the ^ and $ tokens match newline characters as well as the start and end of the stream. If this flag is not specified, the ^ token will only ever match at the start of a stream, and the $ token will only ever match at the end of a stream within the guidelines of the PCRE specification.

**HS_FLAG_SINGLEMATCH**
Compile flag: Set single-match only mode.

This flag sets the expression’s match ID to match at most once. In streaming mode, this means that the expression will return only a single match over the lifetime of the stream, rather than reporting every match as per standard Hyperscan semantics. In block mode or vectored mode, only the first match for each invocation of hs_scan() or hs_scan_vector() will be returned.

If multiple expressions in the database share the same match ID, then they either must all specify HS_FLAG_SINGLEMATCH or none of them specify HS_FLAG_SINGLEMATCH. If a group of expressions sharing a match ID specify the flag, then at most one match with the match ID will be generated per stream.

Note: The use of this flag in combination with HS_FLAG_SOM_LEFTMOST is not currently supported.

**HS_FLAG_ALLOWEMPTY**
Compile flag: Allow expressions that can match against empty buffers.

This flag instructs the compiler to allow expressions that can match against empty buffers, such as .?, .*, (a|). Since Hyperscan can return every possible match for an expression, such expressions generally execute very slowly; the default behaviour is to return an error when an attempt to compile one is made. Using this flag will force the compiler to allow such an expression.

**HS_FLAG_UTF8**
Compile flag: Enable UTF-8 mode for this expression.

This flag instructs Hyperscan to treat the pattern as a sequence of UTF-8 characters. The results of scanning invalid UTF-8 sequences with a Hyperscan library that has been compiled with one or more patterns using this flag are undefined.

**HS_FLAG_UCP**
Compile flag: Enable Unicode property support for this expression.

This flag instructs Hyperscan to use Unicode properties, rather than the default ASCII interpretations, for character mnemonics like \w and \s as well as the POSIX character classes. It is only meaningful in conjunction with HS_FLAG_UTF8.

**HS_FLAG_PREFILTER**
Compile flag: Enable prefiltering mode for this expression.

This flag instructs Hyperscan to compile an “approximate” version of this pattern for use in a prefiltering application, even if Hyperscan does not support the pattern in normal operation.

The set of matches returned when this flag is used is guaranteed to be a superset of the matches specified by the non-prefiltering expression.

If the pattern contains pattern constructs not supported by Hyperscan (such as zero-width assertions, back-references or conditional references) these constructs will be replaced internally with broader constructs that may match more often.

Furthermore, in prefiltering mode Hyperscan may simplify a pattern that would otherwise return a “Pattern too large” error at compile time, or for performance reasons (subject to the matching guarantee above).

It is generally expected that the application will subsequently confirm prefilter matches with another regular expression matcher that can provide exact matches for the pattern.

Note: The use of this flag in combination with HS_FLAG_SOM_LEFTMOST is not currently supported.

**HS_FLAG_SOM_LEFTMOST**
Compile flag: Enable leftmost start of match reporting.

This flag instructs Hyperscan to report the leftmost possible start of match offset when a match is reported for this expression. (By default, no start of match is returned.)

Enabling this behaviour may reduce performance and increase stream state requirements in streaming mode.

**HS_FLAG_COMBINATION**
Compile flag: Logical combination.

This flag instructs Hyperscan to parse this expression as logical combination syntax. Logical constraints consist of operands, operators and parentheses. The operands are expression indices, and operators can be ‘!’(NOT), ‘&’(AND) or ‘|’(OR). For example: (101&102&103)|(104&!105) ((301|302)&303)&(304|305)

**HS_FLAG_QUIET**
Compile flag: Don’t do any match reporting.

This flag instructs Hyperscan to ignore match reporting for this expression. It is designed to be used on the sub-expressions in logical combinations.

## CPU feature support flags
**HS_CPU_FEATURES_AVX2**
CPU features flag - Intel(R) Advanced Vector Extensions 2 (Intel(R) AVX2)

Setting this flag indicates that the target platform supports AVX2 instructions.

**HS_CPU_FEATURES_AVX512**
CPU features flag - Intel(R) Advanced Vector Extensions 512 (Intel(R) AVX512)

Setting this flag indicates that the target platform supports AVX512 instructions, specifically AVX-512BW. Using AVX512 implies the use of AVX2.

## CPU tuning flags
**HS_TUNE_FAMILY_GENERIC**
Tuning Parameter - Generic

This indicates that the compiled database should not be tuned for any particular target platform.

**HS_TUNE_FAMILY_SNB**
Tuning Parameter - Intel(R) microarchitecture code name Sandy Bridge

This indicates that the compiled database should be tuned for the Sandy Bridge microarchitecture.

**HS_TUNE_FAMILY_IVB**
Tuning Parameter - Intel(R) microarchitecture code name Ivy Bridge

This indicates that the compiled database should be tuned for the Ivy Bridge microarchitecture.

**HS_TUNE_FAMILY_HSW**
Tuning Parameter - Intel(R) microarchitecture code name Haswell

This indicates that the compiled database should be tuned for the Haswell microarchitecture.

**HS_TUNE_FAMILY_SLM**
Tuning Parameter - Intel(R) microarchitecture code name Silvermont

This indicates that the compiled database should be tuned for the Silvermont microarchitecture.

**HS_TUNE_FAMILY_BDW**
Tuning Parameter - Intel(R) microarchitecture code name Broadwell

This indicates that the compiled database should be tuned for the Broadwell microarchitecture.

**HS_TUNE_FAMILY_SKL**
Tuning Parameter - Intel(R) microarchitecture code name Skylake

This indicates that the compiled database should be tuned for the Skylake microarchitecture.

**HS_TUNE_FAMILY_SKX**
Tuning Parameter - Intel(R) microarchitecture code name Skylake Server

This indicates that the compiled database should be tuned for the Skylake Server microarchitecture.

**HS_TUNE_FAMILY_GLM**
Tuning Parameter - Intel(R) microarchitecture code name Goldmont

This indicates that the compiled database should be tuned for the Goldmont microarchitecture.

## Compile mode flags
**HS_MODE_BLOCK**
Compiler mode flag: Block scan (non-streaming) database.

**HS_MODE_NOSTREAM**
Compiler mode flag: Alias for HS_MODE_BLOCK.

**HS_MODE_STREAM**
Compiler mode flag: Streaming database.

**HS_MODE_VECTORED**
Compiler mode flag: Vectored scanning database.

**HS_MODE_SOM_HORIZON_LARGE**
Compiler mode flag: use full precision to track start of match offsets in stream state.

This mode will use the most stream state per pattern, but will always return an accurate start of match offset regardless of how far back in the past it was found.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

**HS_MODE_SOM_HORIZON_MEDIUM**
Compiler mode flag: use medium precision to track start of match offsets in stream state.

This mode will use less stream state than HS_MODE_SOM_HORIZON_LARGE and will limit start of match accuracy to offsets within 2^32 bytes of the end of match offset reported.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

**HS_MODE_SOM_HORIZON_SMALL**
Compiler mode flag: use limited precision to track start of match offsets in stream state.

This mode will use less stream state than HS_MODE_SOM_HORIZON_LARGE and will limit start of match accuracy to offsets within 2^16 bytes of the end of match offset reported.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

-----------------

# API Reference: Files
## File: hs.h
The complete Hyperscan API definition.

Hyperscan is a high speed regular expression engine.

This header includes both the Hyperscan compiler and runtime components. See the individual component headers for documentation.

## File: hs_common.h
The Hyperscan common API definition.

Hyperscan is a high speed regular expression engine.

This header contains functions available to both the Hyperscan compiler and runtime.

Defines

**HS_SUCCESS**
The engine completed normally.

**HS_INVALID**
A parameter passed to this function was invalid.

This error is only returned in cases where the function can detect an invalid parameter it cannot be relied upon to detect (for example) pointers to freed memory or other invalid data.

**HS_NOMEM**
A memory allocation failed.

**HS_SCAN_TERMINATED**
The engine was terminated by callback.

This return value indicates that the target buffer was partially scanned, but that the callback function requested that scanning cease after a match was located.

**HS_COMPILER_ERROR**
The pattern compiler failed, and the hs_compile_error_t should be inspected for more detail.

**HS_DB_VERSION_ERROR**
The given database was built for a different version of Hyperscan.

**HS_DB_PLATFORM_ERROR**
The given database was built for a different platform (i.e., CPU type).

**HS_DB_MODE_ERROR**
The given database was built for a different mode of operation. This error is returned when streaming calls are used with a block or vectored database and vice versa.

**HS_BAD_ALIGN**
A parameter passed to this function was not correctly aligned.

**HS_BAD_ALLOC**
The memory allocator (either malloc() or the allocator set with hs_set_allocator()) did not correctly return memory suitably aligned for the largest representable data type on this platform.

**HS_SCRATCH_IN_USE**
The scratch region was already in use.

This error is returned when Hyperscan is able to detect that the scratch region given is already in use by another Hyperscan API call.

A separate scratch region, allocated with hs_alloc_scratch() or hs_clone_scratch(), is required for every concurrent caller of the Hyperscan API.

For example, this error might be returned when hs_scan() has been called inside a callback delivered by a currently-executing hs_scan() call using the same scratch region.

Note: Not all concurrent uses of scratch regions may be detected. This error is intended as a best-effort debugging tool, not a guarantee.

**HS_ARCH_ERROR**
Unsupported CPU architecture.

This error is returned when Hyperscan is able to detect that the current system does not support the required instruction set.

At a minimum, Hyperscan requires Supplemental Streaming SIMD Extensions 3 (SSSE3).

**HS_INSUFFICIENT_SPACE**
Provided buffer was too small.

This error indicates that there was insufficient space in the buffer. The call should be repeated with a larger provided buffer.

Note: in this situation, it is normal for the amount of space required to be returned in the same manner as the used space would have been returned if the call was successful.

Typedefs

typedef **hs_database_t**

    A Hyperscan pattern database.

	Generated by one of the Hyperscan compiler functions:

    hs_compile()
    hs_compile_multi()
    hs_compile_ext_multi()

typedef **hs_error_t**

    A type for errors returned by Hyperscan functions.

typedef ( * **hs_alloc_t**)(size_t size)

	The type of the callback function that will be used by Hyperscan to allocate more memory at runtime as required, for example in hs_open_stream() to allocate stream state.

If Hyperscan is to be used in a multi-threaded, or similarly concurrent environment, the allocation function will need to be re-entrant, or similarly safe for concurrent use.

**Return**
A pointer to the region of memory allocated, or NULL on error.
**Parameters**
- size -
The number of bytes to allocate.

typedef ( * **hs_free_t**)(void *ptr)
The type of the callback function that will be used by Hyperscan to free memory regions previously allocated using the hs_alloc_t function.

**Parameters**
- ptr -
The region of memory to be freed.

**Functions**

hs_error_t **hs_free_database**(hs_database_t * db)

	Free a compiled pattern database.

T	he free callback set by hs_set_database_allocator() (or hs_set_allocator()) will be used by this function.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- db -
A compiled pattern database. NULL may also be safely provided, in which case the function does nothing.

hs_error_t **hs_serialize_database**(const hs_database_t * db, char ** bytes, size_t * length)

    Serialize a pattern database to a stream of bytes.

	The allocator callback set by hs_set_misc_allocator() (or hs_set_allocator()) will be used by this function.

**Return**

	HS_SUCCESS on success, HS_NOMEM if the byte array cannot be allocated, other values may be returned if errors are detected.
**Parameters**
- db -
	A compiled pattern database.

- bytes -

	On success, a pointer to an array of bytes will be returned here. These bytes can be subsequently relocated or written to disk. The caller is responsible for freeing this block.

- length -

	On success, the number of bytes in the generated byte array will be returned here.

hs_error_t **hs_deserialize_database**(const char * bytes, const size_t length, hs_database_t ** db)

	Reconstruct a pattern database from a stream of bytes previously generated by hs_serialize_database().

	This function will allocate sufficient space for the database using the allocator set with hs_set_database_allocator() (or hs_set_allocator()); to use a pre-allocated region of memory, use the hs_deserialize_database_at() function.

**Return**
	HS_SUCCESS on success, other values on failure.
**Parameters**
- bytes -
	A byte array generated by hs_serialize_database() representing a compiled pattern database.

- length -
	The length of the byte array generated by hs_serialize_database(). This should be the same value as that returned by hs_serialize_database().

- db -
	On success, a pointer to a newly allocated hs_database_t will be returned here. This database can then be used for scanning, and eventually freed by the caller using hs_free_database().

hs_error_t **hs_deserialize_database_at**(const char * bytes, const size_t length, hs_database_t * db)
	Reconstruct a pattern database from a stream of bytes previously generated by hs_serialize_database() at a given memory location.

	This function (unlike hs_deserialize_database()) will write the reconstructed database to the memory location given in the db parameter. The amount of space required at this location can be determined with the hs_serialized_database_size() function.

**Return**
	HS_SUCCESS on success, other values on failure.
**Parameters**
- bytes -
	A byte array generated by hs_serialize_database() representing a compiled pattern database.

- length -
	The length of the byte array generated by hs_serialize_database(). This should be the same value as that returned by hs_serialize_database().

- db -

	Pointer to an 8-byte aligned block of memory of sufficient size to hold the deserialized database. On success, the reconstructed database will be written to this location. This database can then be used for pattern matching. The user is responsible for freeing this memory; the hs_free_database() call should not be used.

hs_error_t **hs_stream_size**(const hs_database_t * database, size_t * stream_size)
	Provides the size of the stream state allocated by a single stream opened against the given database.

**Return**
	HS_SUCCESS on success, other values on failure.
**Parameters**
- database -

    Pointer to a compiled (streaming mode) pattern database.

- stream_size -

	On success, the size in bytes of an individual stream opened against the given database is placed in this parameter.

hs_error_t **hs_database_size**(const hs_database_t * database, size_t * database_size)
	Provides the size of the given database in bytes.

**Return**
	HS_SUCCESS on success, other values on failure.
**Parameters**
- database -

	Pointer to compiled pattern database.

- database_size -

	On success, the size of the compiled database in bytes is placed in this parameter.

hs_error_t **hs_serialized_database_size**(const char * bytes, const size_t length, size_t * deserialized_size)
Utility function for reporting the size that would be required by a database if it were deserialized.

This can be used to allocate a shared memory region or other “special” allocation prior to deserializing with the hs_deserialize_database_at() function.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- bytes -
Pointer to a byte array generated by hs_serialize_database() representing a compiled pattern database.

- length -
The length of the byte array generated by hs_serialize_database(). This should be the same value as that returned by hs_serialize_database().

- deserialized_size -
On success, the size of the compiled database that would be generated by hs_deserialize_database_at() is returned here.

hs_error_t **hs_database_info**(const hs_database_t * database, char ** info)
Utility function providing information about a database.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- database -
Pointer to a compiled database.

- info -
On success, a string containing the version and platform information for the supplied database is placed in the parameter. The string is allocated using the allocator supplied in hs_set_misc_allocator() (or malloc() if no allocator was set) and should be freed by the caller.

hs_error_t **hs_serialized_database_info**(const char * bytes, size_t length, char ** info)
Utility function providing information about a serialized database.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- bytes -
Pointer to a serialized database.

- length -
Length in bytes of the serialized database.

- info -
On success, a string containing the version and platform information for the supplied serialized database is placed in the parameter. The string is allocated using the allocator supplied in hs_set_misc_allocator() (or malloc() if no allocator was set) and should be freed by the caller.

hs_error_t **hs_set_allocator**(hs_alloc_t alloc_func, hs_free_t free_func)
Set the allocate and free functions used by Hyperscan for allocating memory at runtime for stream state, scratch space, database bytecode, and various other data structure returned by the Hyperscan API.

The function is equivalent to calling hs_set_stream_allocator(), hs_set_scratch_allocator(), hs_set_database_allocator() and hs_set_misc_allocator() with the provided parameters.

This call will override any previous allocators that have been set.

Note: there is no way to change the allocator used for temporary objects created during the various compile calls (hs_compile(), hs_compile_multi(), hs_compile_ext_multi()).

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- alloc_func -
A callback function pointer that allocates memory. This function must return memory suitably aligned for the largest representable data type on this platform.

- free_func -
A callback function pointer that frees allocated memory.

hs_error_t **hs_set_database_allocator**(hs_alloc_t alloc_func, hs_free_t free_func)
Set the allocate and free functions used by Hyperscan for allocating memory for database bytecode produced by the compile calls (hs_compile(), hs_compile_multi(), hs_compile_ext_multi()) and by database deserialization (hs_deserialize_database()).

If no database allocation functions are set, or if NULL is used in place of both parameters, then memory allocation will default to standard methods (such as the system malloc() and free() calls).

This call will override any previous database allocators that have been set.

Note: the database allocator may also be set by calling hs_set_allocator().

Note: there is no way to change how temporary objects created during the various compile calls (hs_compile(), hs_compile_multi(), hs_compile_ext_multi()) are allocated.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- alloc_func -
A callback function pointer that allocates memory. This function must return memory suitably aligned for the largest representable data type on this platform.

- free_func -
A callback function pointer that frees allocated memory.

hs_error_t **hs_set_misc_allocator**(hs_alloc_t alloc_func, hs_free_t free_func)
Set the allocate and free functions used by Hyperscan for allocating memory for items returned by the Hyperscan API such as hs_compile_error_t, hs_expr_info_t and serialized databases.

If no misc allocation functions are set, or if NULL is used in place of both parameters, then memory allocation will default to standard methods (such as the system malloc() and free() calls).

This call will override any previous misc allocators that have been set.

Note: the misc allocator may also be set by calling hs_set_allocator().

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- alloc_func -
A callback function pointer that allocates memory. This function must return memory suitably aligned for the largest representable data type on this platform.

- free_func -
A callback function pointer that frees allocated memory.

hs_error_t **hs_set_scratch_allocator**(hs_alloc_t alloc_func, hs_free_t free_func)
Set the allocate and free functions used by Hyperscan for allocating memory for scratch space by hs_alloc_scratch() and hs_clone_scratch().

If no scratch allocation functions are set, or if NULL is used in place of both parameters, then memory allocation will default to standard methods (such as the system malloc() and free() calls).

This call will override any previous scratch allocators that have been set.

Note: the scratch allocator may also be set by calling hs_set_allocator().

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- alloc_func -
A callback function pointer that allocates memory. This function must return memory suitably aligned for the largest representable data type on this platform.

- free_func -
A callback function pointer that frees allocated memory.

hs_error_t **hs_set_stream_allocator**(hs_alloc_t alloc_func, hs_free_t free_func)
Set the allocate and free functions used by Hyperscan for allocating memory for stream state by hs_open_stream().

If no stream allocation functions are set, or if NULL is used in place of both parameters, then memory allocation will default to standard methods (such as the system malloc() and free() calls).

This call will override any previous stream allocators that have been set.

Note: the stream allocator may also be set by calling hs_set_allocator().

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- alloc_func -
A callback function pointer that allocates memory. This function must return memory suitably aligned for the largest representable data type on this platform.

- free_func -
A callback function pointer that frees allocated memory.

const char* **hs_version**(void)
Utility function for identifying this release version.

**Return**
A string containing the version number of this release build and the date of the build. It is allocated statically, so it does not need to be freed by the caller.
hs_error_t **hs_valid_platform**(void)
Utility function to test the current system architecture.

Hyperscan requires the Supplemental Streaming SIMD Extensions 3 instruction set. This function can be called on any x86 platform to determine if the system provides the required instruction set.

This function does not test for more advanced features if Hyperscan has been built for a more specific architecture, for example the AVX2 instruction set.

**Return**
HS_SUCCESS on success, HS_ARCH_ERROR if system does not support Hyperscan.

## File: hs_compile.h
The Hyperscan compiler API definition.

Hyperscan is a high speed regular expression engine.

This header contains functions for compiling regular expressions into Hyperscan databases that can be used by the Hyperscan runtime.

**Defines**

**HS_EXT_FLAG_MIN_OFFSET**
Flag indicating that the hs_expr_ext::min_offset field is used.

**HS_EXT_FLAG_MAX_OFFSET**
Flag indicating that the hs_expr_ext::max_offset field is used.

**HS_EXT_FLAG_MIN_LENGTH**
Flag indicating that the hs_expr_ext::min_length field is used.

**HS_EXT_FLAG_EDIT_DISTANCE**
Flag indicating that the hs_expr_ext::edit_distance field is used.

**HS_EXT_FLAG_HAMMING_DISTANCE**
Flag indicating that the hs_expr_ext::hamming_distance field is used.

**HS_FLAG_CASELESS**
Compile flag: Set case-insensitive matching.

This flag sets the expression to be matched case-insensitively by default. The expression may still use PCRE tokens (notably (?i) and (?-i)) to switch case-insensitive matching on and off.

**HS_FLAG_DOTALL**
Compile flag: Matching a . will not exclude newlines.

This flag sets any instances of the . token to match newline characters as well as all other characters. The PCRE specification states that the . token does not match newline characters by default, so without this flag the . token will not cross line boundaries.

**HS_FLAG_MULTILINE**
Compile flag: Set multi-line anchoring.

This flag instructs the expression to make the ^ and $ tokens match newline characters as well as the start and end of the stream. If this flag is not specified, the ^ token will only ever match at the start of a stream, and the $ token will only ever match at the end of a stream within the guidelines of the PCRE specification.

**HS_FLAG_SINGLEMATCH**
Compile flag: Set single-match only mode.

This flag sets the expression’s match ID to match at most once. In streaming mode, this means that the expression will return only a single match over the lifetime of the stream, rather than reporting every match as per standard Hyperscan semantics. In block mode or vectored mode, only the first match for each invocation of hs_scan() or hs_scan_vector() will be returned.

If multiple expressions in the database share the same match ID, then they either must all specify HS_FLAG_SINGLEMATCH or none of them specify HS_FLAG_SINGLEMATCH. If a group of expressions sharing a match ID specify the flag, then at most one match with the match ID will be generated per stream.

Note: The use of this flag in combination with HS_FLAG_SOM_LEFTMOST is not currently supported.

**HS_FLAG_ALLOWEMPTY**
Compile flag: Allow expressions that can match against empty buffers.

This flag instructs the compiler to allow expressions that can match against empty buffers, such as .?, .*, (a|). Since Hyperscan can return every possible match for an expression, such expressions generally execute very slowly; the default behaviour is to return an error when an attempt to compile one is made. Using this flag will force the compiler to allow such an expression.

**HS_FLAG_UTF8**
Compile flag: Enable UTF-8 mode for this expression.

This flag instructs Hyperscan to treat the pattern as a sequence of UTF-8 characters. The results of scanning invalid UTF-8 sequences with a Hyperscan library that has been compiled with one or more patterns using this flag are undefined.

**HS_FLAG_UCP**
Compile flag: Enable Unicode property support for this expression.

This flag instructs Hyperscan to use Unicode properties, rather than the default ASCII interpretations, for character mnemonics like \w and \s as well as the POSIX character classes. It is only meaningful in conjunction with HS_FLAG_UTF8.

**HS_FLAG_PREFILTER**
Compile flag: Enable prefiltering mode for this expression.

This flag instructs Hyperscan to compile an “approximate” version of this pattern for use in a prefiltering application, even if Hyperscan does not support the pattern in normal operation.

The set of matches returned when this flag is used is guaranteed to be a superset of the matches specified by the non-prefiltering expression.

If the pattern contains pattern constructs not supported by Hyperscan (such as zero-width assertions, back-references or conditional references) these constructs will be replaced internally with broader constructs that may match more often.

Furthermore, in prefiltering mode Hyperscan may simplify a pattern that would otherwise return a “Pattern too large” error at compile time, or for performance reasons (subject to the matching guarantee above).

It is generally expected that the application will subsequently confirm prefilter matches with another regular expression matcher that can provide exact matches for the pattern.

Note: The use of this flag in combination with HS_FLAG_SOM_LEFTMOST is not currently supported.

**HS_FLAG_SOM_LEFTMOST**
Compile flag: Enable leftmost start of match reporting.

This flag instructs Hyperscan to report the leftmost possible start of match offset when a match is reported for this expression. (By default, no start of match is returned.)

Enabling this behaviour may reduce performance and increase stream state requirements in streaming mode.

**HS_FLAG_COMBINATION**
Compile flag: Logical combination.

This flag instructs Hyperscan to parse this expression as logical combination syntax. Logical constraints consist of operands, operators and parentheses. The operands are expression indices, and operators can be ‘!’(NOT), ‘&’(AND) or ‘|’(OR). For example: (101&102&103)|(104&!105) ((301|302)&303)&(304|305)

**HS_FLAG_QUIET**
Compile flag: Don’t do any match reporting.

This flag instructs Hyperscan to ignore match reporting for this expression. It is designed to be used on the sub-expressions in logical combinations.

**HS_CPU_FEATURES_AVX2**
CPU features flag - Intel(R) Advanced Vector Extensions 2 (Intel(R) AVX2)

Setting this flag indicates that the target platform supports AVX2 instructions.

**HS_CPU_FEATURES_AVX512**
CPU features flag - Intel(R) Advanced Vector Extensions 512 (Intel(R) AVX512)

Setting this flag indicates that the target platform supports AVX512 instructions, specifically AVX-512BW. Using AVX512 implies the use of AVX2.

**HS_TUNE_FAMILY_GENERIC**
Tuning Parameter - Generic

This indicates that the compiled database should not be tuned for any particular target platform.

**HS_TUNE_FAMILY_SNB**
Tuning Parameter - Intel(R) microarchitecture code name Sandy Bridge

This indicates that the compiled database should be tuned for the Sandy Bridge microarchitecture.

**HS_TUNE_FAMILY_IVB**
Tuning Parameter - Intel(R) microarchitecture code name Ivy Bridge

This indicates that the compiled database should be tuned for the Ivy Bridge microarchitecture.

**HS_TUNE_FAMILY_HSW**
Tuning Parameter - Intel(R) microarchitecture code name Haswell

This indicates that the compiled database should be tuned for the Haswell microarchitecture.

**HS_TUNE_FAMILY_SLM**
Tuning Parameter - Intel(R) microarchitecture code name Silvermont

This indicates that the compiled database should be tuned for the Silvermont microarchitecture.

**HS_TUNE_FAMILY_BDW**
Tuning Parameter - Intel(R) microarchitecture code name Broadwell

This indicates that the compiled database should be tuned for the Broadwell microarchitecture.

**HS_TUNE_FAMILY_SKL**
Tuning Parameter - Intel(R) microarchitecture code name Skylake

This indicates that the compiled database should be tuned for the Skylake microarchitecture.

**HS_TUNE_FAMILY_SKX**
Tuning Parameter - Intel(R) microarchitecture code name Skylake Server

This indicates that the compiled database should be tuned for the Skylake Server microarchitecture.

**HS_TUNE_FAMILY_GLM**
Tuning Parameter - Intel(R) microarchitecture code name Goldmont

This indicates that the compiled database should be tuned for the Goldmont microarchitecture.

**HS_MODE_BLOCK**
Compiler mode flag: Block scan (non-streaming) database.

**HS_MODE_NOSTREAM**
Compiler mode flag: Alias for HS_MODE_BLOCK.

**HS_MODE_STREAM**
Compiler mode flag: Streaming database.

**HS_MODE_VECTORED**
Compiler mode flag: Vectored scanning database.

**HS_MODE_SOM_HORIZON_LARGE**
Compiler mode flag: use full precision to track start of match offsets in stream state.

This mode will use the most stream state per pattern, but will always return an accurate start of match offset regardless of how far back in the past it was found.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

**HS_MODE_SOM_HORIZON_MEDIUM**
Compiler mode flag: use medium precision to track start of match offsets in stream state.

This mode will use less stream state than HS_MODE_SOM_HORIZON_LARGE and will limit start of match accuracy to offsets within 2^32 bytes of the end of match offset reported.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

**HS_MODE_SOM_HORIZON_SMALL**
Compiler mode flag: use limited precision to track start of match offsets in stream state.

This mode will use less stream state than HS_MODE_SOM_HORIZON_LARGE and will limit start of match accuracy to offsets within 2^16 bytes of the end of match offset reported.

One of the SOM_HORIZON modes must be selected to use the HS_FLAG_SOM_LEFTMOST expression flag.

**Typedefs**

typedef **hs_compile_error_t**
A type containing error details that is returned by the compile calls (hs_compile(), hs_compile_multi() and hs_compile_ext_multi()) on failure. The caller may inspect the values returned in this type to determine the cause of failure.

Common errors generated during the compile process include:

- Invalid parameter

An invalid argument was specified in the compile call.

- Unrecognised flag

An unrecognised value was passed in the flags argument.

- Pattern matches empty buffer

By default, Hyperscan only supports patterns that will always consume at least one byte of input. Patterns that do not have this property (such as /(abc)?/) will produce this error unless the HS_FLAG_ALLOWEMPTY flag is supplied. Note that such patterns will produce a match for every byte when scanned.

- Embedded anchors not supported

Hyperscan only supports the use of anchor meta-characters (such as ^ and $) in patterns where they could only match at the start or end of a buffer. A pattern containing an embedded anchor, such as /abc^def/, can never match, as there is no way for abc to precede the start of the data stream.

- Bounded repeat is too large

The pattern contains a repeated construct with very large finite bounds.

- Unsupported component type

An unsupported PCRE construct was used in the pattern.

- Unable to generate bytecode

This error indicates that Hyperscan was unable to compile a pattern that is syntactically valid. The most common cause is a pattern that is very long and complex or contains a large repeated subpattern.

- Unable to allocate memory

The library was unable to allocate temporary storage used during compilation time.

- Allocator returned misaligned memory

The memory allocator (either malloc() or the allocator set with hs_set_allocator()) did not correctly return memory suitably aligned for the largest representable data type on this platform.

- Internal error

An unexpected error occurred: if this error is reported, please contact the Hyperscan team with a description of the situation.

typedef **hs_platform_info_t**
A type containing information on the target platform which may optionally be provided to the compile calls (hs_compile(), hs_compile_multi(), hs_compile_ext_multi()).

A hs_platform_info structure may be populated for the current platform by using the hs_populate_platform() call.

typedef **hs_expr_info_t**
A type containing information related to an expression that is returned by hs_expression_info() or hs_expression_ext_info.

typedef **hs_expr_ext_t**
A structure containing additional parameters related to an expression, passed in at build time to hs_compile_ext_multi() or hs_expression_ext_info.

These parameters allow the set of matches produced by a pattern to be constrained at compile time, rather than relying on the application to process unwanted matches at runtime.

**Functions**

hs_error_t **hs_compile**(const char * expression, unsigned int flags, unsigned int mode, const hs_platform_info_t * platform, hs_database_t ** db, hs_compile_error_t ** error)
The basic regular expression compiler.

This is the function call with which an expression is compiled into a Hyperscan database which can be passed to the runtime functions (such as hs_scan(), hs_open_stream(), etc.)

**Return**
HS_SUCCESS is returned on successful compilation; HS_COMPILER_ERROR on failure, with details provided in the error parameter.
**Parameters**
- expression -
The NULL-terminated expression to parse. Note that this string must represent ONLY the pattern to be matched, with no delimiters or flags; any global flags should be specified with the flags argument. For example, the expression /abc?def/i should be compiled by providing abc?def as the expression, and HS_FLAG_CASELESS as the flags.

- flags -
Flags which modify the behaviour of the expression. Multiple flags may be used by ORing them together. Valid values are:

    HS_FLAG_CASELESS - Matching will be performed case-insensitively.
    HS_FLAG_DOTALL - Matching a . will not exclude newlines.
    HS_FLAG_MULTILINE - ^ and $ anchors match any newlines in data.
    HS_FLAG_SINGLEMATCH - Only one match will be generated for the expression per stream.
    HS_FLAG_ALLOWEMPTY - Allow expressions which can match against an empty string, such as .*.
    HS_FLAG_UTF8 - Treat this pattern as a sequence of UTF-8 characters.
    HS_FLAG_UCP - Use Unicode properties for character classes.
    HS_FLAG_PREFILTER - Compile pattern in prefiltering mode.
    HS_FLAG_SOM_LEFTMOST - Report the leftmost start of match offset when a match is found.
- mode -
Compiler mode flags that affect the database as a whole. One of HS_MODE_STREAM or HS_MODE_BLOCK or HS_MODE_VECTORED must be supplied, to select between the generation of a streaming, block or vectored database. In addition, other flags (beginning with HS_MODE_) may be supplied to enable specific features. See Compile mode flags for more details.

- platform -
If not NULL, the platform structure is used to determine the target platform for the database. If NULL, a database suitable for running on the current host platform is produced.

- db -
On success, a pointer to the generated database will be returned in this parameter, or NULL on failure. The caller is responsible for deallocating the buffer using the hs_free_database() function.

- error -
If the compile fails, a pointer to a hs_compile_error_t will be returned, providing details of the error condition. The caller is responsible for deallocating the buffer using the hs_free_compile_error() function.

hs_error_t **hs_compile_multi**(const char *const * expressions, const unsigned int * flags, const unsigned int * ids, unsigned int elements, unsigned int mode, const hs_platform_info_t * platform, hs_database_t ** db, hs_compile_error_t ** error)
The multiple regular expression compiler.

This is the function call with which a set of expressions is compiled into a database which can be passed to the runtime functions (such as hs_scan(), hs_open_stream(), etc.) Each expression can be labelled with a unique integer which is passed into the match callback to identify the pattern that has matched.

**Return**
HS_SUCCESS is returned on successful compilation; HS_COMPILER_ERROR on failure, with details provided in the error parameter.
**Parameters**
- expressions -
Array of NULL-terminated expressions to compile. Note that (as for hs_compile()) these strings must contain only the pattern to be matched, with no delimiters or flags. For example, the expression /abc?def/i should be compiled by providing abc?def as the first string in the expressions array, and HS_FLAG_CASELESS as the first value in the flags array.

- flags -
Array of flags which modify the behaviour of each expression. Multiple flags may be used by ORing them together. Specifying the NULL pointer in place of an array will set the flags value for all patterns to zero. Valid values are:

    HS_FLAG_CASELESS - Matching will be performed case-insensitively.
    HS_FLAG_DOTALL - Matching a . will not exclude newlines.
    HS_FLAG_MULTILINE - ^ and $ anchors match any newlines in data.
    HS_FLAG_SINGLEMATCH - Only one match will be generated by patterns with this match id per stream.
    HS_FLAG_ALLOWEMPTY - Allow expressions which can match against an empty string, such as .*.
    HS_FLAG_UTF8 - Treat this pattern as a sequence of UTF-8 characters.
    HS_FLAG_UCP - Use Unicode properties for character classes.
    HS_FLAG_PREFILTER - Compile pattern in prefiltering mode.
    HS_FLAG_SOM_LEFTMOST - Report the leftmost start of match offset when a match is found.
- ids -
An array of integers specifying the ID number to be associated with the corresponding pattern in the expressions array. Specifying the NULL pointer in place of an array will set the ID value for all patterns to zero.

- elements -
The number of elements in the input arrays.

- mode -
Compiler mode flags that affect the database as a whole. One of HS_MODE_STREAM or HS_MODE_BLOCK or HS_MODE_VECTORED must be supplied, to select between the generation of a streaming, block or vectored database. In addition, other flags (beginning with HS_MODE_) may be supplied to enable specific features. See Compile mode flags for more details.

- platform -
If not NULL, the platform structure is used to determine the target platform for the database. If NULL, a database suitable for running on the current host platform is produced.

- db -
On success, a pointer to the generated database will be returned in this parameter, or NULL on failure. The caller is responsible for deallocating the buffer using the hs_free_database() function.

- error -
If the compile fails, a pointer to a hs_compile_error_t will be returned, providing details of the error condition. The caller is responsible for deallocating the buffer using the hs_free_compile_error() function.

hs_error_t **hs_compile_ext_multi**(const char *const * expressions, const unsigned int * flags, const unsigned int * ids, const hs_expr_ext_t *const * ext, unsigned int elements, unsigned int mode, const hs_platform_info_t * platform, hs_database_t ** db, hs_compile_error_t ** error)
The multiple regular expression compiler with extended parameter support.

This function call compiles a group of expressions into a database in the same way as hs_compile_multi(), but allows additional parameters to be specified via an hs_expr_ext_t structure per expression.

**Return**
HS_SUCCESS is returned on successful compilation; HS_COMPILER_ERROR on failure, with details provided in the error parameter.
**Parameters**
- expressions -
Array of NULL-terminated expressions to compile. Note that (as for hs_compile()) these strings must contain only the pattern to be matched, with no delimiters or flags. For example, the expression /abc?def/i should be compiled by providing abc?def as the first string in the expressions array, and HS_FLAG_CASELESS as the first value in the flags array.

- flags -
Array of flags which modify the behaviour of each expression. Multiple flags may be used by ORing them together. Specifying the NULL pointer in place of an array will set the flags value for all patterns to zero. Valid values are:

    HS_FLAG_CASELESS - Matching will be performed case-insensitively.
    HS_FLAG_DOTALL - Matching a . will not exclude newlines.
    HS_FLAG_MULTILINE - ^ and $ anchors match any newlines in data.
    HS_FLAG_SINGLEMATCH - Only one match will be generated by patterns with this match id per stream.
    HS_FLAG_ALLOWEMPTY - Allow expressions which can match against an empty string, such as .*.
    HS_FLAG_UTF8 - Treat this pattern as a sequence of UTF-8 characters.
    HS_FLAG_UCP - Use Unicode properties for character classes.
    HS_FLAG_PREFILTER - Compile pattern in prefiltering mode.
    HS_FLAG_SOM_LEFTMOST - Report the leftmost start of match offset when a match is found.
- ids -
An array of integers specifying the ID number to be associated with the corresponding pattern in the expressions array. Specifying the NULL pointer in place of an array will set the ID value for all patterns to zero.

- ext -
An array of pointers to filled hs_expr_ext_t structures that define extended behaviour for each pattern. NULL may be specified if no extended behaviour is needed for an individual pattern, or in place of the whole array if it is not needed for any expressions. Memory used by these structures must be both allocated and freed by the caller.

- elements -
The number of elements in the input arrays.

- mode -
Compiler mode flags that affect the database as a whole. One of HS_MODE_STREAM, HS_MODE_BLOCK or HS_MODE_VECTORED must be supplied, to select between the generation of a streaming, block or vectored database. In addition, other flags (beginning with HS_MODE_) may be supplied to enable specific features. See Compile mode flags for more details.

- platform -
If not NULL, the platform structure is used to determine the target platform for the database. If NULL, a database suitable for running on the current host platform is produced.

- db -
On success, a pointer to the generated database will be returned in this parameter, or NULL on failure. The caller is responsible for deallocating the buffer using the hs_free_database() function.

- error -
If the compile fails, a pointer to a hs_compile_error_t will be returned, providing details of the error condition. The caller is responsible for deallocating the buffer using the hs_free_compile_error() function.

hs_error_t **hs_free_compile_error**(hs_compile_error_t * error)
Free an error structure generated by hs_compile(), hs_compile_multi() or hs_compile_ext_multi().

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- error -
The hs_compile_error_t to be freed. NULL may also be safely provided.

hs_error_t **hs_expression_info**(const char * expression, unsigned int flags, hs_expr_info_t ** info, hs_compile_error_t ** error)
Utility function providing information about a regular expression. The information provided in hs_expr_info_t includes the minimum and maximum width of a pattern match.

Note: successful analysis of an expression with this function does not imply that compilation of the same expression (via hs_compile(), hs_compile_multi() or hs_compile_ext_multi()) would succeed. This function may return HS_SUCCESS for regular expressions that Hyperscan cannot compile.

Note: some per-pattern flags (such as HS_FLAG_ALLOWEMPTY, HS_FLAG_SOM_LEFTMOST) are accepted by this call, but as they do not affect the properties returned in the hs_expr_info_t structure, they will not affect the outcome of this function.

**Return**
HS_SUCCESS is returned on successful compilation; HS_COMPILER_ERROR on failure, with details provided in the error parameter.
**Parameters**
- expression -
The NULL-terminated expression to parse. Note that this string must represent ONLY the pattern to be matched, with no delimiters or flags; any global flags should be specified with the flags argument. For example, the expression /abc?def/i should be compiled by providing abc?def as the expression, and HS_FLAG_CASELESS as the flags.

- flags -
Flags which modify the behaviour of the expression. Multiple flags may be used by ORing them together. Valid values are:

    HS_FLAG_CASELESS - Matching will be performed case-insensitively.
    HS_FLAG_DOTALL - Matching a . will not exclude newlines.
    HS_FLAG_MULTILINE - ^ and $ anchors match any newlines in data.
    HS_FLAG_SINGLEMATCH - Only one match will be generated by the expression per stream.
    HS_FLAG_ALLOWEMPTY - Allow expressions which can match against an empty string, such as .*.
    HS_FLAG_UTF8 - Treat this pattern as a sequence of UTF-8 characters.
    HS_FLAG_UCP - Use Unicode properties for character classes.
    HS_FLAG_PREFILTER - Compile pattern in prefiltering mode.
    HS_FLAG_SOM_LEFTMOST - Report the leftmost start of match offset when a match is found.
- info -
On success, a pointer to the pattern information will be returned in this parameter, or NULL on failure. This structure is allocated using the allocator supplied in hs_set_allocator() (or malloc() if no allocator was set) and should be freed by the caller.

- error -
If the call fails, a pointer to a hs_compile_error_t will be returned, providing details of the error condition. The caller is responsible for deallocating the buffer using the hs_free_compile_error() function.

hs_error_t **hs_expression_ext_info**(const char * expression, unsigned int flags, const hs_expr_ext_t * ext, hs_expr_info_t ** info, hs_compile_error_t ** error)
Utility function providing information about a regular expression, with extended parameter support. The information provided in hs_expr_info_t includes the minimum and maximum width of a pattern match.

Note: successful analysis of an expression with this function does not imply that compilation of the same expression (via hs_compile(), hs_compile_multi() or hs_compile_ext_multi()) would succeed. This function may return HS_SUCCESS for regular expressions that Hyperscan cannot compile.

Note: some per-pattern flags (such as HS_FLAG_ALLOWEMPTY, HS_FLAG_SOM_LEFTMOST) are accepted by this call, but as they do not affect the properties returned in the hs_expr_info_t structure, they will not affect the outcome of this function.

**Return**
HS_SUCCESS is returned on successful compilation; HS_COMPILER_ERROR on failure, with details provided in the error parameter.
**Parameters**
expression -
The NULL-terminated expression to parse. Note that this string must represent ONLY the pattern to be matched, with no delimiters or flags; any global flags should be specified with the flags argument. For example, the expression /abc?def/i should be compiled by providing abc?def as the expression, and HS_FLAG_CASELESS as the flags.

- flags -
Flags which modify the behaviour of the expression. Multiple flags may be used by ORing them together. Valid values are:

    HS_FLAG_CASELESS - Matching will be performed case-insensitively.
    HS_FLAG_DOTALL - Matching a . will not exclude newlines.
    HS_FLAG_MULTILINE - ^ and $ anchors match any newlines in data.
    HS_FLAG_SINGLEMATCH - Only one match will be generated by the expression per stream.
    HS_FLAG_ALLOWEMPTY - Allow expressions which can match against an empty string, such as .*.
    HS_FLAG_UTF8 - Treat this pattern as a sequence of UTF-8 characters.
    HS_FLAG_UCP - Use Unicode properties for character classes.
    HS_FLAG_PREFILTER - Compile pattern in prefiltering mode.
    HS_FLAG_SOM_LEFTMOST - Report the leftmost start of match offset when a match is found.
- ext -
A pointer to a filled hs_expr_ext_t structure that defines extended behaviour for this pattern. NULL may be specified if no extended parameters are needed.

- info -
On success, a pointer to the pattern information will be returned in this parameter, or NULL on failure. This structure is allocated using the allocator supplied in hs_set_allocator() (or malloc() if no allocator was set) and should be freed by the caller.

- error -
If the call fails, a pointer to a hs_compile_error_t will be returned, providing details of the error condition. The caller is responsible for deallocating the buffer using the hs_free_compile_error() function.

hs_error_t **hs_populate_platform**(hs_platform_info_t * platform)
Populates the platform information based on the current host.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- platform -
On success, the pointed to structure is populated based on the current host.

struct **hs_compile_error**
\#include &lt;hs_compile.h&gt;
A type containing error details that is returned by the compile calls (hs_compile(), hs_compile_multi() and hs_compile_ext_multi()) on failure. The caller may inspect the values returned in this type to determine the cause of failure.

Common errors generated during the compile process include:

- Invalid parameter

An invalid argument was specified in the compile call.

- Unrecognised flag

An unrecognised value was passed in the flags argument.

- Pattern matches empty buffer

By default, Hyperscan only supports patterns that will always consume at least one byte of input. Patterns that do not have this property (such as /(abc)?/) will produce this error unless the HS_FLAG_ALLOWEMPTY flag is supplied. Note that such patterns will produce a match for every byte when scanned.

- Embedded anchors not supported

Hyperscan only supports the use of anchor meta-characters (such as ^ and $) in patterns where they could only match at the start or end of a buffer. A pattern containing an embedded anchor, such as /abc^def/, can never match, as there is no way for abc to precede the start of the data stream.

- Bounded repeat is too large

The pattern contains a repeated construct with very large finite bounds.

- Unsupported component type

An unsupported PCRE construct was used in the pattern.

- Unable to generate bytecode

This error indicates that Hyperscan was unable to compile a pattern that is syntactically valid. The most common cause is a pattern that is very long and complex or contains a large repeated subpattern.

- Unable to allocate memory

The library was unable to allocate temporary storage used during compilation time.

- Allocator returned misaligned memory

The memory allocator (either malloc() or the allocator set with hs_set_allocator()) did not correctly return memory suitably aligned for the largest representable data type on this platform.

- Internal error

An unexpected error occurred: if this error is reported, please contact the Hyperscan team with a description of the situation.

**Public Members**

char\* **message**
A human-readable error message describing the error.

int **expression**
The zero-based number of the expression that caused the error (if this can be determined). If the error is not specific to an expression, then this value will be less than zero.

struct **hs_platform_info**
\#include &lt;hs_compile.h&gt;
A type containing information on the target platform which may optionally be provided to the compile calls (hs_compile(), hs_compile_multi(), hs_compile_ext_multi()).

A hs_platform_info structure may be populated for the current platform by using the hs_populate_platform() call.

**Public Members**

unsigned int **tune**
Information about the target platform which may be used to guide the optimisation process of the compile.

Use of this field does not limit the processors that the resulting database can run on, but may impact the performance of the resulting database.

unsigned long long **cpu_features**
Relevant CPU features available on the target platform

This value may be produced by combining HS_CPU_FEATURE_* flags (such as HS_CPU_FEATURES_AVX2). Multiple CPU features may be or’ed together to produce the value.

unsigned long long **reserved1**
Reserved for future use.

unsigned long long **reserved2**
Reserved for future use.

struct **hs_expr_info**
\#include &lt;hs_compile.h&gt;
A type containing information related to an expression that is returned by hs_expression_info() or hs_expression_ext_info.

**Public Members**

unsigned int **min_width**
The minimum length in bytes of a match for the pattern.

Note: in some cases when using advanced features to suppress matches (such as extended parameters or the HS_FLAG_SINGLEMATCH flag) this may represent a conservative lower bound for the true minimum length of a match.

unsigned int **max_width**
The maximum length in bytes of a match for the pattern. If the pattern has an unbounded maximum length, this will be set to the maximum value of an unsigned int (UINT_MAX).

Note: in some cases when using advanced features to suppress matches (such as extended parameters or the HS_FLAG_SINGLEMATCH flag) this may represent a conservative upper bound for the true maximum length of a match.

char **unordered_matches**
Whether this expression can produce matches that are not returned in order, such as those produced by assertions. Zero if false, non-zero if true.

char **matches_at_eod**
Whether this expression can produce matches at end of data (EOD). In streaming mode, EOD matches are raised during hs_close_stream(), since it is only when hs_close_stream() is called that the EOD location is known. Zero if false, non-zero if true.

Note: trailing \b word boundary assertions may also result in EOD matches as end-of-data can act as a word boundary.

char **matches_only_at_eod**
Whether this expression can only produce matches at end of data (EOD). In streaming mode, all matches for this expression are raised during hs_close_stream(). Zero if false, non-zero if true.

struct **hs_expr_ext**
\#include &lt;hs_compile.h&gt;
A structure containing additional parameters related to an expression, passed in at build time to hs_compile_ext_multi() or hs_expression_ext_info.

These parameters allow the set of matches produced by a pattern to be constrained at compile time, rather than relying on the application to process unwanted matches at runtime.

**Public Members**

unsigned long long **flags**
Flags governing which parts of this structure are to be used by the compiler. See hs_expr_ext_t flags.

unsigned long long **min_offset**
The minimum end offset in the data stream at which this expression should match successfully. To use this parameter, set the HS_EXT_FLAG_MIN_OFFSET flag in the hs_expr_ext::flags field.

unsigned long long **max_offset**
The maximum end offset in the data stream at which this expression should match successfully. To use this parameter, set the HS_EXT_FLAG_MAX_OFFSET flag in the hs_expr_ext::flags field.

unsigned long long **min_length**
The minimum match length (from start to end) required to successfully match this expression. To use this parameter, set the HS_EXT_FLAG_MIN_LENGTH flag in the hs_expr_ext::flags field.

unsigned **edit_distance**
Allow patterns to approximately match within this edit distance. To use this parameter, set the HS_EXT_FLAG_EDIT_DISTANCE flag in the hs_expr_ext::flags field.

unsigned **hamming_distance**
Allow patterns to approximately match within this Hamming distance. To use this parameter, set the HS_EXT_FLAG_HAMMING_DISTANCE flag in the hs_expr_ext::flags field.

## File: hs_runtime.h
The Hyperscan runtime API definition.

Hyperscan is a high speed regular expression engine.

This header contains functions for using compiled Hyperscan databases for scanning data at runtime.

**Defines**

**HS_OFFSET_PAST_HORIZON**
Callback ‘from’ return value, indicating that the start of this match was too early to be tracked with the requested SOM_HORIZON precision.

**Typedefs**

typedef **hs_stream_t**
The stream identifier returned by hs_open_stream().

typedef **hs_scratch_t**
A Hyperscan scratch space.

typedef ( * **match_event_handler**)(unsigned int id, unsigned long long from, unsigned long long to, unsigned int flags, void *context)
Definition of the match event callback function type.

A callback function matching the defined type must be provided by the application calling the hs_scan(), hs_scan_vector() or hs_scan_stream() functions (or other streaming calls which can produce matches).

This callback function will be invoked whenever a match is located in the target data during the execution of a scan. The details of the match are passed in as parameters to the callback function, and the callback function should return a value indicating whether or not matching should continue on the target data. If no callbacks are desired from a scan call, NULL may be provided in order to suppress match production.

This callback function should not attempt to call Hyperscan API functions on the same stream nor should it attempt to reuse the scratch space allocated for the API calls that caused it to be triggered. Making another call to the Hyperscan library with completely independent parameters should work (for example, scanning a different database in a new stream and with new scratch space), but reusing data structures like stream state and/or scratch space will produce undefined behavior.

**Return**
Non-zero if the matching should cease, else zero. If scanning is performed in streaming mode and a non-zero value is returned, any subsequent calls to hs_scan_stream() for that stream will immediately return with HS_SCAN_TERMINATED.
**Parameters**
- id -
The ID number of the expression that matched. If the expression was a single expression compiled with hs_compile(), this value will be zero.

- from -
    If a start of match flag is enabled for the current pattern, this argument will be set to the start of match for the pattern assuming that that start of match value lies within the current ‘start of match horizon’ chosen by one of the SOM_HORIZON mode flags.
    If the start of match value lies outside this horizon (possible only when the SOM_HORIZON value is not HS_MODE_SOM_HORIZON_LARGE), the from value will be set to HS_OFFSET_PAST_HORIZON.
    This argument will be set to zero if the Start of Match flag is not enabled for the given pattern.
- to -
The offset after the last byte that matches the expression.

- flags -
This is provided for future use and is unused at present.

- context -
The pointer supplied by the user to the hs_scan(), hs_scan_vector() or hs_scan_stream() function.

**Functions**

hs_error_t **hs_open_stream**(const hs_database_t * db, unsigned int flags, hs_stream_t ** stream)
Open and initialise a stream.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- db -
A compiled pattern database.

- flags -
Flags modifying the behaviour of the stream. This parameter is provided for future use and is unused at present.

- stream -
On success, a pointer to the generated hs_stream_t will be returned; NULL on failure.

hs_error_t **hs_scan_stream**(hs_stream_t * id, const char * data, unsigned int length, unsigned int flags, hs_scratch_t * scratch, match_event_handler onEvent, void * ctxt)
Write data to be scanned to the opened stream.

This is the function call in which the actual pattern matching takes place as data is written to the stream. Matches will be returned via the match_event_handler callback supplied.

**Return**
Returns HS_SUCCESS on success; HS_SCAN_TERMINATED if the match callback indicated that scanning should stop; other values on error.
**Parameters**
- id -
The stream ID (returned by hs_open_stream()) to which the data will be written.

- data -
Pointer to the data to be scanned.

- length -
The number of bytes to scan.

- flags -
Flags modifying the behaviour of the stream. This parameter is provided for future use and is unused at present.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch().

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- ctxt -
The user defined pointer which will be passed to the callback function when a match occurs.

hs_error_t **hs_close_stream**(hs_stream_t * id, hs_scratch_t * scratch, match_event_handler onEvent, void * ctxt)
Close a stream.

This function completes matching on the given stream and frees the memory associated with the stream state. After this call, the stream pointed to by id is invalid and can no longer be used. To reuse the stream state after completion, rather than closing it, the hs_reset_stream function can be used.

This function must be called for any stream created with hs_open_stream(), even if scanning has been terminated by a non-zero return from the match callback function.

Note: This operation may result in matches being returned (via calls to the match event callback) for expressions anchored to the end of the data stream (for example, via the use of the $ meta-character). If these matches are not desired, NULL may be provided as the match_event_handler callback.

If NULL is provided as the match_event_handler callback, it is permissible to provide a NULL scratch.

**Return**
Returns HS_SUCCESS on success, other values on failure.
**Parameters**
- id -
The stream ID returned by hs_open_stream().

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch(). This is allowed to be NULL only if the onEvent callback is also NULL.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- ctxt -
The user defined pointer which will be passed to the callback function when a match occurs.

hs_error_t **hs_reset_stream**(hs_stream_t * id, unsigned int flags, hs_scratch_t * scratch, match_event_handler onEvent, void * context)
Reset a stream to an initial state.

Conceptually, this is equivalent to performing hs_close_stream() on the given stream, followed by a hs_open_stream(). This new stream replaces the original stream in memory, avoiding the overhead of freeing the old stream and allocating the new one.

Note: This operation may result in matches being returned (via calls to the match event callback) for expressions anchored to the end of the original data stream (for example, via the use of the $ meta-character). If these matches are not desired, NULL may be provided as the match_event_handler callback.

Note: the stream will also be tied to the same database.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- id -
The stream (as created by hs_open_stream()) to be replaced.

- flags -
Flags modifying the behaviour of the stream. This parameter is provided for future use and is unused at present.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch(). This is allowed to be NULL only if the onEvent callback is also NULL.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- context -
The user defined pointer which will be passed to the callback function when a match occurs.

hs_error_t **hs_copy_stream**(hs_stream_t ** to_id, const hs_stream_t * from_id)
Duplicate the given stream. The new stream will have the same state as the original including the current stream offset.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- to_id -
On success, a pointer to the new, copied hs_stream_t will be returned; NULL on failure.

- from_id -
The stream (as created by hs_open_stream()) to be copied.

hs_error_t **hs_reset_and_copy_stream**(hs_stream_t * to_id, const hs_stream_t * from_id, hs_scratch_t * scratch, match_event_handler onEvent, void * context)
Duplicate the given ‘from’ stream state onto the ‘to’ stream. The ‘to’ stream will first be reset (reporting any EOD matches if a non-NULL onEvent callback handler is provided).

Note: the ‘to’ stream and the ‘from’ stream must be open against the same database.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- to_id -
On success, a pointer to the new, copied hs_stream_t will be returned; NULL on failure.

- from_id -
The stream (as created by hs_open_stream()) to be copied.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch(). This is allowed to be NULL only if the onEvent callback is also NULL.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- context -
The user defined pointer which will be passed to the callback function when a match occurs.

hs_error_t **hs_compress_stream**(const hs_stream_t * stream, char * buf, size_t buf_space, size_t * used_space)
Creates a compressed representation of the provided stream in the buffer provided. This compressed representation can be converted back into a stream state by using hs_expand_stream() or hs_reset_and_expand_stream(). The size of the compressed representation will be placed into used_space.

If there is not sufficient space in the buffer to hold the compressed representation, HS_INSUFFICIENT_SPACE will be returned and used_space will be populated with the amount of space required.

Note: this function does not close the provided stream, you may continue to use the stream or to free it with hs_close_stream().

**Return**
HS_SUCCESS on success, HS_INSUFFICIENT_SPACE if the provided buffer is too small.
**Parameters**
- stream -
The stream (as created by hs_open_stream()) to be compressed.

- buf -
Buffer to write the compressed representation into. Note: if the call is just being used to determine the amount of space required, it is allowed to pass NULL here and buf_space as 0.

- buf_space -
The number of bytes in buf. If buf_space is too small, the call will fail with HS_INSUFFICIENT_SPACE.

- used_space -
Pointer to where the amount of used space will be written to. The used buffer space is always less than or equal to buf_space. If the call fails with HS_INSUFFICIENT_SPACE, this pointer will be used to write out the amount of buffer space required.

hs_error_t **hs_expand_stream**(const hs_database_t * db, hs_stream_t ** stream, const char * buf, size_t buf_size)
Decompresses a compressed representation created by hs_compress_stream() into a new stream.

Note: buf must correspond to a complete compressed representation created by hs_compress_stream() of a stream that was opened against db. It is not always possible to detect misuse of this API and behaviour is undefined if these properties are not satisfied.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- db -
The compiled pattern database that the compressed stream was opened against.

- stream -
On success, a pointer to the expanded hs_stream_t will be returned; NULL on failure.

- buf -
A compressed representation of a stream. These compressed forms are created by hs_compress_stream().

- buf_size -
The size in bytes of the compressed representation.

hs_error_t **hs_reset_and_expand_stream**(hs_stream_t * to_stream, const char * buf, size_t buf_size, hs_scratch_t * scratch, match_event_handler onEvent, void * context)
Decompresses a compressed representation created by hs_compress_stream() on top of the ‘to’ stream. The ‘to’ stream will first be reset (reporting any EOD matches if a non-NULL onEvent callback handler is provided).

Note: the ‘to’ stream must be opened against the same database as the compressed stream.

Note: buf must correspond to a complete compressed representation created by hs_compress_stream() of a stream that was opened against db. It is not always possible to detect misuse of this API and behaviour is undefined if these properties are not satisfied.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- to_stream -
A pointer to a valid stream state. A pointer to the expanded hs_stream_t will be returned; NULL on failure.

- buf -
A compressed representation of a stream. These compressed forms are created by hs_compress_stream().

- buf_size -
The size in bytes of the compressed representation.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch(). This is allowed to be NULL only if the onEvent callback is also NULL.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- context -
The user defined pointer which will be passed to the callback function when a match occurs.

hs_error_t **hs_scan**(const hs_database_t * db, const char * data, unsigned int length, unsigned int flags, hs_scratch_t * scratch, match_event_handler onEvent, void * context)
The block (non-streaming) regular expression scanner.

This is the function call in which the actual pattern matching takes place for block-mode pattern databases.

**Return**
Returns HS_SUCCESS on success; HS_SCAN_TERMINATED if the match callback indicated that scanning should stop; other values on error.
**Parameters**
- db -
A compiled pattern database.

- data -
Pointer to the data to be scanned.

- length -
The number of bytes to scan.

- flags -
Flags modifying the behaviour of this function. This parameter is provided for future use and is unused at present.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch() for this database.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- context -
The user defined pointer which will be passed to the callback function.

hs_error_t **hs_scan_vector**(const hs_database_t * db, const char *const * data, const unsigned int * length, unsigned int count, unsigned int flags, hs_scratch_t * scratch, match_event_handler onEvent, void * context)
The vectored regular expression scanner.

This is the function call in which the actual pattern matching takes place for vectoring-mode pattern databases.

**Return**
Returns HS_SUCCESS on success; HS_SCAN_TERMINATED if the match callback indicated that scanning should stop; other values on error.
**Parameters**
- db -
A compiled pattern database.

- data -
An array of pointers to the data blocks to be scanned.

- length -
An array of lengths (in bytes) of each data block to scan.

- count -
Number of data blocks to scan. This should correspond to the size of of the data and length arrays.

- flags -
Flags modifying the behaviour of this function. This parameter is provided for future use and is unused at present.

- scratch -
A per-thread scratch space allocated by hs_alloc_scratch() for this database.

- onEvent -
Pointer to a match event callback function. If a NULL pointer is given, no matches will be returned.

- context -
The user defined pointer which will be passed to the callback function.

hs_error_t **hs_alloc_scratch**(const hs_database_t * db, hs_scratch_t ** scratch)
Allocate a “scratch” space for use by Hyperscan.

This is required for runtime use, and one scratch space per thread, or concurrent caller, is required. Any allocator callback set by hs_set_scratch_allocator() or hs_set_allocator() will be used by this function.

**Return**
HS_SUCCESS on successful allocation; HS_NOMEM if the allocation fails. Other errors may be returned if invalid parameters are specified.
**Parameters**
- db -
The database, as produced by hs_compile().

- scratch -
On first allocation, a pointer to NULL should be provided so a new scratch can be allocated. If a scratch block has been previously allocated, then a pointer to it should be passed back in to see if it is valid for this database block. If a new scratch block is required, the original will be freed and the new one returned, otherwise the previous scratch block will be returned. On success, the scratch block will be suitable for use with the provided database in addition to any databases that original scratch space was suitable for.

hs_error_t **hs_clone_scratch**(const hs_scratch_t * src, hs_scratch_t ** dest)
Allocate a scratch space that is a clone of an existing scratch space.

This is useful when multiple concurrent threads will be using the same set of compiled databases, and another scratch space is required. Any allocator callback set by hs_set_scratch_allocator() or hs_set_allocator() will be used by this function.

**Return**
HS_SUCCESS on success; HS_NOMEM if the allocation fails. Other errors may be returned if invalid parameters are specified.
**Parameters**
- src -
The existing hs_scratch_t to be cloned.

- dest -
A pointer to the new scratch space will be returned here.

hs_error_t **hs_scratch_size**(const hs_scratch_t * scratch, size_t * scratch_size)
Provides the size of the given scratch space.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- scratch -
A per-thread scratch space allocated by hs_alloc_scratch() or hs_clone_scratch().

- scratch_size -
On success, the size of the scratch space in bytes is placed in this parameter.

hs_error_t **hs_free_scratch**(hs_scratch_t * scratch)
Free a scratch block previously allocated by hs_alloc_scratch() or hs_clone_scratch().

The free callback set by hs_set_scratch_allocator() or hs_set_allocator() will be used by this function.

**Return**
HS_SUCCESS on success, other values on failure.
**Parameters**
- scratch -
The scratch block to be freed. NULL may also be safely provided.

