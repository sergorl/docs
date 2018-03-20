There may be several contexts in which data can be pruned:

* A fully validating node may get rid of some data it has already validated to
free space.
* A partially validating node (similar to SPV) may not be interested in either
receiving or keeping all the data.
* When a new node joins the network, it may temporarily behave as a partially
validating node to make it available for use faster, even if it ultimately becomes
a fully validating node.
