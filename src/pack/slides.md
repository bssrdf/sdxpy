---
template: slides
title: "A Package Manager"
---

## The Problem

-   Packages and their dependencies change over time

-   If A and B require different versions of C
    it might not be possible to use A and B together

-   Need a systematic way to find globally-consistent sets of packages

---

## Identifying Versions

-   [%g semantic_versioning "Semantic versioning" %] uses three integers `X.Y.Z`

    -   `X` is the major version (breaking changes)

    -   `Y` is the minor version (new features)

    -   `Z` is the [%g patch "patch" %] (bug fixes)

-   Notation

    -   `>=1.2.3` means "any version from 1.2.3 onward"

    -   `<4` means "any version before 4.anything"

    -   `1.0-3.1` means "any version in the specified range"

---

## A Simplified Version

[%inc triple.json %]

---

## Multiple Dimensions

-   Imagine a multi-dimensional grid with one axis per package

-   Possible combinations are points in this grid

[% figure
   slug="pack-allowable"
   img="allowable.svg"
   alt="Allowable versions"
   caption="Finding allowable combinations of package versions."
%]

-   But how do we find the legal ones?

---

## Estimate Work

-   Our example has 3×3×2=18 combinations

-   Adding one more package with two versions doubles
    the [%g search_space "search space" %]

-   A [%g combinatorial_explosion "combinatorial explosion" %]

-   Brute force solutions are impractical even for small problems

-   But worth implementing as a starting point and for testing

---

## Brute Force

-   Generate all possible combinations of package versions

-   Then eliminate ones that aren't compatible with the manifest

[%inc exhaustive.py mark=main %]

---

## Generating Possibilities

-   Create a list of the available versions of each package

-   Generate their [%g cross_product "cross product" %]

[%inc exhaustive.py mark=possible %]

[% figure
   slug="pack-product"
   img="product.svg"
   alt="Generating a cross-product"
   caption="Generating all possible combinations of items."
%]

---

## Checking Validity

-   Compare every entry X against every other entry Y

-   If they are the same package, keep looking

-   If package X's requirements say nothing about package Y,
    keep searching

-   If X *does* depend on Y
    but this particular version of X doesn't list this particular version of Y
    as a dependency,
    rule out this combination

-   If combination is still a candidate, add to list

---

## Checking Validity

[%inc exhaustive.py mark=compatible %]

-   Finds 3 valid combinations among our 18 possibilities

[%inc exhaustive.out %]

---

## Manual Generation

-   Create our own cross-product function
    in preparation for doing something more efficient

[%inc manual.py mark=start %]

-   Create a list of lists of available versions

-   Create an empty accumulator

-   Do some magic

---

## Manual Generation

-   Each call to `_make_possible` handles one package's worth of work

-   Loop over available versions of current package

-   Add that package to the combination in progress

-   If more packages, recurse

-   Otherwise, append to accumulator

[%inc manual.py mark=make %]

---

## Incremental Search

-   Generate-and-discard is inefficient

-   Stop immediately if a partial combination of packages is illegal

[%inc incremental.py mark=main %]

-   `reverse` to allow experimentation

---

## Finding Possibilities

[%inc incremental.py mark=find %]

---

## Finding Possibilities

1.  The manifest that tells us what's compatible with what.

2.  The names of the packages we've haven't considered yet.

3.  An accumulator to hold all the valid combinations we've found so far.

4.  The partially-completed combination we're going to extend next.

5.  A count of the number of combinations we've considered so far,
    which we will use as a measure of efficiency.

---

## Savings

-   We only create 11 candidates instead of 18

[%inc incremental.sh %]
[%inc incremental.out %]

-   Reversing the order cuts it to 9

[%inc incremental_reverse.sh %]
[%inc incremental_reverse.out %]

---

## Using a Theorem Prover

-   An automated theorem prover can do much better 

    -   But the algorithms quickly become very complex

-   Prove that a set of logical propositions (e.g., dependencies) are satisfiable

-   We will use the [Z3 theorem prover][z3]

    -   Whose documentation is unfortunately a barrier to entry

---

## Using Z3

[%inc z3_equal.py mark=setup %]

-   `A`, `B`, and `C` don't have values

-   Instead, each represents the set of possible Boolean values

---

## Using Z3

-   Specify constraints such as `A == B`

[%inc z3_equal.py mark=solve %]

-   Then ask Z3 to find a [%g model "model" %] that satisfies those constraints

[%inc z3_equal.out %]

---

## Unsatisfiable

-   Require `A` to equal `B` and `B` to equal `C`
    but `A` and `C` to be unequal

[%inc z3_unequal.py mark=solve %]
[%inc z3_unequal.out %]

---

## Packaging

-   Represent our package versions

[%inc z3_triple.py mark=setup %]

---

## Packaging

-   We want one version of `A`

[%inc z3_triple.py mark=top %]

-   But the versions of `A` are mutually exclusive

[%inc z3_triple.py mark=exclusive %]

-   Do the same for `B` and `C`

---

## Dependencies

-   Add inter-package dependencies and solve

[%inc z3_triple.py mark=depends %]
[%inc z3_triple.out %]

---

## Summary

[% figure
   slug="pack-concept-map"
   img="concept_map.svg"
   alt="Concept map for package manager."
   caption="Concept map."
%]
