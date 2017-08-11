# A biased CPLEX FAQ
This is a list of frequently asked questions that may get lost in the
bottomless pit that is the [CPLEX forum](https://www.ibm.com/developerworks/community/forums/html/forum?id=11111111-0000-0000-0000-000000002059). The list is biased because most of the questions are my questions and usually apply to the C callable library.

Please check that the answer still applies to the version of CPLEX you are using.

Do you have anything to add? Send me an [email](http://rocarvaj.uai.cl).

1. [How to limit cut generation to root node only?](#how-to-limit-cut-generation-to-root-node-only)
2. [How to identify different types of fathomed nodes?](#how-to-identify-different-types-of-fathomed-nodes)
3. [How to add information to a node, but without having to manually branch as CPLEX?](#how-to-add-information-to-a-node-but-not-having-to-manually-branch-as-cplex)
4. [How can I turn off all presolve options for MIP?](#how-can-i-turn-off-all-presolve-options-for-mip)
5. [How to perform customized strong branching?](#how-to-perform-customized-strong-branching)
6. [How are pseudocosts initialized in CPLEX?](#how-are-pseudocosts-initialized-in-cplex)
7. [Deterministic behavior of CPLEX: ticks or seconds?](#deterministic-behavior-of-cplex-ticks-or-seconds)

### How to limit cut generation to root node only?
**A:** Implement a cut callback function which does the following:

1. Query depth of node using `CPXXgetcallbacknodeinfo()` with `whichinfo = CPX_CALLBACK_INFO_NODE_DEPTH` in the C API.
2. If `depth==0`, let CPLEX take it's default action using `*useraction_p = CPX_CALLBACK_DEFAULT`.. Else, abort cut loop using `*useraction_p = CPX_CALLBACK_ABORT_CUT_LOOP` and proceed to branching.

Answer by AnirudhSubramanyam ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=cd574f99-5f7f-4bd6-94c4-e5db4c420389&ps=25)).

**A2:** A way to do this without callbacks is the following:

1. Set the node limit to 1 and call solve(). This will process the root node with default cut settings (or whatever cut settings you specified).
2. Clear the node limit and disable all cuts. Then call solve() again. This should proceed right where the previous step left off (right after the root node) and should not try to separate any cuts anymore.

### How to identify different types of fathomed nodes?
I want to identify different types of nodes in the B&B tree:

* Fathomed (by bound)
* Integer
* Infeasible

**A:** I think you will have to use node user data with a combination of callbacks:

* In a branch callback create the branches that CPLEX would create but attach a user data object to each node (function CPXbranchcallbackbranchasCPLEX is a convenient way to do that).
* In a node select callback just select the node that CPLEX would suggest but change the node's user data to reflect that the node has been selected (functions CPXcallbacksetuserhandle and CPXcallbacksetnodeuserhandle can be used to change a node's user object).
* In a solve callback just use the default solving strategy but update the node's user object to indicate that the solve callback was invoked. Nodes that are LP infeasible can be detected in the solve callback.
* In a delete callback check the user object of the node. If the solve callback was not invoked for the node and the node was never selected by the node selection callback then the node was fathomed by bound. If the node was selected but the solve callback was not invoked on it then it was proven infeasible by node presolve. If the solve callback was invoked then you already know whether it was feasible or infeasible.
* Integer feasible nodes can be detected in the branch callback by testing that CPLEX would not create any branches.

Answer by DanielJunglas ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=ac79e5b1-97c6-454b-9614-48f92afe83bb&ps=25)).

### How to add information to a node, but without having to manually branch as CPLEX?
**A:** In a branch callback use the `CPXbranchcallbackbranchasCPLEX` function and just add your node handle to
the node.

### How can I turn off all presolve options for MIP?
**A:** You should turn off presolve (`CPX_PARAM_PREIND`, `CPX_PARAM_RELAXPREIND` and `CPX_PARAM_PRSLVND`), probing (`CPX_PARAM_PROBE`), cplex cuts (`CPX_PARAM_EACHCUTLIM` and `CPX_PARAM_FRACCUTS` and `CPX_PARAM_CUTSFACTOR`) and cplex heuristics (`CPX_PARAM_HEURFREQ`) to use CPLEX as a "pure" LP solver for the purposes of branch and bound.

Answer by AnirudhSubramanyam ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=bde0e565-e4a7-4d74-bf90-001a8829f2d6&ps=25))

*Note:* This might not be a full answer.

### How to perform customized strong branching?
**A:** In a C API branch callback, you would query the nodelp with `CPXgetcallbacknodelp()`. Then, query the optimal basis and dual norms with `CPXgetbasednorms()`. Then do:

1. load in the basis and dual norms with `CPXcopybasednorms()`,
2. add the additional branching constraints with `CPXaddrows()`,
3. solve the LP with `CPXdualopt()` and query solution information with `CPXsolution()`,
4. remove the branching constraints with `CPXdelrows()`,
5. repeat 1-4 until all branching candidates have been evaluated,
6. load in the basis and dual norms with `CPXcopybasednorms()`,
7. call `CPXdualopt()` to restore the internal LP state that CPLEX needs to proceed.

I think this should work, but maybe CPLEX does not like you to work directly on the nodelp. If this is the case, then you just need to copy the nodelp locally and work on this local copy instead (which then saves you steps 6 and 7).

### How are pseudocosts initialized in CPLEX?
**Q:** When calling `CPXgetcallbackpseudocosts()` just before branching at the root note, I see that the pseudocosts are already initialized. Are they initialized using strong branching?

**A:** With the default variableselection, they are indeed initialized using strong branching. Even if there is a user branch callback in place.

Answer by Ed Klotz ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=b9f3d077-b2ab-4a70-b01c-6412a4bd3a95&ps=25)).

