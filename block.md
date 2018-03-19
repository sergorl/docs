
## Структура блока блокчейна

1. **Нормальная** полная форма блока:
```rust
pub struct Block {
	/// The header with metadata and commitments to the rest of the data
	pub header: BlockHeader,
	/// List of transaction inputs
	pub inputs: Vec<Input>,
	/// List of transaction outputs
	pub outputs: Vec<Output>,
	/// List of kernels with associated proofs (note these are offset from
	/// tx_kernels)
	pub kernels: Vec<TxKernel>,
}
```


**TxKernel** - cтруктура, содержащая доказательство равенства суммы входов и суммы выходов:
```rust
pub struct TxKernel {
	/// Options for a kernel's structure or use
	pub features: KernelFeatures,

	/// Options for a kernel's structure or use
	pub struct KernelFeatures: u8 {
		/// No flags
		const DEFAULT_KERNEL = 0b00000000;
		/// Kernel matching a coinbase output
		const COINBASE_KERNEL = 0b00000001;
	}

	/// Fee originally included in the transaction this proof is for.
	pub fee: u64,
	/// This kernel is not valid earlier than lock_height blocks
	/// The max lock_height of all *inputs* to this transaction
	pub lock_height: u64,
	/// Remainder of the sum of all transaction commitments. If the transaction
	/// is well formed, amounts components should sum to zero and the excess
	/// is hence a valid public key.
	pub excess: Commitment,
	/// The signature proving the excess is a valid public key, which signs
	/// the transaction fee.
	pub excess_sig: Signature,
}
```

**Заголовок** блока:
```rust
pub struct BlockHeader {
	/// Version of the block
	pub version: u16,
	/// Height of this block since the genesis block (height 0)
	pub height: u64,
	/// Hash of the block previous to this in the chain.
	pub previous: Hash,
	/// Timestamp at which the block was built.
	pub timestamp: time::Tm,
	/// Merklish root of all the commitments in the TxHashSet
	pub output_root: Hash,
	/// Merklish root of all range proofs in the TxHashSet
	pub range_proof_root: Hash,
	/// Merklish root of all transaction kernels in the TxHashSet
	pub kernel_root: Hash,
	/// Nonce increment used to mine this block.
	pub nonce: u64,
	/// Proof of work data.
	pub pow: Proof,
	/// Total accumulated difficulty since genesis block
	pub total_difficulty: Difficulty,
	/// Total accumulated sum of kernel offsets since genesis block.
	/// We can derive the kernel offset sum for *this* block from
	/// the total kernel offset of the previous block header.
	pub total_kernel_offset: BlindingFactor,
}
```

2. **Компактная** форма:
```rust
pub struct CompactBlock {
	/// The header with metadata and commitments to the rest of the data
	pub header: BlockHeader,
	/// Nonce for connection specific short_ids
	pub nonce: u64,
	/// List of full outputs - specifically the coinbase output(s)
	pub out_full: Vec<Output>,
	/// List of full kernels - specifically the coinbase kernel(s)
	pub kern_full: Vec<TxKernel>,
	/// List of transaction kernels, excluding those in the full list
	/// (short_ids)
	pub kern_ids: Vec<ShortId>,
}

/// 6-битный ключ для идентификации inputs/outputs/kernels
/// Формируется из хэша заголовка и его поля nonce с применением SipHash-функции 
pub struct ShortId([u8; 6]);
```

**Компактная форма** может быть получена из **нормальной формы** следующим образом:
- cохраняются только outputs, содержащие OutputFeatures::COINBASE_OUTPUT для формирования списка Vec<Output>;
- сохраняются только kernels, содержащие KernelFeatures::COINBASE_KERNEL для формирования списка Vec<TxKernel>;
- генерируется случайное число nonce (оно используется для формирования ShortId из kernels, неподходящих под описание выше);
- kernels, не содержащие KernelFeatures::COINBASE_KERNEL заменются ShortId, формируя список Vec<ShortId>.
Все cписки outputs/kernels/kern_ids хранятся в отсортированном лексикографическом порядке.
	
**Нормальная форма** может быть получена из блока **компактной формы**, прошедшего успешно валидацию, и некоторого списка транзакций, та как в этом случае при формировании нового блока из блока компактной формы ипользуется только заголовок BlocHeader.

Новый блок **нормальной формы** може быть создан из:
- заголовка предыдущего блока,
- некоторого списка транзакций,
- некоторого Output'a и соотвествующего ему TxKernel'a,
- и некоторого парамера сложности Difficulty.
В процессе создании нового блока:
1. каждая транзакция валидируется и в случае успеха помещается внутрь блока (заполняя соответствующие поля-списки inputs/outputs/kernels в лексикографическом порядке);
2. а BlindingFactor'ы соответствующих транзакций суммируются и помещаются в заголовок блока total_kernel_offset;
3. текущая сложность складывается со сложностью предыдущего заголовка и поомещается в новый заголовок;
4. высота нового блока стноавится равна высоте предыдущего + 1;
5. фиксируется время создания блока.


