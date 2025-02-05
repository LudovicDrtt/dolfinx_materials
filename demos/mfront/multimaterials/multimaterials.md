---
jupytext:
  formats: md:myst,ipynb
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# Multiple behaviors on subdomains and interface conditions

In this demo, we show how to define a problem containing different subdomains, potentially separated by an interface. In each subdomain, a different MFront behavior is called.

We consider a geometry containing a matrix phase and inclusions. Both subdomains will be separated by an elastic interface behavior. In particular, the displacement field is continuous inside both phases, but discontinuous across the interface. To tackle this case, we will build a formulation involving the matrix and the inclusion submeshes with a standard Continuous Galerkin formulation in both of them. The two domains will then be tied by the formulation of an elastic behavior on the interface.

```{image} multimaterials.gif
:align: center
:width: 500px
```

```{seealso}
This demo builds upon the [COMET cohesive zone models tutorials](https://bleyerj.github.io/comet-fenicsx/tours/interfaces/contents.html). While the latter focus on a damageable interface behavior and elastic behavior inside the phases, the present demo assumes an elastic behavior of the interface and different nonlinear plastic behaviors inside both phases. Both approaches can obviously be combined together.
```

```{admonition} Download sources
:class: download

* {Download}`Python script<./multimaterials.py>`
* {Download}`Jupyter notebook<./multimaterials.ipynb>`
* {Download}`Utility module<./utils.py>`
* {Download}`MFront files<./behaviors.zip>`
```

```{code-cell} ipython3
import numpy as np
from mpi4py import MPI
import gmsh
import ufl
from petsc4py import PETSc
from dolfinx import fem, io, mesh, cpp
from dolfinx.cpp.nls.petsc import NewtonSolver
from dolfinx_materials.quadrature_map import QuadratureMap
from dolfinx_materials.material.mfront import MFrontMaterial
from dolfinx_materials.solvers import (
    BlockedNonlinearMaterialProblem,
)
from utils import (
    interface_int_entities,
    transfer_meshtags_to_submesh,
    interpolate_submesh_to_parent,
)
```

## Meshing and subdomains
We first create the mesh and define the different tags for identifying physical domains and interfaces.

```{code-cell} ipython3
:tags: [hide-input]

def create_matrix_inclusion_mesh(L, W, R, hsize):
    comm = MPI.COMM_WORLD

    gmsh.initialize()
    gdim = 2
    model_rank = 0
    if comm.rank == model_rank:
        gmsh.option.setNumber("General.Terminal", 0)  # to disable meshing info
        gmsh.model.add("Model")

        gmsh.model.occ.addRectangle(0.0, 0.0, 0.0, L, W, tag=1)
        gmsh.model.occ.addDisk(0.4, 0.0, 0.0, R, R, tag=2)
        gmsh.model.occ.addDisk(0.6, W, 0.0, R, R, tag=3)
        gmsh.model.occ.fragment([(gdim, 1)], [(gdim, 2), (gdim, 3)], removeObject=True)

        gmsh.model.occ.synchronize()

        gmsh.model.occ.remove([(gdim, 5), (gdim, 4)], recursive=True)

        gmsh.model.occ.synchronize()

        gmsh.option.setNumber("Mesh.CharacteristicLengthMin", hsize)
        gmsh.option.setNumber("Mesh.CharacteristicLengthMax", hsize)

        gmsh.model.addPhysicalGroup(gdim, [1], 1, name="Matrix")
        gmsh.model.addPhysicalGroup(gdim, [2, 3], 2, name="Inclusions")

        gmsh.model.addPhysicalGroup(gdim - 1, [3], 1, name="left")
        gmsh.model.addPhysicalGroup(gdim - 1, [7], 2, name="right")
        gmsh.model.addPhysicalGroup(gdim - 1, [1, 5], 3, name="interface")
        gmsh.model.addPhysicalGroup(gdim - 1, [2, 9, 8, 4, 10, 6], 4, name="sides")
        gmsh.model.mesh.generate(gdim)

    partitioner = cpp.mesh.create_cell_partitioner(mesh.GhostMode.shared_facet)
    domain, cells, facets = io.gmshio.model_to_mesh(
        gmsh.model, MPI.COMM_WORLD, model_rank, gdim=gdim, partitioner=partitioner
    )
    gmsh.finalize()
    return (domain, cells, facets)
```

```{code-cell} ipython3
length = 1.0
width = 0.5
radius = 0.25
hsize = 0.01
domain, cells, facets = create_matrix_inclusion_mesh(length, width, radius, hsize)
MATRIX_TAG = 1  # tag of matrix phase
INCL_TAG = 2  # tag of inclusion phase
INT_TAG = 3  # tag of interface
LEFT_TAG = 1  # tag of left boundary
RIGHT_TAG = 2  # tag of right boundary
interface_facets = facets.find(INT_TAG)

tdim = domain.topology.dim
fdim = tdim - 1
```

