### Interfacing with HIPS

__HIPS__ (_Hierarchical Iterative Parallel Solver_) is a scientific library that provides an efficient parallel iterative solver for very large sparse linear systems. HIPS is available as free software under the CeCILL-C licence. The interface that we realized is compatible with release __1.2 beta.rc4__ $\codered$ of HIPS.

HIPS implements two solver classes which are the iteratives class (GMRES, PCG) and the Direct class. Concerning preconditionners, HIPS implements a type of multilevel ILU. For further informations on those preconditionners see \cite{A:LaBRI::sisc06,A:LaBRI::HRR07} $\codered$.

__Installation of HIPS__

To install HIPS, first download the HIPS package at \cite{HIPS} $\codered$, unpack it and go to the HIPS source directory. The installation of HIPS is machine dependence. For example, to install HIPS on a linux cluster copy the file __Makefile\_Inc\_Examples/makefile.inc.gnu__ $\codered$ on the root directory of HIPS with the name __makefile.inc__. After this, edit __makefile.inc__ to set values of different variables and type `:::bash make all`.

__Using HIPS as the interface to FreeFem++__

Before calling the HIPS solver inside FreeFem++, you must compile file hips\_FreeFem.cpp to create dynamic library hips\_FreeFem.so. To do this, move to the directory src/solver of FreeFem++ and edit the file makefile.inc to specify the following variables: $\codered$

```freefem
HIPS_DIR :  Directory of HIPS
HIPS_INCLUDE : -I$(HIPS_DIR)/SRC/INCLUDE : Directory for HIPS headers $\codered$
LIB_DIR : -L$(HIPS_DIR)/LIB : Librairies directory $\codered$
LIBHIPSSEQUENTIAL : $(HIPS_DIR)/LIB/libhipssequential.a: HIPS utilities library
LIBHIPS : $(HIPS_DIR)/LIB/libhips.a: HIPS library
FREEFEM : FreeFem++ directory
FREEFEM_INCLUDE : FreeFem headers for sparse linear solver
METIS : METIS directory
METIS_LIB : METIS library
MPI : MPI directory
MPI_INCLUDE : MPI headers
```

After specifies all the variables, in the command line in the directory
$src/solver$ type `:::freefem make hips` to create hips\_FreeFem.so. Like with pARMS, the calling of HIPS in FreeFem++ can be done in three different manners. We will present only one example where the user specifies the parameters through keywords `:::freefem lparams` and `:::freefem dparams`.

__Laplacian 3D solve with HIPS__

Let us consider the 3D Laplacian example inside FreeFem++ package where after discretization we want to solve the linear equation with Hips. Example \ref{hips:laplacian} $\codered$ is Laplacian3D using Hips as linear solver. We first load Hips solver at line 2. From line 4 to 15 we specify the parameters for the Hips solver and in line 46 of example \ref{hips:laplacian} $\codered$ we set these parameters in the linear solver.

In Table 1.10 \ref{hipslabel} $\codered$ results of running example 1.1 5\ref{hips:laplacian} $\codered$ on Cluster Paradent of Grid5000 are reported. We can see in this running example the efficiency of parallelism.

__Example Laplacian3D.edp__

```freefem
load "msh3"
load "hips_FreeFem" //load library
int nn=10,iii;
int[int] iparm(14);
real[int] dparm(6);
for(iii=0;iii<14;iii++)iparm[iii]=-1;
for(iii=0;iii<6;iii++) dparm[iii]=-1;
iparm[0]=0; //use iterative solver
iparm[1]=1; //PCG as Krylov method
iparm[4]=0; //Matrix are symmetric
iparm[5]=1; //Pattern are also symmetric
iparm[9]=1; //Scale matrix
dparm[0]=1e-13;//Tolerance to convergence
dparm[1]=5e-4; //Threshold in ILUT
dparm[2]=5e-4; //Threshold for Schur preconditionner
mesh Th2=square(nn,nn);
 fespace Vh2(Th2,P2);
 Vh2 ux,uz,p2;
 int[int] rup=[0,2], rdown=[0,1], rmid=[1,1,2,1,3,1,4,1];
real zmin=0,zmax=1;
mesh3 Th=buildlayers(Th2,nn,
 zbound=[zmin,zmax],
 reffacemid=rmid,
 reffaceup = rup,
 reffacelow = rdown);
savemesh(Th,"copie.mesh");
mesh3 Th3("copie.mesh");
fespace Vh(Th,P2);
func ue = 2*x*x + 3*y*y + 4*z*z + 5*x*y+6*x*z+1;
func uex= 4*x+ 5*y+6*z;
func uey= 6*y + 5*x;
func uez= 8*z +6*x;
func f= -18. ;
Vh uhe = ue; //
cout << " uhe min: " << uhe[].min << " max:" << uhe[].max << endl;
Vh u,v;
macro Grad3(u) [dx(u),dy(u),dz(u)] // EOM
varf va(u,v)= int3d(Th)(Grad3(v)' *Grad3(u)) //') for emacs
 + int2d(Th,2)(u*v)
 - int3d(Th)(f*v)
 - int2d(Th,2) ( ue*v + (uex*N.x +uey*N.y +uez*N.z)*v )
 + on(1,u=ue);
real cpu=clock();
matrix Aa;
Aa=va(Vh,Vh);
varf l(unused,v)=int3d(Th)(f*v);
Vh F; F[]=va(0,Vh);
if(mpirank==0){
 cout << "Taille " << Aa.n << endl;
 cout << "Non zeros " << Aa.nbcoef << endl;
}
if(mpirank==0)
cout << "CPU TIME FOR FORMING MATRIX = " << clock()-cpu << endl;
set(Aa,solver=sparsesolver,dparams=dparm, lparams=iparm); //Set hips as linear solver
u[]=Aa^-1*F[];
```

