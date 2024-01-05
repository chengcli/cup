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


What is cmake
-------------


Canoe's build system
--------------------


Set the compiler
~~~~~~~~~~~~~~~~


Configure your project
~~~~~~~~~~~~~~~~~~~~~~


Fetch dependencies
~~~~~~~~~~~~~~~~~~


Examples
--------
