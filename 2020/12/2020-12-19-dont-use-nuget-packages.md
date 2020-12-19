# Dev-log: December 19th 2020

## Don't use NuGet packages

NuGet packages are a great way for people to use your code. Except when it's not. Let's go over some problems with NuGet packages and why I personally do not recommend them *for some projects*.

### Why you should not use NuGet packages

1. **It hurts when developers downstream have issues**

When a developer is using your code through a NuGet package, he/she will eventually encounter some issues. What does the developer do in this situation? Probably report an issue and you, the author of the NuGet package, have to to investigate and most likely fix it yourself. This is not ideal. Why? Because developers should be able to report **and** investigate *themselves*. Developers can only really do this if they have the source code.

Yes, it is possible to decompile the packaged NuGet binaries and investigate like so by disabling "Just my Code" in VisualStudio. But this is not the real source code, it's decompiled source code. Ignoring the legal issues of decompiling source code (you really should not), the binary is probably built in `release` configuration making debugging slightly harder. Worse, any comments are gone. This also doesn't work in Ahead-of-Time (AOT) complilation scenarios which is common for mobile applications and games. It only works for Just-in-Time (JIT) compliation scenarios. In an (native) AOT context, the code is no longer intermediate-language (IL) code but rather assembly code for an instruction set architecture (ISA) such as `x86_64` or `amd64`. Good luck debugging a running game or iOS app without the source code!

2. **It hurts open-source development**

If a developer can investigate and fix a problem themself *because they have the source code* then they can more easily submit a pull-request upstream if they are so kind. This is not easily possible with NuGet packages simply because developers don't have the source code. This means that NuGet packages hurt open-source; if you want your project to grow in fashion where you are encouraging contributors, NuGet packages are just not that great.

3. **It's dependency hell all over again**

Let's say a developer is working on `CompanyName.Project1`. The developer adds `PackageA` with a dependency on `PackageZ` version `v2.1.505.2`. So far so good. The developer then also adds `PackageB` with a dependency of `PackageZ` version `v3.0.1304.1`. Uh-oh. The problem presented here is small and possibly easily manageable to solve. In real situations the complexity of mis-matching dependency versions is very much not small and not at all easily manageable to solve as their can be non-trivial dependency graphs. Even if you think you successfully managed to add NuGet packages where your project compiles, a runtime error like "Could not load file or assembly '***.dll' or one of its dependencies. The specified module could not be found." can be very real.

### So, what to use instead of NuGet packages?

There is no silver bullet here. However, if your project agrees with all the points above I personally recommend using Git submodules or simply just copying source code over. Be warned however, really think about the positives and consequences for your project before you take such a decision.