### Submesh creation

We define three submeshes: two submeshes (of codim. 0) corresponding to the matrix and inclusion 2D domains and one submesh (of codim. 1) corresponding to the facet restriction on the interface

```{code-cell} ipython3
subdomain2, subdomain2_cell_map, subdomain2_vertex_map, _ = mesh.create_submesh(
    domain, tdim, cells.find(INCL_TAG)
)
subdomain1, subdomain1_cell_map, subdomain1_vertex_map, _ = mesh.create_submesh(
    domain, tdim, cells.find(MATRIX_TAG)
)
interface_mesh, interface_cell_map, _, _ = mesh.create_submesh(
    domain, fdim, interface_facets
)
```

Now that we have defined submeshes, we need to transfer (facets) meshtags from those defined on the original domain to their subdomain counterpart. This function is available in {download}`./utils.py`.

```{code-cell} ipython3
subdomain1_facet_tags, subdomain1_facet_map = transfer_meshtags_to_submesh(
    domain, facets, subdomain1, subdomain1_vertex_map, subdomain1_cell_map
)
subdomain2_facet_tags, subdomain2_facet_map = transfer_meshtags_to_submesh(
    domain, facets, subdomain2, subdomain2_vertex_map, subdomain2_cell_map
)
```

### Entity map and integration measures

Similarly to the previous CZM tour, *entity maps* must be defined to link integration of quantities defined on the subdomains.

```{code-cell} ipython3
cell_imap = domain.topology.index_map(tdim)
num_cells = cell_imap.size_local + cell_imap.num_ghosts
domain_to_subdomain1 = np.full(num_cells, -1, dtype=np.int32)
domain_to_subdomain1[subdomain1_cell_map] = np.arange(
    len(subdomain1_cell_map), dtype=np.int32
)
domain_to_subdomain2 = np.full(num_cells, -1, dtype=np.int32)
domain_to_subdomain2[subdomain2_cell_map] = np.arange(
    len(subdomain2_cell_map), dtype=np.int32
)

subdomain1.topology.create_connectivity(fdim, tdim)
subdomain2.topology.create_connectivity(fdim, tdim)

facet_imap = domain.topology.index_map(facets.dim)
num_facets = facet_imap.size_local + facet_imap.num_ghosts
domain_to_interface = np.full(num_facets, -1)
domain_to_interface[interface_cell_map] = np.arange(len(interface_cell_map))
```

Before setting up the `entity_maps` dictionary, we need a specific treatment for integrating terms on the interface. The `interface_int_integration` manually defines integration quantities on the interface. Besides, interface terms seen from one specific subdomain only exist on one side. As the assembler complains about this, there is a specific tweak to map cells from one side of the interface to the other side, thereby modifying the `domain_to_subdomain` maps. Most importantly, cells in subdomain 1 correspond to the `"+"` side of the interface and cells in subdomain 2 to the `"-"` side.

```{code-cell} ipython3
interface_entities, domain_to_subdomain1, domain_to_subdomain2 = interface_int_entities(
    domain, interface_facets, domain_to_subdomain1, domain_to_subdomain2
)

entity_maps = {
    interface_mesh: domain_to_interface,
    subdomain1: domain_to_subdomain1,
    subdomain2: domain_to_subdomain2,
}
```

We are now in position to define the various integration measures. The key point here is that the `dInt` interface measure is defined using prescribed integration entities which have been defined earlier. This is done by passing them to `subdomain_data` as follows.

```{code-cell} ipython3
dx = ufl.Measure("dx", domain=domain, subdomain_data=cells)
ds = ufl.Measure("ds", domain=domain, subdomain_data=facets)
dx_int = ufl.Measure("dx", domain=interface_mesh)
dInt = ufl.Measure(
    "dS",
    domain=domain,
    subdomain_data=[(INT_TAG, interface_entities)],
    subdomain_id=INT_TAG,
)
```

