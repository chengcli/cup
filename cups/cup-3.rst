Plane-parallel Radiative Transfer
=================================

CUP: 3
Author: Cheng Li
Type: Feature
Created: 05-01-2024

.. toctr::
    :local:
    :depth: 2

Overview
--------


This CUP discusses how to obtain the radiation fields given an atmospheric profile,
and the boundary conditions. The radiation fields consist of the intensities at
multiple viewing angles and the radiative fluxes at the top and bottom boundaries
of an atmospheric layer. Both of them are the solutions of the 1D radiative transfer
equation (RTE) in plane-parallel geometry.

The RTE is an integro-differential equation as a function of the optical depth.
To solve it numerically, we employ the discrete ordinate method, which discretizes
the atmosphere into multiple layers and the viewing angles into multiple directions.
Specifically, we use the C version of 
`DISORT <https://www.libradtran.org/doku.php?id=download>`_
to solve the RTE [1]_. We made changes to the original srouce code to adapt it to the
``cmake`` and C++ environment.


Changes to the Original C-DISORT
--------------------------------

The most significant changes to the original C-DISORT are:


1. Add head guards to the header file indicating that the source code is written in C.

Here is an example of adding head guards to the header file:

.. code-block:: c

    #ifdef __cplusplus
    extern "C" {
    #endif

    ...

    #ifdef __cplusplus
    }
    #endif

``__cplusplus`` is a preprocessor directive that checks if the code is being compiled
in a C++ environment, which is only defined in C++ compilers.
``extern "C"`` tells the C++ compiler to use C linkage for the enclosed code. In C++,
function names are mangled (changed) during compilation to support function overloading,
a feature not present in C. By specifying extern "C", the C++ compiler ensures that the
function names inside the block are not mangled, making them compatible with C.
The last ``__cplusplus`` closes the block that started with extern "C".


2. Add error handling to the source code such that C-DISORT does not exit
   when an error occurs. Instead, it returns an error code to the caller.

C-DISORT uses ``exit()`` to exit the program when an error occurs. However, ``exit()``
is not compatible with modern C++ exception handling. This becomes exceedingly problematic
when C-DISORT is wrapped in a Python interface. When an error occurs, 
the Python interpreter will crash if ``exit()`` is called. To solve this issue,
we modified the source code to return an error code to the caller instead of exiting:

.. code-block:: diff

    -void c_check_inputs(disort_state *ds,
    +int c_check_inputs(disort_state *ds,
            int           scat_yes,
            int           deltam,
            int           corint,
    @@ -6212,7 +6216,8 @@ void c_check_inputs(disort_state *ds,
       }
     
       if (inperr) {
    -    c_errmsg("DISORT--input and/or dimension errors",DS_ERROR);
    +    c_errmsg("DISORT--input and/or dimension errors",DS_WARNING);
    +    return 1;
       }


This code snippet changes the error code ``DS_ERROR`` to ``DS_WARNING`` and returns
``1`` to the caller. The caller can then decide what to do based on the error code.

On the caller side, usually it is the ``c_disort`` function, it checks the error code
and returns an error code as well if an error occurs:


.. code-block:: diff

    -void c_disort(disort_state  *ds,
    +int c_disort(disort_state  *ds,
            disort_output *out)
     {
       static int
    @@ -507,7 +507,11 @@ void c_disort(disort_state  *ds,
       }
     
       /* Check input dimensions and variables */
    -  c_check_inputs(ds,scat_yes,deltam,corint,tauc,callnum);
    +  int err = c_check_inputs(ds,scat_yes,deltam,corint,tauc,callnum);
    +  if (err) {
    +    free(tauc);
    +    return err;
    +  }

The revised C-DISORT program has been wrapped in a Python interface, which can be installed
via ``pip``:


.. code-block:: bash

    pip install pydisort

The documentation of the Python interface and the example program 
can be found `here <https://pydisort.readthedocs.io/en/latest/>`_.


Optical Properties
------------------


Absorber
~~~~~~~~

RadiationBand
~~~~~~~~~~~~~

Radiation
~~~~~~~~~


Opacity
-------


Line-by-line Opacity
~~~~~~~~~~~~~~~~~~~~


Correlated-K Opacity
~~~~~~~~~~~~~~~~~~~~



Radiative Transfer
------------------


Boundary Conditions
~~~~~~~~~~~~~~~~~~~


Intensity
~~~~~~~~~


Radiative Flux
~~~~~~~~~~~~~~


Actinic Flux
~~~~~~~~~~~~


Examples
--------


References
----------
