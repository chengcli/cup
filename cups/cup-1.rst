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

in which the ``=>`` symbol indicates that the reaction is irreversible, and A, B, C are
the reactants and products of the reaction. A is the parent molecule that absorbs the photon
and B, C are the daughter molecules, also called the photofragments. Sometimes,
absorption of a photon does not lead to the dissociation of the parent molecule. It is legit to write 
the reaction as:

.. math::

    \text{A} \Rightarrow \text{A},

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

.. _yaml_input:

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

.. _photolysis_rate:

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

In the above code, the photolysis cross section is stored in a 3D array ``m_crossSection``, where the first dimension
is the temperature, the second dimension is the wavelength, and the third dimension is the branch of the photolysis reaction.
There are ``m_ntemp`` temperatures and ``m_nwave`` wavelengths. 
The ``m_temp_wave_grid`` is a one-dimensional array that stores the temperature
grid and the wavelength grid sequentially. The ``interpn`` function finds a multi-linear interpolation of the cross sections
at :math:`T=` ``data.temperature`` and :math:`\lambda=` ``data.wavelength[i]``, where ``i`` is the index of the wavelength grid,
for all branches of the photolysis reaction. The ``cross1`` and ``cross2`` arrays store the interpolated cross sections.
Because we use an interpolation, the grid of the cross section does not need to be the same as the grid 
of photon wavelengths. 

A biproduct of the rate calculation is the effective stoichiometric coefficients of the photofragments, which are stored in the
``m_net_products`` map. The ``m_net_products`` map is a map from the name of the photofragment to its effective stoichiometric
coefficient. The effective stoichiometric coefficient is defined as the ratio of the number of photofragments produced to the
number of photons absorbed. The ``m_branch`` vector stores the stoichiometric coefficients of the
photofragments for each branch of the photolysis reaction. Each element in the ``m_branch`` vector is a map, which
maps the name of the photofragment to its stoichiometric coefficient. The ``m_branch`` vector is initialized in the
``PhotolysisRate`` constructor from the ``branches`` fields in the :ref:`YAML input file <yaml_input>`.
Later, the ``m_net_products`` values are passed to a higher level class, ``Kinetics``,
to calculate the net production rate of the photofragments. Finally, it remains to say what ``PhotolysisData`` is.


PhotolysisData
~~~~~~~~~~~~~~

We discussed briefly that ``Cantera`` groups reactions with the same expression of the rate coefficient to speed up the evaluation.
In the :ref:`code snippet <photolysis_rate>` of the ``PhotolysisRate`` class, we see 

.. code-block:: C++

    unique_ptr<MultiRateBase> newMultiRate() const override {
        return make_unique<MultiRate<PhotolysisRate, PhotolysisData>>();
    }

does the trick. When ``newMultiRate`` is called, it creates a ``MultiRate`` object that takes two template parameters,
``PhotolysisRate`` and ``PhotolysisData``. The ``PhotolysisData`` class is a struct that stores all the 
**shared information** used by the ``PhotolysisRate`` class. In a photolysis reaction, the shared information
across all photolysis reactions includes the temperature, the photon actinic flux, and the wavelength. As a result,
the ``PhotolysisData`` class is defined as:

.. code-block:: C++

    struct PhotolysisData : public ReactionData {
        ...
        vector<double> actinicFlux;
        vector<double> wavelength;
    };

We do not need to explicitly store the temperature because it is already stored in the base class ``ReactionData``.
Looking again at the :ref:`code snippet <photolysis_rate>` of the ``PhotolysisRate`` class:

.. code-block:: C++

    double evalFromStruct(PhotolysisData const& data) {
        ...
        double coord[2] = {data.temperature, data.wavelength[iwmin]};
        ...
    }

the ``evalFromStruct`` method of the ``PhotolysisRate`` class takes a ``PhotolysisData`` object as an argument
and access the temperature and the wavelength from the ``data`` object.


MultiRate
~~~~~~~~~

Though the rate evalution method is defined in the ``PhotolysisRate`` class, the execution is **not** done
by the ``PhotolysisRate`` class because of the ``MultiRate`` evaluation.
The ``MultiRate`` class is a template class that takes two template parameters, ``RateType`` and ``DataType``:

.. code-block:: C++

    template<class RateType, class DataType>
    class MultiRate : public MultiRateBase {
        CT_DEFINE_HAS_MEMBER(has_update, updateFromStruct)
        CT_DEFINE_HAS_MEMBER(has_ddT, ddTScaledFromStruct)
        CT_DEFINE_HAS_MEMBER(has_ddP, perturbPressure)
        CT_DEFINE_HAS_MEMBER(has_ddM, perturbThirdBodies)
        ...
      protected:
        ...
        vector<pair<size_t, RateType&>> m_rxn_rates;
        DataType m_shared;
    }

