# ILC CPU and memory usage

ILC, the NativeAOT compiler, runs in several stages. Each of those stages has specific CPU 
and memory requirements. While there may be some overlaps between these stages and this is 
a deliberate over-simplification, let's divide them into three general sections.

# Marking, scanning and whole program analysis

The NativeAOT compiler starts by finding compilation roots (eg. Main method, exported symbols, 
other symbols from trimming descriptors) and then recursively walking the dependencies until 
a whole program view is produced.

As of .NET 9 Preview 5 this stage is largely single-threaded and bound by the compiled 
program size. Some specific code constructs are more expensive than others (eg. generic virtual 
methods, LINQ over value types, deep class hierarchies, etc.).

To keep the compile time in this stage in check, a good advice is to check the output of 
[sizoscope](https://github.com/MichalStrehovsky/sizoscope) and keep tabs on which code 
ends up in the final output. Large number of generic instantiations is a typical red flag.

# Code compilation

Compiling IL code through RyuJIT into a native code that ends up in the final output is 
an expensive stage of the compilation. It is, however, run largely in parallel. On modern 
machines with many CPU cores this effectively helps to reduce the compilation time.

On .NET 9 Preview 5 this parallelization is currently limited to up to 24 threads, or 
number of CPU cores if it's smaller. This upper bound ensures that memory usage from 
the compilation stays in check. Running too many compilations in parallel could unintentially 
exhaust the memory and slow down the overall compilation time as a result, so some 
balance needs to be maintained.

# Object file emitting

Once the whole program is analyzed and code compiled, the final output needs to be written. 
This encompasses the final executable but also the debugging information necessary for the 
native debuggers.

This stage is currently single-threaded and very memory consumption intensive.

The CPU usage during this phase is currently dominated by `ObjectNode.GetData` calls. While 
the code itself was already produced in the code compilation phase, some data tables (eg.
virtual methods slot tables and reflection metadata) are only lazily created here.

Prior to .NET 9 a custom library based on LLVM assembler code was used to produce the object 
files and debugging information. This had the benefit of relying on a time-proven and battle 
tested foundation that made it easier to support multiple platforms. However, it also came 
with its own shortcomings. There's a disparity between the representation of various concepts 
in the ILC compiler and the LLVM assembler. This disparity results in larger than necessary 
memory usage which is also harder to track in unmanaged code.

In .NET 9 the object file writing code was completely rewritten. The main objective of the 
rewrite was to remove the LLVM dependency and to use memory more efficiently. The rewrite 
achieved that by removing significant amount of memory copying, and improved management of 
relocation tables. It also allows a more fine grained control of the finicky details of 
some object file formats that can be used for further optimizations.

The object writer supports three general object file formats - COFF (Windows), Mach-O (macOS), 
and ELF (Linux, FreeBSD). The general structure of the object file format is pretty similar 
across the formats even though the on-disk structure is different.

## Unwind tables

One particular part of the object file writer emits the unwind tables. The unwind tables are used for walking the call stack during exception handling, garbage collection, and other operations. The format of these tables depends on the platform.

Windows have their own table based format that's described in the [PE file format specification](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format). There's a PDATA table that maps code address to unwind data. The actual unwind data are in the XDATA table. Format of these tables differs slightly for each platform. Notably, ARM64 has a way to represent common function prolog/epilog using compact codes. This is currently not used by ILC and it may be worth exploring to get some size benefits.

macOS and Linux both historically use the same standardized form - DWARF. The format is a bit verbose but also incredibly powerful. In fact, it's so powerful that [academic paper proved that it's Turning complete](https://static.usenix.org/event/woot11/tech/final_files/Oakley.pdf).

Because DWARF itself is very generic and verbose, Apple has come up with a proprietary solution that augments it - compact unwinding tables. The general concept is the same as in Windows - represent the common prologs/epilogs with a simple 32-bit code, and fallback everything more complext to the DWARF format. Just like Windows the actual code format differs per platform. The compact unwinding tables serve a secondary purpose too. DWARF itself doesn't have an index for quickly locating an unwind information given a code address. The solution that Apple devised is to use the compact unwinding tables as the index and reserve a single code with 24-bit offset as a way to map a code address into link inside the DWARF table. The downside of this approach is that the size of the DWARF section is limited to around 16 MiB.

.NET ARM64 JIT currently generate prologs are incompatible with the Apple Compact Unwinding encoding. There's an [issue filed](https://github.com/dotnet/runtime/issues/76371) to investigate how can we improve that in the future. Notably, while evaluating the benefits of compact unwinding we found that typical programs contain roughly between 20% and 30% leaf methods (ie. methods that don't call any other method) with a prolog that doesn't save any registers on the stack except for LR (Link Register) and FP (Frame Pointer). Turns out that this specific sequence can be represented by compact unwinding, so ILC detects it and emits the compact unwinding code there. The large proliferation of these methods [begs the question whether we can get rid of the prolog/epilog](https://github.com/dotnet/runtime/issues/88823) and get code size improvements in addition to the size optimization of the unwind tables.

The 16 MiB size limit for DWARF section on Apple platforms is generally problematic. It places an additional limit on the executable code size that is difficult to express and document. Some versions of Apple linker are known to silently produce corrupted output when this limit is hit which is notoriously difficult to diagnose. We also cannot reliably detect it in the ILC compiler since the limit is imposed on the final executable, not on the object files. An error can be produced when the DWARF section size hits the limit in the generated object file but it doesn't guaranteed that the final executable is not broken. 

# Experiments

When we embarked on the journey of profiling the ILC compiler, the expectation was that each of the above phases will be clearly visible in the profile. That we will see the memory usage increase steadily during the mark and compile phase and then suddenly quickly go up for a short period while the object file was emitted.

Turns out, it's never so simple. WinForms in particular have rather deep object hierarchies which hit [unexpected bottlenecks in the marking phase](https://github.com/dotnet/runtime/issues/103034). There were few other similar issues that were hit along the way:
- [Hash codes that produced guaranteed collisions](https://github.com/dotnet/runtime/issues/103070)
- [Type checks that unexpectedly dominate processing time](https://github.com/dotnet/runtime/pull/103199)
- [Caches that perform worse than if the cache was not present](https://github.com/dotnet/runtime/issues/103285)
- [Repeated dictionary lookups that can be optimized](https://github.com/dotnet/runtime/pull/103202)

Most of these issues are clearly visible in the profiler when compiling project of this size. They are often easy to fix with just few lines of code. Ever wondered when your knowledge of algorithms and O-notation comes useful in the real world? Then this is precisely the area. Simple changes may result in huge speed ups. Fixing the above problems saves more than 3 minutes, or 30%, on the compilation times.

Other notable insight is that DATAS reduced the peak memory usage from ~20 GiB to ~15 GiB (private working set). 

Lastly, there are some expensive `ObjectNode.GetData` calls that are run in single-threaded fashion in the Object File Emitting phase. This is an area for potential improvement in compile times. We implemented a [limited parallelization of the `GetData` calls](https://github.com/filipnavara/runtime/commit/44cc7e653300cce8db142c285ab0651c1635b06b). This resulted in a speed up of 20+ seconds. Unfortunately, the prototype doesn't guarantee full determinsic output as is. A more limited approach focusing on precalculating just the expensive data (eg. virtual method slots in `EETypeNode`) may be a viable alternative to explore that should yield at least 60% of the time savings produced with the simple prototype.
