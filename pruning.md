## MMR
***Merkle Mountain Range (MMR)*** - бинарное дерево, узлы которого есть хэши некоторых сущностей, а сами узлы хранятся в некотором массиве в порядке post-order. Так на рисунке ниже нумерация узлов соответствует их индексации в массиве, которая и отображает post-order.
![](https://github.com/sergorl/docs/blob/master/mmr.png)

Важно прояснить следующие понятия для **MMR-дерева**:
* Все родители являются пиками дерева (как минимум дерева их трёх элемнтов).
* Узлы, имеющие одного общего родителя являются братьями.
* Все узлы имеют высоту.
* Child-узлы имеют нулевую высоту.

## Pruning и PMMR
***Pruning*** – техника сокращения размерности бинарного дерева для упрощения проверки наличия некоторого элемента-узла в структуре дерева.

There may be several contexts in which data can be pruned:
* A fully validating node may get rid of some data it has already validated to
free space.
* A partially validating node (similar to SPV) may not be interested in either
receiving or keeping all the data.
* When a new node joins the network, it may temporarily behave as a partially
validating node to make it available for use faster, even if it ultimately becomes
a fully validating node.

Есть несколько ситуаций, в которых применяется **pruning**:
* Полностью прошедший валидацию узел может избавиться от избыточных данных для освобождения памяти;
* Частично прошедший валидацию узел может быть не заинтересован в приёме или хранениии данных;
* Когда новый узел, присоединясь к сети, может проходить частичную валидацию для ускорения работы. В дальнейшем подобный узел может  пройти валидацию полностью.

**Pruning** возможно осуществлять только для **сhild-узлов**, высота которых равна нулю.

***Merkle Proof*** - процедура проверки наличия узла в **MMR-дереве**.

**Pruning** полезен для **Merkle Proof**, так как позволяет устранить избыточность узлов, то есть удалить узлы, которые не участвуют в процедуре, повысив эффективность проверки. 

***Prunable Merkle Mountain Range (PMMR)*** - **MMR-дерево**, предварительно прошедшее **pruning**. Такое дерево существует для некоторого стартового **child-узла** из полного MMR-дерева и задаётся двумя массивами:
* массивом пиков и
* массивом узлов-братьев для соответсвущих пиков.

![](https://github.com/sergorl/docs/blob/master/pmmr.png)

