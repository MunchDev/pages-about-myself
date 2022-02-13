---
layout: post
title: About CMake Build System
description: >
  This post describes the CMake build system, as well as lists some of the most popular template CMake
  files for C++ projects
image: /assets/img/coding/cmake-sample.png
categories: [coding]
tags: [cmake, cpp]
sitemap: false
---

_This is not a complete guide to CMake, but rather some of the practices that I find most relevant today._
_Hence, basic knowledge about CMake is assumed._

* toc
{:toc}

## One or more `CMakeLists.txt` files?

Most beginner tutorial will tell you to create a single `CMakeLists.txt` file in the root of the project.
However, this is mostly for convenience and simplicity, for the sake of the tutorials themselves. On the other
hand, you will most certainly find all major open-source projects that use `CMake` will use a whole sets of
`CMakeLists.txt` and its companion config files, so much so you find it impossible to replicate this
well-tuned build system in your own _tiny_ project.

Hence, in this post, I'm describing all the practices using a single `CMakeLists.txt` file, and maybe with a
few more (optional) configuration files.

## Compulsory information found in a `CMakeLists.txt` file

This post assumes you're writing a C++ project. As such, not all information will be applicable should you
write in C.
{:.note}

Opening any top-level `CMakeLists.txt` file will you find this (and its variation) at the top:

```cmake
// file: "CMakeLists.txt"
cmake_minimum_required(VERSION 3.21)
project(<name> VERSION 1.0.0 LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
```

The first line is self-explanatory.

The second line declares a project named `EXAMPLE` with version `1.0.0` and written in C++. Note that the
`project` command also sets the following CMake variables: `<name>_VERSION`, `<name>_VERSION_MAJOR`,
`<name>_VERSION_MINOR`, and `<name>_VERSION_PATCH`, where `name` is the name of the project.

The third line sets the C++ standard to be used, in this case `C++17`.

Setting C++ standard is not strictly compulsory. However, if it is not set, the compiler will use a default
standard (`C++14` for the most recent version of many popular compilers), and some newer feature in your code
might not work.
{:.note}

## CMake practices: C++ libraries

The rest of the template looks like this for building C++ libraries:

```cmake
// file: "CMakeLists.txt"
add_library(<name> ... path to source files ...)
add_library(<name>::<name> ALIAS <name>)

option(BUILD_SHARED_LIBS "Build shared library" ON)
include(GenerateExportHeader)
generate_export_header(<name>
    EXPORT_MACRO_NAME <name>_API
    EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/include/<name>/core/common.h
)

target_include_directories(<name>
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)

set_target_properties(<name> PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)

include(GNUInstallDirs)
install(TARGETS <name>
    EXPORT <name>-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${LIBLEGACY_INCLUDE_DIRS}
)
install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/<name>
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(EXPORT <name>-targets
    FILE <name>-targets.cmake
    NAMESPACE <name>::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_SOURCE_DIR}/cmake/<name>-config.cmake.in
    ${CMAKE_BINARY_DIR}/cmake/<name>-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)
write_basic_package_version_file(
    ${CMAKE_BINARY_DIR}/cmake/<name>-config-version.cmake
    VERSION ${<name>_VERSION}
    COMPATIBILITY AnyNewerVersion
)
install(
    FILES
        ${CMAKE_BINARY_DIR}/cmake/<name>-config.cmake
        ${CMAKE_BINARY_DIR}/cmake/<name>-config-version.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)
```

A mouthful of commands? Don't worry, I'm explaining everything below!

```cmake
add_library(<name> ... path to source files ...)
add_library(<name>::<name> ALIAS <name>)
```

The first line declares all the source files (not including any public/private header files) used to build
the library. For maximum compatibility, use absolute path for all files (using the special variable
`CMAKE_CURRENT_SOURCE_DIR`, which is the same as the location of the `CMakeLists.txt` file).

The next line just defines an alias, which is preferred by the community.

