Adding photolysis mechanism to Cantera
======================================

CUP: 1
Author: Cheng Li
Type: Feature
Created: 04-01-2024

.. toctree::
   :local:
   :depth: 2

Overview
--------

`Cantera <https://cantera.org>`_ is an open-source chemical kinetics solver written in modern C++.
It is capable of solving a wide range of chemical kinetics/transport problems, both in 0D and 1D.
However, ``Cantera`` does not natively support photochemical reactions. Adding photolsis to Cantera
requires some deep understanding of the ``Cantera`` code base. 
This CUP discusses the design of adding photolysis reactions to ``Cantera``.

Photochemical Reactions
-----------------------

A photochemical reaction is written as:

.. math::

    \text{A} \Rightarrow \text{B} + \text{C} + \ldots,

in which the ``=>`` symbol indicates that the reaction is irreversible, and ``A``, ``B``, ``C`` are
the reactants and products of the reaction. ``A`` is the parent molecule that absorbs the photon
and ``B``, ``C`` are the daughter molecules, also called the photofragments. Sometimes,
absorption of a photon does not lead to the dissociation of the parent molecule. It is legit to write 
the reaction as:

.. math::

    \text{A} \Rightarrow \text{A}

when the parent molecule does not dissociate. Since the outcome/yields of a photochemical reaction
depend on the wavelength of the photon and the temperature, the stoichiometric coefficients of the 
photofragments are variable. In many photochemistry models, the photolysis of a parent molecule is
implemented as a set of individual reactions, each of which corresponds to a specific combination of
fragments. For example, the photolysis of CH\ :sub:`4` can be written as:

.. math::

    \begin{align}
    \text{CH}_4 & \Rightarrow \text{CH}_3 + \text{H} \\
    \text{CH}_4 & \Rightarrow (1)\text{CH}_2 + \text{H}_2 \\
    \text{CH}_4 & \Rightarrow (3)\text{CH}_2 + \text{H} \\
    & \ldots
    \end{align}

Each of the above reactions is called a branch of the photolysis reaction. The total photolysis rate
of CH\ :sub:`4` is the sum of the rates of all branches. The fraction of each branch out of the total
is called the quantum yield or the branching ratio. So, there are two ways to specify the photolysis
cross section of a parent molecule: (1) the total cross section and the branching ratios of all
branches; (2) the cross section of each branch. ``Kinetics7`` format uses the second way
and ``Kinetics9`` format uses the first way.

Because the quantum yields are wavelength and temperature dependent, we can combine all branches
into one net reaction without specifying the quantum yields of the branches. For example, the
net photolysis reaction of CH\ :sub:`4` can be written as:

.. math::

    \text{CH}_4 \Rightarrow \text{CH}_3 + (1)\text{CH}_2 + \text{H}_2 + (3)\text{CH}_2 + \text{H} + \ldots

This way allows a more compact representation of the photolysis reaction such that the total photolysis rate
of CH\ :sub:`4` is the rate of this net reaction. It is important to note that the effective stoichiometric
coefficients of the products in the above reaction should be computed from the quantum yields of each branch.
As a result, we provide the follwing information to specify a photochemical reaction:

    1. The chemical equation of the net reaction
    2. The photofragments of each branch
    3. The cross section of each branch

``Cantera`` uses the `YAML <https://yaml.org/>`_ format to specify a chemical reaction. The following is an example
of the photolysis of CH\ :sub:`4`:

.. code-block:: yaml

    reactions:
      - equation: CH4 => CH4 + CH3 + (1)CH2 + (3)CH2 + CH + H2 + H
        type: photolysis
        branches:
          - name: b1
            product: "CH4:1"
          - name: b2
            product: "CH3:1 H:1"
          - name: b3
            product: "(1)CH2:1 H2:1"
          - name: b4
            product: "(3)CH2:1 H:2"
          - name: b5
            product: "CH:1 H2:1 H:1"
        cross-section:
          - format: KINETICS7
            temperature-range: [0., 300.]
            filenames: [../data/cross/CH4.dat]

Here, we define the net reaction equation in the ``equation`` field. Reaction balance check is not performed
if the reaction is a photolysis reaction, specified by the ``type`` field. The ``branches`` field is a list,
containing the information of all branches. Each branch is defined by a ``name`` and a ``product`` string, which
is a space-separated list of the photofragments. The string before the colon is the name of the photofragment
and the number after the colon is the stoichiometric coefficient of the photofragment. Finally, the ``cross-section``
field contains temperature-dependent cross sections, which are grouped into disjoint temperature ranges.
The cross sections of each temperature range are stored in a list of files, specified by the ``filenames`` field.
The ``format`` field specifies the format of the cross section files. Currently, we support following formats: 

  - KINETICS7
  - VULCAN


Implement Photolysis in Cantera
-------------------------------