## Nonlinear behavior formulation
$\newcommand{\bu}{\boldsymbol{u}}
\newcommand{\bv}{\boldsymbol{v}}
\newcommand{\jump}[1]{[\![#1]\!]}$
We now define the relevant function spaces. As hinted before, the unknown $\bu$ will consist of two displacements $(\bu^{(1)},\bu^{(2)})$ respectively belonging to a continuous Lagrange space defined on subdomains 1 and 2. We use a `MixedFunctionSpace` for this, meaning that we will end up with a block system. For easier post-processing, the computed displacement will be stored as a `DG` function, with jumps being non zero only at the interface.

```{code-cell} ipython3
def strain(u):
    return ufl.as_vector(
        [
            u[0].dx(0),
            u[1].dx(1),
            0.0,
            1 / np.sqrt(2) * (u[1].dx(0) + u[0].dx(1)),
            0.0,
            0.0,
        ]
    )


V1 = fem.functionspace(subdomain1, ("Lagrange", 1, (tdim,)))
V2 = fem.functionspace(subdomain2, ("Lagrange", 1, (tdim,)))
V = fem.functionspace(domain, ("DG", 1, (tdim,)))  # for post-processing only
W = ufl.MixedFunctionSpace(V1, V2)
u1 = fem.Function(V1, name="Displacement_1")
u2 = fem.Function(V2, name="Displacement_2")
u = fem.Function(V, name="Displacement")
v1, v2 = ufl.TestFunctions(W)
du1, du2 = ufl.TrialFunctions(W)
```

### Material laws on subdomains

Second, we define two different `MFrontMaterial` on the two subdomains. In this example, we use two plastic behaviors with different yield surfaces and hardening laws. In the matrix, a von Mises criterion is used with an exponential Voce hardening whereas in the stiffer inclusions, we use a Hosford criterion and linear isotropic hardening. 

```{important}
It is perfectly possible to use behaviors with different internal state variables, and even with different gradients/fluxes etc. They are really independent from each other and will only be combined by summing their contribution to the resulting weak form. As a result, we can also combine a MFront implementation and a JAX implementation for instance.
```

```{code-cell} ipython3

material1 = MFrontMaterial(
    "src/libBehaviour.so",
    "IsotropicPlasticMisesFlowVoce",
    material_properties={
        "young_modulus": 70e3,
        "poisson_ratio": 0.3,
        "R0": 200.0,
        "Rinf": 450,
        "b": 100,
    },
)
material2 = MFrontMaterial(
    "src/libBehaviour.so",
    "IsotropicPlasticHosfordFlowLinear",
    material_properties={
        "young_modulus": 90e3,
        "poisson_ratio": 0.25,
        "hardening_slope": 10.0,
        "R0": 200.0,
    },
)
```

As a result, we define two different `QuadratureMap` defined on both subdomains. Note that the registered gradients involve the two different displacements `u1` and `u2` respectively. Note that when assembling mixed forms it is more convenient that integration measures are defined on the similar parent domain (the full mesh) and to pass the entity_maps dictionary when compiling the forms. We thus redefine the `qmap` measures metadata accordingly. Finally, we define the contributions of both subdomains to the total residual form.

```{code-cell} ipython3
deg_quad = 1
qmap1 = QuadratureMap(subdomain1, deg_quad, material1)
qmap1.register_gradient("Strain", strain(u1))
qmap1.dx = qmap1.dx(domain=domain, subdomain_data=cells)
sig1 = qmap1.fluxes["Stress"]

qmap2 = QuadratureMap(subdomain2, deg_quad, material2)
qmap2.register_gradient("Strain", strain(u2))
qmap2.dx = qmap2.dx(domain=domain, subdomain_data=cells)
sig2 = qmap2.fluxes["Stress"]

Res_matrix = ufl.dot(sig1, strain(v1)) * qmap1.dx(1)
Res_inclusions = ufl.dot(sig2, strain(v2)) * qmap2.dx(2)
```

### Interface behavior

As regards the elastic interface of stiffness $K$, its contribution to the total residual is given by:

$$
\int_{\Gamma} K\jump{\bu}\cdot\jump{\bv}\,\text{d}S
$$

where we define the displacement $\jump{\bu} = \bu^{(2)} - \bu^{(1)}$ with $(1)$ denoting subdomain 1 (the matrix) and $(2)$ denoting subdomain 2 (the inclusions). Note that we need to restrict quantities since we work with a facet measure on the interface $\Gamma$. Although only one side exist for each subdomain, cells of a given subdomain from one side have been mapped to the other side, as discussed before. As a result, it does not really matter which side is used here. For consistency, we use the the `"+"` side for subdomain 1 and the `"-"` side for subdomain 2.

```{code-cell} ipython3
def jump(u1, u2):
    # As cell("+") are mapped to cell("-") when defining the cell maps, it does not really matter which side ("+"/"-") is used
    return u2("+") - u1("-")

K = fem.Constant(domain, 1e5)

Res_interface = K * ufl.dot(jump(u1, u2), jump(v1, v2)) * dInt
```

### Total residual and jacobian

Finally, the total residual is the sum of all three residuals. Since we work with a `MixedFunctionSpace`, we use `ufl.extract_blocks` to extract the blocks corresponding to both `u1` and `u2`. We then compute the corresponding Jacobian with both `qmap.derivative` in the corresponding trial functions. Both the residual and tangent blocked forms are compiled by passing the `entity_maps` dictionary to `fem.form`.

```{code-cell} ipython3

Res = Res_matrix + Res_inclusions + Res_interface
Res_blocked_compiled = fem.form(ufl.extract_blocks(Res), entity_maps=entity_maps)

Jac = qmap1.derivative(Res, u1, du1) + qmap2.derivative(Res, u2, du2)
Jac_blocked_compiled = fem.form(ufl.extract_blocks(Jac), entity_maps=entity_maps)
```

### Boundary conditions

We apply an imposed displacement on the right boundary and fix the left boundary. We use a virtual test field with unit value on the boundary to compute the consistent reaction force, see [](https://bleyerj.github.io/comet-fenicsx/tips/computing_reactions/computing_reactions.html).

```{code-cell} ipython3
Uimp = fem.Constant(domain, (1.0, 0.0))
left_dofs = fem.locate_dofs_topological(V1, fdim, subdomain1_facet_tags.find(1))
right_dofs = fem.locate_dofs_topological(V1, fdim, subdomain1_facet_tags.find(2))

bcs = [
    fem.dirichletbc(np.zeros((tdim,)), left_dofs, V1),
    fem.dirichletbc(Uimp, right_dofs, V1),
]

v_reac1 = fem.Function(V1)
fem.set_bc(v_reac1.x.array, bcs)
v_reac2 = fem.Function(V2)
fem.set_bc(v_reac2.x.array, bcs)
virtual_work_form = fem.form(
    ufl.replace(Res, {v1: v_reac1, v2: v_reac2}),
    entity_maps=entity_maps,
)
```

## Resolution

Next, we define the custom nonlinear problem. Here, we work with a blocked system and thus rely on `BlockedNonlinearMaterialProblem` which expects a list of behaviors to update and a list of functions corresponding to the blocked solution. We then define a classical Newton solver.

```{code-cell} ipython3
problem = BlockedNonlinearMaterialProblem(
    [qmap1, qmap2], Res_blocked_compiled, Jac_blocked_compiled, [u1, u2], bcs
)

newton = NewtonSolver(MPI.COMM_WORLD)
newton.rtol = 1e-5
newton.atol = 1e-5
newton.convergence_criterion = "residual"
newton.max_it = 20
```

### Time-stepping

Upon time-stepping, post-processing steps are needed. First, the subdomain displacements `u1` and `u2` are reconstructed into a single `DG` displacement field `u` which is more convenient to handle in Paraview. Note that although this field is discontinuous on the whole mesh, non-zero jumps will occur only at the interface. Second, we recover plastic strain variables in both subdomains as `DG0` fields. Since this step involves a projection, we need to pass the `entity_maps` dictionary to the `project_on` method to compute the corresponding forms of the projection subproblem. Note that both plastic strains are not recombined into a single field, although it is possible in the present case.  

```{code-cell} ipython3
:tags: [hide-output]

file_results = io.VTKFile(
    domain.comm,
    f"multimaterials_results.pvd",
    "w",
)

N = 30
Exx = np.linspace(0, 15e-3, N + 1)

file_results.write_function(u, 0)
p1 = qmap1.project_on("EquivalentPlasticStrain", ("DG", 0), entity_maps=entity_maps)
p1.name = "PlasticStrain1"
file_results.write_function(p1, 0)
p2 = qmap2.project_on("EquivalentPlasticStrain", ("DG", 0), entity_maps=entity_maps)
p2.name = "PlasticStrain2"
file_results.write_function(p2, 0)

Force = np.zeros_like(Exx)
for i, exx in enumerate(Exx[1:]):
    Uimp.value[0] = exx * length

    converged, it = problem.solve(newton)

    interpolate_submesh_to_parent(u, u1, subdomain1_cell_map)
    interpolate_submesh_to_parent(u, u2, subdomain2_cell_map)
    file_results.write_function(u, i + 1)
    p1 = qmap1.project_on("EquivalentPlasticStrain", ("DG", 0), entity_maps=entity_maps)
    p1.name = "PlasticStrain1"
    file_results.write_function(p1, i + 1)
    p2 = qmap2.project_on("EquivalentPlasticStrain", ("DG", 0), entity_maps=entity_maps)
    p2.name = "PlasticStrain2"
    file_results.write_function(p2, i + 1)

    Force[i + 1] = fem.assemble_scalar(virtual_work_form)

file_results.close()
```

## Results 

We finally plot the resulting load-displacement curve.

```{code-cell} ipython3
import matplotlib.pyplot as plt

plt.figure()
plt.plot(Exx, Force, "-oC3")
plt.xlabel("Imposed horizontal strain")
plt.ylabel("Reaction force")
plt.show()
```
