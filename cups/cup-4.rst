Problem Generator and Unit Tests
================================

:CUP:
  4

:Author:
  Cheng Li

:Type:
  Feature

:Created:
  01-07-2017

:Builds-on:
  :ref:`CUP2 <cup2>`

.. toctree::

Overview
--------

In :ref:`CUP2 <cup2>`, we discussed that the design principle of ``Canoe`` is
to allow dissembling a big program into small pieces, and then assemble them
together in any arbitrary way to tackle another problem. In this CUP, we discuss
the concepts of **unit test** and **problem generator**. 
The former permits test small pieces of code, and the latter 
permits generating the initial condition for a new problem.


Unit Test
---------

A unit test encompasses a small piece of code that tests a single function or a
certain behavior of a module. The unit test is the starting point of assembling a
bigger program. A unit test is a crucial step to ensure that the code is
behaving as expected.

``Canoe`` uses ``gtest`` (google test) as the unit test framework.
All test subroutines are placed under ``tests/`` folder. According to the
``gtest`` convention, the name of the test subroutine should start with
``TEST_F`` for testing member functions of a class or ``TEST`` 
for testing standalone functions. For example, the following code snippet
defines a test subroutine for the interpolation function ``interp_weno5``:

.. code-block:: c++

    TEST(interp_weno5, test_case1) {
      double phim2 = 1.0;
      double phim1 = 2.0;
      double phi = 3.0;
      double phip1 = 4.0;
      double phip2 = 5.0;
      double result = interp_weno5(phim2, phim1, phi, phip1, phip2);
      double expected_result = 2.5000000000000004;
      EXPECT_NEAR(result, expected_result, 1.E-10);
    }

in which, the ``interp_weno5`` function is a function that yields the
interpolated value *at the cell interface* using the WENO5 scheme,
based on the *cell averaged quantities*, schematically denoted as::

    | x_{-2} | x_{-1} | x_0 | x_1 | x_2 |
                       ^
                       |
                       return value

The function ``EXPECT_NEAR`` is a ``gtest`` macro that checks whether the
``result`` is close to the ``expected_result`` within a tolerance of ``1.E-10``.

You can define as many test subroutines as you want, all starting with the
macro definition ``TEST``. The first argument of the ``TEST`` macro is the name
of the test subroutine, and the second argument is the name of the test case.

The main function of the unit test reads as follows:

.. code-block:: c++

    int main(int argc, char **argv) {
      testing::InitGoogleTest(&argc, argv);
      return RUN_ALL_TESTS();
    }

As the name suggests, the ``InitGoogleTest`` function initializes the ``gtest``
framework, and the ``RUN_ALL_TESTS`` function runs all the test subroutines.

After compiling the code, all unit tests will be built into executable files
under the ``build/tests/`` folder. To run the unit tests, simply type
``ctest`` in the ``build/tests`` folder. The output should look like this::

    Test project /Users/chengcli/Development/canoe/build/tests
        Start 1: test_weno.release
    1/5 Test #1: test_weno.release ................   Passed    0.02 sec
        Start 2: test_glog.release
    2/5 Test #2: test_glog.release ................   Passed    0.02 sec
        Start 3: test_absorber.release
    3/5 Test #3: test_absorber.release ............   Passed    0.07 sec
        Start 4: test_yaml_read.release
    4/5 Test #4: test_yaml_read.release ...........   Passed    0.02 sec
        Start 5: test_mesh.release
    5/5 Test #5: test_mesh.release ................***Failed    0.07 sec

    80% tests passed, 1 tests failed out of 5

    Total Test time (real) =   0.19 sec

    The following tests FAILED:
        5 - test_mesh.release (Failed)
    Errors while running CTest

The output shows that 4 out of 5 tests passed, and the test ``test_mesh.release``
failed. The ``ctest`` command also generates a log file ``Testing/Temporary/LastTest.log``
that stores the detailed information of the test.

For a more sophisticated example of unit test, take a look at the
``tests/test_impl.cpp`` file. This file contains a test class that
tests the implementation of the ``MeshBlock::Impl`` class, which
is an extension module of the ``MeshBlock`` class. The structure of the unit test
class is as follows:

.. code-block:: c++

    class TestImpl: public ::testing::Test {
    protected:
      virtual void SetUp() {
        // set up the test fixture
      }

      virtual void TearDown() {
        // tear down the test fixture
      }

      // declare member variables
    };

    TEST_F(TestImpl, TestName) {
      // test subroutine
    }

The ``SetUp`` and ``TearDown`` functions are called before and after each test
subroutine, respectively. The ``TEST_F`` macro is used to test member functions
of a class. The first argument of the ``TEST_F`` macro is the name of the test
class, and the second argument is the name of the test subroutine.
Here is a snippet of the implementation of the ``TestImpl`` class:

.. _test_impl:

.. code-block:: c++

    class TestImpl: public ::testing::Test {
     protected:
      ParameterInput *pinput;
      Mesh *pmesh;

      virtual void SetUp() {
        IOWrapper infile;
        infile.Open("test_thermodynamics.inp", IOWrapper::FileMode::read);

        pinput = new ParameterInput;
        pinput->LoadFromFile(infile);

        ...

        IndexMap::InitFromAthenaInput(pinput);
        Thermodynamics::InitFromAthenaInput(pinput);

        int restart = false;
        int mesh_only = false;
        pmesh = new Mesh(pinput, mesh_only);
        ...
        pmesh->Initialize(restart, pinput);
      }

      virtual void TearDown() {
        delete pinput;
        delete pmesh;
        Thermodynamics::Destroy();
        IndexMap::Destroy();
      }
    }

    TEST_F(TestImpl, GatherPrimitive) {
      // test subroutine
      ...
    }

    TEST_F(TestImpl, GatherConserved) {
      // test subroutine
      ...
    }

In the above code snippet, the ``SetUp`` function reads the input file
``test_thermodynamics.inp`` and initializes the ``pinput`` and ``pmesh``
variables. The ``TearDown`` function deletes the ``pinput`` and ``pmesh``
variables and destroys the ``Thermodynamics`` and ``IndexMap`` classes.

The ``TEST_F`` macro defines two test subroutines ``GatherPrimitive`` and
``GatherConserved``, which test the procedures of gathering the primitive
and conserved variables, respectively. A bigger program can thus be assembled
by combining incrementally more test subroutines.

Finally, the instructions for compiling the unit tests are specified in the
``tests/CMakeLists.txt`` file:

.. code-block:: cmake

    setup_test(test_weno)
    ...
    if (${NVAPOR} EQUAL 2)
      if (${NCLOUD} EQUAL 4)
        setup_test(test_impl)
        ...
      endif()
    endif()

The ``setup_test`` is a predefined macro that sets up the compilation of the
unit test. The argument of the ``setup_test`` macro should be the name of the
cpp file that contains the test subroutines. Be aware that certain unit tests
may require specific compilation flags. For example, the ``test_impl`` unit
test requires the ``NVAPOR`` variable to be 2 and the ``NCLOUD`` variable to
be 4. The ``setup_test`` macro checks whether the ``NVAPOR`` and ``NCLOUD``
variables satisfy the requirements. If not, the unit test will not be compiled.

Problem Generator
-----------------

Once smaller functions are tested, the next step is to assemble them into
the solver that ``Canoe`` provides. The **problem generator** is a file that
generates the initial condition for a specific problem.
Examples of problem generators can be found under the ``examples/`` folder.
Each example problem is a folder that may contain the following files:
  
  - ``<problem>.cpp``: the problem generator
  - ``<problem>.inp``: the input file for the problem
  - ``<problem>.yaml``: the configuration file for the problem

We will leave the discussion of the input file and the configuration file
to other CUPs. Here we focus on the problem generator.
An example snippet of the problem generator is shown below:

.. code-block:: c++

    void MeshBlock::ProblemGenerator(ParameterInput *pin) {
      ...
      // construct atmosphere from bottom up
      for (int k = ks; k <= ke; ++k)
        for (int j = js; j <= je; ++j) {
          air.SetZero();
          air.w[iH2O] = xH2O;
          air.w[IPR] = Ps;
          air.w[IDN] = Ts;
          ...
          for (int i = is; i <= ie; ++i) {
            air.w[IVX] = 1. * sin(2. * M_PI * rand() / RAND_MAX);
            AirParcelHelper::distribute_to_conserved(this, k, j, i, air);
            pthermo->Extrapolate(&air, pcoord->dx1f(i),
                                 Thermodynamics::Method::PseudoAdiabat, grav,
                                 1.e-5);
          }
          ...
          pimpl->prad->CalFlux(this, k, j, is, ie + 1);
        }
    }

The ``ProblemGenerator`` function is a member function of the ``MeshBlock``,
which is a high-level manager class that manages the data and the operations
on a mesh grid. The ``ProblemGenerator`` function is automatically called
inside

.. code-block:: c++

    pmesh->Initialize(restart, pinput);

function of the ``Mesh`` class (see the code snippet in the :ref:`TestImpl <test_impl>` class).

Inside the body of the ``ProblemGenerator`` function, the initial condition
is constructed by setting the conserved variables of each cell. The
``AirParcelHelper::distribute_to_conserved`` function converts the primitive
variables to the conserved variables. The ``pthermo->Extrapolate`` function
extrapolates the primitive variables to another level along a pseudo-adiabat.
At the end, the ``pimpl->prad->CalFlux`` function calculates the radiation
fluxes from level ``is`` to ``ie + 1`` at the ``k``-th and ``j``-th cell.

Similar to the unit tests, the problem generator is compiled by adding
the following line to the ``CMakeLists.txt`` file when building the
executables of the ``Canoe``:

.. code-block:: cmake

    setup_problem(<problem>)

where ``<problem>`` is the name of the problem generator file without the
``.cpp`` extension.
