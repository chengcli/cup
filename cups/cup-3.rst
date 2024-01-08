Plane-parallel Radiative Transfer
=================================

:CUP:
  3

:Author:
  Cheng Li

:Type:
  Feature

:Created:
  01-05-2024

.. toctree::
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

Thus, any phase function must satisfy the following **normalization condition**:

.. _normalization_condition:

.. math::

    \int_{-1}^{1} P(\mu) \mathrm{d}\mu = 2

We use a finite number of Legendre polynomials to represent the phase function:

.. math::

    P(\mu) = \sum_{n=0}^N \tilde{\omega}_n P_n(\mu)

where :math:`P_n(\mu)` is the :math:`n`-th order Legendre polynomial and the expansion
coefficients :math:`\tilde{\omega}_n` are called the **phase moments**. The phase 
function moments are given by the orthogonality relation:

.. math::

    \tilde{\omega}_n = \frac{2n+1}{2} \int_{-1}^{1} P(\mu) P_n(\mu) \mathrm{d}\mu

Based on the previous :ref:`normalization condition <normalization_condition>`, the
zero-th order phase moment is always 1:

.. math::

    \tilde{\omega}_0 = 1

because :math:`P_0(\mu) = 1`. The first-order phase moment measures the asymmetry of
the phase function in the forward and backward directions. In isotropic scattering,
and Rayleigh scattering, the first-order phase moment is zero. We define the *asymmetry
factor* (:math:`g`) as one third of the first-order phase moment:

.. math::

    g = \frac{1}{3} \tilde{\omega}_1

The :math:`g`-factor increases as the diffraction peak of the phase function becomes
more forward-peaked and turns negative when the phase function is back-scattering dominant.

Overall, the attenuation coefficient :math:`\kappa`, 
single-scattering albedo :math:`\omega_0`,
and the phase function moments :math:`\tilde{\omega}_n` are the three fundamental optical
properties that enable the calculation of the radiative transfer in a medium.


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

to calculate the optical properties of the absorber.
In the above code snippet, ``wave1`` and ``wave2`` are the lower and upper wavenumbers
(or wavelengths) of the spectral bin, ``var`` stores the TPX (temperature,
pressure, composition) state of the 
air parcel. The ``GetAttenuation`` or ``GetSingleScatteringAlbedo`` function returns
a real number, while the ``GetPhaseMomentum`` function returns an array of real numbers,
stored in the ``pp`` array of size ``np``.

Any concrete absorber class should inherit from the ``Absorber`` class and
provides the customized implementation of the above three methods. For example, 
the following code snippet shows the implementation of the ``HitranAbsorber`` class for
calculating the attenuation coefficient.

.. code-block:: C++

    Real HitranAbsorber::GetAttenuation(Real wave1, Real wave2,
                                    AirParcel const &var) const {
      // first axis is wavenumber, second is pressure, third is temperature anomaly
      Real val, coord[3] = {wave1, var.w[IPR], var.w[IDN] - getRefTemp(var.w[IPR])};
      interpn(&val, coord, kcoeff_.data(), axis_.data(), len_, 3, 1);

      Real dens = var.w[IPR] / (Constants::Rgas * var.w[IDN]);
      Real x0 = 1.;
      if (mySpeciesId(0) == 0) {
        for (int n = 1; n <= NVAPOR; ++n) x0 -= var.w[n];
      } else {
        x0 = var.w[mySpeciesId(0)];
      }
      return 1.E-3 * exp(val) * dens * x0;  // ln(m*2/kmol) -> 1/m
    }

The ``HitranAbsorber`` class is a derived class of the ``Absorber`` class.
It loads the absorption cross section computed from the 
`HITRAN database <https://hitran.org/>`_ and calculates the optical properties given 
the state of an air parcel. The grided absorption cross section is stored in the 
``kcoeff_`` array, and the ``axis_`` array stores the coordinates of the grid points.
The ``interpn`` function is the same function as we used in the
:ref:`photolysis rate <photolysis_rate>` calculation.
There are other paculiarities in the code above, such as the ``mySpeciesId``,
``NVAPOR`` that we shall defer the discussion to later.


