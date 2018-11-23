# HiLeMMS Definition

##  Assumptions

#### Regarding capabilities

1. The Cartesian grid method (CGM) is assumed to be employed to describe the geometry of either embedded solid bodies or envelops of flow field. To reduce memory cost, multi-block technique might be employed if possible. In the future we might be able to support other technique, e.g., finite difference LBM with multi-block structure mesh. 
2. By allowing the flexibility in defining macroscopic variables, components (i.e., distribution function), equilibrium term, force term and relaxation times as needed, most of requirements are satisfied for research works in the lattice Boltzmann modelling. 
3. The equilibrium and force terms, and relaxation time are assumed to be dependent on distributions, macroscopic variables and their gradients. Extra single parameters can be necessary for cases such as external force. 
4. In many cases, it is necessary to allow users to define their own functions for calculating gradients of macroscopic variables. 

#### Regarding implementation

1. The central idea here is to provide a unified set of facilities for a user or an application developer to utilise existing efforts, e.g., adaptive mesh refinement capability provided by a few libraries.

2. Except for a few enumerated types,  we may not need to define and/or allocate actual data structures as the backend code will be responsible for actually allocating the memory.  A wrapper or code generation tool will be response for translating the requirement to the backend code.

3. For allowing an application developer  to develop new functionalities as described by the above assumptions, it is sufficient to define  a kernel function that will be executed for each computational grid in the physical/particle velocity space. A warp function will call this kernel function and specify the range for it. 

4. We may need to have two types of wraps functions, one will go through each spatial grid, the other will also loop over the particle velocity space. 

5. Four elementary indexation may be necessary for defining kernel function, i.e., indexation over the particle velocity space, $\xi=(c_x,c_y,c_z)$, physical space, $\Omega=(x,y,z)$,  the indexation over the component space $\mathcal{C}$, and the indexation over the macrosopic ID space $\mathcal{M}$.

6. The latter two kind of indexation schemes are due to the flexibility of allowing an application developer to define components and macroscopic etc. A backend code including its wrapper will not know these information beforehand, thus cannot know which component a scheme will operate on, the application developer has to use the indexation mechanism to pass the exact information to the backend code.

7. It may be better not to overuse the sophisticated syntax of C++, i.e., OOP and template considering that they may need to be parsed by very advanced code generation tool. Due to the same reason, wrapper may be preferred over code generation.  

   


## API definition

####  First version : Interface

In this version, we will aim to implement a IDSL which provides a unified interface and necessary wrappers for various backend codes. These backend codes may cover different mesh techniques. 

**We note that, although there appears no advanced ingredients in this version (cf. the second version), the task could be not easy if a backend code does not allow the flexibility stated by Capability 2**. If this is the case,  considered amount of efforts will be needed to either modify the backend code or develop the wrapper/code generation tool.

Meanwhile, if the capability 2 is realised, we will obtain a natural foundation for implementing the ingredients defined in the Version 2. 

```c++
void DefineCase(std::string caseName, const int spaceDim)
//caseName: case name
//spaceDim: 2D or 3D application
```

**Description:** The case name will be used as the basis for naming the input file, intermediary files (if any) and output files. 

```c++
void DefineProblemDomain(const int blockNum,
                         const int* blockSize)
//blockNum:total number of blocks.
//blockSize: array of integers specifying the block blockSize
```

**Description:** Defining a "problem domain" comprising of blocks which enclose fluids and other embedded solid bodies if any. A simplest strategy for LBM is to cut a single block and construct the problem domain, which will be supported in the first instance.  In some cases, using multi-block technique could reduce computational costs.

```c++
void DefineBlockBoundary(int blockIndex,
                         BoundarySurface surface,
                         BoundaryType type,
                         Real* value)
//blockIndex: block index
//surface: which surface to set.
//type: boundary condition type
//value: Specified value for the boundary condition
void DefineBlockBoundary(int blockIndex,
                         BoundarySurface surface,
                         BoundaryType type,
                         void* boundaryFunc,
                         int* parameterList)
//boundaryFunc: pointing to a function calculating the
//boundary value
//parameterList: ID list of macroscopic variables, which
//is needed by the boundaryFunc
```

