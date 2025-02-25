# smartsim-openFOAM

This repository provides instructions and source files
for an example of including the SmartRedis client in
OpenFOAM so that OpenFOAM can leverage the machine learning,
data analysis, and data visualization capabilities of SmartSim.

The example in this repository adapts the
[TensorFlowFoam](https://github.com/argonne-lcf/TensorFlowFoam)
work which utilized a deep neural network to predict steady-state
turbulent viscosities of the Spalart-Allmaras (SA) model.
The example in this repository highlights that a machine learning
model can be evaluated using SmartSim from within an OpenFOAM object
with minimal external library code.  In this particular case,
only four SmartRedis client API calls are needed to
initialize a client connection,
send tensor data for evaluation, execute the TensorFlow model,
and retrieve the model inference result.  In the following
sections, instructions will be given for building and running
the aforementioned example.


![TFF-SmartSim-inference](https://user-images.githubusercontent.com/13009163/135544687-ea219754-3c79-4140-987b-8f0c81020cea.png)

Absolute value of the difference between the ML inferred value of eddy viscosity using SmartSim vs the Spalart Allmaras (SA) calculated value from the original TensorFlowFoam project repository. The geometry upstream from the step has been truncated where the error is lowest.

## Prerequisites

### OpenFOAM-5.x and ThirdParty-5.x

In this example, a deep neural network is used to predict steady-state
turbulent viscosities of the Spalart-Allmaras (SA) model.  The
evaluation of the deep neural network is incorporated into OpenFOAM
via a custom turbulence model that is dynamically linked to the
simulation.  As a result, the base OpenFOAM-5.x and ThirdParty-5.x
applications can be built using typical build instructions.  However,
in the case that a system (e.g. Cray XC) is being used that is
not listed in the OpenFOAM make options, check the ``README`` of the
``builds`` directory of this repository for custom build instructions
and files.

### SmartSim and SmartRedis

An installation of SmartSim is required to run this example.
SmartSim can be installed via ``pip`` or from source using the public
[installation instructions](https://www.craylabs.org/docs/installation.html#smartsim).

An installation of SmartRedis is also required to run this example.
The SmartRedis Python client, which is used in the ``driver.py`` file
to place the machine learning model in SmartSim orchestrator, can
be installed via ``pip`` or as an editable install from source using
the public
[installation instructions](https://www.craylabs.org/docs/installation.html#smartredis).
The SmartRedis C++ client library is also required and can be built and installed
using the public
[installation instructions](https://www.craylabs.org/docs/installation.html#smartredis).

### SimpleFoam_ML

As part of the [TensorFlowFoam](https://github.com/argonne-lcf/TensorFlowFoam)
work, the OpenFOAM program ``simpleFoam_ML`` was written to
call a custom turbulence model that utilizies a TensorFlow model
at the beginning of the simulation and then the turbulent
eddy-viscosity is held constant for the rest of the simulation.
In this example, ``simpleFoam_ML`` is also used. To build
the ``simpleFoam_ML`` executable, execute the following commands:

```bash
cd /path/to/OpenFOAM-5.x
source etc/bashrc
cd /path/to/smartsim-openFOAM/simpleFoam_ML
wclean && wmake
```

The executable for SimpleFoam_ML
will be installed in a subdirectory of the
directory referenced by the environment variable
``FOAM_APPBIN``.  Before proceeding, verify
that the ``simpleFoam_ML`` executable
exists in the aformentioned directory.

### SA_Detailed turbulence model

The OpenFOAM cases that are used to generate training data utilize a custom turbulence model for enhanced data output.  To build this turbulence model, execute the following commands:

```bash
cd /path/to/OpenFOAM-5.x
source etc/bashrc
cd /path/to/smartsim-openFOAM/turbulence_models/SA_Detailed
wmake .
```

After executing these commands, verify that ``SA_Detailed.so`` is in ``$FOAM_USER_LIBBIN``.

### ML_SA_CG turbulence model with SmartRedis

The TensorFlow machine learning model is incorporated into the OpenFOAM
simulation via a custom turbulence model built as a dynamic library.
The directory ``turbulence_models/ML_SA_CG`` in this repository
contains the source files
for the custom turbulence model and the files needed to build the
associated dynamic library.  These files have been adapted from the
[TensorFlowFoam](https://github.com/argonne-lcf/TensorFlowFoam)
``OF_Model``.

To build the dynamic library, execute the following commands:

```bash
export SMARTREDIS_PATH=path/to/SmartRedis
cd /path/to/OpenFOAM-5.x
source etc/bashrc
cd /path/to/smartsim-openFOAM/turbulence_models/ML_SA_CG
wmake libso .
```

Note that the environment variable ``SMARTREDIS_PATH`` is used
to locate the ``SmartRedis`` include and install directories
and should point to the top level ``SmartRedis`` directory.

The dynamic library for the custom turbulence model
will be installed in a subdirectory of the
directory referenced by the environment variable
``FOAM_USER_LIBBIN``. Before proceeding, verify
that the ``ML_SA_CG.so`` file
exists in the aformentioned directory.

## Running OpenFOAM with SmartSim

A ``SmartSim`` script (``driver.py``) is provided in this
repository to execute the OpenFOAM case with
the TensorFlow model.  The aforementioned ``SmartSim``
script will deploy a ``SmartSim`` ``Orchestrator``,
generate training data, train the machine
learning model, and run the OpenFOAM simulation.
The results of the simulation
will be in the experiment directory ``openfoam_ml``.

On machines with the Slurm workload manager,
``run.sh`` can be used to execute the SmartSim script
``driver.py``.  Parameters inside of ``run.sh`` can
be modified to fit compute resource  availability.
The variables ``OF_PATH`` and ``SMARTREDIS_LIB_PATH``
must be set to the correct directories before
executing ``run.sh``.  To run ``run.sh``, execute:

```bash
sbatch run.sh
```

On machines with the Cobalt workload manager (e.g. Theta),
``run-theta.sh`` can be used to execute the SmartSim script
``driver.py``.  Parameters inside of ``run-theta.sh`` can
be modified to fit compute resource  availability.
The variables ``OF_PATH`` and ``SMARTREDIS_LIB_PATH``
must be set to the correct directories before
executing ``run-theta.sh``.  To run ``run-theta.sh``, execute:

```bash
qsub run-theta.sh
```

## Notes on the inclusion of SmartRedis in OpenFOAM

In this section, a brief description
of the ``SmartRedis`` code that was added to
OpenFOAM is presented.  With the notes herein,
users should be able to include SmartRedis in any
part of the OpenFOAM code.

Aside from the addition of an include statement in
``ML_SA_CG.H``, all of the ``SmartRedis``
content is in ``ML_SA_CG.C``.

In line 447 of ``ML_SA_GC.C``, the ``SmartRedis``
client is initialized to utilize a ``SmartSim``
``Orchestrator`` in cluster configuration.  To
use to a non-cluster ``Orchestrator``,
simply change the boolean input to the constructor
to false.

```c++
  // Initialize the SmartRedis client
  SmartRedis::Client client(true);
```

In line 449 through line 458, a key is contructed
for the tensor that will be sent to the database.
This logic creates the tensor key ``model_input_rank_``
that is suffixed by the MPI rank if the
``MPI_COMM_WORLD`` has been initialized.  Using
the MPI rank as a suffix is a simple and convenient
way to prevent key collisions in a parallel application.


```c++
    // Get the MPI rank for key creation
    int rank = 0;
    int init_flag = 0;

    MPI_Initialized(&init_flag);
    if(init_flag==1)
        MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    // Create a key for the tensor storage
    std::string input_key = "model_input_rank_" + std::to_string(rank);
```

In line 460, the tensor is put into the ``SmartSim`` ``Orchestrator``
using the ``put_tensor()`` API call.  The ``input_vals`` variable
is a tensor that contains the velocity, position, and step height
for each point in the mesh.  The ``input_dims`` is a vector
containing the ``input_vals`` dimensions.  Finally, the
last two variable inputs to ``put_tensor()`` indicate the data
type and memory layout associated with the ``input_vals`` variable.

```c++
    // Put the input tensor into the database
    client.put_tensor(input_key, input_vals, input_dims,
                      SmartRedis::TensorType::flt,
                      SmartRedis::MemoryLayout::nested);
```

In line 466 through 469, a key is constructed for the
tensor output of the ML model, and the TensorFlow model
that is stored in the database is executed.

```c++
    // Run the model
    std::string model_key = "ml_sa_cg_model";
    std::string output_key = "model_output_rank_" + std::to_string(rank);
    client.run_model(model_key, {input_key}, {output_key});
```

Finally, in line 473, the tensor output of the
TensorFlow model is retrieved and stored in a vector
named ``data``.  The API function ``unpack_tensor()``
is used to place the tensor output data into an
existing block of memory, whereas the API function
``get_tensor()`` can be used to place the results
into a block of memory managed by the ``SmartRedis``
client.  Following the retrieval of the
TensorFlow model ouptut tensor, the data can
then be used in OpenFOAM simulation.

```c++
    // Get the output tensor
    std::vector<float> data(num_cells, 0.0);
    client.unpack_tensor(output_key, data.data(),
                         {num_cells},
                         SmartRedis::TensorType::flt,
                         SmartRedis::MemoryLayout::contiguous);
```
