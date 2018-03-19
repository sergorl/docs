# docs
simple docs

Just code snippet:

```rust
struct PMMRHandle<T>
where
	T: PMMRable,
{
	backend: PMMRBackend<T>,
	last_pos: u64,
}
```