**Description:** Defining the block boundary condition with fixed values. Two enumerated types are concerned: BoundarySurface type distinguishing surface types, including various corner types and BoundaryType distinguishing boundary condition types. For the second version trying to support variable boundary condition, we assume the value depends on macroscopic variable, coordinates and time.

```c++
void AddEmbeddedBody(SolidBodyType type,
                     Real* centrePos,
                     Real* controlParas)
//type: Circle/Sphere,Ellipse/Ellipsoid, superquadrics, ...
//centerPos: the position vector of centre point
//controlParas: control parameters,
//e.g., radius for Circle/Sphere,...
void AddEmbeddedBody(int vertexNum, Real* vertexCoords)
//Add 2D polygon
//vertexNum: total number of vertexes
//vertexCoords: Coordinates of each vertex
void AddEmbeddedBody(std::string STLFileName)
//Add 3D polyhedron through STL file
```

**Description:** Adding embedded solid bodies into the computational domain. 

```c++
void DefineEmbeddedBoundary(BoundaryType type, Real* value)
```

**Description:** Defining the type of boundary condition for embedded solid body. 

```c++
void DefineComponents(std::vector<std::string> compoNames,
                      std::vector<int> compoId,
                      std::vector<std::string> lattNames)
//compoNames: A vector of string containing component names
//The vector size implies the number of components.
//compoId: A vector containing the component IDs
//must be consecutive and starting from 0
//lattNames: lattice structure associated with each component
void DefineComponents(std::vector<std::string> compoNames,
                      std::vector<int> compoId,
                      int lattDim,
                      std::vector<std::vector<Real>> weights,
                      std::vector<std::vector<Real>> lattPts)
//lattDim: the dimension of lattice structure
//weighs and lattPts as indicated by name.
```

**Description:**  Define the component needed for the simulation. For multi-phase flows or even some thermal flows, we need to use more than one component in general. 

```c++
void DefineMacroVars(VariableTypes types,
                     std::vector<std::string> names,
                     std::vector<int> varId,
                     std::vector<int> compoId)
//types: types of macroscopic variables
//names: A vector containing names of macroscopic variables
//varId: A vector containing the ID of variables
//compoId: which component the variables belong to
```

**Description:** Define the macroscopic variables needed in the simulation. By specifying types, a backend code will provide default algorithms of calculating them (e.g., density velocity etc.). 

```c++
DefineEquilibriumTerm(EquilibriumType types,
                      std::vector<int> compoId)
//types: which kind of equilibrium function to use
//compoId: which component to act on
```

**Description:** Define the equilibrium which can be calculated by using algorithms provided by the backend code. 


```c++
DefineForceTerm(ForceType types,
                std::vector<int> compoId)
//types: which kind of force function to use
//compoId: which component to act on
```
**Description:**  Define the force term which can be calculated by using algorithms provided by the backend code. 


```c++
Iterate(SchemeType scheme,
        const int steps,
        const int checkPointPeriod)
//scheme: which scheme to use, for implementing such as
//finite difference scheme.
Iterate(SchemeType scheme,
        const Real convergenceCriteria,
        const int checkPointPeriod)

```

**Description:** Two versions of wraps for running transient or steady simulations.  This wrap will be running all the necessary steps. 

### Second version: Customizable ingredients

In this version, we will implement the functionalities allowing customisable ingredients including equilibrium term, force terms etc. 

####  Variables and Indexing space

According to the assumptions  Capability 2 and 3, and those of implementation, we can have the following type of variables and the indexation needed by them.

|            Variable Type             | $\xi$ | $\Omega$ | $\mathcal{C}$ | $\mathcal{M}$ | Index Type |
| :----------------------------------: | :---: | :------: | :-----------: | :-----------: | :--------: |
|   $f$, $f^{eq}$, $g$ (force term)    |   ✔︎   |    ✔︎     |       ✔︎       |       ✕       |     A      |
|  $c_x$, $c_y$, $c_z$, $w$ (weight)   |   ✔︎   |    ✕     |       ✔︎       |       ✕       |     B      |
| $\rho$,  $u$, $v$, $w$, $\sigma$ etc |   ✕   |    ✔︎     |       ✔︎       |       ✔︎       |     C      |
|            $x$, $y$, $z$             |   ✕   |    ✔︎     |       ✕       |       ✕       |     D      |

