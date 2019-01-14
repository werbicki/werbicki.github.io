---
layout: post
title:  "Visual Studio MSBuild support for Boost Versioned Library Layout"
date:   2018-10-08 12:00:00 -0700
categories: [C++]
tags: [Visual Studio, MSBuild, Boost]
---
MSBuild provides a flexible way to programatically discover build variables and apply them to your Visual Studio build. The Boost libraries use the Boost.Build system, a variant of Jam to perform the build and part of that system allows for versioned library names - a very powerful naming convention for libraries! Replicating this within Visual Studio allows for multiple builds to co-exist within the same OutDir making library management much easier.

Boost uses the parameter layout to determine how to produce library names using Boost.Build.

```
--layout=<layout> Determines whether to choose library names
                  and header locations such that multiple
                  versions of Boost or multiple compilers can
                  be used on the same system.
```

| Layout | Description |
| versioned | Names of boost binaries include the Boost version number, name and version of the compiler and encoded build properties.  Boost headers are installed in a subdirectory of <HDRDIR> whose name contains the Boost version number |
| tagged | Names of boost binaries include the encoded build properties such as variant and threading, but do not including compiler name and version, or Boost version. This option is useful if you build several variants of Boost, using the same compiler |
| system | Binaries names do not include the Boost version number or the name and version number of the compiler.  Boost headers are installed directly into <HDRDIR>.  This option is intended for system integrators who are building distribution packages |

The default value is 'versioned' on Windows, and 'system' on Unix.