RadiationBand
~~~~~~~~~~~~~

Absorbers are contained in a ``RadiationBand`` class, which manages the absorbers
and carries out the radiative transfer calculation. The ``RadiationBand`` class is
a rich class with many members and methods. Here we only show some important ones:

.. code-block:: C++

    class RadiationBand {
     public:  // public access data
      ....
      //! band optical depth
      AthenaArray<Real> btau;

      //! band single scattering albedo
      AthenaArray<Real> bssa;

      //! band phase function moments
      AthenaArray<Real> bpmom;

      //! band upward flux
      AthenaArray<Real> bflxup;

      //! band downward flux
      AthenaArray<Real> bflxdn;

      //! band top-of-the-atmosphere radiance
      AthenaArray<Real> btoa;
      ...

     public:  // inbound functions
      ...
      void SetSpectralProperties(AirColumn &air, Real const *x1f, Real gH0 = 0,
                                 int k = 0, int j = 0);

      //! Calculate band radiative fluxes
      RadiationBand const *CalBandFlux(MeshBlock const *pmb, int k, int j, int il, int iu);

      //! Calculate band radiances
      RadiationBand const *CalBandRadiance(MeshBlock const *pmb, int k, int j);
      ...

     protected:
      ...
      //! radiative transfer solver
      std::shared_ptr<RTSolver> psolver_;

      //! spectral grid
      std::shared_ptr<SpectralGridBase> pgrid_;

      //! all absorbers
      std::vector<std::shared_ptr<Absorber>> absorbers_;
      ...
    }

The general convention of ``Harp`` is that protected member names end with an underscore.
We divide the whole spectral range into several disjoint bands, with each having its own
``RadiationBand`` object. The ``RadiationBand`` object stores the optical properties
in public member arrays, such as ``btau``, ``bssa``, ``bpmom``, ``bflxup``, ``bflxdn``,
and ``btoa`` because they are the primary outputs of the radiative transfer calculation
used by other modules such as I/O and dynamics.
This is not particularly a safe practice, but it is convenient.


.. note::

    Future versions of ``Harp`` may restrict the access of ``btau``, ``bssa`` and ``bpmom``


The ``RadiationBand``
object also has protected members that other modules should not access directly, such as
the radiative transfer solver ``psolver_``, the spectral grid within the band ``pgrid_``,
and the absorbers ``absorbers_``.

The ``AthenaArray`` is a templated class that implements a multi-dimensional array
with a parenthesis operator for accessing the elements. The ``AthenaArray`` class
is part of the ``Athena++`` hydrodynamics code, and it is used in ``Harp`` for
storing a multi-dimensional data cube. Accessing the elements of an ``AthenaArray``
is similar to accessing the elements of a Python array, except for using parenthesis
instead of square brackets. For example, to access the element at ``(k,j,i)``,
a 3D ``AthenaArray`` object ``a`` is accessed as ``a(k,j,i)``.


There are three main methods of the ``RadiationBand`` class:

    - ``SetSpectralProperties``: sets the optical properties of the air column
    - ``CalBandFlux``: calculates the radiative fluxes
    - ``CalBandRadiance``: calculates the intensity (radiance)

``SetSpectralProperties`` takes an ``AirColumn`` object and the coordinates of the
lower boundary of the air column as input, and sets all optical properties of the
internal variables of the ``RadiationBand`` object. The ``AirColumn`` object is
simply a vector of ``AirParcel`` objects, which store the TPX state of
the air parcel.

A convention throughout ``Harp`` is that the ``i`` index is the vertical index,
and ``k`` and ``j`` are unspecified indices. 
They can represent the horizontal dimensions or other indices that identify 
the vertical column.

