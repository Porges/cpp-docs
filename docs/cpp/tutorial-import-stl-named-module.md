---
title: "Tutorial: Import the standard library (STL) using modules (C++)"
ms.date: 11/18/2022
ms.topic: "tutorial"
author: "tylermsft"
ms.author: "twhitney"
helpviewer_keywords: ["modules [C++]", "modules [C++]", "named modules [C++], import standard library (STL) using named modules"]
description: Learn how to import the C++ standard library (STL) using modules
---
# Tutorial: Import the C++ standard library using modules

Learn how to import the C++ standard library using C++ library modules. Which is significantly faster to compile and more robust than using header files or header units or precompiled headers (PCH).

In this tutorial, you'll learn about:

- How to import the standard library as a module from the command line.
- The performance and usability benefits of modules
- The two standard library modules `std` and `std.compat` and the difference between them

## Prerequisites

This tutorial requires Visual Studio 2022 17.5 or later.

> [!WARNING]
> This tutorial is for a preview feature. The feature is subject to change between preview releases. You shouldn't use preview features in production code.

## An introduction to the standard library modules

Header files suffer from problems caused by dependencies, semantics that can change depending on macro definitions, semantics that can change depending on the order you include header files, and slow compilation. Modules solve these problems. It's now possible to import the standard library as a module instead of as a tangle of header files. This is significantly faster and more robust than including header files.

The C++23 standard library introduces two named modules: `std` and `std.compat`.

- `std` exports the declarations and names defined in the C++ standard library that are in namespace `std` such as `std::vector` and `std::sort`. It also exports the contents of C wrapper headers such as `<cmath>`, `<cstdio>`, `<cstdlib>` that provide `std::byte`, `std::printf()`, and so on. The C functions defined in the *global namespace* such as `::printf()` and `::fopen` aren't exported. This improves the situation where previously including a C wrapper header like `<cstdio>` which provides `std::` qualified versions of the C runtime functions *also* included the actual C header, like `stdio.h`, which meant the C global namespace versions were also brought in. This is no longer a problem if you import `std`.
- `std.compat` exports everything in `std`, and adds the global namespace counterparts of the C runtime such as `::printf`, `::fopen`, `::size_t`, `::strlen`, and so on. This module makes it easier when working with a codebase that refers to many C runtime functions/types in the global namespace.

The compiler imports the entire standard library when you use `import std;` or `import std.compat;` and does it faster than bringing in a single header file. That is, it's faster to bring in the entire standard library with `import std;` (or `import std.compat`) than it is to `#include <vector>`, for example.

