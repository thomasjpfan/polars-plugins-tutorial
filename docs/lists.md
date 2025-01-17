# 9. Lists at last

According to [one YouTube talk](https://youtu.be/u5mIDz5ldmI?si=4AtnyyAwdVk33bYu),
the `list` namespace is one of Polars' main selling points.
If you're also a fan of it, this section will teach you how to extend it even further.

## Motivation

Say you have
```python
In [10]: df = pl.DataFrame({
    ...:     'values': [[1, 3, 2], [5, 7]],
    ...:     'weights': [[.5, .3, .2], [.1, .9]]
    ...: })

In [11]: df
Out[11]:
shape: (2, 2)
┌───────────┬─────────────────┐
│ values    ┆ weights         │
│ ---       ┆ ---             │
│ list[i64] ┆ list[f64]       │
╞═══════════╪═════════════════╡
│ [1, 3, 2] ┆ [0.5, 0.3, 0.2] │
│ [5, 7]    ┆ [0.1, 0.9]      │
└───────────┴─────────────────┘
```

Can you calculate the mean of the values in `'values'`, weighted by the values in `'weights'`?

So:

- `.5*1 + .3*3 + .2*2 = 1.8`
- `5*.1 + 7*.9 = 6.8`

I don't know of an easy way to do this with Polars expressions. There probably is a way - but
as you'll see here, it's not that hard to write a plugin, and it's probably faster too.

## Weighted mean

On the Python side, this'll be similar to `sum_i64`:

```python
def weighted_mean(expr: IntoExpr, weights: IntoExpr) -> pl.Expr:
    expr = parse_into_expr(expr)
    return expr.register_plugin(
        lib=lib,
        symbol="weighted_mean",
        is_elementwise=True,
        args=[weights]
    )
```

On the Rust side, we're going to start by writing the following
to `src/utils.rs`:

```rust
use polars::prelude::*;

pub(crate) fn binary_amortized_elementwise<'a, T, K, F>(
    ca: &'a ListChunked,
    weights: &'a ListChunked,
    mut f: F,
) -> ChunkedArray<T>
where
    T: PolarsDataType,
    T::Array: ArrayFromIter<Option<K>>,
    F: FnMut(&Series, &Series) -> Option<K> + Copy,
{
    unsafe {
        ca.amortized_iter()
            .zip(weights.amortized_iter())
            .map(|(lhs, rhs)| match (lhs, rhs) {
                (Some(lhs), Some(rhs)) => f(lhs.as_ref(), rhs.as_ref()),
                _ => None,
            })
            .collect_ca(ca.name())
    }
}
```
and then adding
```rust
use crate::utils::binary_amortized_elementwise;
```
to the top of `src/expressions.rs`, after the previous imports.

Don't worry about understanding it.
Some of its details (such as `.as_ref()` to get a `Series` out of an `UnstableSeries`) are arguably
implementation details. Hopefully a more generic version of this utility like this can be added to
Polars itself, so that plugin authors won't need to worry about it.

Let's concern ourselves with just using it to accomplish our task!
We just need to write a function which accepts two `Series`, computes their dot product, and then
divides by the sum of the weights:

```rust
#[polars_expr(output_type=Float64)]
fn weighted_mean(inputs: &[Series]) -> PolarsResult<Series> {
    let values = inputs[0].list()?;
    let weights = &inputs[1].list()?;

    let out: Float64Chunked = binary_amortized_elementwise(
        values,
        weights,
        |values_inner: &Series, weights_inner: &Series| -> Option<f64> {
            let values_inner = values_inner.i64().unwrap();
            let weights_inner = weights_inner.f64().unwrap();
            let out_inner: Float64Chunked = binary_elementwise(
                values_inner,
                weights_inner,
                |opt_value: Option<i64>, opt_weight: Option<f64>| match (opt_value, opt_weight) {
                    (Some(value), Some(weight)) => Some(value as f64 * weight),
                    _ => None,
                },
            );
            match (out_inner.sum(), weights_inner.sum()) {
                (Some(weighted_sum), Some(weights_sum)) => Some(weighted_sum / weights_sum),
                _ => None,
            }
        },
    );
    Ok(out.into_series())
}
```
That's it! This version only accepts `Int64` values - see section 2 for how you could make it more
generic.

To try it out, we compile with `maturin develop` (or `maturin develop --release` if you're 
benchmarking), and then we should be able to run `run.py`:

```python
import polars as pl
import minimal_plugin as mp

df = pl.DataFrame({
    'values': [[1, 3, 2], [5, 7]],
    'weights': [[.5, .3, .2], [.1, .9]]
})
print(df.with_columns(weighted_mean = mp.weighted_mean('values', 'weights')))
```
to see
```
shape: (2, 3)
┌───────────┬─────────────────┬───────────────┐
│ values    ┆ weights         ┆ weighted_mean │
│ ---       ┆ ---             ┆ ---           │
│ list[i64] ┆ list[f64]       ┆ f64           │
╞═══════════╪═════════════════╪═══════════════╡
│ [1, 3, 2] ┆ [0.5, 0.3, 0.2] ┆ 1.8           │
│ [5, 7]    ┆ [0.1, 0.9]      ┆ 6.8           │
└───────────┴─────────────────┴───────────────┘
```

## Gimme challenge

Could you implement a weighted standard deviation calculator?