<table>
	<thead>
		<tr>
			<th colspan="3">Table 11.10 : Legend of table 11.10 are give in table 11.9 \ref{legtableparm} $\codered$</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td align='center'>$n=4 \times 10^6$</td>
	        <td align='center'>$nnz=118 \times 10^6$</td>
	        <td align='center'>Te=221.34</td>
	    </tr>
	    <tr>
	        <td align='center'>np</td>
	        <td align='center'>nit</td>
	        <td align='center'>time</td>
	    </tr>
	    <tr>
	        <td align='center'>8</td>
	        <td align='center'>190</td>
	        <td align='center'>120.34</td>
	    </tr>
	    <tr>
	        <td align='center'>16</td>
	        <td align='center'>189</td>
	        <td align='center'>61.08</td>
	    </tr>
	    <tr>
	        <td align='center'>32</td>
	        <td align='center'>186</td>
	        <td align='center'>31.70</td>
	    </tr>
	    <tr>
	        <td align='center'>64</td>
	        <td align='center'>183</td>
	        <td align='center'>23.44</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.11 : Significations of lparams corresponding to HIPS interface</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td style="font-weight: bold">Entries of iparm</td>
	        <td style="font-weight: bold">Significations of each entries</td>
	    </tr>
	    <tr>
	        <td>iparm[0]</td>
	        <td>Strategy use for solving (Iterative=0 or Hybrid=1 or Direct=2). Defaults values are : Iterative</td>
	    </tr>
	    <tr>
	        <td>iparm[1]</td>
	        <td>Krylov methods. If iparm[0]=0, give type of Krylov methods: 0 for GMRES, 1 for PCG</td>
	    </tr>
	    <tr>
	        <td>iparm[2]</td>
	        <td>Maximum of iterations in outer iteration: default value 1000</td>
	    </tr>
	    <tr>
	        <td>iparm[3]</td>
	        <td>Krylov subspace dimension in outer iteration: default value 40</td>
	    </tr>
	    <tr>
	        <td>iparm[4]</td>
	        <td>Symmetric(=0 for symmetric) and 1 for unsymmetricmatrix: default value 1 (unsymmetric matrix)</td>
	    </tr>
	    <tr>
	        <td>iparm[5]</td>
	        <td>Pattern of matrix are symmetric or not: default value 0</td>
	    </tr>
	    <tr>
	        <td>iparm[6]</td>
	        <td>Partition type of input matrix: default value 0</td>
	    </tr>
	    <tr>
	        <td>iparm[7]</td>
	        <td>Number of level that use the HIPS locally consistentfill-in: Default value 2</td>
	    </tr>
	    <tr>
	        <td>iparm[8]</td>
	        <td>Numbering in indices array will start at 0 or 1: Default value 0</td>
	    </tr>

	    <tr>
	        <td>iparm[9]</td>
	        <td>Scale matrix. Default value 1</td>
	    </tr>
	    <tr>
	        <td>iparm[10]</td>
	        <td>Reordering use inside subdomains for reducingfill-in: Only use for iterative. Default value 1</td>
	    </tr>
	    <tr>
	        <td>iparm[11]</td>
	        <td>Number of unknowns per node in the matrix non-zeropattern graph: Default value 1</td>
	    </tr>
	    <tr>
	        <td>iparm[12]</td>
	        <td>This value is used to set the number of time the normalization is applied to the matrix: Default 2.</td>
	    </tr>
	    <tr>
	        <td>iparm[13]</td>
	        <td>Level of informations printed during solving: Default 5.</td>
	    </tr>
	    <tr>
	        <td>iparm[14]</td>
	        <td>HIPS_DOMSIZE Subdomain size</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.12 : Significations of dparams corresponding to HIPS interface</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>dparm[0]</td>
	        <td>HIPS_PREC: Relative residual norm: Default=1e-9</td>
	    </tr>
	    <tr>
	        <td>dparm[1]</td>
	        <td>HIPS_DROPTOL0: Numerical threshold in ILUT for interior domain (important : set 0.0 in HYBRID: Default=0.005)</td>
	    </tr>
	    <tr>
	        <td>dparm[2]</td>
	        <td>HIPS_DROPTOL1 : Numerical threshold in ILUT for Schur preconditioner: Default=0.005</td>
	    </tr>
	    <tr>
	        <td></td>
	        <td></td>
	    </tr>
	    <tr>
	        <td>dparm[3]</td>
	        <td>HIPS_DROPTOLE : Numerical threshold for coupling between the interior level and Schur: Default 0.005</td>
	    </tr>
	    <tr>
	        <td>dparm[4]</td>
	        <td>HIPS_AMALG : Numerical threshold for coupling between the interior level and Schur: Default=0.005</td>
	    </tr>
	    <tr>
	        <td>dparm[5]</td>
	        <td>HIPS_DROPSCHUR : Numerical threshold for coupling between the interior level and Schur: Default=0.005</td>
	    </tr>
	</tbody>
</table>

### Interfacing with HYPRE

__HYPRE__ (High Level Preconditioner) is a suite of parallel preconditioner developed at Lawrence Livermore National Lab \cite{HYPRE} $\codered$ .

There are two main classes of preconditioners developed in HYPRE: AMG (Algebraic MultiGrid) and Parasails (Parallel Sparse Approximate Inverse).

Now, suppose we want to solve $Ax=b$.
At the heart of AMG there is a series of progressively coarser(smaller) representations of the matrix $A$. Given an approximation $\hat{x}$ to the solution $x$, consider solving the residual equation $Ae=r$ to find the error $e$, where $r=b-A\hat{x}$. A fundamental principle of AMG is that it is an algebraically smooth error. To reduce the algebraically smooth errors further, they need to be represented by a smaller defect equation (coarse grid residual equation) $A_ce_c=r_c$, which is cheaper to solve. After solving this coarse equation, the solution is then interpolated in fine grid represented here by matrix $A$. The quality of AMG depends on the choice of coarsening and interpolating operators.

The _sparse approximate inverse_ approximates the inverse of a matrix $A$ by a sparse matrix $M$. A technical idea to construct matrix $M$ is to minimize the Frobenuis norm of the residual matrix $I-MA$. For more details on this preconditioner technics see \cite{chow} $\codered$.

HYPRE implement three Krylov subspace solvers: GMRES, PCG and BiCGStab.

__Installation of HYPRE__

To install HYPRE, first download the HYPRE package at \cite{HYPRE} $\codered$, unpack it and go to the HYPRE/src source directory and do `:::bash ./configure` to configure Hypre. After this just type `:::bash make all` to create __libHYPRE.a__.

__Using HYPRE as interface to FreeFem++__

Before calling HYPRE solver inside FreeFem++, you must compile the file hypre\_FreeFem.cpp to create dynamic library hypre\_FreeFem.so.
To do this, move to the directory src/solver of FreeFem++, edit the file makefile.inc to specify the following variables:

```freefem
HYPRE_DIR : Directory of HYPRE
HYPRE_INCLUDE = -I$(HYPRE_DIR)src/hypre/include/ : Directory for header of HYPRE
HYPRE_LIB = -L$(HIPS_DIR)/src/lib/ -lHYPRE : Hypre Library
FREEFEM : FreeFem++ directory
FREEFEM_INCLUDE : FreeFem header for sparse linear solver
METIS : METIS directory
METIS_LIB : METIS library
MPI : MPI directory
MPI_INCLUDE : MPI headers
```

Like with pARMS, the calling of HIPS in FreeFem++ can be done in three manners.
We will present only one example where the user specifies its parameters through keywords `:::freefem lparams` and `:::freefem dparams`.

__Laplacian 3D solve with HYPRE__

Let us consider again the 3D Laplacian example inside FreeFem++ package where after discretization we want to solve the linear equation with Hypre. Example \ref{hypre:laplacian} $\codered$ is the Laplacian3D using Hypre as linear solver. Example \ref{hypre:laplacian} $\codered$ is the same as \ref{hips:laplacian} $\codered$, so we just show here the lines where we set some Hypre parameters.

We first load the Hypre solver at line 2. From line 4 to 15 we specifies the parameters to set to Hypre solver and in line 43 we set parameters to Hypre solver.