```cmake
option(BUILD_SHARED_LIBS "Build shared library" ON)
include(GenerateExportHeader)
generate_export_header(<name>
    EXPORT_MACRO_NAME <name>_API
    EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/include/<name>/core/common.h
)
```

The first line is self-explanatory, it simply allows everyone to build the library as a static library versus
a dynamic library (by specifying `-DBUILD_SHARED_LIBS`). Not knowing which one to choose? Please visit
[this helpful guide](https://stackoverflow.com/questions/140061/when-to-use-dynamic-vs-static-libraries)!

Then, a very useful module `GenerateExportHeader` is included, which defines macros that you can use to change
the visibility of classes and functions in your header file, or to mark something deprecated. More on this will
be explained later.

The next line (or three lines) uses the module to generate a header file that will be included to make the
above-mentioned macros available.

```cmake
target_include_directories(<name>
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)
```

This particular command specifies all the header files (beside system ones) that your project needs to build
the library. The `PUBLIC` and `PRIVATE` tags specifies whether header files will be included in another
project that depends on your library. The third line includes all your public header files, while the forth
line includes the header file generated by `GenerateExportHeader`, so that you can use the API macros.

The fifth line is needed so that your project's users know where to find your public header files. Private header files are usually stored in the same folder as source files, so that sixth line includes them.

Change this line to wherever you put your private header files.
{:.note}

```cmake
set_target_properties(<name> PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)
```

This command simply organises your build directory.

```cmake
include(GNUInstallDirs)
install(TARGETS <name>
    EXPORT <name>-targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ... include path of your project dependencies ...
)
install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/<name>
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(EXPORT <name>-targets
    FILE <name>-targets.cmake
    NAMESPACE <name>::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)
```

This particular section allows users to install your library on their system. The first line includes
`GNUInstallDirs` - a widely-used module for all CMake projects. The first `install` command simply specifies
the target to be exported, including the destionations to all build artifacts. The line starting with
`INCLUDES DESTINATION` allows you to specify the `include` path of your dependencies.

The second `install` command installs your `include` directory in the right place so that your users can look
for your public header files.

The third `install` command will generate a file named `<name>-targets.cmake`, which will contains essential
command to build your project correctly.

```cmake
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_SOURCE_DIR}/cmake/<name>-config.cmake.in
    ${CMAKE_BINARY_DIR}/cmake/<name>-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)
write_basic_package_version_file(
    ${CMAKE_BINARY_DIR}/cmake/<name>-config-version.cmake
    VERSION ${<name>_VERSION}
    COMPATIBILITY AnyNewerVersion
)
install(
    FILES
        ${CMAKE_BINARY_DIR}/cmake/<name>-config.cmake
        ${CMAKE_BINARY_DIR}/cmake/<name>-config-version.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/<name>
)
```

The last section requires the use of `CMakePackageConfigHelpers` module. Basically, this section will creates
a file named `<name>-config.cmake` so that CMake's `find_package` command will know where to find your
projects.

It's a lot to know right? If you don't want to understand the code, please just copy everything and change
`<name>` to the name of your project!

Moreover, one more config file template is needed. Create another folder named `cmake` (this folder can be
used to store other CMake files if needed) and create an empty file named `<name>-config.cmake.in`. The
content is as follows:

```cmake
// file: "cmake/<name>-config.cmake.in"

@PACKAGE_INIT@

find_package(libmodern REQUIRED)
find_package(liblegacy 3.0 REQUIRED)

if(NOT TARGET yart::yart)
    include(${CMAKE_CURRENT_LIST_DIR}/yart-targets.cmake)
endif()
```

## Conclusion

CMake is definitely not easy, but I hope that I provide a minimal template that you can use for your project.
Once again, if you decide to use multiple `CMakeLists.txt` files, some modifications are needed. In the next
part, I will be talking about a CMake template for building a C++ executable.

This post adapted from <https://unclejimbo.github.io/2018/06/08/Modern-CMake-for-Library-Developers/>
{:.note}
