# docs
simple docs

Just code snippet:

```
struct PMMRHandle<T>
where
	T: PMMRable,
{
	backend: PMMRBackend<T>,
	last_pos: u64,
}
```