### Deterministic behavior of CPLEX: ticks or seconds?
[A post by Marc-André Carle](http://www.thequestforoptimality.com/deterministic-behavior-of-cplex-ticks-or-seconds/) on CPLEX deterministic ticks. He asks if:

1. CPLEX will yield the same tick count when ran multiple times on the same computer;
2. CPLEX will yield the same tick count when ran on different computers.

Tobias Acheterberg posts a very clarifying comment:

> Hi Marc-Andre,
> 
> the answers to your two experiments are “yes” for the first, and “no” for the second.
> 
> It is very typical for CPLEX workloads that the bottleneck for computations is the memory bus. If you have a cache miss, then the CPU has to wait very long to load the required data. Consequently, the deterministic ticks that CPLEX measures is basically the number of memory accesses that CPLEX performs (as a proxy for the number of cache misses). Yes: we have instrumented all of our code to count or estimate memory accesses. A lot of work but a big success as this does not only enable the deterministic time limit and deterministic parameter tuning (i.e., you tune parameters for a given set of models, and when you do the same again you will get exactly the same recommendations as in the first tuning session), but also lots of under-the-hood performance improvements.
> 
> For the deterministic time this means that two runs using the same CPLEX binary with the same settings (including that if the the “threads” parameter is still set to 0, then the machines must have the same number of cores) and the same data will produce the same deterministic time. This is independent on the load of the machine. If you have other jobs running in the background the wall-clock run-time can be significantly higher, but the deterministic time will not change. This is very useful for computational experiments, because it means that it is no longer necessary to have exclusive access to the machine. Moreover, it allows to conduct long running automated tuning sessions on machines that are used for other tasks as well.
>
> The fact that the wall-clock run-time of CPLEX is mostly determined by the number of cache misses means that the deterministic time is a very good proxy for wall-clock time. The ratio of wall-clock time to deterministic time depends of course on the hardware, but it is not very dependent on the input data. This allows to set reasonable deterministic time limits: just measure the deterministic/wall-clock time ratio on your machine for some reasonable CPLEX work loads and from then on use a deterministic time limit by multiplying your wall-clock time limit with the ratio that you observed. As a consequence, CPLEX users are able to produce algorithms that use limited time optimization runs as sub-procedure and that will still show deterministic behavior. And when those algorithms are run on faster hardware (of the same architecture), the algorithm will behave identically, except that it will run faster.

### Is there a book that describes the algorithms CPLEX uses?
**A:** While there is no publication regarding the details of the implementations in CPLEX, the following publications contain plenty of useful information.


**THE SIMPLEX ALGORITHM**
* Bixby, Robert, Progress in Linear Programming. ORSA Journal on Computing, Volume 6, Number 1, Winter 1994. [Aspects of CPLEX implementation, with a good list of further references.]
* Nazareth, Larry, Computer Solution of Linear Programs, Oxford University Press, 1987. [Good general reference.]
* Wolfe, Philip, The Simplex Method for Quadratic Programming, Econometrica, Vol 27, No. 3, July 1959.
* Cottle, Richard, ed. The Basic George B. Dantzig, Stanford University Press, 2003 [General reference on numerous topics, but specifically Ch. 21 for QP simplex method]


**THE BARRIER METHOD**
* Lustig, I.J., Marsten, R. and Shanno, D.F. (1994). Interior Point Methods for Linear Programming: Computational State of the Art, ORSA Journal on Computing, Volume 6(1), 1-14. [Overview of the implementation found in CPLEX.]


**BRANCH AND BOUND**
* Land, A. and Powell, S., Computer codes for problems of integer programming, in P.L. Hammer, E.L. Johnson, and B.H. Korte, editors, Discrete Optimization II, Annals of Discrete Mathematics Volume 5, North Holland, Amsterdam, 1979, pages 221-269.
* Forrest, J., Hirst, J., and Tomlin J., Practical Solution of Large Mixed Integer Programming Problems with Umpire. Management Science, Volume 20, Number 5, January 1974, pages 736-772.
* Hoffman, K.L. and Padberg, M., (1985) LP-Based Combinatorial Problem Solving. Annals of Operations Research Volume 4, pages 145-194.
* Bixby, Robert E., Fenelon, Mary, Zonghao Gu, Rothberg, Ed, Wunderling, Roland, MIP: Theory and Practice Closing the Gap. In: M. J. D. Powell and S. Scholtes (eds.). System Modelling and Optimization: Methods, Theory and Applications. Kluwer. [Available in PDF format at MIPLIB - Mixed Integer Problem Library]
* More generally, George L. Nemhauser and Laurence A. Wolsey's book Integer and Combinatorial Optimization, 1999, from Wiley-Interscience Series in Discrete Mathematics and Optimization, has a huge number of MIP references at the back.
* Wolsey, Integer Programming, John Wiley & Sons 1998 [good fundamental branch & bound reference]


**PRESOLVE**
* Savelsbergh, M.W.P. (1994), Preprocessing and probing techniques for mixed integer programming problems. ORSA Journal on Computing, Volume 6, Number 4, pages 445-454.
* Andersen, E.D. and Andersen, K.D. (1995), Presolving in linear programming. Mathematical Programming Volume 71, pages 221-245.
* Brearley, A.L., Mitra, G., and Williams, H.P. (1975). Analysis of mathematical programming problems prior to applying the simplex algorithm. Mathematical Programming Volume 8, pages 54--83.


**PROBING**
*Savelsbergh, M.W.P. (1994). Preprocessing and Probing for Mixed Integer Programming Problems, ORSA Journal on Computing, Volume 6, pages 445-454.


**PARALLEL**
* For a discussion of the parallel simplex method, you can download this technical report:  
CRPC-TR95706 Parallelizing the Dual Simplex Method, Robert E. Bixby, Alexander Martin, December 1995, Revised July 1997. Submitted November 1997.
* Lustig, I.J. and Rothberg, E. (1996). "Gigaflops in Linear Programming", Operations Research Letters 18(4), 157-165.


**QUADRATIC CONSTRAINTS**
* E. D. Andersen, C. Roos, and T. Terlaky, On implementing a primal-dual interior-point method for conic quadratic optimization.

Source: [IBM Support](https://www-304.ibm.com/support/docview.wss?uid=swg21400019)