It should be noted that the meaning of the entries of these vectors is different from those of Hips. In the case of HYPRE, the meaning of differents entries of vectors `:::freefem iparm` and `:::freefem dparm` are given in tables \ref{communipramsHypre} $\codered$ to \ref{AMGHypre} $\codered$.

In Table \ref{hyprelabel} $\codered$ the results of running example \ref{hypre:laplacian} $\codered$ on Cluster Paradent of Grid5000 are reported. We can see in this running example the efficiency of parallelism, in particular when AMG are use as preconditioner.

__Example Laplacian3D.edp__

```freefem
load "msh3"
load "hipre_FreeFem" // Load librairie
int nn=10,iii;
int[int] iparm(20);
real[int] dparm(6);
for(iii=0;iii<20;iii++)iparm[iii]=-1;
for(iii=0;iii<6;iii++) dparm[iii]=-1;
iparm[0]=2; // PCG as krylov method
iparm[1]=0; // AMG as preconditionner 2: if ParaSails
iparm[7]=7; // Interpolation
iparm[9]=6; // AMG Coarsen type
iparm[10]=1; // Measure type
iparm[16]=2; // Additive schwarz as smoother
dparm[0]=1e-13; // Tolerance to convergence
dparm[1]=5e-4; // Threshold
dparm[2]=5e-4; // Truncation factor
.
.
.
set(Aa,solver=sparsesolver,dparams=dparm, lparams=iparm);
```

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.13 : Definitions of common entries of iparms and dparms vectors for every preconditioner in HYPRE</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>iparms[0]</td>
	        <td>Solver identification: 0: BiCGStab, 1: GMRES, 2: PCG. Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[1]</td>
	        <td>Preconditioner identification: 0: BOOMER AMG, 1: PILUT, 2: Parasails, 3: Schwartz Default=0</td>
	    </tr>
	    <tr>
	        <td>iparms[2]</td>
	        <td>Maximum of iteration: Default=1000</td>
	    </tr>
	    <tr>
	        <td>iparms[3]</td>
	        <td>Krylov subspace dim: Default= 40</td>
	    </tr>
	    <tr>
	        <td>iparms[4]</td>
	        <td>Solver print info level: Default=2</td>
	    </tr>
	    <tr>
	        <td>iparms[5]</td>
	        <td>Solver log: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[6]</td>
	        <td>Solver stopping criteria only for BiCGStab : Default=1</td>
	    </tr>
	    <tr>
	        <td>dparms[0]</td>
	        <td>Tolerance for convergence: Default=$1.0e-11$</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.14 : Definitions of other entries of iparms and dparms if preconditioner is BOOMER AMG</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>iparms[7]</td>
	        <td>AMG interpolation type: Default=6</td>
	    </tr>
	    <tr>
	        <td>iparms[8]</td>
	        <td>Specifies the use of GSMG - geometrically smooth coarsening and interpolation: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[9]</td>
	        <td>AMG coarsen type: Default=6</td>
	    </tr>
	    <tr>
	        <td>iparms[10]</td>
	        <td>Defines whether local or global measures are used: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[11]</td>
	        <td>AMG cycle type: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[12]</td>
	        <td>AMG Smoother type: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[13]</td>
	        <td>AMG number of levels for smoothers: Default=3</td>
	    </tr>
	    <tr>
	        <td>iparms[14]</td>
	        <td>AMG number of sweeps for smoothers: Default=2</td>
	    </tr>
	    <tr>
	        <td>iparms[15]</td>
	        <td>Maximum number of multigrid levels: Default=25</td>
	    </tr>
	    <tr>
	        <td>iparms[16]</td>
	        <td>Defines which variant of the Schwartz method isused:<br>
			0: hybrid multiplicative Schwartz method (no overlap across processor boundaries)<br>
			1: hybrid additive Schwartz method (no overlap across processor boundaries)<br>
			2: additive Schwartz method<br>
			3: hybrid multiplicative Schwartz method (with overlap across processor boundaries)<br>
			Default=1
			</td>
	    </tr>
	    <tr>
	        <td>iparms[17]</td>
	        <td>Size of the system of PDEs: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[18]</td>
	        <td>Overlap for the Schwarz method: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[19]</td>
	        <td>Type of domain used for the Schwarz method<br>
			0: each point is a domain<br>
			1: each node is a domain (only of interest in ``systems'' AMG)<br>
			2: each domain is generated by agglomeration (default)</td>
	    </tr>
	    <tr>
	        <td>dparms[1]</td>
	        <td>AMG strength threshold: Default=0.25</td>
	    </tr>
	    <tr>
	        <td>dparms[2]</td>
	        <td>Truncation factor for the interpolation: Default=1e-2</td>
	    </tr>
	    <tr>
	        <td>dparms[3]</td>
	        <td>Sets a parameter to modify the definition of strength for diagonal dominant portions of the matrix: Default=0.9</td>
	    </tr>
	    <tr>
	        <td>dparms[3]</td>
	        <td>Defines a smoothing parameter for the additive Schwartz method. Default=1</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.15 : Definitions of other entries of iparms and dparms if preconditioner is PILUT</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>iparms[7]</td>
	        <td>Row size in Parallel ILUT: Default=1000</td>
	    </tr>
	    <tr>
	        <td>iparms[8]</td>
	        <td>Set maximum number of iterations: Default=30</td>
	    </tr>
	    <tr>
	        <td>dparms[1]</td>
	        <td>Drop tolerance in Parallel ILUT: Default=$1e-5$</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.16 : Definitions of other entries of iparms and dparms if preconditioner is ParaSails</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>iparms[7]</td>
	        <td>Number of levels in Parallel Sparse Approximate inverse: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[8]</td>
	        <td>Symmetric parameter for the ParaSails preconditioner:<br>
			0: nonsymmetric and/or indefinite problem, and nonsymmetric preconditioner<br>
			1: SPD problem, and SPD (factored) preconditioner<br>
			2: nonsymmetric, definite problem, and SPD (factored) preconditioner<br>
			Default=0</td>
	    </tr>
	    <tr>
	        <td>dparms[1]</td>
	        <td>Filters parameters. The filter parameter is used to drop small nonzeros in the preconditioner, to reduce the cost of applying the preconditioner: Default=0.1</td>
	    </tr>
	    <tr>
	        <td>dparms[2]</td>
	        <td>Threshold parameter: Default=0.1</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="2">Table 11.17 : Definitions of other entries of iparms and dparms if preconditionner is Schwartz</th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td>iparms[7]</td>
	        <td>Defines which variant of the Schwartz method isused:<br>
			0: hybrid multiplicative Schwartz method (no overlap across processor boundaries)<br>
			1: hybrid additive Schwartz method (no overlap across processor boundaries)<br>
			2: additive Schwartz method<br>
			3: hybrid multiplicative Schwartz method (with overlap across processor boundaries)<br>
			Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[8]</td>
	        <td>Overlap for the Schwartz method: Default=1</td>
	    </tr>
	    <tr>
	        <td>iparms[9]</td>
	        <td>Type of domain used for the Schwartz method<br>
			0: each point is a domain<br>
			1: each node is a domain (only of interest in "systems" AMG)<br>
			2: each domain is generated by agglomeration (default)</td>
	    </tr>
	</tbody>
