# Dev-log: July 9th 2021

## Native Ahead of Time Compilation (nAOT), part 1: Introduction

## Motivation: Why use nAOT?

Recently I have been experimenting away from MonoGame for creating games in C#. There are actually quite a few options for creating games in C# today (some better than others) such as:

- Unity
- Godot
- Stride (formerly Xenko)
- Wave Engine
- Cry Engine

However these are *engines*. I'm not interested in using an engine. I am interested in learning and exploring the knowledge required to make a game from scratch; *no-engine* if you will. Here are couple "frameworks" in C# which glue together some fundementally APIs such as windowing + graphics:

- Veldrid
- MonoGame
- FNA
- Raylib-cs

This is great. However, I'm still not quite satisfied. To give someone a copy of your game for Windows/macOS/Linux, that person needs to download the .NET runtime. Or, you need to bundle a trimmed .NET runtime with your game. Hundreds of megabytes for just a base starting point doesn't sound like much but to me it's not acceptable. I'm specifically looking for and striving for "minimalism" in the way I approach programming as an art form. Let's walk through how traditional .NET applications operate and work our way up to a solution to this unsatfisfactory situation with Native Ahead of Time Compilation (nAOT).

### Traditional .NET

C# has been around since 2001. Circa 2001 Microsoft basically copied Java with J++ and later J# (precursors of C#). In more ways similar than not, around 2001 C# *was for all intents and purposes* Java. Java has the concept of *bytecode* which is what your *source code* gets compiled to. That *byte code* then gets compiled into *native machine code* for the target platform at **runtime of your program** and then executed by your CPU in the common [fetch/decode/execute loop](https://en.wikipedia.org/wiki/Instruction_cycle) paradigm. You know that short delay you have when you start your C# program? Yeah, the majority of that time is Common Intermediate Language (CIL) getting compiled into native assembly code. This type of compilation is called Just in Time (JIT) compilation and requires a virtual machine. That virtual machine and other parts of the Common Language Runtime (CLR), such as the Garbage Collector (GC), are [primarly written in C/C++/Assembly source code](https://github.com/dotnet/runtime/tree/main/src/coreclr). The compiled version of the runtime is that thing you and your friends need to download and install to run your programs called the [*.NET Runtime* or *.NET Core Runtime* or *.NET Framework Runtime*](https://dotnet.microsoft.com/download).

