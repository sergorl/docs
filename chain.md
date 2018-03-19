Блок, который не имеет известного предка в самой длинной цепочке блоков:
```rust
#[derive(Debug, Clone)]
struct Orphan {
	block: Block,
	opts:  Options,
	added: Instant,
}
```
