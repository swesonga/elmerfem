# TemperateIce Solver
## General Information
- **Solver Fortran File:** ``TemperateIce.F90``
- **Solver Name:** ``TemperateIceSolver``
- **Required Output Variable(s):** ``Temp`` [user defined]
- **Required Input Variable(s):** ``Flow Solution Name = Flow Solution`` [else ``Flow Solution`` is default]
- **Optional Output Variable(s):** ``Temp Homologous``, ``Temp Residual`` [heading name (here ``Temp``) must coincide with variable name]
- **Optional Input Variable(s):** Variable contianing Deformational Heat [in Source]
- **Solver Keywords:** 
  - ``Apply Dirichlet`` (Logical) True [switch in lower/uper limit contraint]
  - ``Temp Upper Limit`` (Real) PressureMeltingPoint [in corresponding ``Material`` section]
  - ``Temp Lower Limit`` (Real) 0.0 [in corresponding ``Material`` section]
  - ``Temp Volume Source`` (Real) DeformationalHeat  [in corresponding ``Body Force`` section]
## General Description
This solver treats the heat transfer problem with respect to an upper limit of the temperature (usually with ice the pressure-melting point, <img src="https://render.githubusercontent.com/render/math?math=T\le\,T_{\text{pm}}" title="Temperature below pressure melting">). Optionally, such a limit (and furthermore also a lower limit, e.g., T > 0 K) is introduced by solving the consequent variational inequality problem using an algorithm that - in comparison to the free surface problem - can be interpreted as a contact problem solver. In case of temperature, it basically introduces additional heat sinks/sources in order to comply with the constraints.

The volumetric heat source term can be estimated from the ice flow deformational heat using the [DeformationalHeat](./DeformationalHeat.md) Solver.

### Looping option
It is possible in some cases that the nonlinear convergence tolerance can be reached before all nodes have been properly constrained. At this point the solver can exit with some nodes retaining temperatures above the upper limit. A new option was added to check for nodes with temperatures above the limit and to continue looping if such nodes exist. Using this option may cause more iterations but should ensure the upper limit is properly applied at all nodes. To use it add the following line to the temperate ice solver section in the sif file (default is False):

``Loop While Unconstrained Nodes = Logical True``

## Known bugs
- timestep was not initialized; caused excessive heating if transient simulations with more than one iteration at the steady state level were done. Fixed June 2012

## SIF contents
The required keywords in the SIF file for this solver are given below. The MATC functions used here are explained and given on [this page](http://elmerfem.org/elmerice/wiki/doku.php?id=tips:thermoprop). If doing performance critical simulations, we though recommend to stick to ready implemented Fortran versions documented in [IceProperties](../../UserFunctions/Documentation/IceProperties.md)

```
! Units : MPa - m - yr
$yearinsec = 365.25*24*60*60

!Compute the heat generated by ice deformation
Solver 3
  Equation = DeformationalHeat
  Variable = W
  Variable DOFs = 1

  procedure =  "ElmerIceSolvers" "DeformationalHeatSolver"

  Linear System Solver = direct
  Linear System direct Method = umfpack
End

Solver 4
  Equation = String "Homologous Temperature Equation"
  Procedure =  File "ElmerIceSolvers" "TemperateIceSolver"
  ! Comment next line in parallel, as EliminateDirichlet does
  ! not work in parallel
  !------------------------------------------------------------
  Before Linsolve = "EliminateDirichlet" "EliminateDirichlet"
  Variable = String "Temp"
  Variable DOFs = 1
  Linear System Solver = "Iterative"
  Linear System Iterative Method = "BiCGStab"
  Linear System Max Iterations = 500
  Linear System Convergence Tolerance = 1.0E-07
  Linear System Abort Not Converged = True
  Linear System Preconditioning = "ILU0"
  Linear System Residual Output = 1
  Steady State Convergence Tolerance = 1.0E-04
  Nonlinear System Convergence Tolerance = 1.0E-05
  Nonlinear System Max Iterations = 50
  Nonlinear System Relaxation Factor = Real 9.999E-01
  ! uses the contact algorithm (aka Dirichlet algorithm)
  !-----------------------------------------------------
  Apply Dirichlet = Logical True
  Stabilize = True
  ! those two variables are needed in order to store
  ! the relative or homologous temperature as well
  ! as the residual
  !-------------------------------------------------
  Exported Variable 1 = String "Temp Homologous"
  Exported Variable 1 DOFs = 1
  Exported Variable 2 = String "Temp Residual"
  Exported Variable 2 DOFs = 1
End

Body Force 1
  ...
  Temp Volume Source = Equals W ! The volumetric heat source 
End

Material 1
   ...
   ! the heat capacity as a MATC function of temperature itself
   !-----------------------------------------------------------
   Temp Heat Capacity = Variable Temp
    Real MATC "capacity(tx)*yearinsec^2"
   ! the heat conductivity as a MATC function of temperature itself
   !--------------------------------------------------------------
   Temp Heat Conductivity = Variable Temp
    Real MATC "conductivity(tx)*yearinsec*1.0E-06"
   ! Upper limit - pressure melting point
   !  as a MATC function of the pressure (what else?)
   !-------------------------------------------------
   Temp Upper Limit = Variable Pressure
         Real MATC "pressuremeltingpoint(tx)"
   ! lower limit (to be save) as 0 K
   !--------------------------------
    Temp Lower Limit = Real 0.0
End

!Upper surface
Boundary Condition 1
   ...
  Temp = Real 263.15
End

!Bedrock
Boundary Condition 2
  ...
  !-------------------
  ! geothermal heatflux
  !--------------------
  Temp Flux BC = Logical True
  Temp Heat Flux = Real $56.05E-03*yearinsec*1.0E-6
  !-------------------
  ! frictional heat
  !--------------------
  Temp Load = Variable Velocity 1
    Real Procedure  "ElmerIceUSF" "getFrictionLoads"
End
```
See also [Thermodynamic Properties](http://elmerfem.org/elmerice/wiki/doku.php?id=tips:thermoprop) for heat capacity and conductivity functions.

## Examples
Examples demonstrating the use of the thermal properties of ice can be found in [ELMER_TRUNK]/elmerice/Tests/{TemperateIceTest,TemperateIceTestFct,TemperateIceTestTrans}


## Reference
When used this solver can be cited using the following reference:
Zwinger T. , R. Greve, O. Gagliardini , T. Shiraiwa and M. Lyly, 2007. A full Stokes-flow thermo-mechanical model for firn and ice applied to the Gorshkov crater glacier, Kamchatka. Annals of Glaciol., 45, p. 29-37.
