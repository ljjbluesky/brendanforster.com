---
title: Investigating and Resolving a Complex .NET Core Bug
date: 2020-02-02
summary: A journey I went on to investigate and assist with resolving a .NET Core bug that affected one of the projects I maintain.
---

**Note:** as you'll discover on this post, the word "bug" as used in the title
does not really cover what happened here. I settled on this because "regression
as a result of addressing a different regression" doesn't roll of the tongue as
well.

It's been a while since I looked at .NET Core, and a couple of weeks ago I found
myself dusting off Octokit.net and looking into a wishlist of tasks:

 - introduce `Microsoft.NETFramework.ReferenceAssemblies` so we can build projects
   that target .NET Framework without requiring .NET Framework or Mono
 - remove the requirement for .NET Framework or Mono to to be installed
 - strip out the customizations to the project files
 - move the entire build process over to GitHub Actions

This is a work-in-progress, and I had to pause it to dig into a strange
behaviour that I uncovered when I tried to upgrade the continuous integration
scripts to use .NET Core 3.1. It took a fair while to wrap my head around what
was happening, because it was involving two things that I believed to be stable:

 - .NET Core 3.1 (the latest release of .NET Core and is an Long-Term Support release)
 - [Sourcelink](https://github.com/ctaggart/SourceLink/) - a command line tool
   verifying that the debugging information embedded in assemblies is correct

### The Problem

As part of the Octokit project, we verify the debugging information on each
build:

```
========================================
TestSourceLink
========================================
Executing task: TestSourceLink
Testing sourcelink info in packaging/Octokit.0.37.0-beta0023.nupkg
sourcelink test passed: lib/net45/Octokit.dll
sourcelink test passed: lib/net46/Octokit.dll
sourcelink test passed: lib/netstandard1.1/Octokit.dll
sourcelink test passed: lib/netstandard2.0/Octokit.dll
Testing sourcelink info in packaging/Octokit.Reactive.0.37.0-beta0023.nupkg
sourcelink test passed: lib/net45/Octokit.Reactive.dll
sourcelink test passed: lib/net46/Octokit.Reactive.dll
sourcelink test passed: lib/netstandard1.1/Octokit.Reactive.dll
sourcelink test passed: lib/netstandard2.0/Octokit.Reactive.dll
Finished executing task: TestSourceLink
========================================
```

As soon as I upgraded the tooling to use .NET Core 3.1, this step would fail
with a complex error:

```
========================================
TestSourceLink
========================================
Executing task: TestSourceLink
Testing sourcelink info in packaging/Octokit.0.37.0-netcore-3-1-0024.nupkg
1 Documents without URLs:
bdbf6eb3afe9056e474d2ca2bec98a866c17b8a66405d1463fc9e8b8a832a65c sha256 csharp C:\Users\appveyor\AppData\Local\Temp\1\.NETFramework,Version=v4.5.AssemblyAttributes.cs
1 Documents with errors:
0233caf8ef78de276241dcb9cbc6a50a9ce9524127b181de0170ca837acd6a11 sha256 csharp C:\projects\octokit-net\Octokit\obj\Release\net45\Octokit.AssemblyInfo.cs
https://raw.githubusercontent.com/octokit/octokit.net/bc5d72819434bd8050311d43a2f434e2e0585194/Octokit/obj/Release/net45/Octokit.AssemblyInfo.cs
error: url failed NotFound: Not Found
sourcelink test failed
failed for lib/net45/Octokit.dll
1 Documents without URLs:
50ddf2a6dd2a7d54b76e5119ea0685236fb6e703d82dc5a35d5d3683504f8f0d sha256 csharp C:\Users\appveyor\AppData\Local\Temp\1\.NETFramework,Version=v4.6.AssemblyAttributes.cs
1 Documents with errors:
0233caf8ef78de276241dcb9cbc6a50a9ce9524127b181de0170ca837acd6a11 sha256 csharp C:\projects\octokit-net\Octokit\obj\Release\net46\Octokit.AssemblyInfo.cs
https://raw.githubusercontent.com/octokit/octokit.net/bc5d72819434bd8050311d43a2f434e2e0585194/Octokit/obj/Release/net46/Octokit.AssemblyInfo.cs
error: url failed NotFound: Not Found
sourcelink test failed
failed for lib/net46/Octokit.dll
1 Documents without URLs:
e66dc6241c8fb7ec2a5a1179a4ba4587229ba0c0cf8328f52e061b610c675915 sha256 csharp C:\Users\appveyor\AppData\Local\Temp\1\.NETStandard,Version=v1.1.AssemblyAttributes.cs
1 Documents with errors:
0233caf8ef78de276241dcb9cbc6a50a9ce9524127b181de0170ca837acd6a11 sha256 csharp C:\projects\octokit-net\Octokit\obj\Release\netstandard1.1\Octokit.AssemblyInfo.cs
https://raw.githubusercontent.com/octokit/octokit.net/bc5d72819434bd8050311d43a2f434e2e0585194/Octokit/obj/Release/netstandard1.1/Octokit.AssemblyInfo.cs
error: url failed NotFound: Not Found
sourcelink test failed
failed for lib/netstandard1.1/Octokit.dll
1 Documents without URLs:
a6e03ae4df13fe05345e9022d1f1cd24ecae4bfd66db4843697c855d9f9335f4 sha256 csharp C:\Users\appveyor\AppData\Local\Temp\1\.NETStandard,Version=v2.0.AssemblyAttributes.cs
1 Documents with errors:
0233caf8ef78de276241dcb9cbc6a50a9ce9524127b181de0170ca837acd6a11 sha256 csharp C:\projects\octokit-net\Octokit\obj\Release\netstandard2.0\Octokit.AssemblyInfo.cs
https://raw.githubusercontent.com/octokit/octokit.net/bc5d72819434bd8050311d43a2f434e2e0585194/Octokit/obj/Release/netstandard2.0/Octokit.AssemblyInfo.cs
error: url failed NotFound: Not Found
sourcelink test failed
failed for lib/netstandard2.0/Octokit.dll
4 files did not pass in C:/projects/octokit-net/packaging/Octokit.0.37.0-netcore-3-1-0024.nupkg
```

I had several questions about this:

 - The path for this file withing `obj` - is this actually a generated file?
 - This `[platform].AssemblyAttributes.cs` file looks to be generated for each 
   platform - where is this coming from?
 - Sourcelink is looking for thsi file on GitHub - but it's not in version
   control. Can we tell it to not do that? Or just to disable it?

### Reproducing the problem

To sanity check the issue, I reproduced the issue outside of the Octokit
project. I was able to reproduce this using these components:

 - a project file, e.g. `repro.csproj`:

 ```xml
 <Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>netstandard2.0</TargetFrameworks>
    <RootNamespace>dotnetcore_sourcelink_repro</RootNamespace>
    <DebugType>embedded</DebugType>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.SourceLink.GitHub" Version="1.0.0" PrivateAssets="All"/>
  </ItemGroup>

</Project>
 ```

 - a `.config/dotnet-tools.json` file that installs the `sourcelink` command line tool

 ```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "sourcelink": {
      "version": "3.1.1",
      "commands": [
        "sourcelink"
      ]
    }
  }
}
 ```

 - a `global.json` file to pin the project to a specific version of the .NET SDK

```json
{
  "sdk": {
    "version": "3.1.101"
  }
}
```

With those three pieces in place, you can reproduce the issue in the same
manner:

```shellsession
dotnet tool restore
dotnet build -c Release
dotnet sourcelink test bin/Release/netstandard2.0/repro.dll
```
 