</table>

<table>
	<thead>
		<tr>
			<th colspan="3">Table 11.18 : Convergence and time for solving linear system from example 11.4 \ref{exm:segond} $\codered$ </th>
		</tr>
	</thead>
	<tbody>
	    <tr>
	        <td align='center'>n = $4\times10^6$</td>
	        <td align='center'>nnz = $13\times10^6$</td>
	        <td align='center'>Te = 571,29</td>
	    </tr>
	    <tr>
	        <td rowspan="2" align='center'>np</td>
	        <td colspan="2" align='center'>AMG</td>
	    </tr>
	    <tr>
	        <td align='center'>nit</td>
	        <td align='center'>time</td>
	    </tr>
	    <tr>
	        <td align='center'>8</td>
	        <td align='center'>6</td>
	        <td align='center'>1491.83</td>
	    </tr>
	    <tr>
	        <td align='center'>16</td>
	        <td align='center'>5</td>
	        <td align='center'>708.49</td>
	    </tr>
	    <tr>
	        <td align='center'>32</td>
	        <td align='center'>4</td>
	        <td align='center'>296.22</td>
	    </tr>
	    <tr>
	        <td align='center'>64</td>
	        <td align='center'>4</td>
	        <td align='center'>145.64</td>
	    </tr>
	</tbody>
</table>

### Conclusion

With the different runs presented here, we wanted to illustrate the gain in time when we increase the number of processors used for the simulations. We saw that in every case the time for the construction of the finite element matrix is constant. This is normal because until now this phase is sequential in FreeFem++. In contrast, phases for solving the linear system are parallel. We saw on several examples presented here that when we increase the number of processors, in general we decrease the time used for solving the linear systems. But this not true in every case. In several case, when we increase the number of processors the time to convergence also increases. There are two main reasons for this. First, the increase of processors can lead to the increase of volume of exchanged data across processors consequently increasing the time for solving the linear systems.

Furthermore, in decomposition domain type preconditioners, the number of processors generally corresponds to the number of sub domains. In subdomain methods, generally when we increase the number of subdomains we decrease convergence quality of the preconditioner. This can increase the time used for solving linear equations.

To end this, we should note that good use of the preconditioners interfaced in FreeFem++ is empiric, because it is difficult to know what is a good preconditioner for some type of problems. Although, the efficiency of preconditioners sometimes depends on how its parameters are set. For this reason we advise the user to pay attention to the meaning of the parameters in the user guide of the iterative solvers interfaced in FreeFem++.

## Domain decomposition

In the previous section, we saw that the phases to construct a matrix are sequential. One strategy to construct the matrix in parallel is to divide geometrically the domain into subdomains. In every subdomain we construct a local submatrix and after that we assemble every submatrix to form the global matrix.

We can use this technique to solve pde directly in domain $\Omega$. In this case, in every subdomains you have to define artificial boundary conditions to form consistent equations in every subdomains. After this, you solve equation in every subdomains and define a strategy to obtain the global solution.

In terms of parallel programming for FreeFem++, with MPI, this means that the user must be able to divide processors avaible for computation into subgroups of processors and also must be able to realize different type of communications in FreeFem++ script. Here is a wrapper of some MPI functions.

### Communicators and groups

__Groups__

`:::freefem mpiGroup grpe(mpiGroup gp,KN_<long>)`: Create MPI_Group from existing group `:::freefem gp` by
given vector.

__Communicators__

Communicators is an abstract MPI object which allows MPI user to communicate across group of processors. Communicators can be Intra-communicators(involves a single group) or Inter-communicators (involves two groups). When we not specify type of communicator it will be Intra-communicators

__mpiComm cc(mpiComm comm, mpiGroup gp):__ Creates a new communicator.
`:::freefem comm` communicator(handle), `:::freefem gp` group which is a subset of the group of `:::freefem comm` (handle). Return new communicator

__mpiComm cc(mpiGroup gp)__: Same as previous constructor but default `:::freefem comm` here is MPI\_COMM\_WORLD.

__mpiComm cc(mpiComm comm, int color, int key):__ Creates new communicators
based on `:::freefem colors` and `:::freefem key`. This constructor is based on MPI\_Comm\_split routine of MPI.

__mpiComm cc(MPIrank p,int key):__ Same constructor than the last one.
Here `:::freefem colors` and `:::freefem comm` is defined in `:::freefem MPIrank`. This constructor is based on MPI\_Comm\_split routine of MPI.

__Example 11.7 commsplit.edp__

```freefem
int color=mpiRank(comm)%2;
mpiComm ccc(processor(color,comm),0);
mpiComm qpp(comm,);
mpiComm cp(cc,color,0);
```

__mpiComm cc(mpiComm comm, int high):__ Creates an intracommunicator from
an intercommunicator. `:::freefem comm` intercommunicator, `:::freefem high` Used to order the groups within `:::freefem comm` (logical) when creating the new communicator. This constructor is based on MPI\_Intercomm\_merge routine of MPI.

__mpiComm cc(MPIrank p1, MPIrank p2, int tag):__ This constructor creates an intercommuncator from two intracommunicators. `:::freefem p1` defined local (intra)communicator and rank in local\_comm of leader (often 0) while `:::freefem p2` defined remote communicator and rank in peer\_comm of remote leader (often 0). `:::freefem tag` Message tag to use in constructing intercommunicator. This constructor is based on MPI\_Intercomm\_create.

__Example 11.8 merge.edp__

```freefem
mpiComm comm,cc;
int color=mpiRank(comm)%2;
int rk=mpiRank(comm);
int size=mpiSize(comm);
cout << "Color values " << color << endl;
mpiComm ccc(processor((rk<size/2),comm),rk);
mpiComm cp(cc,color,0);
int rleader;
if (rk == 0) { rleader = size/2; }
else if (rk == size/2) { rleader = 0;}
else { rleader = 3; }
mpiComm qqp(processor(0,ccc),processor(rleader,comm),12345);
int aaa=mpiSize(ccc);
cout << "number of processor" << aaa << endl;
```

### Process

In FreeFem++ we wrap MPI process by function call `:::freefem processor` which create internal FreeFem++ object call `:::freefem MPIrank`. This mean that do not use `:::freefem MPIrank` in FreeFem++ script.

`:::freefem processor(int rk)`: Keep process rank inside object `:::freefem MPIrank`. Rank is inside MPI\_COMM\_WORLD.

`:::freefem processor(int rk, mpiComm cc)` and `:::freefem processor(mpiComm cc,int rk)` process rank inside communicator cc.

`:::freefem processor(int rk, mpiComm cc)` and `:::freefem processor(mpiComm cc,int rk)` process rank inside communicator cc.