Using MSBuild it is possible to replicate this functionality to allow for versioned, tagged or system library names when compiling using Visual Studio. Using the following file, copy it into your Solution folder as boost_library_layout.props, and add it to your Project.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ImportGroup Label="PropertySheets" />
  <PropertyGroup>
    <_ConfigurationAndPlatformRegex><![CDATA[<ItemDefinitionGroup Condition="'\$\(Configuration\)\|\$\(Platform\)'==']]>$(Configuration)\|$(Platform)<![CDATA['[^"]*"(?:.*\n)*?.*<ClCompile>(?:.*\n)*?.*<RuntimeLibrary>(.*)</RuntimeLibrary>(?:.*\n)*?.*</ClCompile>(?:.*\n)*?.*</ItemDefinitionGroup>]]></_ConfigurationAndPlatformRegex>
    <_ConfigurationRegex><![CDATA[<ItemDefinitionGroup Condition="'\$\(Configuration\)'==']]>$(Configuration)<![CDATA['[^"]*"(?:.*\n)*?.*<ClCompile>(?:.*\n)*?.*<RuntimeLibrary>(.*)</RuntimeLibrary>(?:.*\n)*?.*</ClCompile>(?:.*\n)*?.*</ItemDefinitionGroup>]]></_ConfigurationRegex>
    <_IsDebug>$([System.Text.RegularExpressions.Regex]::IsMatch($(Configuration),'[Dd]ebug'))</_IsDebug>
    <_HasRuntimeLibraryFromConfigurationAndPlatform>$([System.Text.RegularExpressions.Regex]::Match($([System.IO.File]::ReadAllText($(MSBuildProjectFullPath))), $(_ConfigurationAndPlatformRegex)).Success)</_HasRuntimeLibraryFromConfigurationAndPlatform>
    <_HasRuntimeLibraryFromConfiguration>$([System.Text.RegularExpressions.Regex]::Match($([System.IO.File]::ReadAllText($(MSBuildProjectFullPath))), $(_ConfigurationRegex)).Success)</_HasRuntimeLibraryFromConfiguration>
    <RuntimeLibrary Condition="$(_HasRuntimeLibraryFromConfigurationAndPlatform)">$([System.Text.RegularExpressions.Regex]::Match($([System.IO.File]::ReadAllText($(MSBuildProjectFullPath))), $(_ConfigurationAndPlatformRegex)).Result('$1'))</RuntimeLibrary>
    <RuntimeLibrary Condition="$(_HasRuntimeLibraryFromConfiguration)">$([System.Text.RegularExpressions.Regex]::Match($([System.IO.File]::ReadAllText($(MSBuildProjectFullPath))), $(_ConfigurationRegex)).Result('$1'))</RuntimeLibrary>
    <RuntimeLibrary Condition="!$(_HasRuntimeLibraryFromConfigurationAndPlatform) And !$(_HasRuntimeLibraryFromConfiguration) And $(_IsDebug)">MultiThreadedDebugDLL</RuntimeLibrary>
    <RuntimeLibrary Condition="!$(_HasRuntimeLibraryFromConfigurationAndPlatform) And !$(_HasRuntimeLibraryFromConfiguration) And !$(_IsDebug)">MultiThreadedDLL</RuntimeLibrary>
    <_IsMultiThreaded>$([System.Text.RegularExpressions.Regex]::IsMatch($(RuntimeLibrary),'MultiThreaded.+'))</_IsMultiThreaded>
    <!-- 
        Fix incremental build (Different results when running msbuild within Visual Studio or from console).
        http://connect.microsoft.com/VisualStudio/feedback/details/677499/different-results-when-running-msbuild-within-visual-studio-or-from-console 
    -->
    <DisableFastUpToDateCheck>true</DisableFastUpToDateCheck>
  </PropertyGroup>
  <PropertyGroup Label="UserMacros">
    <SolutionVersion>1.0</SolutionVersion>
    <SolutionStaticOrDynamic></SolutionStaticOrDynamic>
    <!-- Boost Directories -->
    <Boost_BuildRootNoSlash>$([MSBuild]::GetDirectoryNameOfFileAbove($(MSBuildThisFileDirectory), build.root))</Boost_BuildRoot>
    <Boost_BuildRoot>$(BuildRootNoSlash)\</Boost_BuildRoot>
    <Boost_RootDirNoSlash Condition="'$(BuildRootNoSlash)'!=''">$(BuildRoot)</Boost_RootDir>
    <Boost_RootDirNoSlash Condition="'$(BuildRootNoSlash)'==''">$(SolutionDir)..</Boost_RootDir>
    <Boost_RootDir>$(RootDirNoSlash)\</Boost_RootDir>
    <Boost_OutDirSuffix Condition="'$(SolutionStaticOrDynamic.ToLower())'=='dynamic'">.shared</Boost_OutDirSuffix>
    <Boost_OutDirSuffix Condition="'$(SolutionStaticOrDynamic.ToLower())'!='dynamic' AND '$(ConfigurationType)'=='DynamicLibrary'">.shared</Boost_OutDirSuffix>
    <Boost_OutDirSuffix Condition="'$(SolutionStaticOrDynamic.ToLower())'!='dynamic' AND '$(ConfigurationType)'!='DynamicLibrary'">
    </Boost_OutDirSuffix>
    <Boost_BuildDirNoSlash>$(RootDir)_build\$(OS)$(Boost_OutDirSuffix)\$(SolutionName)\$(ProjectName)\$(Configuration)$(Boost_ArchAndModel)</Boost_BuildDirNoSlash>
    <Boost_BuildDir>$(Boost_BuildDirNoSlash)\</Boost_BuildDir>
    <Boost_StageLibDirNoSlash>$(RootDir)_stage\$(OS)$(Boost_OutDirSuffix)\lib</Boost_StageLibDirNoSlash>
    <Boost_StageLibDir>$(Boost_StageLibDirNoSlash)\</Boost_StageLibDir>
    <Boost_StageDirNoSlash Condition="'$(ConfigurationType)'=='StaticLibrary' OR '$(ConfigurationType)'=='DynamicLibrary'">$(Boost_StageLibDirNoSlash)</Boost_StageDirNoSlash>
    <Boost_StageDirNoSlash Condition="'$(ConfigurationType)'!='StaticLibrary' AND '$(ConfigurationType)'!='DynamicLibrary'">$(RootDir)_stage\$(OS)$(Boost_OutDirSuffix)\$(Configuration)$(Boost_ArchAndModel)</Boost_StageDirNoSlash>
    <Boost_StageDir>$(Boost_StageDirNoSlash)\</Boost_StageDir>
    <!-- Boost Targets -->
    <Boost_Layout>Versioned</Boost_Layout>
    <Boost_LibPrefix Condition="'$(ConfigurationType)'=='StaticLibrary'">lib</Boost_LibPrefix>
    <Boost_LibPrefix Condition="'$(ConfigurationType)'!='StaticLibrary'"></Boost_LibPrefix>
    <Boost_Toolset>-vc$(PlatformToolsetVersion)</Boost_Toolset>
    <Boost_Threading Condition="$(_IsMultiThreaded)">-mt</Boost_Threading>
    <Boost_RtStaticOrDynamic Condition="'$(RuntimeLibrary)'=='MultiThreaded' OR '$(RuntimeLibrary)'=='MultiThreadedDebug'">s</Boost_RtStaticOrDynamic>
    <Boost_RtDebugOrRelease Condition="'$(RuntimeLibrary)'=='MultiThreadedDebug' OR '$(RuntimeLibrary)'=='MultiThreadedDebugDLL'">g</Boost_RtDebugOrRelease>
    <Boost_DebugOrRelease Condition="$(_IsDebug)">d</Boost_DebugOrRelease>
    <Boost_Runtime Condition="'$(Boost_RtStaticOrDynamic)$(Boost_RtDebugOrRelease)$(Boost_DebugOrRelease)'!=''">-$(Boost_RtStaticOrDynamic)$(Boost_RtDebugOrRelease)$(Boost_DebugOrRelease)</Boost_Runtime>
    <Boost_ArchAndModel>-$(PlatformShortName)</Boost_ArchAndModel>
    <Boost_BuildProps Condition="'$(Boost_Layout.ToLower())'=='versioned'">$(Boost_Toolset)$(Boost_Threading)$(Boost_Runtime)$(Boost_ArchAndModel)</Boost_BuildProps>
    <Boost_BuildProps Condition="'$(Boost_Layout.ToLower())'=='tagged'">$(Boost_Threading)$(Boost_Runtime)$(Boost_ArchAndModel)</Boost_BuildProps>
    <Boost_BuildProps Condition="'$(Boost_Layout.ToLower())'!='versioned' AND '$(Boost_Layout.ToLower())'!='tagged'"></Boost_BuildProps>
    <Boost_LibSuffix Condition="'$(SolutionVersion)'!='' AND '$(Boost_Layout.ToLower())'=='versioned'">$(Boost_BuildProps)-$(SolutionVersion)</Boost_LibSuffix>
    <Boost_LibSuffix Condition="'$(SolutionVersion)'=='' OR '$(Boost_Layout.ToLower())'!='versioned'">$(Boost_BuildProps)</Boost_LibSuffix>
    <Boost_TargetName>$(Boost_LibPrefix)$(ProjectName)$(Boost_LibSuffix)</Boost_TargetName>
  </PropertyGroup>
  <PropertyGroup>
    <_PropertySheetDisplayName>Boost Library Layout</_PropertySheetDisplayName>
  </PropertyGroup>
  <ItemDefinitionGroup />
  <ItemGroup>
    <BuildMacro Include="SolutionVersion">
      <Value>$(SolutionVersion)</Value>
      <EnvironmentVariable>true</EnvironmentVariable>
    </BuildMacro>
    <BuildMacro Include="SolutionStaticOrDynamic">
      <Value>$(SolutionStaticOrDynamic)</Value>
      <EnvironmentVariable>true</EnvironmentVariable>
    </BuildMacro>
    <BuildMacro Include="Boost_RootDir">
      <Value>$(Boost_RootDir)</Value>
      <EnvironmentVariable>true</EnvironmentVariable>
    </BuildMacro>
    <BuildMacro Include="Boost_Layout">
      <Value>$(Boost_Layout)</Value>
      <EnvironmentVariable>true</EnvironmentVariable>
    </BuildMacro>
  </ItemGroup>
</Project>
```

In your Project, change Target Name to $(Boost_LibPrefix)icuuc$(Boost_LibSuffix).
