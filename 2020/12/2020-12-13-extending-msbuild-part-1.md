# Dev-log: December 13th 2020

## Extending MSBuild (Part 1)

In 2020 I got tired of my `.csproj` files having duplicated configurations. So I was exploring how to share common `ItemGroup`, `PropertyGroup`, and other definitions between multiple `.csproj` files.

### Solution A: `<Import Project="..." />`

Extract common settings into a single project file and use that project file's settings in other projects. For consistency, use the file extension `.props`; the reason will become clear in the next, more advanced, solution to the problem. For now let's take a look at a simple example.

#### Example

`/MyProjects/Common.props`:
```xml
<Project>
    <TargetFramework>net5.0</TargetFramework>
</Project>
```

`/MyProjects/ProjectA/ProjectA.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    ...
    <Import Project="$(MSBuildThisFileDirectory)../Common.csproj" />
    ...
</Project>
```

`/MyProjects/ProjectB/ProjectB.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    ...
    <Import Project="$(MSBuildThisFileDirectory)../Common.csproj" />
    ...
</Project>
```

This effectively makes `ProjectA.csproj` and `ProjectB.csproj` both have `TargetFramework` of `net5.0`. Note that `MSBuildThisFileDirectory` is the directory path of the current `.csproj` **with the final slash**. Also, placing the `Import` XML node earlier or later will have different consequences. If you want all the shared settings in `Common.props` to be applied first, place it at the top. In this case, the current `.csproj` settings can use or override any settings defined in the `Common.props` for the remainder of the `.csproj`.

### Solution B: `Directory.Build.props` and `Directory.Build.targets`

Solution A works. However there is some gotchas: (1) you have to add the `<Import Project=".." />` node for each `.csproj`, and (2) some settings can not be applied correctly such as `BaseIntermediateOutputPath`.

#### Problem: `BaseIntermediateOutputPath` property

The `BaseIntermediateOutputPath` property allows you to change the directory path where `obj` directory is going to be created and populated for your project. However, you can't effectively set it with solution A because the value is used earlier in usual `.csproj` file setups.

An example which **does not work**.

```xml
<Project Sdk="Microsoft.NET.Sdk">
    ...
    <PropertyGroup>
        <BaseIntermediateOutputPath>../obj/</BaseIntermediateOutputPath>
    </PropertyGroup>
    ...
</Project>
```

And an example which **does work**.

```xml
<Project>
    <PropertyGroup>
        <BaseIntermediateOutputPath>../obj/</BaseIntermediateOutputPath>
    </PropertyGroup>
    <Import Project="Sdk.props" Sdk="Microsoft.NET.Sdk" />
    ...
    <Import Project="Sdk.targets" Sdk="Microsoft.NET.Sdk" />
</Project>
```

The reason the second example works is because the value of `BaseIntermediateOutputPath` property is used in the `Sdk.props` project. When you do `<Project Sdk="Microsoft.NET.Sdk">` at the top of your `.csproj` file you are effectively implictly importing the `Sdk.props` project at the top, then your `.csproj` is applied, and finally the `Sdk.targets` project is imported at the bottom. In such a case, setting the `BaseIntermediateOutputPath` property has no effect.

#### `.props` vs `.targets`

So what's the difference between `.props` and `.targets` file extensions? It's just a convention: a `.props` file contains property definitions for *early* things in the build process and a `.targets` file contains definitions for *later* things.

#### `Directory.Build.props` and `Directory.Build.targets`

To overcome the gotchas mentioned and combine the example found in solution A we can use a special file called `Directory.Build.props`. Let's take a look at an example.

`/MyProjects/Directory.Build.props`:
```xml
<Project>
    <TargetFramework>net5.0</TargetFramework>
    <BaseIntermediateOutputPath>$(MSBuildThisFileDirectory)obj/</BaseIntermediateOutputPath>
</Project>
```

`/MyProjects/ProjectA/ProjectA.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    ...
</Project>
```

`/MyProjects/ProjectB/ProjectB.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    ...
</Project>
```

This works because `Sdk.props`, which is an implictly imported project at the top of each `.csproj`'s definitions in this case, is importing the `Directory.Build.props` at the top of it's own definitions. This means that we can set the `BaseIntermediateOutputPath` property correctly before it's used.

Also, it allows us to remove explict `<Import Project=".." />` XML node in each `.csproj`'s definitions. This is because `Sdk.props` will search from the current directory of the `.csproj` and recursively up until it finds a `Directory.Build.props` file.

`Directory.Build.targets` is the same story except it's for `Sdk.targets`. All-in-all, solution B is a win-win!

#### So, what should I use? `Directory.Build.props` or `Directory.Build.targets`?