`:::freefem processorblock(int rk)`: This function is exactlly the same than `:::freefem processor(int rk)` but is use in case of blocking communication.

`:::freefem processorblock(int rk, mpiComm cc)`: This function is exactly the same as `:::freefem processor(int rk,mpiComm cc)` but uses a synchronization point.

### Points to Points communicators

In FreeFem++ you can call MPI points to points communications functions.

`:::freefem Send(processor(int rk,mpiComm cc),Data D)` : Blocking send of `:::freefem Data D` to processor of `:::freefem rank rk` inside communicator `:::freefem cc`. Note that `:::freefem Data D` can be: `:::freefem int`, `:::freefem real`, `:::freefem complex`, `:::freefem int[int]`, `:::freefem real[int]`, `:::freefem complex[int]`, `:::freefem Mesh`, `:::freefem Mesh3`, `:::freefem Matrix`.

`:::freefem Recv(processor(int rk,mpiComm cc),Data D)`: Receive `:::freefem Data D` from process of rank `:::freefem rk` in communicator `:::freefem cc`. Note that `:::freefem Data D` can be: `:::freefem int`, `:::freefem real`, `:::freefem complex`, `:::freefem int[int]`, `:::freefem real[int]`, `:::freefem complex[int]`, `:::freefem Mesh`, `:::freefem Mesh3`, `:::freefem Matrix` and should be the same type than corresponding send.

`:::freefem Isend(processor(int rk,mpiComm cc),Data D)` : Non blocking send of `:::freefem Data D` to processor of `:::freefem rank rk` inside communicator `:::freefem cc`. Note that `:::freefem Data D` can be: int, real, complex, int[int], real[int], complex[int], Mesh, Mesh3, Matrix.

`:::freefem Recv(processor(int rk,mpiComm cc),Data D)`: Receive corresponding to send.

### Global operations

In FreeFem++ you can call MPI global communication functions.

`:::freefem broadcast(processor(int rk,mpiComm cc),Data D)`: Process `:::freefem rk`
Broadcast `:::freefem Data D` to all process inside `:::freefem communicator cc`. Note
that `:::freefem Data D` can be: `:::freefem int`, `:::freefem real`, `:::freefem complex`, `:::freefem int[int]`, `:::freefem real[int]`, `:::freefem complex[int]`, `:::freefem Mesh`, `:::freefem Mesh3`, `:::freefem Matrix`.

`:::freefem broadcast(processor(int rk),Data D)`: Process `:::freefem rk` Broadcast `:::freefem Data D` to all process inside

MPI\_COMM\_WORLD. Note that `:::freefem Data D` can be: `:::freefem int`, `:::freefem real`, `:::freefem complex`, `:::freefem int[int]`, `:::freefem real[int]`, `:::freefem complex[int]`, `:::freefem Mesh`, `:::freefem Mesh3`, `:::freefem Matrix`.

`:::freefem mpiAlltoall(Data a,Data b)`: Sends `:::freefem data a` from all to all processes. Receive buffer is `:::freefem Data b`. This is done inside communicator MPI\_COMM\_WORLD.

`:::freefem mpiAlltoall(Data a,Data b, mpiComm cc)`: Sends `:::freefem data a` from all to all processes. Receive buffer is `:::freefem Data b`. This is done inside communicator cc.

`:::freefem mpiGather(Data a,Data b,processor(mpiComm,int rk)`: Gathers together values `:::freefem Data a` from a group of processes. Process of rank `:::freefem rk` get data on communicator `:::freefem rk`. This function is like MPI\_Gather

`:::freefem mpiAllgather(Data a,Data b)`: Gathers `:::freefem Data a` from all processes and distribute it to all in `:::freefem Data b`. This is done inside communicator MPI\_COMM\_WORLD. This function is like MPI\_Allgather

`:::freefem mpiAllgather(Data a,Data b, mpiComm cc)`: Gathers `:::freefem Data a` from all processes and distribute it to all in `:::freefem Data b`. This is done inside `:::freefem communicator cc`. This function is like MPI\_Allgather

`:::freefem mpiScatter(Data a,Data b,processor(int rk, mpiComm cc))`: Sends `:::freefem Data a` from one process whith rank `:::freefem rk` to all other processes in group represented by communicator `:::freefem mpiComm cc`.

`:::freefem mpiReduce(Data a,Data b,processor(int rk, mpiComm cc),MPI_Op op)`, $\codered$ ERREUR DE FRAPPE QQ PART ? Reduces values `:::freefem Data a` on all processes to a single value `:::freefem Data b` on process of rank `:::freefem rk` and communicator `:::freefem cc`.
Operation use in reduce is: `:::freefem MPI_Op op` which can be: `:::freefem mpiMAX`, `:::freefem mpiMIN`, `:::freefem mpiSUM`, `:::freefem mpiPROD`, `:::freefem mpiLAND`, `:::freefem mpiLOR`, `:::freefem mpiLXOR`, `:::freefem mpiBAND`, `:::freefem mpiBXOR`, `:::freefem mpiMAXLOC`, `:::freefem mpiMINLOC`.

Note that, for all global operations, only `:::freefem int[int]` and `:::freefem real[int]` are data type take in account in FreeFem++.

## HPDDM solvers

The corresponding examples are in the directory `:::freefem examples++-hpddm`. Real valued problems (diffusion, heat, elasticity and Stokes) and complex valued problems (Maxwell and Helmholtz) are given in both 2D and 3D. We detail here the 3D elasticity problem and the 3D time-dependent heat problem.

__Example elasticity-3d.edp__

A three dimensional elasticity problem is defined. The solver is a domain decomposition method. Domain decomposition methods are a natural framework for parallel computers. The scripts in the directory `:::freefem examples++-hpddm` run on multicores computers (from 2 to tens of thousands of cores). Recall that like in any MPI code the number of MPI processes, `:::freefem mpisize`,  is given in the command line via the option `:::freefem -np`. We focus on the script `:::freefem elasticity-3d.edp` but the other scripts have the same structure. The command line to run the example on four processes with `:::freefem ffglut` visualization is: `:::freefem ff-mpirun -np 4 elasticity-3d.edp -glut ffglut`

$\codered$

