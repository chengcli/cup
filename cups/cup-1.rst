Adding photolysis mechanism to Cantera
======================================

CUP: 1
Author: Cheng Li
Type: Feature
Created: 04-01-2024

.. toctree::
   :local:
   :depth: 2

Summary
-------

`Cantera <https://cantera.org>`_ is an open-source chemical kinetics solver written in modern C++.
It is capable of solving a wide range of chemical kinetics/transport problems, both in 0D and 1D.
However, ``Cantera`` does not natively support photochemical reactions. Adding photolsis to Cantera
requires some deep understanding of the ``Cantera`` code base. 
This CUP discusses the design of adding photolysis reactions to ``Cantera``.

Photochemical reactions
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
The ``format`` field specifies the format of the cross section files. Currently, we support two formats: ``KINETICS7``
and ``VULCAN``.

The starting point is the ``Kinetics`` class, which is the manager class that solves 
a kinetic reaction network. A kinetic reaction is a chemical reaction that takes the form of:

.. math::

    A + B + \ldots \Leftrightarrow C + \ldots

or


``Cantera`` supports two kinds of kinetic reactions, reversible and 
irreversible. The reversible reaction is indicated by the ``<=>`` symbol and the 
irreversible reaction is indicated by the ``=>`` symbol. All photolysis reactions are 
irreversible reactions. 

Important Cantera Classes
-------------------------

Reaction
~~~~~~~~


ReactionRate
~~~~~~~~~~~~


MultiRate
~~~~~~~~~


Kinetics
~~~~~~~~


New Photolysis Classes
----------------------


PhotolysisData
~~~~~~~~~~~~~~


PhotolysisRate
~~~~~~~~~~~~~~


Example
-------


Cross Section Formats
---------------------