Here are some *guidelines* I came up with from experience:

- I don't care: Use `Directory.Build.props` xor `Directory.Build.targets`, preferably `Directory.Build.props`. Definitions are replaced for their values at execution time so it doesn't really matter in this case.
- I want to apply default definitions for a `.csproj`: Use `Directory.Build.props`. This ensures that the default definitions can be overriden in a `.csproj`.
- I want to override definitions in a `.csproj`: Use `Directory.Build.targets`. This ensures that whatever definitions are set in a `.csproj` they are always overriden.
- I want to include items conditioned on a property: Use `Directory.Build.props`. Every property (e.g. a XML node in a `PropertyGroup`) is processed before any item (e.g. a XML node in a `ItemGroup`) regardless of the order they are defined. By using `Directory.Build.props` you allow the `.csproj` to customize the items brought in by the import which is what developers expect to be able to do.

Some exceptions I found:

- The `RootNamespace` property does not work correctly in a `Directory.Build.props` file. I either have to define the `RootNamespace` property in each `.csproj` or `Directory.Build.targets`. See: https://github.com/dotnet/sdk/issues/11811

## Practical examples of extending MSBuild

Here are some examples which I use in my MSBuild customization: https://github.com/lithiumtoast/my-msbuild

### 1. Property: `GitRepositoryPath`.

The directory path of the Git repository.

How it works: Searches the current `.csproj`'s directory and up recursively until it finds a `.gitignore`. This assumes of course that you always have only one `.gitignore` file.

Why: I like to do some C# build tasks relative to the root directory of the Git repository.

`Directory.Build.props`:
```xml
<PropertyGroup>
    <GitRepositoryPath>$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildProjectDirectory), .gitignore))/</GitRepositoryPath>
</PropertyGroup>
```

### 2. Changing `bin` and `obj` directories to be at the root of the Git repository.

Builds on example #1.

How it works: Creates build artifacts in folders at the root of your Git repository. E.g., `/MyGitRepo/bin/ProjectA/Debug/net5.0/`

Why: I like doing this because I can always find the latest intermediate and output build artifacts regardless of where or how the project's are placed. This also becomes convenient when I want to delete artifacts to re-claim disk space or force a re-build. I have trust issues sometimes when cleaning in VisualStudio/Rider/CLI.

`Directory.Build.props`:
```xml
<PropertyGroup>
    <OutputPath>$(GitRepositoryPath)bin/$(MSBuildProjectName)/$(Configuration)</OutputPath>
    <BaseIntermediateOutputPath>$(GitRepositoryPath)obj/$(MSBuildProjectName)/$(Configuration)</BaseIntermediateOutputPath>
    <AppendTargetFrameworkToOutputPath>true</AppendTargetFrameworkToOutputPath>
    <AppendRuntimeIdentifierToOutputPath>true</AppendRuntimeIdentifierToOutputPath>
</PropertyGroup>
```

Changing the `AppendTargetFrameworkToOutputPath` property to `false` prevents adding a `netcoreapp3.1`, `net5.0`, `netstandard2.0`, etc to the output path. Changing the `AppendRuntimeIdentifierToOutputPath` property to `false` prevents adding a `win-x86`, `osx-x64`, `linux-x64`, etc to the output path. I recommend having these properties as `false` only when working on projects which are *not* targeting multiple frameworks *nor* multiple platforms. I.e, you most likely want these properties to be `true` to prevent problems when building multiple target-framework and multiple platform builds.

### 3. Changing NuGet package output directory to be at the root of the Git repository.

Builds on example #1.

How it works: Creates NuGet packages artifacts (`.nupkg`, `.snupkg`) in a folder at the root of your Git repository. E.g., `/MyGitRepo/nupkg/ProjectA.1.0.0.nupkg`

I like doing this because I can always find the latest build NuGet packages regardless of where or how the project's are placed. This also becomes convenient when I want to move the NuGet packages or delete them to re-claim disk space. 

`Directory.Build.props`:
```xml
<PropertyGroup>
    <IsPackable>false</IsPackable>
    <PackageOutputPath>$(GitRepositoryPath)nupkg</PackageOutputPath>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>
```

Changing the `IsPackable` property to `true` allows projects to be packed into packages. I usually use a default of `false` and change the property to `true` in each `.csproj` to optionally opt-in per project.  
Changing the `IncludeSymbols` to `false` prevents creating packages with symbol information which are used for debugging purposes. I usually use a default of `true` because I want debug symbols for all my NuGet packages so others (including myself!) can step in with a debugger to code in a NuGet package.  
The `SymbolPackageFormat` property is the file extension (without the `.`) of the package including symbols.


