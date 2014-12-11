# A biased CPLEX FAQ
This is a list of frequently asked questions that may get lost in the
bottomless pit that is the [CPLEX forum](https://www.ibm.com/developerworks/community/forums/html/forum?id=11111111-0000-0000-0000-000000002059). The list is biased because most of the questions are my questions and usually apply to the C callable library.

Please check that the answer still applies to the version of CPLEX you are using.

Do you have anything to add? Send me an [email](http://rocarvaj.uai.cl).

### How to limit cut generation to root node only?
Implement a cut callback function which does the following:

1. Query depth of node using `CPXXgetcallbacknodeinfo()` with `whichinfo = CPX_CALLBACK_INFO_NODE_DEPTH` in the C API.
2. If `depth==0`, let CPLEX take it's default action using `*useraction_p = CPX_CALLBACK_DEFAULT`.. Else, abort cut loop using `*useraction_p = CPX_CALLBACK_ABORT_CUT_LOOP` and proceed to branching.

Answer by AnirudhSubramanyam ([link](https://www.ibm.com/developerworks/community/forums/html/topic?id=cd574f99-5f7f-4bd6-94c4-e5db4c420389&ps=25)).