Because named modules don't expose macros, macros like `assert`, `errno`, `offsetof`, `va_arg`, and others aren't available when you import `std` or `std.compat`. See [Standard library named module considerations](#standard-library-named-module-considerations) for workarounds.

## A little about C++ modules

Header files are how declarations and definitions have been shared between source files in C++. Prior to standard library modules, you would include the part of the standard library you needed with a directive like `#include <vector>`. Header files are fragile and difficult to compose because their semantics may change depending on the order you include them, or whether certain macros are or aren't defined. They also slow compilation because they have to be reprocessed by every source file that includes them.

C++20 introduces a modern alternative called *modules*. In C++23, we were able to capitalize on module support to introduce named modules for the standard library.

Like header files, modules allow you to share declarations and definitions across source files. But unlike header files, modules aren't fragile and are easier to compose because their semantics don't change due to macro definitions or the order in which they're imported. The compiler can process modules significantly faster than it can process `#include` files, and uses less memory at compile time as well. Macro definitions defined in a named module aren't exposed, nor are private implementation details.

For more information about modules, see [Overview of modules in C++](modules-cpp.md) That article also discusses consuming the C++ standard library as modules, but uses an older and experimental way of doing it. This article demonstrates the new and best way to consume the standard library. For more information about alternative ways to consume the standard library, see [Different ways to consume the standard library](#different-ways-to-consume-the-standard-library) later in this article.

## Import the standard library with `std`

The following demonstrates how to consume the standard library as a module using the command line compiler. The Visual Studio IDE experience for importing the standard library as a module is still being implemented.

The standard library is imported into your application with the statement `import std;` or `import std.compat;`. But first, you must compile the standard library named modules. The following steps demonstrate how.

### Example: `std`

1. Open a x86 Native Tools Command Prompt for VS: from the Windows **Start** menu, type *x86 native* and the prompt should appear in the list of apps. Ensure that the prompt is for Visual Studio 2022 preview version 17.5 or above. You'll get errors if you use the wrong version of the prompt. The examples used in this tutorial are for the CMD shell. If you use PowerShell, substitute `"$Env:VCToolsInstallDir\modules\std.ixx"` for `"%VCToolsInstallDir\modules\std.ixx"`.
1. Create a directory such as `C:\STLModules`, and make it the current directory.
1. Compile the `std` named module with the following command:

    ```dos
    cl /std:c++latest /EHsc /nologo /W4 /MTd /c "%VCToolsInstallDir%\modules\std.ixx"
    ```

    If you get errors, ensure that you're using the correct version of the command prompt. If you're still having issues, please file a bug at [Visual Studio Developer Community](https://developercommunity.visualstudio.com/home).

    You should compile the `std` named module using the same compiler settings that you intend to use with the code that imports it. If you have a multi-project solution, you can compile the standard library named module once, and then refer to it from all of your projects by using the [`/reference`](../build/reference/module-reference.md) compiler option.

    Using the compiler command above, the compiler outputs two files:
    - `std.ifc` is the compiled binary representation of the named module interface that the compiler consults to process the `import std;` statement. This is a compile-time only artifact. It doesn't ship with your application.
    - `std.obj` contains the implementation of the named module. Add `std.obj` to the command line when you compile the sample app so that functionality you use from the standard library can be statically linked into your application.

    The key command-line switches in this example are:

    | Switch | Meaning |
    |---|---|
    | [`/std:c++:latest`](../build/reference/std-specify-language-standard-version.md) | Use the latest version of the C++ language standard and library. Although module support is available under `/std:c++20`, you need the latest standard library to get support for standard library named modules. |
    | [`/EHsc`](../build/reference/eh-exception-handling-model.md) | Use C++ exception handling, except for functions marked `extern "C"`. |
    | [`/MTd`](../build/reference/md-mt-ld-use-run-time-library.md) | Use the static multithreaded debug run-time library. |
    | [`/c`](../build/reference/c-compile-without-linking.md) | Compile without linking. |

1. To try out importing the `std` library, start by creating a file named `importExample.cpp` with the following content:

    ```cpp
    // requires /std:c++latest
    
    import std;
    
    int main()
    {
        std::cout << "Import the STL library for best performance\n";
        std::vector<int> v{5, 5, 5};
        for (const auto& e : v)
        {
            std::cout << e;
        }
    }
    ```

    In the preceding code, `import std;` replaces `#include <vector>` and `#include <iostream>`. The statement `import std;` makes all of the standard library available with one statement. If you're concerned about performance, it may help to know that importing the entire standard library is often significantly faster than processing a single standard library header file such as `#include <vector>`.

1. Compile the example by using the following command in the same directory as the previous step:

    ```dos
    cl /std:c++latest /EHsc /nologo /W4 /MTd importExample.cpp std.obj
    ```

    We didn't have to specify `std.ifc` on the command line because the compiler automatically looks for the `.ifc` file that matches the module name specified by an `import` statement. When the compiler encounters `import std;`, it finds `std.ifc` since we put it in the same directory as the source code--relieving us of the need to specify it on the command line. If the `.ifc` file is in a different directory than the source code, use the [`/reference`](../build/reference/module-reference.md) compiler switch to refer to it.

    If you're building a single project, you can combine the steps of building the `std` standard library named module and the step of building your application by adding `"%VCToolsInstallDir%\modules\std.ixx"` to the command line. Make sure to put it before any `.cpp` files that consume it. By default, the output executable's name is taken from the first input file. Use the `/Fe` compiler option to specify the output file name you want. This tutorial shows compiling the `std` named module as a separate step because you only need to build the standard library named module once, and then you can refer to it from your project, or from multiple projects. But it may be convenient to build everything together, as shown by this command line:

    ```dos
    cl /FeimportExample /std:c++latest /EHsc /nologo /W4 /MTd "%VCToolsInstallDir%\modules\std.ixx" importExample.cpp
    ```

    Given the previous command line, the compiler produces an executable named `importExample.exe`. When you run it, it produces the following output:

    ```output
    Import the STL library for best performance
    555
    ```

    The key command-line switches in this example are the same as the previous example, except that the `/c` switch isn't used.

## Import the standard library and global C functions with `std.compat`

The C++ standard library includes the ISO C standard library. The `std.compat` module provides all of the functionality of the `std` module like `std::vector`, `std::cout`, `std::printf`, `std::scanf`, and so on. But it also provides the global namespace versions of these functions such as `::printf`, `::scanf`, `::fopoen`, `::size_t`, and so on.

The `std.compat` named module is a compatibility layer provided to ease migrating existing code that refers to C runtime functions in the global namespace. If you want to avoid adding names to the global namespace, use `import std;`. If you need to ease migrating a codebase that uses many unqualified (that is, global namespace) C runtime functions, use `import std.compat;`. This provides the global namespace C runtime names so that you don't have to qualify all the global name mentions with `std::`. If you don't have any existing code that uses the global namespace C runtime functions, then you don't need to use `import std.compat;`. If you only call a few C runtime functions in your code, it may be better to use `import std;` and qualify the few global namespace C runtime names that need it with `std::`, for example, `std::printf()`. You'll know that you should consider using `import std.compat;` if you see an error like `error C3861: 'printf': identifier not found` when you try to compile your code.

### Example: `std.compat`

Before you can use `import std.compat;` in your code, you must compile the named module `std.compat.ixx`. The steps are similar to for building the `std` named module. The steps include building the `std` named module first because `std.compat` depends on it:

1. Open a Native Tools Command Prompt for VS: from the Windows **Start** menu, type *x86 native* and the prompt should appear in the list of apps. Ensure that the prompt is for Visual Studio 2022 preview version 17.5 or above. If you use the wrong version of the prompt, you'll get compiler errors.
1. Create a directory to try this example, such as `C:\STLCompatModules`, and make it the current directory.
1. Compile the `std` and `std.compat` named modules with the following command:

    ```dos
    cl /std:c++latest /EHsc /nologo /W4 /MTd /c "%VCToolsInstallDir%\modules\std.ixx" "%VCToolsInstallDir%\modules\std.compat.ixx"
    ```

    You should compile `std` and `std.compat` using the same compiler settings that you intend to use with the code that imports them. If you have a multi-project solution, you can compile them once, and then refer to them from all of your projects using the [`/reference`](../build/reference/module-reference.md) compiler option.

    If you get errors, ensure that you're using the correct version of the command prompt. If you're still having issues, please file a bug at [Visual Studio Developer Community](https://developercommunity.visualstudio.com/home). While this feature is still in preview, you can find a list of known problems under [Standard library header units and modules tracking issue 1694](https://github.com/microsoft/stl/issues/1694).

    The compiler outputs four files from the previous two steps:
    - `std.ifc` is the compiled binary named module interface that the compiler consults to process the `import std;` statement. The compiler also consults `std.ifc` to process `import std.compat;` because `std.compat` builds on `std`. This is a compile-time only artifact. It doesn't ship with your application.
    - `std.obj` contains the implementation of the standard library.
    - `std.compat.ifc` is the compiled binary named module interface that the compiler consults to process the `import std.compat;` statement. This is a compile-time only artifact. It doesn't ship with your application.
    - `std.compat.obj` contains implementation. However, most of the implementation is provided by `std.obj`. Add `std.obj` to the command line as well when you compile the sample app so that the functionality you use from the standard library can be statically linked into your application.

1. To try out importing the `std.compat` library, create a file named `stdCompatExample.cpp` with the following content:

    ```cpp
    import std.compat;
    
    int main()
    {
        printf("Import std.compat to get global names like printf()\n");
        
        std::vector<int> v{5, 5, 5};
        for (const auto& e : v)
        {
            printf("%i", e);
        }
    }
    ```

    In the preceding code, `import std.compat;` replaces `#include <cstdio>` and `#include <vector>`. The statement `import std.compat;` makes the standard library and C runtime functions available with one statement. If you're concerned about performance, the performance of modules is such that importing this named module, which includes the C++ standard library and C runtime library global namespace functions, is faster than even processing a single include like `#include <vector>`.

1. Compile the example by using the following command:

    ```dos
    cl /std:c++latest /EHsc /nologo /W4 /MTd stdCompatExample.cpp std.obj std.compat.obj
    ```

    We didn't have to specify `std.compat.ifc` on the command line because the compiler automatically looks for the `.ifc` file that matches the module name in an `import` statement. When the compiler encounters `import std.compat;` it finds `std.compat.ifc` since we put it in the same directory as the source code--relieving us of the need to specify it on the command line. If the `.ifc` file is in a different directory than the source code, use the [`/reference`](../build/reference/module-reference.md) compiler switch to refer to it.

    Even though we're importing `std.compat`, you must also link against `std.obj` because that is where the standard library implementation lives that `std.compat` builds upon.

    If you're building a single project, you can combine the step of building the `std` and `std.compat` standard library named modules with the step of building your application by adding `"%VCToolsInstallDir%\modules\std.ixx"` and `"%VCToolsInstallDir%\modules\std.compat.ixx"` (in that order) to the command line. Make sure to put them before any `.cpp` files that consume them, and specify `/Fe` to name the built `exe` as shown in this example:

    ```dos
    cl /FestdCompatExample /std:c++latest /EHsc /nologo /W4 /MTd "%VCToolsInstallDir%\modules\std.ixx" "%VCToolsInstallDir%\modules\std.compat.ixx" stdCompatExample.cpp
    ```

    I've shown them as separate steps in this tutorial because you only need to build the standard library named modules once, and then you can refer to them from your project, or from multiple projects. But it may be convenient to build them all at once.

     The compiler command above produces an executable named `stdCompatExample.exe`. When you run it, it produces the following output:

    ```output
    Import std.compat to get global names like printf()
    555
    ```

## Different ways to consume the standard library

In this tutorial, you've seen how to import the standard library as a named module. This is the fastest and most robust way to consume the C++ standard library in your application.

There are other ways to consume the standard library that may make sense depending on your situation. Here's a summary so that you're aware of the other options. These are arranged in terms of compiler processing speed and robustness, with `#include` being the slowest and least robust, and `import` being the fastest and most robust.

| Method | Summary |
|---|---|
| `#include` | This is the 'classic' way of doing things. Disadvantages: exposes macros and internal implementation (functions and types that start with an underscore which library implementors use to indicate it's internal and shouldn't use), is fragile (the order of #includes can modify behavior or break code and are affected by macro definitions), is slow and gets slower when the same file is included in multiple files because the header file has to be reprocessed for each file that includes it. |
| [Precompiled header](../build/creating-precompiled-header-files.md) | A precompiled header (PCH) improves compile time by creating a compiler memory snapshot of a set of header files. This is an improvement on rebuilding header files over and over. PCH files have restrictions that make them difficult to build and maintain. In terms of speed and robustness, PCH files are faster than `#include` but slower than `import`.|
| [Header units](../build/walkthrough-header-units.md) | This is a new feature in C++20 that allows you to import 'well-behaved' header files as modules. Header units are faster than `#include`, and are easier, significantly smaller, and faster than pre-compiled header files (PCH). Header units are an 'in-between' step meant to help transition to named modules in cases where you rely on macros defined in header files, since named modules don't expose macros. Header units are slower than importing a named module. Header units aren't affected by macro defines unless they're specified on the command line when the header unit is built--making them more robust than header files. Header units expose the macros and internal implementation defined in them just as header file do, which named modules don't. As a rough approximation of file size, a 250-megabyte PCH file might be represented by an 80-megabyte header unit file. |
| [Modules](modules-cpp.md) | This is the fastest and most robust way to import functionality. Support for importing modules was introduced in C++20. The C++23 standard library introduces the two named modules described in this topic. When you import `std`, you get the standard names such as `std::vector`, `std::cout`, but no extensions, no internal helpers such as `_Sort_unchecked`, and no macros. The order of imports doesn't matter because there are no macro or other side-effects. As a rough approximation of file size, a 250-megabyte PCH file might be represented by an 80-megabyte header unit file, which might be represented by a 25-megabyte module. The reason named modules are faster is because when a named module is compiled into an `.ifc` file and an `.obj` file, the compiler emits a structured representation of the source code that can be loaded quickly when the module is imported. The compiler can do some work (like name resolution) before emitting the `.ifc` file because of how named modules are order-independent and macro-independent--so this work doesn't have to be done when the module is imported. In contrast, when a header file is consumed with `#include`, its contents must be preprocessed and compiled again and again in every translation unit. Precompiled headers, which are compiler memory snapshots, can mitigate those costs, but not as well as named modules. |

## Standard library named module considerations

Versioning for named modules is the same as for headers. The `.ixx` named module files live alongside the headers, for example: `"%VCToolsInstallDir%\modules\std.ixx`, which resolves to `C:\Program Files\Microsoft Visual Studio\2022\Preview\VC\Tools\MSVC\14.35.32019\modules\std.ixx` in the version of the tools used at the time of this writing. Select the version of the named module to use the same way you choose the version of the header file to use--by the directory you refer to them from.

Don't mix and match importing header units and named modules. That is, don't `import <vector>;` and `import std;` in the same file, for example. You'll get a compiler error.

Don't mix and match importing header units and name modules. That is, don't `#include <vector>` and `import std;` in the same file, for example.

You don't have to defend against importing a module multiple times. No header guard `#ifndef` required. The compiler knows when it has already imported a named module and ignores duplicate attempts to do so.

If you need to use the `assert()` macro, then `#include <assert.h>`.

If you need to use the `errno` macro, `#include <assert.h>`. Because named modules don't expose macros, this is the workaround if you need to check for errors from `<cmath>` functions.

`INT_MIN` is defined by `<limits.h>` and `<climits>` and works with header files/header units. However, those two headers won't work as a named module because named modules don't expose macros. You can use `std::numeric_limits<int>::min()` instead.

## Summary

In this tutorial, you've imported the standard library using modules. Next, learn about creating and importing your own modules in [Named modules tutorial in C++](tutorial-named-modules-cpp.md).

## See also

[Overview of modules in C++](modules-cpp.md)\
[A Tour of C++ Modules in Visual Studio](https://devblogs.microsoft.com/cppblog/a-tour-of-cpp-modules-in-visual-studio)\
[Moving a project to C++ named Modules](https://devblogs.microsoft.com/cppblog/moving-a-project-to-cpp-named-modules)\