Microsoft has a name for these *class* of applications; they are called ["Global and general purpose"](https://github.com/dotnet/designs/blob/main/accepted/2020/form-factors.md#global-and-general-purpose). The word *class* might be confusing being used in a purely mathmatical abstract sense so let's adopt Microsoft's language here for calling it *form factor*. There are some nice advantages to this form factor which I'm calling **traditional .NET**.

(This list may be updated over time. If you find anything incorrect, please open an issue.)

|#|Strengths (helpful to your organization)|
|:---:|:---:|
|1|Relatively small file size applications because they only have the code you wrote/imported in the form of Intermediate Language (IL) code. The common stuff is part of the .NET runtime to which it is essentially shared for all .NET applications on your computer.|
|2|Portable applications (at least most of the time) where you can simply copy and paste the `.exe`/`.dll` and the other files of your application accross Windows/macOS/Linux because each platform has the .NET runtime installed which contains all the platform specifics. Note that in practice this isn't really true for most applications, especially ones using graphical user interface technologies such as WinForms and WPF, because these applications rely heavily on interopability with Windows specific APIs such as GDI+ or DirectX.|
|3|Upgrades to the JIT compiler or .NET runtime will allow your code written/shipped yesterday to run faster/better today without any changes/redeployment required by you directly. Note that in practice upgrades to the .NET runtime sometimes require code changes because of how APIs break/change.|
|4|JIT compilation will only compile the Intermediate Language (IL) code to machine code which is actually required to execute because it is done on the fly. Additionally, some optimizations can only be done by this dynamic analysis at runtime to which the code may be slow at first, but eventually the JIT compiler will make the correct optimizations for the code, especially if said code is executed frequently.|
|5|The talent / experience pool for workers is quite high. It's C# as people know it.|

|#|Weaknesses (harmful to your organization)|
|:---:|:---:|
|1|Users have to download and install the .NET runtime. This is north of hundreds of megabytes approaching or sometimes exceeding half a gigabyte. If some users have different versions of the runtime installed, things might work differently for said users compared to others. This is usually mitigiated by having users download and install a very specific version of the runtime for the sole purpose of your application.|
|2|Some non-zero time is spent at the startup of the application to compile Intermediate Language (IL) code to native machine code. Additionally, some non-zero time is spent at runtime to compile Intermediate Language (IL) code to native machine code which was not done at startup. The JIT compiler also heavily uses memory caching for knowing what IL code has already been compiled to machine code. All together this is significant for games and other real-time applications, where time is spent not running the game at full potential of the hardware or in best effort to conserve battery life.|

|#|Opportunites (helpful to your organization)|
|:---:|:---:|
|1|Verifying your code is malware free is relatively straight forward and simple with intermediate language (IL) code.|

|#|Threats (harmful to your organization)|
|:---:|:---:|
|1|Decompiling your code for nefarious or legimate purposes is relatively straight forward and simple with intermediate language (IL) code with such projects as [DotPeek](https://www.jetbrains.com/decompiler/) and [ILSpy](https://github.com/icsharpcode/ILSpy). In some cases the source code could even be reconstructed to be identical to the original line by line. Third parties provide some solutions to obfuscate or functionally make decompiling more difficult such as [SmartAssembly](https://www.red-gate.com/products/dotnet-development/smartassembly/), [Dotfuscator](https://www.preemptive.com/products/dotfuscator), and many others. Such efforts are usually a game of cat and mouse as such projects as https://github.com/dnSpy/dnSpy and https://github.com/de4dot/de4dot.|

### Native Ahead of Time Compilation (nAOT) with .NET

By compiling the source code to intermediate language (IL) and then directly to machine code for the target platform (operating system + architecture), most of the weaknesses and threats above are addressed. However there are some consequences which might make it unsuitable for specific scenarios. However for games, which are a class of real-time systems, the trade offs are often justified and warranted.

(This list may be updated over time. If you find anything incorrect, please open an issue.)

|#|Strengths (helpful to your organization)|
|:---:|:---:|
|1|Minimal to zero dependencies, inluding no .NET runtime. This allows stand-alone applications getting down to 5 megabyes or less and optionally with additional features removed from .NET starting around or less than 1 megabyte.|
|2|Code can become as hard as to decompile as C/C++ applications because the compiled artifacts are all machine code as opposed to intermediate language or a mix thereof. This is especially true if meta-data is stripped from the build artifacts. Meta-data is mostly used for reflection APIs.|
|3|Native speeds are achievable comparable to C/C++ applications because the compiled artifacts are all machine code as opposed to intermediate language or a mix thereof. Most noticeable is that startup times are in milliseconds rather than seconds because there is no just-in-time (JIT) warmup phase.|
|4|Battery life for embedded or mobile class of devices is more easily conserved with machine code that gets executed directly compared to intermediate language (IL) being compiled to machine code and then executed with the help of a virtual machine at runtime.|

|#|Weaknesses (harmful to your organization)|
|:---:|:---:|
|1|Reflection APIs such as those found in `System.Reflection` are finicky or straight up unavailable. Depending on your stance of wether such APIs are good or bad, this might actually not be a weakness.|
|2|No dynamically loading intermediate language (IL) code at runtime using `Assembly.LoadFile` or similar. In some contexts, loading machine code at runtime is okay.|
|3|To take advantages of advancements or upgrades in the compiler toolchain the application will need to be re-compiled and de-deployed. This would also be true for library dependencies if they are originally written in C#.|
|3|The talent / experience pool for workers will naturally be more narrow. There is a lot of nuance for getting code with .NET working under limitations of nAOT. E.g. in some cases, features or APIs known and used previously throughout .NET ecosystem are optionally turned off on purpose such as reflection APIs all together or exception throwing.|

|#|Opportunites (helpful to your organization)|
|:---:|:---:|
|1|Targeting constrained devices such as phones, watches, or consoles for deployment is more realistically in reach. .NET applications can truely approach a strategy of "write once, deploy everywhere". This allows the .NET ecosystem to continue to evolve and grow.|

|#|Threats (harmful to your organization)|
|:---:|:---:|
|1|?|

## Getting started: Hello World with nAOT

See https://github.com/dotnet/runtimelab/tree/feature/NativeAOT/samples/HelloWorld




