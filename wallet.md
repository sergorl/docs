
## Сценарий №1. Кошелёк ***A*** пересылает монеты кошельку ***B***

Взаимодействие кошельков строится на основе передачи/приёма контейнера **PartialTx** по ***http***-протоколу. Основная часть контейнера содержит непосредственно саму транзакцию и криптографические параметры (сигнатуру, blind_excess, nonce, kernel_offset):

```rust
/// Helper in serializing the information required during an interactive aggsig
/// transaction
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
 	- выходы транзакции **OutputData**, содержащие необходимое количество монет;
	- количество пересылаемых монет **+** вознаграждение **fee**;
	- [Identifier](https://github.com/beam-mw/grin/blob/master/keychain/src/extkey.rs) **change_key**.

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
Выход транзакции **OutputData** содержит поле-флаг **OutputStatus**:

```rust
/// Status of an output that's being tracked by the wallet. Can either be
/// unconfirmed, spent, unspent, or locked (when it's been used to generate
/// a transaction but we don't have confirmation that the transaction was
/// broadcasted or mined).
pub enum OutputStatus {
	Unconfirmed,
	Unspent,
	Locked,
	Spent,
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


3. Создаёт контейнер **PartialTx**, внутри которого выставляется флаг **PartialTxPhase::SenderInitiation**. В процессе создания контейнера, внутри него, формируются два публичных ключа:
    - **pub_excess** и 
    - **pub_nonce**.

На стадии **PartialTxPhase::SenderInitiation** такой контейнер не содержит сигнатуру **secp::Signature**.   

4. Отправляет контейнер **PartialTx** кошельку ***B*** по ****http****-протоколу и ждёт ответа:

- кошелёк **A** выбрасывает ошибку: 
    - в случае неудачи при установлении соединения между кошельками, 
    - в случае превышения награды **fee** количества предеавемых монет
    
- кошелёк **A** получает ответ и вычитывает (десериализует) из него:
    - транзакцию;
    - количество пересылаемых монет;
    - два публичных ключа  **recp_pub_blinding** и **recp_pub_nonce** типа **PublicKey**, полученных от кошелька ***B***;
    - **BlindingFactor kernel_offset**;
    - сигнатуру **Signature**.
    
5. Верифицирует сигнатуру, полученную на предыщуем шаге от кошелька ***B***. В случае неуспеха кошелёк ***A*** завершает взаимодействие с кошельком ***B***. 

6. Кошелёк ***A*** создаёт свою сигнатуру, используя:
    - **UUID** транзакции;
    - **recp_pub_nonce**, полученный от кошелька ***B*** на предыдущем шаге;
    - вознаграждение **fee** (используется как сообщение, которое подписывают);
    - **lock_height**.

7. Кошелёк ***A*** создаёт новый контейнер **PartialTx** уже с флагом **PartialTxPhase::SenderConfirmation** и кладёт внутрь него свою сигнатуру, полученную на предыдущем шаге. 

8. Далее кошелёк ***A*** отправляет новый контейнер **PartialTx** кошельку ***B*** и ждёт ответа:

    - в случае успеха кошелёк ***А*** блокирует выходы **OutputData**;
    - ??? в случае неудачи кошелёк ***А*** удаляет выход **OutputData**, соответствующий **change_key**, из списка выходов.

    

