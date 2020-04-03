---
layout: post
title:  "How to organize a C/C++ project"
thumbnail: assets/images/cpp-logo.svg
date: 2020-03-30
categories: cpp
---
Back to basics. I remember when starting C++ programming that I had a hard time
figuring out what is the correct way how to organize my projects. Most text books
I read or lectures I attended were more focused on teaching you either programming
principles or language features. So in this post I would like to share what is
in my opinion the best project structure. 

![Yocto](/assets/images/cpp-logo.svg){: .center-image }

The answer you usually get when asking how to structure your project is that you
should look at a well established project / library and stick to something 
similar. However when looking at projects like [Boost](https://github.com/boostorg/),
[Abseil](https://github.com/abseil/abseil-cpp), [JSON for Modern C++](https://github.com/nlohmann/json) I always first had bit a problem to get the essentials because there was
so much going on or because the library was header-only, but I wanted build
shared library, and I had to work on a couple of projects till I got the hang of
it.

So let's save you this time and look at a simple example that still should cover
the essentials of every project.

I am going to use CMake as my build tool to give a concrete example. But the
organization should be independent of the tool or IDE you use.

```sh
myproject
├── CMakeLists.txt
├── extern (any gitsubmodules or third-party sources)
├── include
│   └── mylib
│       └── public-header.hpp
├── src
│   └── mylib
│       ├── implementation.cpp
│       └── private-header.hpp
└── test
    └── test-myclass.cpp
```

Already when starting with a project it is important to define which parts of
code will be part of the public interface of the library and only place those
parts in the `include` directory (when you are building only an application you
basically do not need this directory). Then in the `src` folder you place your
private headers and the compilation units for your project.

I also usually keep the unit and integration tests of the libray in a separate
`test` folder.

I found it good practice to mirror the namespaces of my libraries as subdirectories
in the `src` and `include` folder.
The includes in your files look then like this:
```cpp
#include "mylib/mysubname/myheader.hpp"

namespace mylib::mysubname {

struct MyObject {
...
```

A corresponding `CMakeLists.txt` file for a library that used links to *Boost*
internally and *Threads* externally, and uses [googletest](https://github.com/google/googletest)
 for unit tests looks like this:
```cmake
cmake_minimum_required(VERSION 3.11)

project(mylib VERSION 2020.1
              LANGUAGES CXX
              HOMEPAGE_URL "https://github.com/dtrussel/cpp_project_template")

#####################################################################
# DEPENDENCIES
#####################################################################

find_package(Threads REQUIRED)
find_package(Boost REQUIRED)

include(FetchContent)

FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        release-1.10.0)

FetchContent_GetProperties(googletest)
if(NOT catch_POPULATED)
  FetchContent_Populate(googletest)
  add_subdirectory(${googletest_SOURCE_DIR} ${googletest_BINARY_DIR}
    EXCLUDE_FROM_ALL)
endif()

#####################################################################
# LIBRARY
#####################################################################

add_library(${PROJECT_NAME}
  src/mylib/implementation.cpp)

add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

set_target_properties(${PROJECT_NAME} PROPERTIES
  VERSION ${PROJECT_VERSION})

target_include_directories(${PROJECT_NAME}
  PUBLIC include
  PRIVATE src)

target_link_libraries(${PROJECT_NAME}
  PUBLIC Threads::Threads
  PRIVATE Boost::boost)

target_compile_options(${PROJECT_NAME}
  PRIVATE -Wall -Wextra -pedantic -Werror)

target_compile_features(${PROJECT_NAME}
  PRIVATE cxx_std_17)

include(GNUInstallDirs)

install(TARGETS ${PROJECT_NAME}
  EXPORT MyLibTargets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

#####################################################################
# UNIT TESTS
#####################################################################

add_executable(${PROJECT_NAME}-tests
  test/test-myclass.cpp)

target_link_libraries(${PROJECT_NAME}-tests
  PRIVATE gtest_main Threads::Threads ${PROJECT_NAME}::${PROJECT_NAME})

target_compile_options(${PROJECT_NAME}-tests
  PRIVATE -Wall -Wextra -pedantic -Werror)

target_compile_features(${PROJECT_NAME}
  PRIVATE cxx_std_17)

```

In my next post I will go more into detail regarding the CMake file. But for now
I hope you got a short and easy introduction into how to structure your C/C++ projects.

