Canoe as a vessel for models
============================

CUP: 2
Author: Cheng Li
Type: Feature
Created: 05-01-2024

.. toctree::
   :local:
   :depth: 2

Overview
--------

We are trying to solve the following three problems:


1. How to build a numerical model as a lego toy, so that it can be easily
   disintegrated into small pieces and reassembled into a new toy.
2. How to make personal modifications to an existing model, without
   interfering with the original development process.
3. How to leverage the experience and knowledge from the industry, so that
   an academic model is built with the same quality as an industrial model.


Here comes `Canoe <https://github.com/chengcli/canoe>`_, a vessel for building
your own numerical models. ``Canoe`` was originally developed for atmospheric
simulations, but it has evolved into a general purpose tool for building
any numerical models. It provides a collection of tools and a system for integrating
existing numerical models into a new model. Each component in ``Canoe``
can be tested independently, or the model can be used as a whole. Adding a new
component to the model is as easy as adding a new lego block to a toy.
This CUP discusses the design of ``Canoe`` and how to use it to build your own
numerical model.


What is a build system
----------------------

We primarily target the numerical model that requires a "build" process.
The "build" proccess compiles a set of source files into an executable program.
Therefore, programs written in C, C++, Fortran, rust, and other compiled languages
belongs to this category. Programs written in Python, Matlab, and other interpreted
languages do not require a build process, and they are not the target of this CUP.
However, ``Canoe`` can be used to build a Python interface whose backend is written
in C or C++.

A build system is a set of rules and tools that automate the build process. The most
one is ``make``, which is a tool for building programs from instructions written in
a ``makefile``. ``make`` is good for small projects, but it is not scalable for large 
projects that consist of many source codes and dependencies. ``cmake`` is a more
powerful tool for building large projects. It is a meta-build system that generates
``makefile`` for ``make``. ``cmake`` is also a cross-platform, cross-language tool
that can generate ``makefile`` for Linux, Mac, and Windows and for code written in
C, C++, Fortran, and other languages. Many popular open-source projects use ``cmake``
as their build system. Moreover, if a model is built with ``cmake``, it can be easily
integrated into other projects that use ``cmake`` as their build system. Therefore,
we choose to use ``cmake`` as the build system ``Canoe``.


How to cmake
------------

At the core of ``cmake`` is the ``CMakeLists.txt`` file, a configuration file which 
defines build settings. Every folder in a ``cmake`` project that contains source code
to be built should contain a ``CMakeLists.txt`` file. ``CMakeLists.txt`` is a text
file containing a set of commands. To illustrate, consider a simple project with two source 
files, ``main.cpp`` and ``functions.cpp``. The ``CMakeLists.txt`` for this project may look like 
this:

.. code-block:: cmake

   cmake_minimum_required(VERSION 3.10)
   project(SimpleProject)
   add_executable(App main.cpp functions.cpp)

Here, ``cmake_minimum_required`` specifies the minimum CMake version required. ``project`` 
names the project, and ``add_executable`` tells CMake to create an executable named 
``App`` from ``main.cpp`` and ``functions.cpp``.

To internal build this project, you would create a ``build`` directory at the root
directory of the source code, change to the ``build`` directory, and
run ``cmake`` followed by the path to the project folder that contains the ``CMakeLists.txt``
file. For example, on Linux or Mac:


Internal build method
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   mkdir build
   cd build
   cmake ..

In which, ``..`` refers to the parent directory of the current directory (build).
You can use also an **external build** method, in which you create the ``build`` directory
**outside** the project folder and run ``cmake`` followed by the path to the project folder:


External build method
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   mkdir build
   cd build
   cmake /path/to/project

For simple tests, an internal build is sufficient. However, for large projects that may
contain many different builds, versions, and configurations, an external build is preferred.

Regardless of internal or external builds, the ``cmake`` processes the ``CMakeLists.txt`` file
to generate standard build files suitable for your environment (e.g., Makefiles on Unix,
project files on Windows, etc). When you run `cmake` followed by the path to
``CMakeLists.txt``, it detects your environment, compiler, and available libraries, 
adapting the build process accordingly. This abstraction allows developers to write code 
without worrying about platform-specific build details.


Canoe's build system
--------------------

``Canoe`` uses ``cmake`` as its build system and manages the dependencies using commands
in the ``CMakeLists.txt`` files. The ``CMakeLists.txt`` file at the root
directory of ``Canoe`` contains the following commands:

.. code-block:: cmake

    ...
    include(${CMAKE_SOURCE_DIR}/cmake/yamlpp.cmake)
    include(${CMAKE_SOURCE_DIR}/cmake/gtest.cmake)
    include(${CMAKE_SOURCE_DIR}/cmake/athena.cmake)
    ...


In which, the ``include`` command includes the ``yamlpp.cmake``, ``gtest.cmake``, and
``athena.cmake`` files, which are located in the ``cmake`` directory. These files
contain the commands for downloading and building the dependencies. For example,
the ``yamlpp.cmake`` file contains the following commands:

.. code-block:: cmake

    ...
    FetchContent_Declare(
      yamlcpp
      DOWNLOAD_EXTRACT_TIMESTAMP TRUE
      URL https://github.com/jbeder/yaml-cpp/archive/refs/tags/yaml-cpp-0.7.0.tar.gz
    )
    FetchContent_MakeAvailable(yamlcpp)


When ``Canoe`` builds, it downloads the ``yaml-cpp`` source code from the URL specified
in the ``yamlcpp.cmake`` file and builds the dependency. The ``FetchContent_MakeAvailable``
command makes the ``yaml-cpp`` library available to any other sources inside the ``Canoe`` 
project. Because ``yaml-cpp`` is a header-only library, the library only contains the 
header files. If a dependency is a library that requires compilation, ``Canoe`` will
build the library and link it to the ``Canoe`` project. For example, the ``gtest.cmake``
downloads the ``google test`` source code and builds the ``gtest`` library with headers
and binary files.

Since dependent libraries are downloaded and built on the fly, ``Canoe`` requires internet
connection to build. However, once the dependencies are built, ``Canoe`` can be built
again offline. The ``Canoe`` build system is designed to be cross-platform and cross-language.
It is currently tested on Linu and Mac, and it can build C, C++, and Fortran programs.


Set command line options
~~~~~~~~~~~~~~~~~~~~~~~~

All ``cmake`` related commands are placed in the ``cmake`` folder at the root directory.
One of them is the ``compiler.cmake`` file, which contains the commands for setting the
compiler flags. By default, ``Canoe`` detects the system compiler and sets the appropriate
flags. You can override the default setting by command line options. For example, to
set the compiler to ``clang`` and enable the ``-Wall`` flag, you can run the following

.. code-block:: bash

   cmake -DCMAKE_CXX_COMPILER=clang -DCMAKE_CXX_FLAGS="-Wall" ..

The command line options are passed to the ``cmake`` via the ``-D`` flag. 
You can chain multiple options together by repeating the ``-D`` flags.
Here is a table of commonly used command line options:

.. list-table::
    :widths: 20 20 40
    :header-rows: 1

    * - Flag
      - Options
      - Description
    * - ``CMAKE_BUILD_TYPE``
      - ``Debug`` | ``Release``
      - set the build type
    * - ``CMAKE_INSTALL_PREFIX``
      - ``/path/to/install``
      - set the installation directory
    * - ``CMAKE_PREFIX_PATH``
      - [``/path/to/dependencies``;``...``]
      - set the paths to the dependencies, separated by ``;``


Configure your project
~~~~~~~~~~~~~~~~~~~~~~


Fetch dependencies
~~~~~~~~~~~~~~~~~~


Examples
--------
