
## Сценарий №1. Кошелёк ***A*** пересылает монеты кошельку ***B***

Взаимодействие кошельков строится на основе передачи/приёма контейнера **PartialTx** по ***http***-протоколу. Основная часть контейнера содержит непосредственно саму транзакцию и криптографические параметры (сигнатуру, blind_excess, nonce, kernel_offset):

```rust
/// Helper in serializing the information required during an interactive aggsig
/// transaction
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct PartialTx {
	pub phase: PartialTxPhase,
	pub id: Uuid,
	pub amount: u64,
	pub public_blind_excess: String,
	pub public_nonce: String,
	pub kernel_offset: String,
	pub part_sig: String,
	pub tx: String,
}
```

Внутри контейнера есть флаг **PartialTxPhase**, который обозначает стадию процесса взаимодействия между двумя кошельками для формирования транзакции:

```rust
pub enum PartialTxPhase {
	SenderInitiation,
	ReceiverInitiation,
	SenderConfirmation,
	ReceiverConfirmation,
}
```

#### Рассмотрим процесс формирования выходов транзакции в сценарии "кошелёк ***A*** пересылает монеты кошельку ***B***"

1. Кошелёк ***A*** на своей стороне формирует:
  	- непосредственно саму транзакцию;
 	- **BlindingFactor**;
 	- выходы транзакций **OutputData**, содержащие необходимое количество монет;
	- количество пересылаемых монет **+** вознаграждение **fee**;
	- [**Identifier change_key**](https://github.com/beam-mw/grin/blob/master/keychain/src/extkey.rs).

```rust
pub struct OutputData {
	/// Root key_id that the key for this output is derived from
	pub root_key_id: keychain::Identifier,
	/// Derived key for this output
	pub key_id: keychain::Identifier,
	/// How many derivations down from the root key
	pub n_child: u32,
	/// Value of the output, necessary to rebuild the commitment
	pub value: u64,
	/// Current status of the output
	pub status: OutputStatus,
	/// Height of the output
	pub height: u64,
	/// Height we are locked until
	pub lock_height: u64,
	/// Is this a coinbase output? Is it subject to coinbase locktime?
	pub is_coinbase: bool,
	/// Hash of the block this output originated from.
	pub block: Option<BlockIdentifier>,
	pub merkle_proof: Option<MerkleProofWrapper>,
}
``` 


2. Осуществляет проверку дупликатов транзакции с помощью [Keychain](https://github.com/beam-mw/grin/blob/master/keychain/src/keychain.rs):
	- генерирует случайный **kernel_offset**, используя **Keychain**;
	- cоздаёт [BlindSum](https://github.com/beam-mw/grin/blob/master/keychain/src/blind.rs);

	```rust
	/// Accumulator to compute the sum of blinding factors. Keeps track of each
	/// factor as well as the "sign" with which they should be combined.
	#[derive(Clone, Debug, PartialEq)]
	pub struct BlindSum {
		pub positive_key_ids: Vec<Identifier>,
		pub negative_key_ids: Vec<Identifier>,
		pub positive_blinding_factors: Vec<BlindingFactor>,
		pub negative_blinding_factors: Vec<BlindingFactor>,
	}
	```
	
	- **blind_offset = BlindSum + BlindingFactor - kernel_offset**;
	- используя **blind_offset** и **Keychain**, создаёт секретный ключ **skey**;
	- создаёт [Uuid](https://ru.wikipedia.org/wiki/UUID) для идентификации будущей транзакции;
	- используя  **skey** и **Uuid**, проверяет транзакцию на наличие дупликатов в контексте **Keychain**.


3. Создаёт контейнер **PartialTx**:


4. Отправляет квазитранзакцию **PartialTx** кошельку **B** по **http**-протоколу и ждёт ответа:
Кошелёк **A** выбрасывает ошибку: 
    - в случае неудачи при установлении соединения между кошельками 
    - в случае превышения награды **fee** количества предеавемых монет
Кошелёк **A** получает ответ и из него вычитвает
    - транзакцию 
    - количестов предаваемых монет
    - два публичных ключа **PublicKey**
    - **BlindingFactor**
    - 

