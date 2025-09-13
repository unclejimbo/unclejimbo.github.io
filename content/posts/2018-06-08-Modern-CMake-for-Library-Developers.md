---
title: Modern CMake for Library Developers
toc: true
tags:
  - C++
  - CMake
categories:
  - C++
date: 2018-06-08
---

CMake has followed the C++ standard on the road to modernization, which leads to simpler package and dependency management. As opposed to the old ways of doing CMake like setting `CMAKE_CXX_FLAGS` directly, modern cmake introduces lots of facilities to handle dependencies more cleanly. But also like C++, CMake is a huge monster now which is very hard to tame. Although there are a few talks and tutorials about modern CMake on the Internet, I still find them hard to follow for the first time.

Here I'll present a detailed explaination on how to use modern CMake, especially for library developers who want to package their libraries for downstream developers to use easily with CMake. Basic knowledge of CMake is preferred, as I'll not cover too much on some basic commands.

<!-- more -->

## Project Structure

> The example code in this blog post is an simplified version of my project [yart](https://github.com/unclejimbo/yet-another-ray-tracer), stripping off the real files and 3rd-party dependencies. If you'd like to see a working example, you could try the code itself, there are only two external dependencies you need to install for it to work.

Now suppose we are building a library `yart`, and the project is structured like bellow,

```
- yart
    - cmake
        - yart-config.cmake.in  // talk this later
    - example
        - main.cpp              // an example program using yart
        - CMakeLists.txt
    - include
        - yart
            - yart.h            // the main api header
    - src
        - yart.cpp              // the implementation file
        - util.h                // some internal utilities not intended for yart users
        - CMakeLists.txt
    - CMakeLists.txt
```

Let's start with the parent level `CMakeLists.txt`. Nothing interesting here, I'd like to use C++17 so CMake 3.10 is required. The `project` command will create a `YART` project as well as versioning variables including `YART_VERSION`, `YART_VERSION_MAJOR`, `YART_VERSION_MINOR` and `YART_VERSION_PATCH`.

```cmake
cmake_minimum_required(VERSION 3.10)

project(YART
    LANGUAGES CXX
    VERSION 0.1.0
)

add_subdirectory(src)
add_subdirectory(examples)
```

## A Song of Targets and Properties

Targets are the main objects CMake manipulate for building a project. Your library is a target, your executable is a target, and you'll also meet some other types of targets when setting up the build system.

And each target has a set of properties attached to them, in an OO sense that they even have access control. You'll soon notice that many commands in CMake has a signature similar to `command(your-target [PUBLIC|INTERFACE|PRIVATE] properties)`. An `INTERFACE` property means that a user will need to respect this property when depending on this target, while a `PRIVATE` property means that this property is only used internally. And `PUBLIC` means both.

### Create a Target

Without further ado, let's set up our library first in `src/CMakeLists.txt`.

```cmake
add_library(yart
    ${CMAKE_CURRENT_SOURCE_DIR}/yart.cpp
)
```

You might have seen or have used `file(GLOB ...)` before, please be advised that you should explicitly list the source files like the example above for the build system to automatically reconfigure CMake when you add a new source file.

```cmake
add_library(yart::yart ALIAS yart)
```

This line enables you to use `yart::yart` in the `example` target, will see late
### Configure the Target

We would like the users to choose whether to build a shared library or a static one,

```cmake
option(BUILD_SHARED_LIBS "Build shared library" ON)
include(GenerateExportHeader)
generate_export_header(yart
    EXPORT_MACRO_NAME YART_API
    EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/include/yart/core/common.h
)
```

There're a lot of interesting thing going on here. In the first command, `BUILD_SHARED_LIBS` is read by CMake to switch between static and shared library, and a user could alter this option in cache.

Well, the `generate_export_header` command creates a header file which helps switch between building shared and static libraries. And here is the generated `common.h` file with msvc, and you should use these macros to export your library symbols like `void YART_API my_api_fcn();`

```cpp
#ifndef YART_API_H
#define YART_API_H

#ifdef YART_STATIC_DEFINE
#  define YART_API
#  define YART_NO_EXPORT
#else
#  ifndef YART_API
#    ifdef yart_EXPORTS
        /* We are building this library */
#      define YART_API __declspec(dllexport)
#    else
        /* We are using this library */
#      define YART_API __declspec(dllimport)
#    endif
#  endif

#  ifndef YART_NO_EXPORT
#    define YART_NO_EXPORT
#  endif
#endif

#ifndef YART_DEPRECATED
#  define YART_DEPRECATED __declspec(deprecated)
#endif

#ifndef YART_DEPRECATED_EXPORT
#  define YART_DEPRECATED_EXPORT YART_API YART_DEPRECATED
#endif

#ifndef YART_DEPRECATED_NO_EXPORT
#  define YART_DEPRECATED_NO_EXPORT YART_NO_EXPORT YART_DEPRECATED
#endif

#if 0 /* DEFINE_NO_DEPRECATED */
#  ifndef YART_NO_DEPRECATED
#    define YART_NO_DEPRECATED
#  endif
#endif

#endif /* YART_API_H */
```

The pattern `$<:>` you see earlier is [generator-expressions](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html) which works just like `if` statement but could be compactly inserted into other cmake commands like this `target_compile_definitions` command. Notice the `PUBLIC` keyword here, it says that this definition is a public property of `yart`.

And now, we'd like to configure the compiler options,

```cmake
target_compile_features(yart PUBLIC cxx_std_17)
target_compile_options(yart PRIVATE
    $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:GNU>>:
        -pipe -march=native
        $<$<CONFIG:Debug>:-O0 -Wall -Wextra>>
    $<$<CXX_COMPILER_ID:MSVC>:
        $<$<CONFIG:Debug>:/Od /Wall /Zi>>
)
```

With `target_compile_features`, you could directly demand a language standard version like I did, or you could require specific c++ features like `target_compile_feature(yart PUBLIC cxx_const_expr)`. The second command specify compile options depending on the compilers and build type, again with generator-expressions.

```cmake
target_include_directories(yart
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)
```

The `target_include_directories` command set up the include directories of `yart`. Public api is located in `$<CMAKE_SOURCE_DIR>/include/`, as well as the generated `common.h` file, and the private header file is in the same directory as `$<CMAKE_CURRENT_SOURCE_DIR>`. Notice that `$<INSTALL_INTERFACE:include>` is needed for users to find `yart` headers after installing `yart` onto their system. The `include` directory is a relative path to `${CMAKE_INSTALL_PREFIX}` which is often `/usr/local` on Linux and `C:\Program Files` on Windows.

And we would also like to organize the build tree a bit and configure where to output generated binaries,

```cmake
set_target_properties(yart PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)
```

\*.lib will go to `ARCHIVE_OUTPUT_DIRECTORY`, \*.so will go to `LIBRARY_OUTPUT_DIRECTORY`, and \*.dll will go to `RUNTIME_OUTPUT_DIRECTORY`.

### Handle 3rd Party Dependencies

Suppose we have two 3rd-party dependencies,

```cmake
find_package(libmodern REQRUIED)
find_package(liblegacy 3.0 REQUIRED)
```

Since `libmodern` is written with modern CMake as well, we could simply do,

```cmake
target_link_libraries(yart PUBLIC libmodern::libmodern)
```

And that's it, modern CMake could handle target dependencies transitively, which means that you could forget about the messy variables and every property needed to use `libmodern` is handled correctly.

On the other hand, `liblegacy` is too old for this, so you have to switch to the old method,

```cmake
target_include_directories(yart PUBLIC $<BUILD_INTERFACE:${LIBLEGACY_INCLUDE_DIRS}>)
target_link_libraries(yart PUBLIC ${LIBLEGACY_LIBRARIES})
```

And be alert that the public include requirement is not handled transitively so that `yart` users won't know anything about it, yet. We'll fix that later on. Notice you could also write a `FindLibLegacy.cmake` file for it and handle all sorts of usage requirements there, and finally export only a `liblegacy::liblegacy` target. You can find a decent example [here](https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/).

### Install and Export the Target

Everything should be able to compile by now. But the library is not readily available for other developers to use right now, after they hit `make install` and write `find_package(yart)` in their `CMakeLists.txt`.

First, we need to ensure everything is installed to the correct places on system.

```
include(GNUInstallDirs)

install(TARGETS yart
    EXPORT yart-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${LIBLEGACY_INCLUDE_DIRS}
)
install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/yart
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
```

`GUNInstallDirs` is a cross-platform solution to install directories. And we issue the `install` command to install our files. It has several signatures, in the above block are `install(TARGETS yart ...)`, which installs the compiled library, and `install(DIRECTORY ...)`, which installs the include directory to the right place. Note `EXPORT yart-targets` line also exports this target to be used later, and the `INCLUDE DESTINATION ${LIBLEGACY_INCLUDE_DIRS}` line injects the include dependencies into `yart-targets` so that our dependency problem mentioned above is solved here.

```cmake
install(EXPORT yart-targets
    FILE yart-targets.cmake
    NAMESPACE yart::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yart
)
```

Talking about `EXPORT`, installing an `EXPORT` target will generate a `yart-targets.cmake` file which contains essential commands to build with `yart` correctly, like bellow,

```cmake
# Create imported target yart::yart
add_library(yart::yart SHARED IMPORTED)

set_target_properties(yart::yart PROPERTIES
    ...
)
```

What's more, the `find_package` command expects a `FindYART.cmake` file or a `yart-config.cmake` (`YARTConfig.cmake`) to find this library. The first method is an old way to hell so we definitely want the second one,

```cmake
include(CMakePackageConfigHelpers)

configure_package_config_file(
    ${CMAKE_SOURCE_DIR}/cmake/yart-config.cmake.in
    ${CMAKE_BINARY_DIR}/cmake/yart-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yart
)

write_basic_package_version_file(
    ${CMAKE_BINARY_DIR}/cmake/yart-config-version.cmake
    VERSION ${YART_VERSION}
    COMPATIBILITY AnyNewerVersion
)

install(
    FILES
        ${CMAKE_BINARY_DIR}/cmake/yart-config.cmake
        ${CMAKE_BINARY_DIR}/cmake/yart-config-version.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/yart
)
```

The `configure_package_config_file` command reads an scaffolding `yart-config.cmake.in` file and creates a `yart-config.cmake` file with the right paths. And let's see what the .in file looks like,

```cmake
@PACKAGE_INIT@

find_package(libmodern REQUIRED)
find_package(liblegacy 3.0 REQUIRED)

if(NOT TARGET yart::yart)
    include(${CMAKE_CURRENT_LIST_DIR}/yart-targets.cmake)
endif()
```

Quite simple right? You need to call `@PACKAGE_INIT@` first, find dependencies and finally include the generated `yart-targets.cmake` file.

A version file is also preferred by users and could be created easily with `write_basic_package_version_file`. Notice that these files are generated in your build tree (`${CMAKE_BINARY_DIR}`) and you need to install them into your system. The install directory is `${CMAKE_INSTALL_LIBDIR}/cmake/yart` which is the default search directory of `find_package`.

One last step, remember that we also have another `example` alongside this library? We need to call the `export` command,

```cmake
export(EXPORT yart-targets
    FILE ${CMAKE_BINARY_DIR}/cmake/yart-targets.cmake
    NAMESPACE yart::
)
```

So that `example` could refer to `yart` without finding the package.

## Use the Target

`example/CMakeLists.txt` couldn't be more simple,

```cmake
add_executable(yart-example
    "${CMAKE_CURRENT_SOURCE_DIR}/main.cpp"
)

target_link_libraries(yart-example PRIVATE
    yart::yart
)
```

While using `yart` from an outside project requires one more magic touch,

```cmake
find_package(yart)
```

## Conclusion

That's it folks. It's still quite tricky to do everything right, but I've covered the most common use cases. If you still find it hard to wrap your head around it, just remember your library is a target and its build and usage requirements are properties set with `target_xxx` commands. Other exported targets and config files are auxillary infrastructures to help down stream developers to use your library easily.

For other bits and pieces, please refer to the document, which is not very intuitive unfortunately. You might also want to refer to

- [Effective CMake Talk](https://www.youtube.com/watch?v=bsXLMQ6WgIk)
- [It's time to Do CMake Right](https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/)
- [Eigen](https://github.com/PX4/eigen), which shows a good example on how to do CMake right
- [The official wiki](https://gitlab.kitware.com/cmake/community/wikis/home), which contains some explanations not showing in the documentation
