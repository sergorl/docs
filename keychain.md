## Криптографические примитивы grin'a

**Identifier** - идентификатор ключей **ExtendedKey** и **ChildKey**. **Identifier** может быть создан:
    1. **blake2b**-хэшированием пуличного ключа **PublicKey** кривой **Secp256k1**;
    2. из секретного ключа **SecretKey** - на основе формирования публичного ключа из данного секретного ключа и применения метода №1 выше.

```rust
pub const IDENTIFIER_SIZE: usize = 10;
pub struct Identifier([u8; IDENTIFIER_SIZE]);
```
