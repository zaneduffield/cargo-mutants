# enucleate

https://github.com/sourcefrog/enucleate

[![Tests](https://github.com/sourcefrog/enucleate/actions/workflows/tests.yml/badge.svg)](https://github.com/sourcefrog/enucleate/actions/workflows/tests.yml)

Enucleate is a mutation testing tool for Rust. It guides you to missing test coverage by finding functions whose
implementation could be replaced by something trivial and the tests would all still pass. 

CAUTION: Enucleate builds and runs code with arbitrary changes. If the test suite has side effects such as
writing or deleting files, running it with mutations may be dangerous: for example it might write to or 
delete files outside of the source tree. Eventually, Enucleate might support running tests inside a jail
or sandbox; for now think first about what side effects the test suite could possibly have and/or run
it in a restricted or disposable environment.

This is inspired by the [Decartes mutation-testing tool for Java](https://github.com/STAMP-project/pitest-descartes/) 
described in https://increment.com/reliability/testing-beyond-coverage/.
I think it's an interesting insight that mutation at the level of a whole function is a practical sweet-spot to discover 
missing tests, while (at least at moderate-size trees) still making it feasible to exhaustively generate every
mutant.

## Manifesto

* Draw attention to code that is not tested or only "pseudo-tested": reached by tests but the tests
  don't actually depend on the behavior of the function.
* Be fast enough to plausibly run on thousand-file trees, although not necessarily instant. It's OK if
  it takes some minutes to run.
* Easy start: no annotations need to be added to the source tree to use Enucleate, 
  you just run it and it should say something useful about any Rust tree. 
  I'd like it to be useful on a tree you haven't seen before, where you're not
  sure of the quality of the tests.
* Understandable output including easily reproducible test failures and a copy of the output.
* Signal to noise: most warnings should indicate something that really should be tested more; 
  false positives should be easy to durably dismiss.
* Opt-in annotations to disable mutation of some methods or to explain how to mutate them usefully.
* Rust only, Cargo only. No need to mutate C code; there are other tests for that.

## Approach

The basic approach is:

* Make a copy of the whole tree into a scratch directory. The same directory is reused 
  across all the mutations to benefit from incremental builds.
* Build a list of possible mutations and for each one:
  * Apply the mutation to the scratch tree.
  * Save a diff of the applied mutation for later reference.
  * Run `cargo test` in the tree, saving output to a log file.
  * If the build fails or the tests fail, that's good: the mutation was somehow caught.
  * If the build and tests succeed, that might mean test coverage was inadequate, or it might mean
    we accidentally generated a no-op mutation. (Doing so is a shortcoming in Enucleate.)

The list of possible mutations is generated by:

* Walk all source files and enumerate all item (top-level) and method functions. For each one:
   * Filter functions that should not be mutated:
      * Already-trivial functions whose result would not be changed by mutation. 
      * Functions marked with an `#[enucleate::skip]` attribute.
      * Functions marked `#[test]`.
      * Functions marked `#[cfg(test)]` or inside a `mod` so marked.
   * Apply every supported mutation:
      * Replace the body with `Default::default()`.
        * If the function returns a `Result`, instead replace the body with `Ok(Default::default())`.
      * (maybe more in future)

The nice thing about `Default` is that it's defined on many commonly-used types including `()`,
so Enucleate does not need to really understand the function return type at this early stage.
Some functions will fail to build because they return a type that does not implement `Default`,
and that's OK.

The file is parsed using the [`syn`](https://docs.rs/syn) crate, but mutations
are applied textually, rather than to the token stream, so that unmutated code
retains its prior formatting, comments, line numbers, etc. This makes it
possible to show a text diff of the mutation and should make it easier to
understand any error messages from the build of the mutated code.

## Further thoughts

To make this faster on large trees, we could keep several scratch trees and test them in parallel, 
which is likely to exploit CPU resources more thoroughly than Cargo's own parallelism: in particular
Cargo tends to fall down to a single task during linking.

One class of functions that have a reason to exist but may not cause a test suite failure if emptied 
out are those that exist for performance reasons, or more generally that have effects other than on the directly
observable side effects of calling the function.  For example, functions to do with managing caches
or memory.

Ideally, these should be tested, but doing so in a way that's not flaky can be difficult. Enucleate
can help in a few ways:

* It helps to at least highlight to the developer that the function is not covered by tests, and
  so should perhaps be treated with extra care, or tested manually.
* A `#[enucleate::skip]` annotation can be added to suppress warnings and explain the decision.
* Sometimes these effects can be tested by making the side-effect observable with, for example,
  a counter of the number of memory allocations or cache misses/hits.
