Modeling Opacity
================

CPU: 4
Author: Cheng Li
Type: Feature
Created: 06-01-2024


Global Circulation Model
------------------------

A global circulation model (GCM) is crucial for understanding the complex, three-dimensional nature of atmospheres on exoplanets.
It is also a necessary tool to investigate how large-scale circulation may alter the thermal structure of the atmosphere and the distribution of chemical tracers within. 
However, the complex nature of solving partial differential equations on a sphere is a
significant technical hurdle for many researchers.

As a result, it is crucially important to have a variety of GCMs available to the community, each constructed with different numerical methods and physical approximations
to allow for a diversity of approaches to the problem.

We target the general audience who have access to a hydrodynamic model in the Cartesian
geometry and present a method to convert it to a spherical geometry.


However, developing a GCM from scratch is a time-consuming task that requires knowledge well beyond a basic understanding of numerical methods and physics.
The major difficulty lies not in the numerical methods but in the spherical geometry, which is unique to geophysical fluid dynamics.
The spherical geometry introduces a number of complexities that are not present in the Cartesian geometry, among which the most important are the coordinate singularity at the poles, the non-uniform grid spacing, and the non-orthogonal grid.

The spectral method, based on the spherical harmonics, is a popular choice for solving partial differential equations on a sphere that overcomes many of the difficulties associated with the spherical geometry.

Yet, the scalability of the spectral method is poor as global communication is required at each time step.

Here, we define the numerical singularity as a point where the general numerical scheme ceases to be well-behaved.
Special treatment is needed to deal with the point at the singularity. i
Traditional lat-lon grid has two singularities at the poles.

This is a major challenge for the spectral method, which is based on the spherical harmonics.

I'm not sure if this is a good idea. The spectral method is not scalable. It is not suitable for large-scale simulations.

The spectral method is not scalable. It is not suitable for large-scale simulations.

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
