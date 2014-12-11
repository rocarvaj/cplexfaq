# A biased CPLEX FAQ
This is a list of frequently asked questions that may get lost in the
bottomless pit that is the [CPLEX forum](https://www.ibm.com/developerworks/community/forums/html/forum?id=11111111-0000-0000-0000-000000002059). The list is biased because most of the questions are my questions and usually apply to the C callable library.

Please check that the answer still applies to the version of CPLEX you are using.

Do you have anything to add? Send me an [email](http://rocarvaj.uai.cl).

### How to limit cut generation to root node only?
**A:** Implement a cut callback function which does the following:

1. Query depth of node using `CPXXgetcallbacknodeinfo()` with `whichinfo = CPX_CALLBACK_INFO_NODE_DEPTH` in the C API.
2. If `depth==0`, let CPLEX take it's default action using `*useraction_p = CPX_CALLBACK_DEFAULT`.. Else, abort cut loop using `*useraction_p = CPX_CALLBACK_ABORT_CUT_LOOP` and proceed to branching.

Answer by AnirudhSubramanyam ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=cd574f99-5f7f-4bd6-94c4-e5db4c420389&ps=25)).

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
