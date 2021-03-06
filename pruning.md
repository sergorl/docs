## Данный материал крайне сырой и таит множество непонятных явлений. Выкладываю его на всеобщее обозрение для попыток прояснить проблемы внутри. ##

## MMR
***Merkle Mountain Range (MMR)*** - бинарное дерево, узлы которого есть хэши некоторых сущностей, а сами узлы хранятся в некотором массиве в порядке post-order. Так на рисунке ниже нумерация узлов соответствует их индексации в массиве, которая и отображает post-order.
![](https://github.com/sergorl/docs/blob/master/mmr.png)

Важно прояснить следующие понятия для **MMR-дерева**:
* Все родители являются пиками дерева (как минимум дерева их трёх элементов).
* Узлы, имеющие одного общего родителя, являются братьями.
* Все узлы имеют высоту.
* Child-узлы имеют нулевую высоту.

## Pruning и PMMR
***Pruning*** – техника сокращения размерности бинарного дерева для упрощения проверки наличия некоторого элемента-узла в структуре дерева.

Согласно **grin**'у есть несколько ситуаций, в которых применяется **pruning** данных:
* Полностью прошедший валидацию узел может избавиться от избыточных данных для освобождения памяти;
   **??? У узла есть валидация ???**
* Частично прошедший валидацию узел  (similar to SPV) может быть не заинтересован в приёме или хранениии данных;
* Когда новый узел, присоединяется к сети, то временно он может вести себя как частично прошедший валидацию для ускорения работы. В дальнейшем подобный узел может пройти валидацию полностью.

**Pruning** возможно осуществлять только для **сhild-узлов**, высота которых равна нулю.

***Prunable Merkle Mountain Range (PMMR)*** - **MMR-дерево**, предварительно прошедшее **pruning**. Такое дерево существует для некоторого стартового **child-узла** из полного MMR-дерева и задаётся двумя массивами:
* массивом пиков и
* массивом узлов-братьев для стартового узла и пиков соответственно.

Данная структура **PMMR-дерева** в виде массивов из пиков и узлов-братьев хранится в полях **MerkleProof**'a у каждого **Input**'a транзакции.

![](https://github.com/sergorl/docs/blob/master/pmmr.png)

Так для узла №4:
* пики = {6, 7, 15}, **??? зачем хранить пики, если узлов-братьев достаточно для верификации ???**
* братья = {5, 3, 14}.

**???** Алгоритм **pruning**'a находится в строчках [423-459](https://github.com/beam-mw/grin/blob/master/core/src/core/pmmr.rs). В данном алгоритме сохранются только начальный узел и пики (как на рисунке выше), которые помещаются в некоторый remove_list структуры, реализующей trait Backend. Странность заключается в том, что те узлы, которые важны при проверке в контексте процедуры **Merkle Proof** помещаются в некий remove_list (**почему список назван remove?!**).

## Merkle Proof
***Merkle Proof*** - процедура проверки наличия именно данного узла в **MMR-дереве**.

```rust
/// A Merkle proof.
/// Proves inclusion of an output (node) in the output MMR.
/// We can use this to prove an output was unspent at the time of a given block
/// as the root will match the output_root of the block header.
/// The path and left_right can be used to reconstruct the peak hash for a given tree
/// in the MMR.
/// The root is the result of hashing all the peaks together.
pub struct MerkleProof {
	/// The root hash of the full Merkle tree (in an MMR the hash of all peaks)
	pub root: Hash,
	/// The hash of the element in the tree we care about
	pub node: Hash,
	/// The full list of peak hashes in the MMR
	pub peaks: Vec<Hash>,
	/// The siblings along the path of the tree as we traverse from node to peak
	pub path: Vec<Hash>,
	/// Order of siblings (left vs right) matters, so track this here for each
	/// path element
	pub left_right: Vec<bool>,
}
```

**Pruning** полезен для **Merkle Proof**, так как позволяет устранить избыточность узлов в **MMR-дереве**, то есть удалить узлы, которые не участвуют в процедуре, повысив эффективность проверки по времени. 

**???** Расмотрим алгоритм **Merkle Proof** в **PMMR-дереве**. Данный алгоритм проверки описан в строчках [208-251](https://github.com/beam-mw/grin/blob/master/core/src/core/pmmr.rs) и является крайне НЕПОНЯТНЫМ, так как предполагет, что для каждого хэша внутри MerkleProof'а каждого входа транзакции хранятся известные для него узлы-братья и все пики дерева (**??? зачем пики вообще нужны если узлов-братьев достаточно для проверки через расчет корня по узлу и сравнения рассчитанного корня с оригиналом-корнем полного дерева ???**). Сам алгоритм фактически состит из двух этапов:

1. Для проверяемого узла из массива братьев изымается его узел-брат и из хэшей этих узлов формируется новый хэш нового узла-родителя. Далее для нового узла процедура повторяется рекурсивно до тех пор пока не опустеет массив узлов-братьев. При этом каждый раз формируется новый MerkleProof, на котором и запускается рекурсивно функция проверки. Странность заключается в том, что когда рекурсия завершится, то
конечный хэш должен являться корнем дерева и вроде бы надо провести сравнение с корнем-оригиналом, который хранится в данном MerkleProof'е, но нет - запускается следующий этап.

2. Итеративно пробегаем по списку всех пиков и считаем новый хэш от (хэша предыдущего пика, и хэша текущего). На первой итерации в качестве первого хэша выступает хэш, полученный на предыдущем этапе (**который уже является корнем дерева!**). Конечный результат сравниваем с корнем дерева: в случае равенства верификация проведена успешно и наоборот. **???**
