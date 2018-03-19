## Хранение и обработка блокчейна

**Orphan** — это блок, который не имеет известного предка в самой длинной цепочке блоков.Это блоки, созданные на другом блоке, который больше не является активным концом самой длинной цепи. Некоторые ноды, возможно, считали, что это лучший блок в определенный момент, но они переключились на другую цепь, которая больше не содержит соответствующий блок. Они действительны, проверены, и их происхождение до блока генезиса полностью известно, они просто не активны в настоящее время. Название 

```rust
struct Orphan {
	block: Block,
	opts:  Options, // ???
	added: Instant, // время присоединения в пул Orphan'ов
}
```

#### Пул Orphan'ов:
- блоки хранятся в HashMap по хэшу Orphan'a и защищены [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html)
- связь родитель-ребёнок осуществлена через HashMap<Hash, Hash, защищенную [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) где, Hash - хэш блока
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

#### Chain предоставляет следующие методы для обработки блокчейна:
- Существует ли блокчейн в БД rocksdb?
```rustpub fn chain_exists(db_root: String) -> bool;```

- Создает хранилище типа "ключ-значение", внутри которого находятся БД: 
1. Thread-safe rocksdb wrapper
		pub struct Store {
			rdb: RwLock<DB>,
		}
2. три PMMR-дерева (внутри TxHashSet):
- PMMRHandle<OutputStoreable>     
- PMMRHandle<RangeProof>
- PMMRHandle<TxKernel>	
3. односвязный список блоков блокчейна				 
```rust pub fn init(
	db_root:      String,
	adapter:      Arc<ChainAdapter>, 
	genesis:      Block,
	pow_verifier: fn(&BlockHeader, u32) -> bool,
) -> Result<Chain, Error>; ```
