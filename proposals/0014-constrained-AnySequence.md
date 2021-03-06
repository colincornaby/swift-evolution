# Constraining `AnySequence.init`

* Proposal: [SE-0014](https://github.com/apple/swift-evolution/blob/master/proposals/0014-constrained-AnySequence.md)
* Author(s): [Max Moiseev](https://github.com/moiseev)
* Status: **Awaiting review** (schedule December 18--21, 2015)
* Review Manager: [Doug Gregor](https://github.com/DougGregor)

## Introduction

In order to allow `AnySequence` delegate calls to the underlying sequence,
its initializer should have extra constraints.

## Motivation

At the moment `AnyCollection` does not delegate calls to `SequenceType` protocol
methods to the underlying base sequence, which results in dynamic downcasts in
places where this behavior is needed (see default implementations of
`SequenceType.dropFirst` or `SequenceType.prefix`). Besides, and this is even
more important, customized implementations of `SequenceType` methods would be
ignored without delegation.

## Proposed solution

See the implementation in [this PR](https://github.com/apple/swift/pull/220).

In order for this kind of delegation to become possible, `_SequenceBox` needs to
be able to 'wrap' not only the base sequence but also it's associated
`SubSequence`. So instead of being declared like this:

~~~~Swift
internal class _SequenceBox<S : SequenceType>
    : _AnySequenceBox<S.Generator.Element> { ... }
~~~~

it would become this:

~~~~Swift
internal class _SequenceBox<
  S : SequenceType
  where
    S.SubSequence : SequenceType,
    S.SubSequence.Generator.Element == S.Generator.Element,
    S.SubSequence.SubSequence == S.SubSequence
> : _AnySequenceBox<S.Generator.Element> { ... }
~~~~

Which, in it's turn, will lead to `AnySequence.init` getting a new set of
constraints as follows.

Before the change:

~~~~Swift
public struct AnySequence<Element> : SequenceType {
  public init<
    S: SequenceType
    where
      S.Generator.Element == Element
  >(_ base: S) { ... }
}
~~~~

After the change:

~~~~Swift
public struct AnySequence<Element> : SequenceType {
  public init<
    S: SequenceType
    where
      S.Generator.Element == Element,
      S.SubSequence : SequenceType,
      S.SubSequence.Generator.Element == Element,
      S.SubSequence.SubSequence == S.SubSequence
  >(_ base: S) { ... }
}
~~~~

These constraints, in fact, should be applied to `SequenceType` protocol itself
(although, that is not currently possible), as we expect every `SequenceType`
implementation to satisfy them already.

## Impact on existing code

New constraints do not affect any built-in types that conform to
`SequenceType` protocol as they are essentially constructed like this
(`SubSequence.SubSequence == SubSequence`). 3rd party collections, if they use
the default `SubSequence` (i.e. `Slice`), should also be fine. Those having
custom `SubSequence`s may stop conforming to the protocol.