```freefem
// run with MPI: ff-mpirun -np 4 script.edp
// NBPROC 4

load "hpddm"                        // HPDDM plugin
macro partitioner()metis// EOM      // metis, scotch, or parmetis
macro dimension()3// EOM            // 2D or 3D
macro vectorialfe()P1// EOM
include "macro_ddm.idp"             // additional DDM functions

macro def(i)[i, i#B, i#C]// EOM     // vector field definition
macro init(i)[i, i, i]// EOM        // vector field initialization
/*# DiffMacros #*/
real Sqrt = sqrt(2.0);
macro epsilon(u)[dx(u), dy(u#B), dz(u#C), (dz(u#B) + dy(u#C)) / Sqrt, (dz(u) + dx(u#C)) / Sqrt, (dy(u) + dx(u#B)) / Sqrt]// EOM
macro div(u)(dx(u) + dy(u#B) + dz(u#C))// EOM
/*# DiffMacrosEnd #*/
func Pk = [vectorialfe, vectorialfe, vectorialfe];             // finite element space

/*# DDMoptions #*/
string deflation = getARGV("-deflation", "geneo");              // coarse space construction
int overlap = getARGV("-overlap", 1);                           // geometric overlap between subdomains
int fakeInterface = getARGV("-interface", 10);                  // interface between subdomains
int s = getARGV("-split", 1);                                   // refinement factor

mpiComm comm;
int p = getARGV("-hpddm_master_p", 1);
bool excluded = splitComm(mpiCommWorld, p, comm, topology = getARGV("-hpddm_master_topology", 0), exclude = (usedARGV("-hpddm_master_exclude") != -1));
/*# DDMoptionsEnd #*/

if(verbosity > 0 && mpirank == 0) {
    cout << " --- " << mpirank << "/" << mpisize;
    cout << " - elasticity-3d.edp - input parameters: refinement factor = " << s << " - overlap = " << overlap << endl;
}

int[int] LL = [2,3, 2,1, 2,2];
meshN ThBorder, Th = cube(1, 1, 1, [x, y, z]);
fespace Wh(Th, Pk);                         // local finite element space
/*# SchwarzMethod #*/
int[int] arrayIntersection;                 // ranks of neighboring subdomains
int[int][int] restrictionIntersection(0);   // local-to-neighbors renumbering
real[int] D;                                // partition of unity
{
    meshN ThGlobal = cube(10 * getARGV("-global", 5), getARGV("-global", 5), getARGV("-global", 5), [10 * x, y, z], label = LL);      // global mesh
    build(Th, ThBorder, ThGlobal, fakeInterface, s, overlap, D, arrayIntersection, restrictionIntersection, Wh, Pk, comm, excluded, 3)
}

real f = -9000.0;
real strain = 100.0;
real Young = 2.0e11; // steel
real poisson = 0.35;
real tmp = 1.0 + poisson;
real mu = Young  / (2.0 * tmp);
real lambda = Young * poisson / (tmp * (1.0 - 2.0 * poisson));
real[int] rhs;                              // local right-hand side
matrix<real> Mat;                           // local operator
{                                           // local weak form
    meshN ThAugmented = Th + ThBorder;
    varf vPb(def(u), def(v)) = intN(ThAugmented)(lambda * div(u) * div(v) + 2.0 * mu * (epsilon(u)' * epsilon(v))) + intN(ThAugmented)(f * vC) + on(1, u = 0.0, uB = 0.0, uC = 0.0);
    fespace WhAugmented(ThAugmented, Pk);
    Mat = vPb(WhAugmented, WhAugmented, tgv = -1);
    real[int] rhsFull = vPb(0, WhAugmented, tgv = -1);
    matrix R = interpolate(Wh, WhAugmented);
    renumbering(Mat, R, rhsFull, rhs);
}
ThBorder = cube(1, 1, 1, [x, y, z]);

dschwarz A(Mat, arrayIntersection, restrictionIntersection, scaling = D);
/*# SchwarzMethodEnd #*/

/*# OsmTwolevel #*/
set(A, sparams = "-hpddm_schwarz_method ras -hpddm_schwarz_coarse_correction balanced -hpddm_variant right -hpddm_verbosity 1 -hpddm_geneo_nu 10");
/*# OsmTwolevelEnd #*/

matrix<real> Opt;                           // local operator with optimized boundary conditions
dpair ret;
{
    int solver = getOption("schwarz_method");
    if(solver == 1 || solver == 2 || solver == 4) { // optimized Schwarz methods
        fespace Ph(Th, P0);
        real kZero = getARGV("-kZero", 10.0);
        Ph transmission = 2 * kZero * mu * (2 * mu + lambda) / (lambda + 3 * mu);
        varf vOptimized(def(u), def(v)) = intN(Th)(lambda * div(u) * div(v) + 2.0 * mu * (epsilon(u)' * epsilon(v))) + intN1(Th, fakeInterface)(transmission * (def(u)' * def(v))) + on(1, u = 0.0, uB = 0.0, uC = 0.0);
        Opt = vOptimized(Wh, Wh, tgv = -1);
    }
    if(mpisize > 1 &&
       isSetOption("schwarz_coarse_correction")) { // two-level Schwarz methods
        if(excluded)
            attachCoarseOperator(mpiCommWorld, A/*, A = noPen, B = overlapRestriction, threshold = 2. * h[].max / diam*/);
        else {
            varf vPbNoPen(def(u), def(v)) = intN(Th)(lambda * div(u) * div(v) + 2.0 * mu * (epsilon(u)' * epsilon(v))) + on(1, u = 0.0, uB = 0.0, uC = 0.0);
            matrix<real> noPen = vPbNoPen(Wh, Wh, solver = CG);
            if(deflation == "geneo") // standard GenEO, no need for RHS -> deduced from LHS (Neumann matrix)
                attachCoarseOperator(mpiCommWorld, A, A = noPen/*, threshold = 2. * h[].max / diam,*/, ret = ret);
            else if(deflation == "dtn") {
                varf vMass(def(u), def(v)) = intN1(Th, fakeInterface)(u * v);
                matrix<real> massMatrix = vMass(Wh, Wh, solver = CG);
                attachCoarseOperator(mpiCommWorld, A, A = noPen, B = massMatrix, pattern = Opt/*, threshold = k,*/, ret = ret);
            }
            else if(deflation == "geneo-2") // GenEO-2 for optimized Schwarz methods, need for RHS (LHS is still Neumann matrix)
                attachCoarseOperator(mpiCommWorld, A, A = noPen, B = Opt, pattern = Opt/*, threshold = 2. * h[].max / diam,*/, ret = ret);
        }
    }
}

/*# SolvePlot #*/
Wh<real> def(u);    // local solution

if(Opt.n > 0)       // optimized Schwarz methods
    DDM(A, u[], rhs, excluded = excluded, ret = ret, O = Opt);
else
    u[] = A^-1 * rhs;

real[int] err(u[].n);
err = A * u[];      // global matrix-vector product
err -= rhs;

plotMPI(Th, u[], "Global solution", Pk, def, real, 3, 1)
plotMPI(Th, err, "Global residual", Pk, def, real, 3, 1)
real alpha = 2000.0;
meshN ThMoved = movemesh3(Th, transfo = [x + alpha * u, y + alpha * uB, z + alpha * uC]);
u[] = mpirank;
plotMPI(ThMoved, u[], "Global moved solution", Pk, def, real, 3, 1)
/*# SolvePlotEnd #*/
```

The macro `:::freefem build` is of particular interest since it handles the data distribution among the `:::freefem mpisize` MPI processes with the following steps:

* The initial mesh `:::freefem ThGlobal` is partitioned  by process 0 into `:::freefem mpisize` submeshes

* The partition is broadcasted to every process $i$ for 0 < $i$ < `:::freefem mpisize`. From then on, all tasks are parallel.

