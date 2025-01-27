# Random distributions

For maximum flexibility when producing random values, we define the
[`Distribution`] trait:

```rust,noplayground
# use rand::Rng;
// a producer of data of type T:
pub trait Distribution<T> {
    // the key function:
    fn sample<R: Rng + ?Sized>(&self, rng: &mut R) -> T;

    // a convenience function defined using sample:
    fn sample_iter<R>(self, rng: R) -> rand::distr::Iter<Self, R, T>
    where
        Self: Sized,
        R: Rng,
    {
        // [has a default implementation]
        # todo!()
    }
}
```

Implementations of [`Distribution`] are *probability distribution*: mappings
from events to probabilities (e.g. for a die roll `P(x = i) = ⅙` or for a Normal
distribution with mean `μ=0`, `P(x > 0) = ½`).

Note that although probability distributions all have properties such as a mean,
a Probability Density Function, and can be sampled by inverting the Cumulative
Density Function, here we only concern ourselves with *sampling random values*.
If you require use of such properties you may prefer to use the [`statrs`] crate.

Rand provides implementations of many different distributions; we cover the most
common of these here, but for full details refer to the [`distr`] module
and the [`rand_distr`] crate.

# Uniform distributions

The most obvious type of distribution is the one we already discussed: one
where each equally-sized sub-range has equal chance of containing the next
sample. This is known as *uniform*.

Rand actually has several variants of this, representing different ranges:

-   [`StandardUniform`] requires no parameters and samples values uniformly according
    to the type. [`Rng::random`] provides a short-cut to this distribution.
-   [`Uniform`] is parametrised by `Uniform::new(low, high)` (including `low`,
    excluding `high`) or `Uniform::new_inclusive(low, high)` (including both),
    and samples values uniformly within this range.
    [`Rng::random_range`] is a convenience method defined over
    [`Uniform::sample_single`], optimised for single-sample usage.
-   [`Alphanumeric`] is uniform over the `char` values `0-9A-Za-z`.
-   [`Open01`] and [`OpenClosed01`] are provide alternate sampling ranges for
    floating-point types (see below).

## Uniform sampling by type

Lets go over the distributions by type:

-   For `bool`, [`StandardUniform`] samples each value with probability 50%.
-   For `Option<T>`, the [`StandardUniform`] distribution samples `None` with
    probability 50%, otherwise `Some(value)` is sampled, according to its type.
-   For integers (`u8` through to `u128`, `usize`, and `i*` variants),
    [`StandardUniform`] samples from all possible values while
    [`Uniform`] samples from the parameterised range.
-   For `NonZeroU8` and other "non-zero" types, [`StandardUniform`] samples uniformly
    from all non-zero values (rejection method).
-   `Wrapping<T>` integer types are sampled as for the corresponding integer
    type by the [`StandardUniform`] distribution.
-   For floats (`f32`, `f64`),

    -   [`StandardUniform`] samples from the half-open range `[0, 1)` with 24 or 53
        bits of precision (for `f32` and `f64` respectively)
    -   [`OpenClosed01`] samples from the half-open range `(0, 1]` with 24 or
        53 bits of precision
    -   [`Open01`] samples from the open range `(0, 1)` with 23 or 52 bits of
        precision
    -   [`Uniform`] samples from a given range with 23 or 52 bits of precision
-   For the `char` type, the [`StandardUniform`] distribution samples from all
    available Unicode code points, uniformly; many of these values may not be
    printable (depending on font support). The [`Alphanumeric`] samples from
    only a-z, A-Z and 0-9 uniformly.
-   For tuples and arrays, each element is sampled as above, where supported.
    The [`StandardUniform`] and [`Uniform`] distributions each support a selection of
    these types (up to 12-tuples and 32-element arrays).
    This includes the empty tuple `()` and array.
    When using `rustc` ≥ 1.51, enable the `min_const_gen` feature to support
    arrays larger than 32 elements.
-   For SIMD types, each element is sampled as above, for [`StandardUniform`] and
    [`Uniform`] (for the latter, `low` and `high` parameters are *also* SIMD
    types, effectively sampling from multiple ranges simultaneously). SIMD
    support requires using the `simd_support` feature flag and nightly `rustc`.
-   For enums, you have to implement uniform sampling yourself. For example, you
    could use the following approach:
    ```rust
    # use rand::{Rng, distr::{Distribution, StandardUniform}};
    pub enum Food {
        Burger,
        Pizza,
        Kebab,
    }

    impl Distribution<Food> for StandardUniform {
        fn sample<R: Rng + ?Sized>(&self, rng: &mut R) -> Food {
            let index: u8 = rng.random_range(0..3);
            match index {
                0 => Food::Burger,
                1 => Food::Pizza,
                2 => Food::Kebab,
                _ => unreachable!(),
            }
        }
    }
    ```

# Non-uniform distributions

The [`rand`] crate provides only two non-uniform distributions:

-   The [`Bernoulli`] distribution simply generates a boolean where the
    probability of sampling `true` is some constant (`Bernoulli::new(0.5)`) or
    ratio (`Bernoulli::from_ratio(1, 6)`).
-   The [`WeightedIndex`] distribution may be used to sample from a sequence of
    weighted values. See the [Sequences] section.

Many more non-uniform distributions are provided by the [`rand_distr`] crate.

### Integers

The [`Binomial`] distribution is related to the [`Bernoulli`] in that it
models running `n` independent trials each with probability `p` of success,
then counts the number of successes.

Note that for large `n` the [`Binomial`] distribution's implementation is
much faster than sampling `n` trials individually.

The [`Poisson`] distribution expresses the expected number of events
occurring within a fixed interval, given that events occur with fixed rate λ.
[`Poisson`] distribution sampling generates `Float` values because `Float`s
are used in the sampling calculations, and we prefer to defer to the user on
integer types and the potentially lossy and panicking associated conversions.
For example, `u64` values can be attained with `rng.sample(Poisson) as u64`.

Note that out of range float to int conversions with `as` result in undefined
behavior for Rust <1.45 and a saturating conversion for Rust >=1.45.

## Continuous non-uniform distributions

Continuous distributions model samples drawn from the real number line ℝ, or in
some cases a point from a higher dimension (ℝ², ℝ³, etc.). We provide
implementations for `f64` and for `f32` output in most cases, although currently
the `f32` implementations simply reduce the precision of an `f64` sample.

The exponential distribution, [`Exp`], simulates time until decay, assuming a
fixed rate of decay (i.e. exponential decay).

The [`Normal`] distribution (also known as Gaussian) simulates sampling from
the Normal distribution ("Bell curve") with the given mean and standard
deviation. The [`LogNormal`] is related: for sample `X` from the log-normal
distribution, `log(X)` is normally distributed; this "skews" the normal
distribution to avoid negative values and to have a long positive tail.

The [`UnitCircle`] and [`UnitSphere`] distributions simulate uniform
sampling from the edge of a circle or surface of a sphere.

The [`Cauchy`] distribution (also known as the Lorentz distribution) is the
distribution of the x-intercept of a ray from point `(x0, γ)` with uniformly
distributed angle.

The [`Beta`] distribution is a two-parameter probability distribution, whose
output values lie between 0 and 1. The [`Dirichlet`] distribution is a
generalisation to any positive number of parameters.

[Sequences]: guide-seq.html
[`Distribution`]: https://docs.rs/rand/latest/rand/distr/trait.Distribution.html
[`distr`]: https://docs.rs/rand/latest/rand/distr/
[`rand`]: https://docs.rs/rand/
[`rand_distr`]: https://docs.rs/rand_distr/
[`Rng::random_range`]: https://docs.rs/rand/latest/rand/trait.Rng.html#method.random_range
[`random`]: https://docs.rs/rand/latest/rand/fn.random.html
[`Rng::random_bool`]: https://docs.rs/rand/latest/rand/trait.Rng.html#method.random_bool
[`Rng::random_ratio`]: https://docs.rs/rand/latest/rand/trait.Rng.html#method.random_ratio
[`Rng::random`]: https://docs.rs/rand/latest/rand/trait.Rng.html#method.random
[`Rng`]: https://docs.rs/rand/latest/rand/trait.Rng.html
[`StandardUniform`]: https://docs.rs/rand/latest/rand/distr/struct.StandardUniform.html
[`Uniform`]: https://docs.rs/rand/latest/rand/distr/struct.Uniform.html
[`Uniform::sample_single`]: https://docs.rs/rand/latest/rand/distr/struct.Uniform.html#method.sample_single
[`Alphanumeric`]: https://docs.rs/rand/latest/rand/distr/struct.Alphanumeric.html
[`Open01`]: https://docs.rs/rand/latest/rand/distr/struct.Open01.html
[`OpenClosed01`]: https://docs.rs/rand/latest/rand/distr/struct.OpenClosed01.html
[`Bernoulli`]: https://docs.rs/rand/latest/rand/distr/struct.Bernoulli.html
[`Binomial`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Binomial.html
[`Exp`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Exp.html
[`Normal`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Normal.html
[`LogNormal`]: https://docs.rs/rand_distr/latest/rand_distr/struct.LogNormal.html
[`UnitCircle`]: https://docs.rs/rand_distr/latest/rand_distr/struct.UnitCircle.html
[`UnitSphere`]: https://docs.rs/rand_distr/latest/rand_distr/struct.UnitSphere.html
[`Cauchy`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Cauchy.html
[`Poisson`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Poisson.html
[`Beta`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Beta.html
[`Dirichlet`]: https://docs.rs/rand_distr/latest/rand_distr/struct.Dirichlet.html
[`statrs`]: https://github.com/statrs-dev/statrs/
[`WeightedIndex`]: https://docs.rs/rand/latest/rand/distr/struct.WeightedIndex.html