.. code-block:: C++

    void RadiationBand::SetSpectralProperties(AirColumn& ac, Real const* x1f,
                                              Real gH0, int k, int j) {
      ...
      for (int i = 0; i < ac.size(); ++i) {
        auto& air = ac[i];
        air.ToMoleFraction();
        tem_[i] = air.w[IDN];

        for (auto& a : absorbers_) {
          for (int m = 0; m < nspec; ++m) {
            auto& spec = pgrid_->spec[m];
            Real kcoeff = a->GetAttenuation(spec.wav1, spec.wav2, air);  // 1/m
            Real dssalb =
                a->GetSingleScatteringAlbedo(spec.wav1, spec.wav2, air) * kcoeff;
            // tau
            tau_(m, i) += kcoeff;
            // ssalb
            ssa_(m, i) += dssalb;
            // pmom
            a->GetPhaseMomentum(mypmom.data(), spec.wav1, spec.wav2, air, npmom);
            for (int p = 0; p <= npmom; ++p) pmom_(m, i, p) += mypmom[p] * dssalb;
          }
        }
      }
      ...
    }

For each air parcel, the ``ToMoleFraction`` method is called to convert the
unit of composition to mole fractions. Then the optical properties of each absorber
are calculated and stored in the ``tau_``, ``ssa_``, and ``pmom_`` arrays.
These arrays are internal arrays of the ``RadiationBand`` object that store 
the optical properties of the air column at each spectral bin (grid point).
They differ from band aggregates such as ``btau``, ``bssa``, and ``bpmom``
in that the latter are the optical properties of the whole band, which are
the sum of the optical properties of the bins. Calculating the optical depth ``btau``
from the attenuation coefficient requires knowing the path length, which is
provided by the ``x1f`` array.


``CalBandFlux`` and ``CalBandRadiance`` are the two methods that perform the
radiative transfer calculation. They transfer the optical properties stored
in the ``RadiationBand`` object to the radiative transfer solver ``psolver_``. 
In many cases, a radiative transfer solver is a 1D solver, so the radiative
transfer calculation is performed column by column.


The ``CalBandFlux`` method calculates the radiative fluxes
while the ``CalBandRadiance`` method calculates the intensity (radiance).
They have similar implementations, so we only show the implementation of the
``CalBandFlux`` method here:


.. code-block:: C++

    RadiationBand const *RadiationBand::CalBandFlux(MeshBlock const *pmb, int k,
                                                    int j, int il, int iu) {
      // reset flux of this column
      for (int i = il; i <= iu; ++i) {
        bflxup(k, j, i) = 0.;
        bflxdn(k, j, i) = 0.;
      }

      psolver_->Prepare(pmb, k, j);
      psolver_->CalBandFlux(pmb, k, j, il, iu);

      return this;
    }

The 3D nature of the ``Harp`` is evident in the ``CalBandFlux`` method. The ``pmb``
argument is a pointer to the ``MeshBlock`` object that contains the data of the
geometric information of the domain, such as the coordinates of the cell centers
and the cell volumes. The ``k``, ``j``, ``il`` and ``iu`` arguments are the
Fortran-style inclusive indices of the cell column over which the radiative transfer
calculation is carried out, i.e., the radiative transfer calculation is carried out for
a vertical column with indices from ``il`` to ``iu`` inclusive. 


.. note::

   Explicitly using the Fortran-style indices ``(k,j,i)`` is not a good practice in modern C++.
   It is the major source of error and it is not performance portable. It is
   advised to use higher-level abstractions such as iterators, ranges, views and zips.
   ``Harp`` bears the burden of using Fortran-style indices because it depends on
   ``Athena++``, which uses Fortran-style indices. Future versions of ``Harp``
   may swap out ``Athena++`` with a performance portable C++ library.


Running the radiative transfer solve for an air column is a two-step process.
First, the ``psolver_`` object calss the ``Prepare`` method to prepare the
optical properties of the column **in the solver**. It is the specifics of the
``psolver_`` object to decide how to prepare them. In some cases, the solver
copies the optical properties from the ``RadiationBand`` object to its internal
memory. In other cases, the solver may use the optical properties stored in the
``RadiationBand`` object directly by referencing the appropriate members of the
``RadiationBand`` object.
Then the ``CalBandFlux`` method of the ``psolver_``
object is called to carry out the radiative transfer calculation.

The calculation steps through all spectral bins within the band, and the output
fluxes are summed and stored in the ``bflxup`` and ``bflxdn`` arrays.
Thus, ``bflxup`` and ``bflxdn`` are initialized to zero before the calculation.

