## Сценарий №2. Кошелёк ***B*** принимает монеты от кошелька ***A***

1. Кошелёк ***B*** получает запрос от кошелька ***B*** и вычитывает (десериализует) из него:
    - транзакцию **tx**;
    - количество пересылаемых монет **amount**;
    - два публичных ключа  **sender_pub_blinding** и **sender_pub_nonce** типа **PublicKey**, полученных от кошелька ***A***;
    - **BlindingFactor kernel_offset**;
    - сигнатуру **Signature**.
    
2. Оценивает вознаграждение **fee** за транзакцию по формуле:   

```rust
let mut tx_weight = -1 * (input_len as i32) + 4 * (output_len as i32) + 1;

if tx_weight < 1 {
  tx_weight = 1;
}

fee = (tx_weight as u64) * use_base_fee
```

Если оценка **fee** на равна вознаграждению внутри самой транзакции **tx.fee()**, то кошелёк ***B*** прекращает взаимодействие с кошельком ***B***, выбрасывая ошибку.

Если оценка вознаграждения **fee** превышает количество пересылаемых монет **amount**, то кошелёк ***B*** также прекращает взаимодействие с кошельком ***B***, выбрасывая ошибку.

3. Составляет выход будущей транзакции **OutputData** с идентификатором **Identifier key_id**:

```rust
OutputData {
    root_key_id: root_key_id.clone(),
    key_id: key_id.clone(),
    n_child: derivation,
    value: out_amount,
    status: OutputStatus::Unconfirmed,
    height: 0,
    lock_height: 0,
    is_coinbase: false,
    block: None,
    merkle_proof: None,
}
```

Идентификатор **Identifier key_id** формируется с помощью сущности **Keychain** и происходит из некоего корня **root_key_id**.

Такой выход пока что имеет статус **неподтверждённого OutputStatus::Unconfirmed**, а также: 
    - не ссылается ни на какой блок внутри блокчейна (соотвественно **height** и **lock_height** = 0)
    - и не содержит **merkle_proof**.
   
4. Формирует сигнатуру **Signature**.

5. Создаёт контейнер **PartialTx**, внутри которого выставляется флаг **PartialTxPhase::ReceiverInitiation**. В процессе создания контейнера, внутри него, формируются два публичных ключа:

   - **pub_excess** и
   - **pub_nonce**.

Внутрь контейнера помещаются:
   - непосредственно сама транзакция;
   - количестов пересылаемых монет **amount**;
   - **BlindingFactor kernel_offset**;
   - cигнатура **Signature**, полученная на предыдущем этапе.
   
