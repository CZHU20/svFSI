[toc]

# PETSc in svFSI

## Install PETSc Locally

- Download [PETSc](https://petsc.org/release/). The following procedure has been tested on version 3.25.2.

- Configure PETSc using the following commands. 

  ```bash
  ./configure \
      --prefix=<install_location> \
      --with-mpi-dir=<path_to_installed_mpi> \
      --with-debugging=0 \
      COPTFLAGS='-O3 -march=native -mtune=native' \
      CXXOPTFLAGS='-O3 -march=native -mtune=native' \
      FOPTFLAGS='-O3 -march=native -mtune=native' \
      --download-fblaslapack \
      --download-suitesparse \
      --download-mumps \
      --download-hypre \
      --download-superlu \
      --download-superlu_dist \
      --download-ml \
      --download-scalapack
  ```
  
- Install PETSc following the command line prompt.

- Add the following lines to `~/.bashrc`
  ```bash
  export PETSC_DIR=<install_location>
  export PETSC_ARCH=<install_arch>
  export PATH=$PETSC_DIR/bin:$PATH
  export LD_LIBRARY_PATH=$PETSC_DIR/lib:$LD_LIBRARY_PATH
  export LIBRARY_PATH=$PETSC_DIR/lib:$LIBRARY_PATH
  ```


## Compile svFSI with PETSc

- Turn on PETSc option in `Code/CMake/SimVascularOptions.cmake`

  ```bash
  option(SV_USE_PETSC "Use PETSC Library with svFSI" ON)
  ```

- Compile svFSI as usual, and you should see the following message showing up during make process.

  ```bash
  Found PETSC!  Here are the details:
     PETSC_DIR = ...
     PETSC_LIBRARY_DIRS = ...
     PETSC_INCLUDE_DIRS = ...
  End of PETSC details
  ```

## General Usage
- There are two ways to pass solver configurations to PETSc in svFSI.

  1. Through svFSI input file (very limited support).

     - Supported linear solver: GMRES, BICG, CG

     - Supported preconditioners: Jacobi (i.e. svfsi) and RCS

     You can supply the same keywords as the native linear solver (i.e. svFSILS), and they will be passed to PETSc. None of the trillions keywords will be accepted.

  2. (**Recommended**) Through a user-supplied configuration file petsc_options.inp, you can supply any standard PETSc command-line options. The following is an example using GMRES(20) with an algebraic multigrid preconditioner. The relative tolerance is set to 1e-6 and the maximum number of iterations is 100.

     ```bash
     -ksp_type gmres
     -ksp_gmres_restart 20
     -pc_type gamg
     -ksp_rtol 1e-6
     -ksp_max_it 100
     ```
     
     ==Note that you CANNOT change the file name `petsc_options.inp`. This file should be placed in the same folder as `svFSI.inp`.==
     
     Options from `petsc_options.inp` will overwrite linear solver settings in `svFSI.inp`. Any setting that's not overwritten will be passed to PETSc.

### Options for linear system solvers

- Available linear system solvers (KSP): https://petsc.org/release/docs/manual/ksp/#tab-kspdefaults
- `-ksp_pc_side <right,left,symmetric>` KSP preconditioner side.
- `-ksp_rtol <rtol>`: relative tolerance
-  `-ksp_atol <atol>`: absolute tolerance
-  `-ksp_max_it <its>`: maximum iteration counts 

### Options for preconditioners

- Available preconditioners (PC): https://petsc.org/release/docs/manual/ksp/#tab-pcdefaults

- Each preconditioner may have associated with it a set of options, which can be set with routines and options database commands provided for this purpose. Such routine names and commands are all of the form ` -pc_<type>_<option> [value]`

- Common preconditioners

  `-pc_type jacobi`: Jacobi

  `-pc_type bjacobi -sub_pc_type ilu`: Block Jacobi with ilu in each block

  `-pc_type gamg`: algebraic multigrid

  `-pc_type sor`: successive over-relaxation.

### Common `petsc_options.inp`

- Direct solver

  ```bash
  -ksp_type preonly
  -pc_type lu
  ```

  or

  ```bash
  -ksp_type none
  -pc_type lu
  ```

  Currently supported by PETSc for sequential matrices, and by several external solvers through PETSc interfaces, see [Using External Linear Solvers](https://petsc.org/release/docs/manual/ksp/#using-external-linear-solvers).

- BICGStab with ilu

  ```bash
  -ksp_type bcgs
  -pc_type bjacobi
  -sub_pc_type ilu
  ```

- CG with incomplete Cholesky

  ```bash
  -ksp_type cg
  -pc_type bjacobi
  -sub_pc_type icc
  ```

- GMRES with additive Schwarz and ilu

  ```bash
  -ksp_type gmres
  -ksp_gmres_restart 30
  -pc_type asm
  -pc_asm_overlap 1
  -sub_pc_type ilu
  ```


### Option file for multiple equations

- If multiple equations are solved consecutively, e.g. FSI + mesh, a prefix will be added to distinguish solver options for different equations. The prefix for each equation is `eq%sym` defined in [svFSI/INITIALIZE.f](https://github.com/SimVascular/svFSI/blob/94a2ff5cab2ad9e89e9bf12e0de36eaaa9b85571/Code/Source/svFSI/INITIALIZE.f#L83). For example, `eq%sym` for FSI and mesh are `FS` and `MS` respectively. The option file is

  ```bash
  -fs_ksp_type gmres
  -fs_pc_type gamg
  -ms_ksp_type cg
  -ms_pc_type jacobi
  ```

  You can also choose to omit the `ms` part and use the settings in `svFSI.inp`.