Similarly, the ``CalBandRadiance`` method calls the ``Prepare`` method first and then
the ``CalBandRadiance`` method of the ``psolver_`` object calculates the radiance.


Radiation
~~~~~~~~~

Because the whole spectral range is divided into several disjoint bands, the
``Radiation`` class is the container of ``RadiationBand`` objects. The ``Radiation``
class is a lightweight class that only stores minimal information about the
radiation bands and fields:

.. code-block:: C++

    class Radiation {
     public:  // public access data
      ...
      //! radiance of all bands
      AthenaArray<Real> radiance;

      //! upward flux of all bands
      AthenaArray<Real> flxup;

      //! downward flux of all bands
      AthenaArray<Real> flxdn;
      ...

     public:  // inbound functions
      ...
      //! \brief Calculate the radiative flux
      void CalFlux(MeshBlock const *pmb, int k, int j, int il, int iu);

      //! \brief Calculate the radiance
      void CalRadiance(MeshBlock const *pmb, int k, int j);
      ...

     protected:
      ...
      //! all radiation bands
      std::vector<RadiationBandPtr> bands_;
      ...
    };

In fact, the band aggregated quantities such as ``bflxup``, ``bflxdn`` and ``btoa``,
can be shallow references to ``flxup``, ``flxdn`` and ``radiance`` of the ``Radiation`` object,
respectively. If a ``RadiationBand`` object is constructed within the ``Radiation`` object,
the ``Radiation`` object is responsible for allocating the memory for
these arrays and the ``RadiationBand`` objects sets up the
correct shallow references. The ``Radiation`` object also stores the ``RadiationBand`` objects
in the ``bands_`` vector. The ``RadiationBand`` objects are allocated when the ``Radiation`` 
object is constructed and deallocated when the ``Radiation`` object is destructed.

Two methods, ``CalFlux`` and ``CalRadiance``, are also thin wrappers of the corresponding
methods of the ``RadiationBand`` objects. They first call the ``SetSpectralProperties``
of each band and then loop over all ``RadiationBand`` objects
and call the corresponding methods of the ``RadiationBand`` objects. For example, the
``CalFlux`` method is implemented as follows:


.. code-block:: C++

    void Radiation::CalFlux(MeshBlock const *pmb, int k, int j, int il, int iu) {
      ...
      for (auto &p : bands_) {
        p->SetSpectralProperties(ac, pcoord->x1f.data(), grav * H0, k, j);
        p->CalBandFlux(pmb, k, j, il, iu);
      }
    }

Here, we see the use of ``pcoord`` to get the coordinates of the cell faces. The
``pcoord`` is a pointer to the ``Coordinates`` object of the ``MeshBlock`` object.

Summary
-------

The core of the ``Harp`` library is the ``RadiationBand`` class. It is responsible for
calculating the optical properties of an air column, storing the optical properties
and carrying out the radiative transfer calculation. The ``Radiation`` class is a
simply a container of ``RadiationBand`` objects. It is entirely possible to use
``RadiationBand`` objects directly without using the ``Radiation`` class. However,
the ``Radiation`` class provides a convenient way to manage multiple ``RadiationBand``
objects and perform the I/O operations. Reading radiation bands from a configuration
file is performed by the ``Radiation`` class. Writing the output of the radiative
transfer calculation is also performed by the ``Radiation`` class.


.. note::

  ``Harp`` offers an experimental Python interface to the ``RadiationBand`` class 
  called ``pyharp``. It is currently a work in progress.


References
----------

.. [1] Buras, Robert, Timothy Dowling, and Claudia Emde.
       "New secondary-scattering correction in DISORT with increased efficiency for 
       forward scattering."
       *Journal of Quantitative Spectroscopy and Radiative Transfer* 112.12 (2011): 2028-2034.

.. [2] Li, Cheng, et al. 
       "A high-performance atmospheric radiation package: With applications to the 
       radiative energy budgets of giant planets."
       *Journal of Quantitative Spectroscopy and Radiative Transfer* 217 (2018): 353-362.

