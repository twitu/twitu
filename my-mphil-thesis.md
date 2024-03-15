## My MPhil thesis

At [TACO lab](https://cse.hkust.edu.hk/~parreaux/), I implemented a type system called HMloc that creates better type error messages. My work got accepted at a major conference and eventually became my MPhil thesis. But it's unlikely that you're ever going to read 😛 it so I'll share the key ideas here.

Note that the type system was primarily designed for languages like OCaml, Haskell, Rust (i.e. Hindley-Milner type system) so familiarity with these languages will help ✌️.

## How do programs fail??

One common failure is operating on the wrong shape of data. In practice this can mean -

* Fitting a 🟪 into a 🟣 shaped hole
* Asking a 🐱 to woof instead of a 🐶
* Multiplying a 3x4 matrix with 6x4 matrix (big fail 👎)

Notice the common theme. You are operating on data that does not meet your expectation of "shape", "color", or "behaviour" etc. In languages, like Python and JavaScript this would result in a "run time error" or "exception". Can we avoid such errors entirely? A very emphatic YES!

In languages with type systems properties like "shape" or "behaviour" are encoded as "types". The type system can infer types for the data and check it against our expectations. For e.g.

* Square != Circle (different classes)
* Cat cannot woof (does not implement bark interface/trait/type class)
* 3x4 matrix and 6x4 matrix have incompatible shapes for multiplication (Vec3, Mat34 and similar granular types)

## Why do programs fail??

Typically, we want data to be of the "expected type". However, due to complexity of large code base, misunderstanding or just plain human error we can end up in a situation where the type of the data is not as expected. 


Type systems generally report an error when they see types that are not equal. However, this may not be the most helpful because the context is missing.

```
pet likes 😴 and 🥛 ▶️ 🐱
woof 🐱 ▶️ ❌
```

Error report: 🐱 cannot woof because it is not a 🐶

There could be many steps before the step that tripped up the type system. Without this context it is difficult to find the best way to fix the program.

## How to report better errors??

This is the KEY idea here. Fixing any one of the steps can fix the program. 

Asking the pet to meow instead
```
pet likes 😴 and 🥛 ▶️ 🐱
meow 🐱 ▶️ 🫶
```

Adopting a different pet
```
pet likes 🏃 and 🦴 ▶️ 🐶
woof 🐶 ▶️ 🫶
```

So can we report better errors? An emphatic YES!

A better type error report shows the context of the failing step. It reports the sequence of all the steps that lead to a mistake. This way the programmer can decide to which step to change. This is particularly useful if the program is large or the steps are in different files of a large code base.

```
pet likes 😴 and 🥛 ▶️ 🐱
...
...
...
woof 🐱 ▶️ ❌
```

HMloc cleverly encodes the sequence of steps related to a type within the type itself. That way when an error occurs the error report can be much more contextual and detailed.

## References

These were the key ideas behind the HMloc type system removed from any of the programming specifics. You're welcome to dive into the gory details using the links below.

* [Conference presentation video recording](https://www.youtube.com/watch?v=sDI9Ln4Ckao?start=25340)
* [Published report](https://dl.acm.org/doi/10.1145/3622812) and [Technical report](https://arxiv.org/abs/2402.12637) which has some extra details
* [GitHub repo](https://github.com/hkust-taco/type-error-messages-paper)
