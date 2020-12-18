# Dev-log: December 17th 2020

## Extending MSBuild: NuGet packages (Part 2)

This log builds upon the previous: [2020-12-13: Extending MSBuild (Part 1)](./2020/12/2020-12-13-extending-msbuild-part-1.md)

**Personal recommendation: Don't use NuGet packages to do this. If you agree, nothing to see here; move along.**

It is possible to package your `.props` and `.targets` into a NuGet package and have the said `.props` and `.targets` files used in the project the NuGet package is installed in. The NuGet package itself does not need to have dependencies nor does it need to have code; it can be just a way to install your MSBuild extensions into a project.

Let's look at an example using a dummy `.csproj` for creating a NuGet package.

### ProjectC (The NuGet Package project)

`/ProjectC/ProjectC.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
    <!-- MSBuild project settings -->
    <PropertyGroup>
        <!-- Not really using .NET Standard but we need something for the dummy C# project. -->
        <TargetFramework>netstandard2.0</TargetFramework>
        <!-- We will provide an empty `AssemblyInfo.cs` so our dummy C# project "has code" to prevent an error. -->
        <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    </PropertyGroup>

    <!-- NuGet https://docs.microsoft.com/en-us/dotnet/core/tools/csproj#nuget-metadata-properties https://docs.microsoft.com/en-us/nuget/reference/msbuild-targets#pack-target-inputs -->
    <PropertyGroup>
        <IsPackable>true</IsPackable>
        <IncludeBuildOutput>false</IncludeBuildOutput>
        <SuppressDependenciesWhenPacking>true</SuppressDependenciesWhenPacking>
        <DevelopmentDependency>true</DevelopmentDependency>
    </PropertyGroup>

    <!-- Build files -->
    <ItemGroup>
        <!-- It's important that the .props and .targets files have a filename which is exactly the same as the package name! Also the PackagePath of "build/" is also important.-->
        <None Include="ProjectC.props" Pack="True" PackagePath="build/" />
        <None Include="ProjectC.targets" Pack="True" PackagePath="build/" />
    </ItemGroup>
</Project>
```

To make this example work we will also need the following, `/ProjectC/AssemblyInfo.cs`:
```cs
// No code; comment is here however for some file content.
```

For the `.props` and `.targets` we will use the following for demonstration purposes. Note that they must have the same file name (without extension) as the package.

`/ProjectC/ProjectC.props`:
```xml
<Project>
    <!-- Custom project settings -->
    <PropertyGroup>
        <GitRepositoryPath>$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildProjectDirectory), .gitignore))/</GitRepositoryPath>
    </PropertyGroup>

    <!-- MSBuild project settings -->
    <PropertyGroup>
        <OutputPath>$(GitRepositoryPath)bin/$(MSBuildProjectName)/$(Configuration)</OutputPath>
        <AppendTargetFrameworkToOutputPath>true</AppendTargetFrameworkToOutputPath>
        <AppendRuntimeIdentifierToOutputPath>true</AppendRuntimeIdentifierToOutputPath>
    </PropertyGroup>
</Project>
```

`/ProjectC/ProjectC.targets`:
```xml
<Project>
    <!-- Not used in this example -->
</Project>
```

To create the NuGet package (`.nupkg`) we simply run the following command in the same directory as the `.csproj`:  
```bash
dotnet pack
```

By this stage you should have a `/ProjectC/bin/Debug/ProjectC.1.0.0.nupkg` file.

### ProjectD (The C# project)

To test the install of the `.nupkg` in your C# project you could upload it, but that's very good for testing as it pollutes `nuget.org`. We could upload it to a personal feed like with `myget.org`. However for the purposes here we will do things locally:

```bash
nuget add "/ProjectC/bin/Debug/ProjectC.1.0.0.nupkg" -source "/NuGet/Packages"
```

This will setup the following structure:
```
/NuGet/Packages
  └─projectc
    └─1.0.0
      ├─projectc.1.0.0.nupkg
      └─<other files>
```

We can then create a NuGet config for the local source. This is necessary because we don't want to install packages from `nuget.org` but rather from our local disk.

`/ProjectD/nuget.config`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="local" value="/NuGet/Packages" />
  </packageSources>
</configuration>
```

Then you install the package locally for the C# project:
```bash
dotnet add package ProjectC -s "local"
```

Since our `.props` example is expecting a `.gitignore` file to exist, if your `/ProjectD/` directory does not have a `.gitignore` you can create it just for the purposes of the example.

`/.gitignore`:
```
bin/
```

If all is well, when you build the C# project (`dotnet build`) the output should be in a folder `bin/` beside the `.gitignore` file. This demonstrates that the MSBuild has changed behvaiour by adding the NuGet package. 

So how does this work? If we take a look at `/ProjectD/obj/ProjectD/csproj.nuget.g.props` which is generated when restoring the C# project we can see the following:

```xml
<ImportGroup Condition=" '$(ExcludeRestorePackageImports)' != 'true' ">
    <Import Project="$(NuGetPackageRoot)projectc/1.0.0/build/ProjectC.props" Condition="Exists('$(NuGetPackageRoot)projectc/1.0.0/build/ProjectC.props')" />
</ImportGroup>
```

This means that the problem we discussed in [part 1](2020-12-13-extending-msbuild-part-1.md) about `BaseIntermediateOutputPath` is still relevant to `.props` and `.targets` in NuGet packages. We can't easily 100% extend MSBuild for a C# project through NuGet packages as one may wish. So because of this, and [among other reasons](2020-12-18-dont-use-nuget-packages.md), I don't recommend distributing your `.props` and `.targets` via NuGet packages. Instead, I recommend Git submodules or simply coping the files over directly.




