# Ideal integration with OSS-Fuzz 
OSS projects have different build and test systems. So, we can not expect them
to have a unified way of implementing and maintaining fuzz targets and integrating
them with OSS-Fuzz. However, we will still try to give recommendations on the preferred ways.

Here are the 4 stages of integraion (starting from the easiest) that will make automated fuzzing
simple, efficient and catch regressions early on in the development cycle. 

## Stage 1: Fuzz Target
The code of the [fuzz target(s)](http://libfuzzer.info/#fuzz-target) should be part of the project's source code repository. 
All fuzz targets should be easily discoverable (e.g. reside in the same directory, or follow the same naming pattern, etc). 

This makes it easy to maintain fuzzer and minimizes breakages that can arise as source code changes over time.

Examples: 
[boringssl](https://github.com/google/boringssl/tree/master/fuzz),
[SQLite](https://www.sqlite.org/src/artifact/ad79e867fb504338),
[s2n](https://github.com/awslabs/s2n/tree/master/tests/fuzz),
[openssl](https://github.com/openssl/openssl/tree/master/fuzz),
[FreeType](http://git.savannah.gnu.org/cgit/freetype/freetype2.git/tree/src/tools/ftfuzzer),
[re2](https://github.com/google/re2/tree/master/re2/fuzzing),
[harfbuzz](https://github.com/behdad/harfbuzz/tree/master/test/fuzzing),
[pcre2](http://vcs.pcre.org/pcre2/code/trunk/src/pcre2_fuzzsupport.c?view=markup),
[ffmpeg](https://github.com/FFmpeg/FFmpeg/blob/master/doc/examples/decoder_targeted.c).


## Stage 2: Seed Corpus
The *corpus* is a set of inputs for the fuzz target (stored as individual files). 
When starting the fuzzing process, one should have a "seed corpus", 
i.e. a set of inputs to "seed" the mutations.
The quality of the seed corpus has a huge impact on the fuzzing efficiency as it allows the fuzzer
to discover new code paths easier. 

The ideal corpus is a minimial set of intputs that provides maximal code coverage. 

For better OSS-Fuzz integration 
the seed corpus should be available in revision control (can be same or different as the source code). 
It should be regularly extended with the inputs that (used to) trigger bugs and/or touch new parts of the code. 

Examples: 
[boringssl](https://github.com/google/boringssl/tree/master/fuzz),
[openssl](https://github.com/openssl/openssl/tree/master/fuzz),
[nss](https://github.com/mozilla/nss-fuzzing-corpus) (corpus in a separate repo) 


## Stage 3: Regression Testing
The fuzz targets should be regularly tested (not necessary fuzzed!) as a part of the project's regression testing process.
One way to do so is to link the fuzz target with a simple driver
(e.g. [this one](https://github.com/llvm-mirror/llvm/tree/master/lib/Fuzzer/standalone))
that runs the provided inputs and use this driver with the seed corpus created in previous step. 
It is recommended use the [sanitizers](https://github.com/google/sanitizers) during regression testing.

Examples: [SQLite](https://www.sqlite.org/src/artifact/d9f1a6f43e7bab45),
[openssl](https://github.com/openssl/openssl/blob/master/fuzz/test-corpus.c)

## Stage 4: Build support
A plethora of different build systems exist in the open-source world.
And the less OSS-Fuzz knows about them, the better it can scale. 

An ideal build integration for OSS-Fuzz would look like this:
* For every fuzz target `foo` in the project, there is a build rule that builds `foo_fuzzer.a`,
an archive that contains the fuzzing entry point (`LLVMFuzzerTestOneInput`)
and all the code it depends on, but not the `main()` function
* The build system supports changing the compiler and passing extra compiler
flags so that the build command for a `foo_fuzzer.a` looks similar to this:

```
CC="clang $FUZZER_FLAGS" CXX="clang++ $FUZZER_FLAGS" make_or_whatever_other_command foo_fuzzer.a
```

In this case, linking the target with e.g. libFuzzer will look like "clang++ foo_fuzzer.a libFuzzer.a".
This will allow to have minimal OSS-Fuzz-specific configuration and thus be more robust. 

There is no point in hardcoding the exact compiler flags in the build system because they 
a) may change and b) are different depending on the fuzzing target and the sanitizer being used. 
