# Inclusive In-Contracts

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Mathis Beer mathis.beer@funkwerk-itk.com                        |
| Implementation: | TODO                                                            |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

When a parent class method's in-contract passes, the in-contracts placed on overriding methods in
child classes should also be required to pass.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

In-contracts on class methods are currently evaluated by disjunctively evaluating the contracts of each
overridden method.
(See [dlang.org](https://dlang.org/spec/contracts.html#in_out_inheritance) §23.3.)
This can cause unintuitive-seeming method behavior.

For example:

```
class Class : Super
{
    override void method(Object object)
    in (object !is null)
    { }
}
```

This does not at all guarantee that `object` is not null. If the superclass allows a null object, `method` will be
silently entered with a null object.

Instead, the in-contract should have to reiterate the super class's in-contract in (optionally) looser form.

This sounds like it would be extra effort. However, I believe the effort will be small. In our codebase,
the vast majority of invariants are some variant of "is not null" tests. We need to reiterate those in-contracts
in the subclass anyways, in order for them to have any effect under inheritance. The one case where additional
work is required is if a subclass intends to take some separate set of possible input values compared
to the parent class. In other words, in this situation:

```
class Super
{
    void method(int i) in (i == 3) { }
}

class Class : Super
{
    override void method(int i) in (i == 4) { }
}
```
The subclass in-contract will need to be changed to have the form `in (i == 3 || i == 4)`.

As an experiment, I created a DMD PR https://github.com/dlang/dmd/pull/11440 that applied this semantic change.
The buildkite testsuite passed, indicating that no such contract usage exists in the opensource ecosystem
tested by buildkite.

Furthermore, the existing behavior is visually misleading. One expects the in-contracts to enumerate the conditions
that must be true for the provided parameters. If there is an additional set of possible conditions that may
alternatively be true, but that do not appear in the Ddoc documentation or in the place of implementation,
the ensuing behavior may be highly unexpected - a function may be entered with parameters that were explicitly
excluded, with no warning.

As a related piece of functionality in D, an overridden method may provide more generic parameter types, and more
specific return types. (Contravariance and covariance.) In that case, more generic types are specified by fully
writing the generic type, not by listing another type and expecting the method to be entered with "either that
type or the type defined in the parent." So the behavior of in-contracts is not consistent with the behavior of the
typesystem under inheritance.

A change that causes additional runtime errors is always questionable. However, this change causes verification code
to loudly fail, where previously verification code (the subclass in-contracts) was silently ignored. I believe
this change will hence expose far more issues than it will cause.

## Prior Work

I raised this issue on the newsgroup: https://forum.dlang.org/post/pvsvhrgjfkakmpnjvqsd@forum.dlang.org

Timon Gehr noted in a reply that Eiffel, which invented Design by Contract, uses the syntax of `require else` to
disjunctively extend the superclass method's in-contract. In other words, subclasses will by default inherit the
in-contract of the parent. (Not so in D.) Furthermore, Eiffel allows adding a disjunctive clause to a specific
in-contract, meaning if an additional contract is added to the parent class, the child class gains it automatically,
even if it has decided to loosen another contract clause.
Because D does not name its in-contracts, this behavior cannot be emulated. In other words, whereas Eiffel operates
on a principle of reuse by inheritance combined with fine-grained extension, D operates on a principle of
extension by redefinition.

There is already some slight protection against this case. Previously, a superclass without a contract would mean
that any subclass contract was not even parsed, because it obviously could not loosen the parent contract any further.
Luckily, this case is now an error. However, this only uncovers the most trivial of bugs.

## Description

The behavior of in-contracts is changed. This concerns https://dlang.org/spec/contracts.html#in_out_inheritance .

The new description of section 23.3, paragraph 1 and 2 is:

1. If a function in a derived class overrides a function from its super class, then the function's in-contract
must be satisfied at least in all cases that also satisfy the parent contract. Overriding functions then becomes a
process of loosening the in contracts.

2. A function without an in contract means that any values of the function parameters are allowed. This implies that
if any function in an inheritance hierarchy has no in contract, then any in contracts on functions overriding it
will error at runtime, because they will not loosen the parent's in contract.

The new behavior is described by the pseudocode:

```
    try
    {
        child in-contract;
    }
    catch (Throwable)
    {
        parent in-contract;
        // if the parent in-contract has not thrown, then the child in-contract has rejected a case that the parent
        // in-contract accepted, violating 23.3.1.
        // The message may also contain the Throwable's msg, indicating which in-contract was violated.
        assert(false, "In-contract was tighter than parent in-contract.");
    }
```

Example:

```
class SuperClass
{
    void compare(Object left, Object right)
    in (true)
    {
        return left is right;
    }
}

class SubClass
{
    override void compare(Object left, Object)
    in (left !is null)
    in (right !is null)
    {
        return left is right;
    }
}

unittest
{
    auto object = new SubClass;

    // because SuperClass assured us that nulls were acceptable, SubClass cannot go back on its word.
    object.compare(null, null).assertThrown("In-contract was tighter than parent in-contract.");
}
```

## Breaking Changes and Deprecations

Methods with in-contracts that were ignored because they were not a superset of the parent's
in-contracts will now fail, whereas they previously passed.

Since those methods by definition contained ignored in-conditions, this is easy to fix by just removing the failing
in-conditions.

However, since there is no way to mark these cases with a deprecation warning, being only detectable at runtime,
I propose a preview flag and silent deprecation period, followed by a period where a revert flag is available.

## Reference

- https://dlang.org/spec/contracts.html#in_out_inheritance
- https://forum.dlang.org/post/ren402$2mvi$1@digitalmars.com
- https://web.archive.org/web/20190802081721/https://www.eiffel.org/doc/eiffel/ET-_Inheritance

## Copyright & License
Copyright (c) 2020 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