#### Indexation type

1. **Relative indexation** means the starting point for indexation is the current  local point, e.g., for the Coordinate $x$, 0 means current point,, -1 the left side, 1, the right side.

2. **Absolute indexation** gives the absolute index of a variable. For example, $\textbf{c}_1$ with an absolute index could  the 1st discrete velocity within the lattice structure. 

In general the indexation schemes in $\xi$ , $\mathcal{C}$ and $\mathcal{M}$ are likely to be absolute, and the one in $\Omega$ may be relative. 

For absolute indexation, the application developer will be responsible for defining loop type of operation. 

For relative indexation, the loop type operation will be conducted automatically by the HiLeMMS loop wrapper and therefore by the underlying library.  

#### Loop over indexing space

Assuming the domain decomposition is used for parallelisation, we will need to have such a loop function that iterates the operation defined by users over the space $\Omega$, i.e., hides the underlying details for parallelism. 

In general a reasonable underlying library will provide similar facilities for this purpose. Hence, the loop function is to provide necessary wrapper or code generation tool for the translation. 

#### Indexing functions

```c++
// A type indexing 
CompoVeloSpaIdx (int componId, int veloId, int i, int j, int k)
```
```c++
//B type indexing
CompoVelodx(int componId, int veloId)
```
```c++
//C type indexing
CompoMacroSpaIdx(int componId, int macroVarId, int i, int j, int k)
```
```c++
// D type indexing
SpaIdx(int i, int j, int k)
```

These functions are the key part, C/C++ macros might be a way  for implementation. 

#### An example kernel function for  a “new“ equilibrium

Assuming the equilibrium reads 
$$
f_\alpha =w_\alpha \rho_\alpha \left( {\textbf{c}_\alpha} \cdot \textbf{V}+\frac{du}{dx}\right)
$$

The code may looks that

```c++
//all ID demostrated as varialbs are absolute indexation.
//those in number literials are relative indexation. 
f[CompoVeloSpaIdx(compoId, xiIdx, 0, 0, 0)] =
w[CompoVeloIdx(compoId,xiIdx)]*macroVars[CompoMacroSpaIdx(compoId,idOfRho,0,0,0)]*(
cx[CompoVeloIdx(compoId,xiIdx]*macroVars[CompoMacroSpaIdx(compoId,idOfu,0,0,0)]+
cy[CompoVeloIdx(compoId,xiIdx]*macroVars[CompoMacroSpaIdx(compoId,idOfv,0,0,0)]+
cz[CompoVeloIdx(compoId,xiIdx]*macroVars[CompoMacroSpaIdx(compoId,idOfw,0,0,0)]+
(macroVars[CompoMacroSpaIdx(compoId,idOfu,0,0,0)]-macroVars[CompoMacroSpaIdx(compoId,idOfu,-1,0,0)])/(X[SpaIdx(0,0,0)]-X[SpaIdx(-1,0,0)])
)
```


## Enumerated types

```c++
enum VariableTypes {
    Variable_Rho = 0,//Density
    Variable_U = 1,//velocity
    Variable_V = 2,//velocity
    Variable_W = 3,//velocity
    Variable_T = 4, //temperature
    Variable_Qx = 5, //heat flux
    Variable_Qy = 6, //heat flux
    Variable_Qz = 7, //heat flux
    Variable_Sigmaxx =8,//shear stress
    Variable_Sigmaxy =9,//shear stress
    Variable_Sigmaxz =10,//shear stress
    Variable_Sigmayy =11,//shear stress
    Variable_Sigmayz =12,//shear stress
    Variable_Sigmazz =13,//shear stress
    // variables that not calculated by using distribution
    Variable_Independent =-1,
    // variables calculated by using self-defined althrithm 
    Variable_SelfDefined = -2
};
```