``Cantera`` uses the ``type`` field to determine the type of a reaction and perform the corresponding 
calculation of the reaction rate coefficient. To let ``Cantera`` recognize that a reaction is a photolysis
reaction, we need to register a new keyword, ``photolysis``, to the class ``ReactionRateFactory``.
The following is the code snippet of the ``ReactionRateFactory`` class that registers the keyword
``photolysis``:

.. code-block:: C++

    ReactionRateFactory::ReactionRateFactory()
    {
      ...
        // PhotolysisRate evaluator
        reg("photolysis", [](const AnyMap& node, const UnitStack& rate_units) {
            return new PhotolysisRate(node, rate_units);
        });
    }

When ``Cantera`` finds a reaction with the ``type`` field set to ``photolysis``, it will call the
``PhotolysisRate`` constructor to create a ``PhotolysisRate`` object.

PhotolysisRate
~~~~~~~~~~~~~~

Evaluation of the photolysis rate coefficient is the most time-consuming part of the kinetic model.
``Cantera`` exploits the similarity of the chemical reaction rate evaluation to improve the performance
of the model. Specifically, ``Cantera`` groups reactions with the same expression of the rate coefficient
together and evaluates the rate coefficients of all reactions in a group at the same time. This is called
a ``MultiRate`` evaluation. We derive a ``PhotolysisRate`` class from the general ``ReactionRate`` class
and provide neccessary information to perform the evaluation of the photolysis rate coefficient. The
actual evaluation of all photolysis rates is handled by the ``MultiRate`` class, to be discussed later.
Given a wavelength range from :math:`\lambda_1` to :math:`\lambda_2`, the photolysis rate coefficient
at temperature :math:`T` is defined as:

.. math::

    k(T;\lambda_1,\lambda_2) = \int_{\lambda_1}^{\lambda_2} \sigma(\lambda, T) \phi(\lambda) d\lambda,

where :math:`\sigma(\lambda, T)` is the cross section of the parent molecule and :math:`\phi(\lambda)`
is the photon actinic flux. This function is implemented in the ``evalFromStruct`` method of the ``PhotolysisRate``.
The following code snippet of the ``evalFromStruct`` method shows how to evaluate the
photolysis rate coefficient, and obtain the effective stoichiometric coefficients of the photofragments
at the same time:

.. code-block:: C++

   class PhotolysisRate : public PhotolysisBase {
     public:
      using PhotolysisBase::PhotolysisBase;  // inherit constructor

      unique_ptr<MultiRateBase> newMultiRate() const override {
        return make_unique<MultiRate<PhotolysisRate, PhotolysisData>>();
      }
      ...

      double evalFromStruct(PhotolysisData const& data) {
        ...
        double coord[2] = {data.temperature, data.wavelength[iwmin]};
        size_t len[2] = {m_ntemp, m_nwave};

        interpn(cross1, coord, m_crossSection.data(), m_temp_wave_grid.data(),
            len, 2, m_branch.size());

        double total_rate = 0.0;
        for (int i = iwmin; i < iwmax; i++) {
          coord[1] = data.wavelength[i+1];
          interpn(cross2, coord, m_crossSection.data(), m_temp_wave_grid.data(),
              len, 2, m_branch.size());

          for (size_t n = 0; n < m_branch.size(); n++) {
            double rate = 0.5 * (data.wavelength[i+1] - data.wavelength[i])
              * (cross1[n] * data.actinicFlux[i] + cross2[n] * data.actinicFlux[i+1]);
            for (auto const& [name, stoich] : m_branch[n]) {
              m_net_products.at(name) += rate * stoich;
            }
            total_rate += rate;
            cross1[n] = cross2[n];
          }
        }

        for (auto& [name, stoich] : m_net_products)
          stoich /= total_rate;
        ...
      }
      ...
   }

In the above code, the photolysis cross section is stored in a 2D array ``m_crossSection``, where the first dimension
is the temperature and the second dimension is the wavelength. There are ``m_ntemp`` temperatures and
``m_nwave`` wavelengths. The ``m_temp_wave_grid`` is a one-dimensional array that stores the temperature
grid and the wavelength grid sequentially. The ``interpn`` function finds a linear interpolation of the cross section
at :math:`T=` ``data.temperature`` and :math:`\lambda=` ``data.wavelength[i]``, where ``i`` is the index of the wavelength grid.
Because we use a linear interpolation, the grid of the cross section does not need to be the same as the grid 
of photon wavelengths. It remains to say what ``PhotolysisData`` is.


PhotolysisData
~~~~~~~~~~~~~~


Reaction
~~~~~~~~


ReactionRate
~~~~~~~~~~~~


MultiRate
~~~~~~~~~


Kinetics
~~~~~~~~

``Kinetics`` is the manager class that solves a kinetic reaction network. A kinetic reaction is a chemical reaction that takes the form of:


New Photolysis Classes
----------------------


Example
-------


Cross Section Formats
---------------------
