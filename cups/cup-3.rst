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

We use ``Harp`` to refer to the radiation module of ``Canoe`` because wrapping the 
C-DISORT program started as a project to study the radiative properties of Jovian
atmospheres [2]_, although it has since been updated and refactored to be more general.


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


A radiative transfer solver like C-DISORT requires knowing the optical properties of 
the atmosphere, namely the absorption and scattering coefficients.
The total opacity of the atmosphere is the sum of the absorption and scattering opacities,
measured as the attenuation coefficient of the radiation field.
The attenuation coefficient (:math:`\kappa`), also known as the extinction coefficient, or
the narrow-beam attenuation coefficient, is usually measured in units of
:math:`\mathrm{m}^{-1}`. The attenuation of a narrow light beam intensity in a medium
of small thickness :math:`\Delta z` is acoording to the Beer-Lambert law:


.. math::

    I(z+\Delta z) = I(z) \exp(-\kappa \Delta z)


In reality, :math:`\kappa` is a sophisticated non-smooth function of wavenumber (
:math:`\nu`), temperature (:math:`T`), pressure (:math:`P`), and composition (:math:`X`).


.. math::

    \kappa = \kappa(\nu, T, P, X)

When the medium is composed of multiple species, the total attenuation coefficient
is the sum of the attenuation coefficients of the individual species.


The attenuation coefficient is related to the absorption coefficient (:math:`\kappa_a`)
and the scattering coefficient (:math:`\kappa_s`) by:

.. math::

    \kappa = \kappa_a + \kappa_s

The absorption coefficient is related to the absorption cross section (:math:`\sigma_a`)
by:

.. math::

    \kappa_a = \sigma_a \rho

where :math:`\rho` is the density of the absorbing species.
There is a similar relation between the scattering coefficient and the scattering cross
section (:math:`\sigma_s`). But normally, we express the scattering as a fraction
of the total attenuation.

The fraction of the total attenuation coefficient that is due to scattering is called
the single-scattering albedo (:math:`\omega_0`):


.. math::

    \omega_0 = \frac{\kappa_s}{\kappa}


The distribution of the scattered radiation is described by the phase function
(:math:`P(\cos\theta, \phi)`), which is the probability density function of the 
scattering angle (:math:`\theta`, :math:`\phi`). The phase function is 
**normalized to unity**, meaning that the phase function of the isotropic scattering is 1:

.. math::

    P(\cos\theta, \phi) = 1

We only consider the azimuthally symmetric phase function, which is a function of
the polar angle only. Let :math:`\mu = \cos\theta`, then the phase function is simply
:math:`P(\mu)`.

Thus, any phase function must satisfy the following conditions:

.. math::

    \int_{-1}^{1} P(\mu) \mathrm{d}\mu = 2

We use the Legendre polynomial expansion to represent the phase function:

.. math::

    P(\mu) = \sum_{n=0}^N \frac{2n+1}{2} P_n(\mu)

where :math:`P_n(\mu)` is the :math:`n`-th order Legendre polynomial.

The attenuation coefficient :math:`\kappa`, single-scattering albedo :math:`\omega_0`,
and the phase function :math:`P(\mu)` are the three fundamental optical properties
that enable the calculation of the radiative transfer in a medium.


Absorber
~~~~~~~~

We abstract the interaction between radiation and matter as a class called
``Absorber``. The ``Absorber`` class has the following methods:

.. code-block:: C++

    class Absorber {
      ...
      //! Get attenuation coefficient [1/m]
      virtual Real GetAttenuation(Real wave1, Real wave2,
                                  AirParcel const& var) const {
        return 0.;
      }

      //! Get single scattering albedo [1]
      virtual Real GetSingleScatteringAlbedo(Real wave1, Real wave2,
                                             AirParcel const& var) const {
        return 0.;
      }

      //! Get phase function [1]
      virtual void GetPhaseMomentum(Real* pp, Real wave1, Real wave2,
                                    AirParcel const& var, int np) const {}
      ...
    };

to calculate the attenuation coefficient, single scattering albedo, and the phase function.
In the above code snippet, ``wave1`` and ``wave2`` are the lower and upper wavenumbers
(or wavelengths) of the spectral band, ``var`` stores the :math:`T, P, X` state of the 
air parcel. The ``GetAttenuation`` or ``GetSingleScatteringAlbedo`` function returns
a real number, while the ``GetPhaseMomentum`` function returns an array of real numbers,
stored in the ``pp`` array. 

The ``np`` is the number of phase function moments to be


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
