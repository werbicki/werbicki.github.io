---
layout: post
title:  "ICU Building ICU using Visual Studio with Boost Versioned Library Layout"
date:   2018-11-22 12:00:00 -0700
categories: [C++]
tags: [ICU, Text, Visual Studio]
---
These instructions are for creating a proper out-of-source build of ICU which is not supported very well by the default Visual Studio Solution file and Project files. The goal is to create versioned library names compatible with Boost which ensures that multiple builds can co-exist in the same stage directory. Everything else depends on this premise.

Download the source .zip file to ensure that the correct line endings are used, and extract to the Libraries\icu4c\icu4c-<version>.shared directory.
Download the data .zip file, extract to the Libraries\icu4c\icu4c-<version>.shared\source directory, and overwrite any existing files.
Open the allinone.sln Solution file located in Libraries\icu4c\icu4c-<version>.shared\source\allinone.
Remove the *_uwp Projects.
* Go to the Solution Explorer, right-click on each Project, and select Remove.
Add the Boost Library Layout Property Sheet to all Projects.
* Locate and copy the latest version of the boost_library_layout.props file to the Libraries\icu4c\icu4c-<version>.shared\source\allinone directory.
* Go to the Property Manager, select all of the Property Sets, right-click on the selection, and select Add Existing Property Sheet… Browse to the boost_library_layout.props file in the Solution directory and select it.
* Expand the common Project and select and expand the first Property Set, right-click on Boost Library Layout and select Properties.
* User Macros, change SolutionVersion to 63.1
* User Macros, change SolutionStaticOrDynamic to Dynamic
* Go to the Solution Explorer, right-click on the Solution, and select Add -> Existing Item. Browse to the boost_library_layout.props file in the Solution directory and select it. This will add it in the Solution Items folder directly underneath the Solution.
* Restart Visual Studio.
Create a custom ICU Property Sheet for all Projects. This will include ICU global properties that might change for each Release or might be changed by a Developer depending on their build environment. This property page defaults the Build Variables for the Version as well as the out-of-source Build location.
* Go to the Property Manager, right-click on the common project, select Add New Project Property Sheet.. Select the name as icu.props, and the Location as the Solution Directory, Libraries\icu4c\icu4c-<version>.shared\source\allinone.
* Alternatively: Locate and copy the latest version of the icu.props file to the Libraries\icu4c\icu4c-<version>.shared\source\allinone directory.
* Select all of the Property Sets, right-click on the selection, and select Add Existing Property Sheet… Browse to the icu.props file in the Solution directory and select it.
* Expand the common Project and select and expand the first Property Set, right-click on Boost Library Layout and select Properties.
* General, change Output Directory to $(Icu_StageDir)\
* General, change Intermediate Directory to $(Icu_BuildDir)\
* User Macros, Add the following Macros, always selecting Set this macro as an environment variable in the build environment:
Name	Value
Icu_MajorVersion	$(_Icu_VersionFromFile)
Icu_Version	$(_Icu_VersionShortFromFile)
Icu_PackageMode	dll
Icu_PackageTargetPrefix	
Icu_PackageTargetExt	dll
Icu_BuildDir	$(Boost_BuildDirNoSlash)
Icu_StageDir	$(Boost_StageDirNoSlash)
Icu_StageLibDir	$(Boost_StageLibDirNoSlash)
* VC++ Directories, change Executable Directories to $(Icu_StageLibDir);$(ExecutablePath)
* VC++ Directories, change Include Directories to $(SolutionDir)..\..\include;$(SolutionDir)..\common;$(IncludePath)
* C/C++ -> General, change Debug Information Format to C7 compatible (/Z7)
* C/C++ -> Pre-processor, change Preprocessor Definitions, add U_TOPSRCDIR="$(OutDir.Replace('\', '\\'))"; to the end of the line.
* C/C++ -> Code Generation, change Runtime Library to Multi-threaded Debug DLL 
* C/C++ -> Browse Information, change Enable Browse Information to No
* Linker -> Advanced, change Target Machine to Not Set
* Go to the Solution Explorer, right-click on the Solution, and select Add -> Existing Item. Browse to the icu.props file in the Solution directory and select it. This will add it in the Solution Items folder directly underneath the Solution.
* Restart Visual Studio.
Retarget to the latest Windows SDK Version.
* On the Visual Studio menu select Project -> Retarget solution, select the newest Windows SDK Version, select all Projects and click OK.
Use Project References to control linking dependencies. Project Dependencies (the order that Projects are built in) are separate from Project References (which get automatically linked into projects).
* For each project, when a library (.dll/.lib) Project is listed as part of the build order, add it as a Reference as well.
* Right-click on the Project, and select Build Dependencies -> Project Dependencies…. Make a note of the Projects selected.
* Expand the Project, right-click on References, and select References…. Under Projects -> Solution, select the same Projects.
Go to the Solution Explorer, select all the Projects, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* General, change Output Directory to <inherit from parent or project defaults>
* General, change Intermediate Directory to <inherit from parent or project defaults>
Select all the Projects except for except for *_uwp, makedata*, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes (Note: if option is not available, try removing it, applying the changes, an error may appear and then you can select it):
* General, change Character Set to Use Unicode Character Set
* C/C++ -> General, change Debug Information Format to <inherit from parent or project defaults>
* C/C++ -> Code Generation, change Runtime Library to <inherit from parent or project defaults>
* C/C++ -> Precompiled Headers, change Precompiled Header to <inherit from parent or project defaults>
* C/C++ -> Precompiled Headers, change Precompiled Header Output File to <inherit from parent or project defaults>
* C/C++ -> Output Files, change ASM List Location to <inherit from parent or project defaults>
* C/C++ -> Output Files, change Object File Name to <inherit from parent or project defaults>
* C/C++ -> Output Files, change Program Database File Name to <inherit from parent or project defaults>
* C/C++ -> Browse Information, change Enable Browse Information to <inherit from parent or project defaults>
* Linker -> General, change Output File to <inherit from parent or project defaults>
* Linker -> General, change Additional Library Directories to <inherit from parent or project defaults>
* Linker -> Debugging, change Generate Program Database File to <inherit from parent or project defaults>
Select Configuration -> Release and Platform -> All Platforms. Make the following changes:
* C/C++ -> Code Generation, change Runtime Library to Multi-threaded DLL
Select Configuration -> All Configurations and Platform -> Win32. Make the following changes:
* Linker -> Advanced, change Target Machine to MachineX86
Select Configuration -> All Configurations and Platform -> x64. Make the following changes:
* Linker -> Advanced, change Target Machine to MachineX64
Individually select each Project in the list below, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* General, change Target Name to the target for each Project based on the Target Name in the following list:
Project	TargetName
common	$(Boost_LibPrefix)icuuc$(Boost_LibSuffix)
ctestfw	$(Boost_LibPrefix)icutest$(Boost_LibSuffix)
i18n	$(Boost_LibPrefix)icu$(ProjectName)$(Boost_LibSuffix)
io	$(Boost_LibPrefix)icu$(ProjectName)$(Boost_LibSuffix)
stubdata	$(Boost_LibPrefix)icudt$(Boost_LibSuffix)
testplug	$(Boost_LibPrefix)$(ProjectName)$(Boost_LibSuffix)
toolutil	$(Boost_LibPrefix)icutu$(Boost_LibSuffix)
Select all of the Projects in the list above, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* Linker -> Input, change Additional Dependencies to %(AdditionalDependencies)
* Linker -> Advanced, change Import Library to <inherit from parent or project defaults>
Select the Projects derb, gen*, icuinfo, icupkg, makeconv, pkgdata, and uconv, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* Linker -> Input, change Additional Dependencies to <inherit from parent or project defaults>
* Custom Build Step -> General, remove Command Line
* Custom Build Step -> General, remove Outputs
Select the Projects cal, cintltst, date, intltst, and iotest, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* Linker -> Input, change Additional Dependencies to <inherit from parent or project defaults>
Select the Project uconv, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* Linker -> Input, change Additional Dependencies to $(Icu_StageLibDir)\libconvmsg$(Boost_LibSuffix).lib;%(AdditionalDependencies)
Expand the uconv Project -> Build Scripts, select makefile.mak, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* Custom Build Tool -> General, change Command Line to "$(VC_PGO_RunTime_Dir)\NMAKE" /nologo /f %(Filename).mak CFG="$(Configuration)" ICUP="$(SolutionDir)..\.." ICUOUT="$(Icu_BuildDir)" ICUTOOLS="$(Icu_StageDir)" DLL_OUTPUT="$(Icu_StageLibDir)" LIB_TARGET="libconvmsg$(Boost_LibSuffix)"
* Custom Build Tool -> General, change Outputs to $(Icu_StageLibDir)\libconvmsg$(Boost_LibSuffix).lib;%(Outputs)
Modify the source/extra/uconv/Makefile.mak files for the uconv project.
* Add LIB_TARGET=$(RESNAME), change RESNAME to LIB_TARGET when referencing the output, and for the pkgdata command line use -p “$(LIB_TARGET)” to set the versioned data package name, -e $(RESNAME) to get the proper entry point, and -L “$(LIB_TARGET)” to specify a different library name from the package name.
Select the Projects makedata, and makedata_uwp, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* General, change Build Log File to <inherit from parent or project defaults>
Select the Proejct makedata project, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* NMake -> Build Command Line, change to "$(VC_PGO_RunTime_Dir)\NMAKE" /nologo /f makedata.mak CFG="$(PlatformShortName)\$(Configuration)" ICUP="$(SolutionDir)..\.." ICUOUT="$(Icu_StageDir)\data" ICUTOOLS="$(Icu_StageDir)" ICUPBIN="$(Icu_StageDir)" ICUMAKE="$(ProjectDir)\" CFGTOOLS=".." ICUSRCDATA="$(SolutionDir)..\..\source\data" ICUSRCDATA_RELATIVE_PATH="$(SolutionDir)..\..\source\data" ICUSRCSTUBDATA="$(SolutionDir)..\..\source\stubdata" ICUSRCTEST="$(SolutionDir)..\..\source\test" TESTDATAOUT="$(Icu_StageDir)\test\testdata\out" TESTDATABLD="$(Icu_BuildDir)\testdata" ICU_PACKAGE_MODE="-m $(Icu_PackageMode)" U_ICUDATA_NAME="icudt$(Icu_MajorVersion)" DLL_OUTPUT="$(Icu_StageLibDir)" ICU_LIB_TARGET="$(Icu_StageLibDir)\$(Icu_PackageTargetPrefix)icudt$(Boost_LibSuffix).$(Icu_PackageTargetExt)"
* NMake -> Rebuild Command Line, change to the same as above but with the addition of CLEAN ALL
* NMake -> Clean Command Line, change to the same as above but with the addition of CLEAN
Modify the source/data/Makefile.mak, source/test/testdata/testdata.mak for the makedata project. Advanced modifications are required to ensure that the build copies the correct files and performs a complete build of the ICU Data and test data. The NMake parameter changes above are enough to fully support these makefile changes.
In Visual Studio, use Batch -> Batch Build... to build all Configurations for all Platforms.
Building ICU as Static Library using Visual Studio with Boost Versioned Library Layout
Having restructured the Visual Studio solution using the method described in Building ICU using Visual Studio with Boost Version, it is simple to switch it over to compile and use as a Static Library.
Duplicate the icu4c\icu4c-<version>.shared directory to icu4c\icu4c-<version>.
Changes need to be made to the allinone.sln Solution file located in Libraries\icu4c\icu4c-<version>\source\allinone.
Modify the Boost Library Layout Property Sheet to all Projects.
* Go to the Property Manager, expand the common Project and select and expand the first Property Set, right-click on Boost Library Layout and select Properties.
* User Macros, change SolutionStaticOrDynamic to Static
Modify the custom ICU Property Sheet for all Projects.
* Go to the Property Manager, expand the common Project and select and expand the first Property Set, right-click on icu and select Properties.
* User Macros, change the following:
Name	Value
Icu_PackageMode	static
Icu_PackageTargetPrefix	$(Boost_LibPrefix)
Icu_PackageTargetExt	lib
Select all of the Projects in the list above, right-click and select Properties. Select Configuration -> All Configurations and Platform -> All Platforms. Make the following changes:
* General, change Configuration Type to Static library (.lib)
* C/C++ -> Pre-processor, change Preprocessor Definitions, add U_STATIC_IMPLEMENTATION; to the beginning of the line.
* Librarian -> General, change Ignore All Default Libraries to Yes. Note: Linker changes to Librarian once the General -> Configuration Type is changed to Static library (.lib) and the changes are applied.
In Visual Studio, use Batch -> Batch Build... to build all Configurations for all Platforms.
