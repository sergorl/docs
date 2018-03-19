## Хранение и обработка блокчейна

Блок может находится в блокчейне, а может быть **Orphan**'ом и тогда должен хранится в **пуле Orphan'ов**, ожидая воможности быть добавленным в блокчейн, так чтобы остальные части системы увидели это добавление.

**Orphan** — это блок, который не имеет известного предка в самой длинной цепочке блоков. Это блоки, созданные на другом блоке, который больше не является активным концом самой длинной цепи. Некоторые ноды, возможно, считали, что это лучший блок в определенный момент, но они переключились на другую цепь, которая больше не содержит соответствующий блок. Они действительны, проверены, и их происхождение до блока генезиса полностью известно, они просто не активны в настоящее время. 

![Orphan](https://github.com/sergorl/docs/blob/master/orphan.png)

```rust
struct Orphan {
	block: Block,
	opts:  Options, 
	added: Instant, // время присоединения в пул Orphan'ов
}
/// Options for block validation
pub struct Options: u32 {
	/// No flags
	const NONE = 0b00000000;
	/// Runs without checking the Proof of Work, mostly to make testing easier.
	const SKIP_POW = 0b00000001;
	/// Adds block while in syncing mode.
	const SYNC = 0b00000010;
	/// Block validation on a block we mined ourselves
	const MINE = 0b00000100;
}
```

#### Пул Orphan'ов:
- блоки хранятся в HashMap по хэшу Orphan'a и защищены [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html)
- связь родитель-ребёнок осуществлена через HashMap<Hash, Hash>, защищенную [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) где, Hash - хэш блока
```rust
struct OrphanBlockPool {
	// blocks indexed by their hash
	orphans: RwLock<HashMap<Hash, Orphan>>,
	// additional index of previous -> hash
	// so we can efficiently identify a child block (ex-orphan) after processing a block
	prev_idx: RwLock<HashMap<Hash, Hash>>,
}
```

#### Chain - cтруктура для хранения и обработки блокчейнов, содержит
- вершину блокчейна **Tip** 
- пул орфанов
- функцию проверки PoW
```rust
struct Chain {
	db_root:      String,
	store:        Arc<ChainStore>,
	adapter:      Arc<ChainAdapter>,

	head:         Arc<Mutex<Tip>>,
	orphans:      Arc<OrphanBlockPool>,
	txhashset:    Arc<RwLock<txhashset::TxHashSet>>,

	// POW verification function
	pow_verifier: fn(&BlockHeader, u32) -> bool,
}
```
Начальная инициализация БД происходит через создание структуры Chain по первому genesis-блоку.

#### Tip - вершина блокчейна, содержит:
- хэши последнего и предпоследнего блоков
- высоту (длину) блокчейна
- общую сложность блокчейна
```rust
struct Tip {
	/// Height of the tip (max height of the fork)
	pub height:           u64,
	/// Last block pushed to the fork
	pub last_block_h:     Hash,
	/// Block previous to last
	pub prev_block_h:     Hash,
	/// Total difficulty accumulated on that fork
	pub total_difficulty: Difficulty,
}
```	

## Как блоки присоединяются к блокчейну?
1. Сначала осуществляется попытка присоединить блок к блокчейну, для этого:
	- проверяется есть ли данный блок уже в цепочке блокчейна;
	- проводится валидация BlockHeader'а данного блока;
	- проводится валидация самого блока;
	- сохраняются в БД метаданные для работы с PMMR-деревьями.
2. Результаты попытки добавления блока могут быть следующими:
	- блок может стать новой вершиной цепочки - осуществляется переход к **шагу №3**,
	- а может сформировать альтернативную fork-цепочку, параллельную текущей - осуществляется переход к **шагу №3**;
	- блок пройдёт валидацию, но не добавится в блокчейн - в таком случае он добавляется в пул Orphan'ов, (возвращая ошибку) и **заканчивая обработку блока и блокчейна**.
3. Происходит уведомление всей системы об обновлениях в блокчейне: обновлении вершины цепочки или формирования альтернативного fork'a блокчейна.
4. Проверяется пул Orphan'ов: если в пуле есть child-блок обработанного блока, то он изымается из пула и осуществляестя попытка добавить его в цепочку. **Подобные child-блоки попадают на шаг №1. Если в пуле нет подходящего child-блока, то обработка завершается.**
Итерация в пуле Orphan'ов осуществляется по хэшу родителя (более старого блока), который индексирует ребёнка (более новый блок). 


## Как заголовки (BlockHeader'ы) присоединяются к блокчейну?
1. Осуществляется обработка BlockHeader'а:
	- по хэшу заголовка проверятся есть ли данный заголовок уже в цепочке BlockHeader'ов блокчейна;
	- происходит валидация заголовка.
2. В случае успеха на предыдущем шаге заголовок добавляется в цепочку BlockHeader'ов.
3. Вершина цепочки BlockHeader'ов меняется на обработанный щаголовок, если его общая сложность (total_difficulty) больше, чем общая сложность у старой вершины.
