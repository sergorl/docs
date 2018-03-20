There may be several contexts in which data can be pruned:

* A fully validating node may get rid of some data it has already validated to
free space.
* A partially validating node (similar to SPV) may not be interested in either
receiving or keeping all the data.
* When a new node joins the network, it may temporarily behave as a partially
validating node to make it available for use faster, even if it ultimately becomes
a fully validating node.

Pruning - процедура удаления избыточных не участвующих в верификации узлов MMR-дерева.

Есть несколько ситуаций, в которых применяется **pruning**:
* Полностью прошедший валидацию узел может избавиться от избыточных данных для освобождения памяти;
* Частично прошедший валидацию узел может быть не заинтересован в приёме или хранениии данных;
* Когда новый узел присоединсяется к сети он может проходить частичную валидацию для ускорения работы. В дальнейшем подобный узел может  пройти валидацию полностью.


Нумерация узлов соответствует их индексации в массиве.

1. Pruning – техника сокращения размерности бинарного дерева для упрощения проверки наличия некоторого элемента (узла) в структуре дерева.

2. Pruning возможно осуществлять только для сhild-узлов, высота которых равна нулю.

3. Pruning позволяет устранить избыточность узлов, то есть удалить узлы, которые не участвуют в процедуре Merkle Proof, повысив эффективность проверки. Так для того, чтобы проверить некоторый сhild-узел, например 4, достаточно знать только пики дерева и родственные узлы-братья (узлы, имеющие общего родителя)
