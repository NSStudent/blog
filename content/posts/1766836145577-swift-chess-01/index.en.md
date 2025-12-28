---
title: "Bitboards: Representing the Chessboard Efficiently"
date: 2025-12-27
draft: false
layout: simple
# imagePosition: 50% 50%;
description: "In this article we dive into one of the fundamental concepts of any modern chess engine or library: bitboards."
tags: ["swift", "chess", "bitboard"]
---

For a while now I've been working on connecting two fields I'm really interested in: Swift programming and chess.

As is well known, these two worlds have been linked since the early days of computing: from Claude Shannon's early work on chess as a computational problem, through the era of iconic machines like Deep Blue facing Kasparov, to the present with engines like Stockfish.

In this article we will dive into one of the fundamental concepts of any modern chess engine or library: **bitboards**.

The explanation is based on three pillars:

- The classic definition described in [ChessProgramming Wiki](https://www.chessprogramming.org/Bitboards)
- The first Swift implementation I found on the topic, which I used as a reference: [Sage](https://github.com/nvzqz/Sage/tree/develop)
- The implementation this article is based on ([link](https://github.com/NSStudent/fischer-core/blob/main/Sources/FischerCore/Bitboard.swift))

The goal is not only to understand *what a bitboard is*, but *why this approach is so powerful* when we work with chess logic.


## What is a Bitboard?

A **bitboard** is, conceptually, a 64-bit integer (`UInt64`) where **each bit represents a square on the chessboard**.

Since the board has exactly 64 squares (8×8), the fit is perfect:

- 1 bit = 1 square
- Bit `1` → there is something on that square
- Bit `0` → the square is empty

The square order relative to the bit is represented in the following image:
![bit-position](/img/squarebit.svg#center)

For example, if we wanted to represent the position of all the white pawns in this image:
![pawns](/img/pawns-position.svg#center)

We would mark occupied squares with a 1 and order them as shown before, resulting in the following number
![bitboardPawns](/img/bitboardPawns.svg#center)


## Why not use an 8×8 matrix?

The most intuitive way to represent a board is something like:

```swift
[[Piece?]] // 8x8
```

This works, but it has several problems when we want **performance**:

- Indirect memory access
- Many conditions (`if`, `guard`)
- Hard to calculate attacks in bulk
- Poor CPU utilization

Bitboards, on the other hand, allow:

- Operations in **a single instruction**
- Heavy use of bitwise operations (`&`, `|`, `^`, `<<`, `>>`)
- Parallel calculation of moves and attacks

In chess, where every move counts, that makes a huge difference.


## A board as a set of bitboards

A key idea is that **there is not just one bitboard**, but several.

For a position on a board we need to represent different piece types:
- 8 white pawns (we already saw how to represent them with a single bitboard).
- 2 white rooks.
- 2 white knights.
- 2 white bishops.
- The white king.
- The white queen.
- The equivalent for the black pieces.

So with 12 bitboards we have what we need to represent a full board position.

## Our implementation: `Bitboard` as a domain type

In the library we do not use `UInt64` directly everywhere. Instead, we encapsulate the concept in a dedicated type:

```swift
public struct Bitboard {
    private(set) var rawValue: UInt64
}
```

This has several advantages:

- Strong typing (we do not mix “normal” integers with bitboards)
- Expressive API
- Safety and readability
- Ability to add chess-specific behavior

A `Bitboard` **is not a number**, it is a domain concept.
 

## Bitwise operations as the language of chess

With the 12 bitboards defined we can answer questions like:

- Which squares are occupied?
- Which squares are free?
- Which squares does this piece attack?

Everything is built on binary operations.

### OR (`|`) – union

Thinking of `bitboards` as the set of the 12 bitboards of a position, we get the set of occupied squares like this:

```swift
public var occupiedSpaces: Bitboard {
    return bitboards.reduce(0, |)
}
```

### NOT (`~`) – complement

Once we have the occupied squares, it is easy to detect the empty ones as the complement:

```swift
public var emptySpaces: Bitboard {
    return ~occupiedSpaces
}
```

We can implement all the binary logical operators we want, here is an example:

```swift
extension Bitboard {
    public static func << (lhs: Bitboard, rhs: Bitboard) -> Bitboard {
        let shifted = lhs.rawValue << rhs.rawValue
        return Bitboard(rawValue: shifted)
    }

    public static func >> (lhs: Bitboard, rhs: Bitboard) -> Bitboard {
        let shifted = lhs.rawValue >> rhs.rawValue
        return Bitboard(rawValue: shifted)
    }

    public static func <<= (lhs: inout Bitboard, rhs: Bitboard) {
        lhs = lhs << rhs
    }

    public static func >>= (lhs: inout Bitboard, rhs: Bitboard) {
        lhs = lhs >> rhs
    }

    public static func & (lhs: Bitboard, rhs: Bitboard) -> Bitboard {
        Bitboard(rawValue: lhs.rawValue & rhs.rawValue)
    }

    public static func | (lhs: Bitboard, rhs: Bitboard) -> Bitboard {
        Bitboard(rawValue: lhs.rawValue | rhs.rawValue)
    }

    public static func ^ (lhs: Bitboard, rhs: Bitboard) -> Bitboard {
        Bitboard(rawValue: lhs.rawValue ^ rhs.rawValue)
    }

    public static func > (lhs: Bitboard, rhs: Bitboard) -> Bool {
        lhs.rawValue > rhs.rawValue
    }

    public static func < (lhs: Bitboard, rhs: Bitboard) -> Bool {
        lhs.rawValue < rhs.rawValue
    }

    public static prefix func ~ (board: Bitboard) -> Bitboard {
        Bitboard(rawValue: ~board.rawValue)
    }
}
```

## Iterating squares: popcount and bit scan

Another key point is **iterating only over active squares**, not all 64.

Instead of:

```swift
for square in 0..<64
```

We use techniques like:

- `popcount` → number of active bits
- extracting the least significant bit
- clearing the bit once processed

In our implementation, the `Bitboard` iterator does exactly that: it keeps popping the **LS1B** (least significant set bit) with `popLSB()` until the board is empty. Internally we use the classic trick described in [BitScan](https://www.chessprogramming.org/BitScan):

```swift
extension Bitboard {

    /// A lookup table of least significant bit indices.
    private static let lsbTable: [Int] = [00, 01, 48, 02, 57, 49, 28, 03,
                                          61, 58, 50, 42, 38, 29, 17, 04,
                                          62, 55, 59, 36, 53, 51, 43, 22,
                                          45, 39, 33, 30, 24, 18, 12, 05,
                                          63, 47, 56, 27, 60, 41, 37, 16,
                                          54, 35, 52, 21, 44, 32, 23, 11,
                                          46, 26, 40, 15, 34, 20, 31, 10,
                                          25, 14, 19, 09, 13, 08, 07, 06]

    /// A lookup table of most significant bit indices.
    private static let msbTable: [Int] = [00, 47, 01, 56, 48, 27, 02, 60,
                                          57, 49, 41, 37, 28, 16, 03, 61,
                                          54, 58, 35, 52, 50, 42, 21, 44,
                                          38, 32, 29, 23, 17, 11, 04, 62,
                                          46, 55, 26, 59, 40, 36, 15, 53,
                                          34, 51, 20, 43, 31, 22, 10, 45,
                                          25, 39, 14, 33, 19, 30, 09, 24,
                                          13, 18, 08, 12, 07, 06, 05, 63]

    /// A lookup table of bitboards for all squares.
    private static let bitboardTable: [Bitboard] = (0 ..< 64).map { Bitboard(rawValue: 1 << $0) }

    /// The De Bruijn multiplier.
    private static let debruijn64: UInt64 = 0x03f79d71b4cb0a89

    /// Returns the index of the lsb value.
    private func index(lsb value: Bitboard) -> Int? {
        guard value != 0 else {
            return nil
        }
        return Bitboard.lsbTable[Int((value.rawValue &* Bitboard.debruijn64) >> 58)]
    }

    /// The least significant bit.
    public var lsb: Bitboard {
        return Bitboard(rawValue: rawValue & (0 &- rawValue))
    }

    /// The index for the least significant bit of `self`.
    public var lsbIndex: Int? {
        return index(lsb: lsb)
    }

    /// The square for the least significant bit of `self`.
    public var lsbSquare: Square? {
        return lsbIndex.flatMap({ Square(rawValue: $0) })
    }

    private var msbShifted: UInt64 {
        var x = rawValue
        x |= x >> 1
        x |= x >> 2
        x |= x >> 4
        x |= x >> 8
        x |= x >> 16
        x |= x >> 32
        return x
    }

    /// The most significant bit.
    public var msb: Bitboard {
        return Bitboard(rawValue: (msbShifted >> 1) + 1)
    }

    /// The index for the most significant bit of `self`.
    public var msbIndex: Int? {
        guard rawValue != 0 else {
            return nil
        }
        return Bitboard.msbTable[Int((msbShifted &* Bitboard.debruijn64) >> 58)]
    }

    /// The square for the most significant bit of `self`.
    public var msbSquare: Square? {
        return msbIndex.flatMap({ Square(rawValue: $0) })
    }

    /// Removes the least significant bit and returns it.
    public mutating func popLSB() -> Bitboard {
        let lsb = self.lsb
        rawValue -= lsb.rawValue
        return lsb
    }

    /// Removes the least significant bit and returns its index, if any.
    public mutating func popLSBIndex() -> Int? {
        return index(lsb: popLSB())
    }

    /// Removes the least significant bit and returns its square, if any.
    public mutating func popLSBSquare() -> Square? {
        return popLSBIndex().flatMap({ Square(rawValue: $0) })
    }
}
```

- `lsb = x & -x` with this operation we isolate the least significant active bit (the 1 that is lowest in the binary number)
- Thanks to the De Bruijn tables and the `private func index(lsb value: Bitboard) -> Int?` function we can get the index to know the square
- We call `popLSB` to remove that bit and continue iterating. 

```swift
public struct Bitboard: Equatable, Hashable {

    private(set) var rawValue: UInt64 = 0

    public init(rawValue: UInt64) {
        self.rawValue = rawValue
    }

    public init() {}

    /// An iterator over the squares set in a `Bitboard`.
    ///
    /// Iterates by popping least significant bits until the bitboard is empty.
    public struct Iterator: IteratorProtocol {

        fileprivate var bitboard: Bitboard

        /// Returns the next square in the bitboard by removing the least significant bit.
        public mutating func next() -> Square? {
            return bitboard.popLSBSquare()
        }
    }
}
```

This greatly reduces the computational cost, especially for move generation, because we only walk the squares with pieces instead of all 64.

## Conclusion

Bitboards are not just an optimization:  
they are a **different way to think about chess in code**.

By representing the board as binary data:

- Chess becomes algebra
- Moves become operations
- The position becomes a compact, efficient state

In upcoming articles we will see how to generate **attack bitboards** for each piece, build **complete positions**, and **legal move generation** from this base.