``PhotolysisRate`` and ``PhotolysisData`` are specializations of ``RateType`` and ``DataType``, respectively.
The higher level manager class, ``Kinetics``, calls the ``newMultiRate`` method of the ``PhotolysisRate`` class
that returns a ``MultiRate`` object. The ``MultiRate`` object stores the pointer to the ``RateType`` object
in ``m_rxn_rates`` and and the shared data, a ``DataType`` object, at ``m_shared`` for all reactions 
belonging to the same ``RateType``. Thus, a ``MultiRate`` object can evaluate the rate of all reactions 
belonging to the same ``RateType`` via the ``m_rxn_rates`` pointers. To facilitate such
mechanism, the ``RateType`` class must have a method called ``updateFromStruct`` that takes a ``DataType`` object.
This is what we have done for the ``PhotolysisRate`` class.


Reaction
~~~~~~~~

A ``ReactionRate`` object could be manipulated by two classes, ``MultiRate`` and ``Reaction``. This may cause
potential problems when one class modifies the ``ReactionRate`` object and the other class is not aware of it.
See a discussion of `issue #1211 <https://github.com/Cantera/cantera/issues/1654>`_.
The ``Reaction`` class stores a ``shared_ptr`` to the ``ReactionRate`` object:

.. code-block:: C++

    class Reaction {
      ...
      protected:
        shared_ptr<ReactionRate> m_rate;
        ...
    }

while the ``MultiRate`` template class stores a reference to the ``ReactionRate`` object.
Both of them point to the same ``ReactionRate`` object. 

.. note::

    The ``Cantera`` official repo opts to make a copy of the ``ReactionRate`` object in the ``MultiRate`` class
    such that the ``ReactionRate`` object is not shared by the two classes.

Sharing the ``ReactionRate`` object is crucial for implementing the photolysis mechanism because
the reaction rate calculation is performed by the ``MultiRate`` which must pass the
effective stoichiometric coefficients back to the ``ReactionRate`` object.

Passing the effective stoichiometric coefficients is done in the ``updateROP`` method
(update the rate of progress) of the ``Kinetics`` class:

.. code-block:: C++

    void BulkKinetics::updateROP() {
        ...
        for (auto i : m_photo_index) {
          if (m_reactions[i]->products.size() > 1) {
            modifyProductStoichCoeff(i, m_reactions[i]->rate()->photoProducts());
          }
        }
    }

The above code snippet loops over all photolysis reactions and checks if there is more than one photolysis product.
If so, it calls the ``modifyProductStoichCoeff`` method of the ``Kinetics`` class to modify the effective
stoichiometric coefficients of the photofragments. The ``photoProducts`` method of the ``ReactionRate`` class
returns ``m_net_products`` calculated by the ``MultiRate`` class. Thus, the ``Reaction``
class and the ``MultiRate`` class must share the same ``ReactionRate`` object.


Kinetics
~~~~~~~~

The entire reaction network is managed by the ``Kinetics`` class. 
Specific kinetics implementations, such as ``GasKinetics`` and ``InterfaceKinetics``, are derived from 
the base class. Here are code snippets of the ``BulkKinetics`` class, which implements
kinetics models for gas phase reactions and its base class, ``Kinetics``:

.. code-block:: C++

    class Kinetics : public KineticsBase
    {
      ...
      protected:
        vector<shared_ptr<Reaction>> m_reactions;
        ...
    }

    class BulkKinetics : public Kinetics
    {
      ...
      protected:
        vector<unique_ptr<MultiRateBase>> m_bulk_rates;
        ...
    }


The ``Kinetics`` class is the entry point to read the reaction network from the input
YAML file. The following code snippet shows how to create a ``Kinetics`` object:

.. _kinetics_input:

.. code-block:: C++

    auto phase = newThermo("ch4_photolysis.yaml");
    auto kin = newKinetics({phase}, "ch4_photolysis.yaml");

The first line creates a ``ThermoPhase`` object from the YAML file. 
The second line creates a ``Kinetics`` object.
During the construction of the ``Kinetics`` object, the ``Kinetics`` class reads the reaction network
from the YAML file and creates ``Reaction`` objects for each reaction.
The ``Reaction`` object creates a ``ReactionRate`` object based on the reaction type.
The ``Kinetics`` class also creates a ``MultiRate`` object for reaction rate evaluation.
It makes sure that the ``Reaction`` object and the ``MultiRate`` object share the same ``ReactionRate`` object.


Examples
--------

Continuing from the previous :ref:`code snippet <kinetics_input>`,
the following example shows how to calculate the forward rate constants of a photolysis reaction:

.. code-block:: C++

    PhotochemTitan() {
      ...
      // set the initial state
      phase->setState_TPX(200.0, OneAtm, "CH4:0.02 N2:0.98");

      // set wavelength
      vector<double> wavelength(10);
      vector<double> actinic_flux(10);

      for (int i = 0; i < 10; i++) {
        wavelength[i] = 20.0 + i * 20.0;
        actinic_flux[i] = 1.0;
      }

      kin->setWavelength(wavelength.data(), wavelength.size());
      kin->updateActinicFlux(actinic_flux.data());

      vector<double> kfwd(kin->nReactions());
      kin->getFwdRateConstants(kfwd.data());
    }

``setWavelength`` and ``updateActinicFlux`` are new methods of the ``Kinetics`` class specifically
for calculating photolysis reactions. This example sets the wavelength and actinic flux manually.
But later, they should be provided by a radiation model of the atmosphere.
