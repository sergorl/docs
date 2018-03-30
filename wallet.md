
## Взаимодействие кошельков

### Сценарий №1. Кошелёк ***A*** посылает транзакцию кошельку ***B***
1. Кошелёк ***A*** на своей стороне формирует:
  	- транзакцию
 	- **BlindingFactor**
 	- выходы транзакций **OutputData**
  
```rust
/// Information about an output that's being tracked by the wallet. Must be
/// enough to reconstruct the commitment associated with the ouput when the
/// root private key is known.
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq, PartialOrd, Eq, Ord)]
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

   - change_key
   - число пересылаемых монет + вознаграждение fee

2. 