* Each process creates the local submesh `:::freefem Th` (if the refinement factor `:::freefem s` defined via the option `:::freefem -split` is larger than 1, each local edge is splitted into $s$ subedges, resulting in each element being split into $s^2$ element in 2D and $s^3$ elements in 3D) so that the collection of these submeshes is an overlapping domain decomposition of a refined mesh. The number of extra layers added to the initial partition is monitored by the option `:::freefem -overlap`.

* Connectivity structures are created
	* `:::freefem D` is the diagonal of the local partition of unity (see below \S~\ref{sub:linear} 11.5.2 $\codered$ for more details)
	* `:::freefem arrayIntersection` is the list of neighbors of the current subdomain
	* For `:::freefem j` in `:::freefem arrayIntersection`, `:::freefem restrictionIntersection[j]` is the list of the degrees of freedom that belong to the intersection of the current subdomain with its neighbor `:::freefem j`.

Then, the variational formulation `:::freefem vPb` of a three dimensional elasticity problem is used to assemble a local matrix `:::freefem Mat`. This matrix along with  `:::freefem D`, `:::freefem arrayIntersection` and `:::freefem restrictionIntersection` are arguments for the constructor of the distributed matrix `:::freefem A`. This is enough to solve the problem with a one-level additive Schwarz method which can be either ASM or RAS.

For some problems it is interesting to use optimized interface conditions. When there are many subdomains, it is usually profitable to add a second level to the solver. Options are set in the sequel of the script:

```freefem
set(A, sparams = "-hpddm_schwarz_method ras -hpddm_schwarz_coarse_correction balanced -hpddm_variant right -hpddm_verbosity 1 -hpddm_geneo_nu 10");
```

