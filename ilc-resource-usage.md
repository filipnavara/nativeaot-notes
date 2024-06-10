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
