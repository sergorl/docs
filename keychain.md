## Криптографические примитивы grin'a

### Identifier
**Identifier** - идентификатор секретных ключей **ExtendedKey** и **ChildKey**. **Identifier** может быть создан двумя методами:
   1. **blake2b**-хэшированием пуличного ключа **PublicKey** кривой **Secp256k1**;
   2. из секретного ключа **SecretKey** - на основе формирования публичного ключа из данного секретного ключа и применения метода №1 выше.

```rust
pub const IDENTIFIER_SIZE: usize = 10;
pub struct Identifier([u8; IDENTIFIER_SIZE]);
```

### ExtendedKey
**ExtendedKey** - секретный ключ-исчтоник, но основе которого можно создавать множество других секретных ключей, используемых для сокрытия количества пересылаемых монет.

```rust
pub struct ExtendedKey {
	/// Child number of the extended key
	pub n_child: u32,
	/// Root key id
	pub root_key_id: Identifier,
	/// Key id
	pub key_id: Identifier,
	/// The secret key
	pub key: SecretKey,
	/// The chain code for the key derivation chain
	pub chain_code: [u8; 32],
}
```

**ExtendedKey** генерируется из некоторого **seed**'a - для этого:
   - используется **blake2b**-хеширования данного **seed**'a,
   - а с помощью кривой **Secp256k1** из результата хэширования предыдущего шага формируется сам секретный ключ **ExtendedKey**.

### ChildKey
**ChildKey** - секретный ключ, формируемый из секретного ключа-источника **ExtendedKey**. Формируется аналогично **ExtendedKey**, но с использованием 32-битного **chain_code** в качестве хэшируемого сообщения в алгоритме **blake2b**.
```rust
pub struct ChildKey {
	/// Child number of the key (n derivations)
	pub n_child: u32,
	/// Root key id
	pub root_key_id: Identifier,
	/// Key id
	pub key_id: Identifier,
	/// The private key
	pub key: SecretKey,
}
```