In the above line, the first option selects the one-level preconditioner `:::freefem ras` (possible choices are `:::freefem ras`, `:::freefem oras`, `:::freefem soras`, `:::freefem asm`, `:::freefem osm` or `:::freefem none`), the second option selects the correction formula for the second level here `:::freefem balanced` (possible options are `:::freefem deflated`, `:::freefem additive` or `:::freefem balanced`), the third option selects right preconditioning, the fourth one is verbosity level of HPDDM (different from the one of FreeFem++), the fifth one prints all possible options of HPPDM and the last one specifies the number of coarse degrees of freedom per subdomain of the GENEO coarse space. All other options of [https://github.com/hpddm/hpddm/blob/master/doc/cheatsheet.pdf](cheatsheet of the HPDDM) \cite{Jolivet:2014:HPD} $\codered$ library can be selected via the FreeFem++ function `:::freefem set`.

In the last part of the script, the global linear system is solved by the domain decomposition method defined above.

```freefem
Wh<real> def(u);    // local solution

if(Opt.n > 0)       // optimized Schwarz methods
    DDM(A, u[], rhs, excluded = excluded, ret = ret, O = Opt);
else
    u[] = A^-1 * rhs;

real[int] err(u[].n);
err = A * u[];      // global matrix-vector product
err -= rhs;

plotMPI(Th, u[], "Global solution", Pk, def, real, 3, 1)
plotMPI(Th, err, "Global residual", Pk, def, real, 3, 1)
real alpha = 2000.0;
meshN ThMoved = movemesh3(Th, transfo = [x + alpha * u, y + alpha * uB, z + alpha * uC]);
u[] = mpirank;
plotMPI(ThMoved, u[], "Global moved solution", Pk, def, real, 3, 1)
```

## Time dependent problem

__Example heat-3d.edp:__ A three dimensional heat problem

\[
\frac{\partial u}{\partial t} - \Delta u = 1,\ \ \ u(0,\cdot) := 0 \text{ in }\Omega\,.
\]

is discretized by an implicit Euler scheme. At each time step $n$, we shall seek $u^n(x,y,z)$ satisfying for all $w\in H^1(\Omega)$:

\[
\int_\Omega \frac{u^n-u^{n-1}}{\delta t}\,w + \nabla u^n \nabla w = \int_\Omega w ,\ \ \ u^0 := 0 \text{ in }\Omega\,.
\]

so that at each time step a linear system

\[
(M+dt*K) u^n[] = M*u^{n-1}[] + \delta t*F
\]

is solved by a domain decomposition method where $M$ is the mass matrix and $K$ is the rigidity matrix. In order to save computational efforts, the domain decomposition method preconditioner is built only once and then reused for all subsequent solves with matrix $A:=M+dt*K$. The distributed matrix vector product with matrix $M$ is made through the call to the function `:::freefem dmv` using the partition of unity associated to matrix $A$.

```freefem hl_lines="90 91 92 93 94 95 96 97 98 99 100 101 102 103"
//  run with MPI:  ff-mpirun -np 4 script.edp
// NBPROC 4

load "hpddm"                        // HPDDM plugin
macro partitioner()metis// EOM      // metis, scotch, or parmetis
macro dimension()3// EOM            // 2D or 3D
include "macro_ddm.idp"             // additional DDM functions

macro def(i)i// EOM                         // scalar field definition
macro init(i)i// EOM                        // scalar field initialization
macro grad(u)[dx(u), dy(u), dz(u)]// EOM    // three-dimensional gradient
func Pk = P2;                               // finite element space

string deflation = getARGV("-deflation", "geneo");              // coarse space construction
int overlap = getARGV("-overlap", 1);                           // geometric overlap between subdomains
int fakeInterface = getARGV("-interface", 10);                  // interface between subdomains
int s = getARGV("-split", 1);                                   // refinement factor
real dt = getARGV("-dt", 0.01);                                 // time step
int iMax = getARGV("-iMax", 10);                                // number of iterations

mpiComm comm;
int p = getARGV("-hpddm_master_p", 1);
bool excluded = splitComm(mpiCommWorld, p, comm, topology = getARGV("-hpddm_master_topology", 0), exclude = (usedARGV("-hpddm_master_exclude") != -1));

if(verbosity > 0 && mpirank == 0) {
    cout << " --- " << mpirank << "/" << mpisize;
    cout << " - heat-3d.edp - input parameters: refinement factor = " << s << " - overlap = " << overlap << endl;
}

int[int] LL = [1,2, 1,1, 1,1];
meshN ThBorder, Th = cube(1, 1, 1, [x, y, z]);
fespace Wh(Th, Pk);                         // local finite element space
int[int] arrayIntersection;                 // ranks of neighboring subdomains
int[int][int] restrictionIntersection(0);   // local-to-neighbors renumbering
real[int] D;                                // partition of unity
{
    meshN ThGlobal = cube(getARGV("-global", 10), getARGV("-global", 10), getARGV("-global", 10), [x, y, z], label = LL);      // global mesh
    build(Th, ThBorder, ThGlobal, fakeInterface, s, overlap, D, arrayIntersection, restrictionIntersection, Wh, Pk, comm, excluded)
}

real[int] rhs;                              // local right-hand side
matrix<real> Mat;                           // local operator
matrix<real> M;                             // local mass matrix
{                                           // local weak form
    meshN ThAugmented = Th + ThBorder;
    varf vPb(u, v) = intN(ThAugmented)(u * v + dt * (grad(u)' * grad(v))) + intN(ThAugmented)(dt * v) + on(1, u = 0.0);
    fespace WhAugmented(ThAugmented, Pk);
    Mat = vPb(WhAugmented, WhAugmented, tgv = -1);
    real[int] rhsFull = vPb(0, WhAugmented, tgv = -1);
    matrix R = interpolate(Wh, WhAugmented);
    varf vPbM(u, v) = intN(ThAugmented)(u * v);
    M = vPbM(WhAugmented, WhAugmented);
    renumbering(M, R, rhsFull, rhs);
    renumbering(Mat, R, rhsFull, rhs);
}
ThBorder = cube(1, 1, 1, [x, y, z]);

dschwarz A(Mat, arrayIntersection, restrictionIntersection, scaling = D);

matrix<real> Opt;                           // local operator with optimized boundary conditions
dpair ret;
{
    int solver = getOption("schwarz_method");
    if(solver == 1 || solver == 2 || solver == 4) { // optimized Schwarz methods
        fespace Ph(Th, P0);
        real kZero = getARGV("-kZero", 10.0);
        Ph transmission = kZero;
        varf vOptimized(u, v) = intN(Th)(u * v + dt * (grad(u)' * grad(v))) + intN1(Th, fakeInterface)(transmission * (u * v)) + on(1, u = 0.0);
        Opt = vOptimized(Wh, Wh, tgv = -1);
    }
    if(mpisize > 1 && isSetOption("schwarz_coarse_correction")) { // two-level Schwarz methods
        if(excluded)
            attachCoarseOperator(mpiCommWorld, A/*, A = noPen, B = overlapRestriction, threshold = 2. * h[].max / diam*/);
        else {
            varf vPbNoPen(u, v) = intN(Th)(u * v + dt * (grad(u)' * grad(v))) + on(1, u = 0.0);
            matrix<real> noPen = vPbNoPen(Wh, Wh, solver = CG);
            if(deflation == "geneo") // standard GenEO, no need for RHS -> deduced from LHS (Neumann matrix)
                attachCoarseOperator(mpiCommWorld, A, A = noPen/*, threshold = 2. * h[].max / diam,*/, ret = ret);
            else if(deflation == "dtn") {
                varf vMass(def(u), def(v)) = intN1(Th, fakeInterface)(u * v);
                matrix<real> massMatrix = vMass(Wh, Wh, solver = CG);
                attachCoarseOperator(mpiCommWorld, A, A = noPen, B = massMatrix, pattern = Opt/*, threshold = k,*/, ret = ret);
            }
            else if(deflation == "geneo-2") // GenEO-2 for optimized Schwarz methods, need for RHS (LHS is still Neumann matrix)
                attachCoarseOperator(mpiCommWorld, A, A = noPen, B = Opt, pattern = Opt/*, threshold = 2. * h[].max / diam,*/, ret = ret);
        }
    }
}
/*# SolvePlot #*/
set(A, sparams = "-hpddm_reuse_preconditioner=1");
Wh<real> def(u) = init(0.0);    // local solution
for(int i = 0; i < iMax; ++i) {
    real[int] newRhs(rhs.n);
    dmv(A, M, u[], newRhs);     // newRhs = M * u[]
    newRhs += rhs;

    if(Opt.n > 0)       // optimized Schwarz methods
        DDM(A, u[], newRhs, excluded = excluded, ret = ret, O = Opt);
    else
        u[] = A^-1 * newRhs;

    plotMPI(Th, u[], "Global solution", Pk, def, real, 3, 0)
}
/*# SolvePlotEnd #*/
```

## Distributed vectors in HPDDM

We give here some hints on the way vectors are distributed among $np$ processes when using FreeFem++ interfaced with HPDDM. The set of degrees of freedom ${\mathcal N}$ is decomposed into $np$ overlapping sets $({\mathcal N}_i)_{1\le i\le np}$. A MPI-process is in charge of each subset. Let $n:=\#{\mathcal N}$ be the number of degrees of freedom of the global finite element space. Let $R_i$ denote the restriction operator from $\R^n$ onto $\R^{\#{\mathcal N}_i}$. We have also defined local diagonal matrices $D_i\in \R^{\#{\mathcal N}_i}\times \R^{\#{\mathcal N}_i}$ so that we have a partition of unity at the algebraic level:

\begin{equation}
	\label{eq:hpddm:14}
  {\mathbf U} = \sum_{i=1}^{np} R_i^T\,D_i\,R_i\,{\mathbf U}\ \ \ \ \forall\ {\mathbf U}\in\R^n\,.
\end{equation}

A global vector ${\mathbf U}\in\R^n$ is actually not stored. Rather, it is stored in a distributed way. Each process $i$, $1\le i\le N$, stores the local vector ${\mathbf U}_i:=R_i {\mathbf U}\in \R^{\#{\mathcal N}_i}$.

It is important to ensure that the result of all linear algebra operators applied to this representation are coherent.

As an example, consider the scalar product of two distributed vectors ${\mathbf U}, {\mathbf V} \in \mathbb{R}^{n}$. Using the partition of unity~\eqref{eq:hpddm:14} $\codered$, we have:

\begin{align*}({\mathbf U}, {\mathbf V}) = \left({\mathbf U}, \sum_{i=1}^{np} R_i^T D_i R_i {\mathbf V}\right) &= \sum_{i=1}^{np} (R_i {\mathbf U}, D_i R_i {\mathbf V})\\
&=\sum_{i=1}^{np} \left({\mathbf U}_i, D_i {\mathbf V}_i\right)\,.
\end{align*}

Thus, the formula for the scalar product is:

\begin{equation*}
({\mathbf U}, {\mathbf V}) = \sum_{i = 1}^{np} (R_i {\mathbf U}, D_i R_i {\mathbf V})\,.
\end{equation*}

Local scalar products are performed concurrently. Thus, the implementation is parallel except for the sum which corresponds to a `:::freefem MPI_Reduce` call across the $np$ MPI processes. Note also that the implementation relies on the knowledge of a partition of unity so that the FreeFem++ syntax is `:::freefem dscalprod(D,u,v)`.

A `:::freefem axpy` procedure $y \leftarrow \alpha\,x+y$ for $x,y\in \mathbb{R}^{n}$ and $\alpha\in\R$ is easily implemented concurrently for distributed vectors in the form:

\[
y_i \leftarrow \alpha\,x_i+y_i\,, \forall\ 1\le i \le np\,.
\]

The matrix vector product is more involved and details are given in the SIAM book  [https://www.ljll.math.upmc.fr/nataf/OT144DoleanJolivetNataf_full.pdf](An Introduction to Domain Decomposition Methods: algorithms, theory and parallel implementation) \cite{Dolean:2015:IDD} $\codered$ and even more details are given in [http://jolivet.perso.enseeiht.fr/thesis.pdf](P. Jolivet's PhD manuscrit).