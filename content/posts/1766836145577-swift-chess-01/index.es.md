---
title: "Bitboards: Representando el tablero de ajedrez de forma eficiente"
date: 2025-12-28
draft: false
layout: simple
# imagePosition: 50% 50%;
description: "En este artículo vamos a entrar en uno de los conceptos fundamentales de cualquier motor o librería de ajedrez moderna: los bitboards."
tags: ["swift", "ajedrez", "bitboard"]
---

Desde hace un tiempo estoy trabajando en relacionar dos campos que me interesan mucho, la programación en Swift y el ajedrez.

Como es bien sabido estos dos mundos han estado ligados desde el principio de la programación: desde los primeros trabajos de Claude Shannon sobre el ajedrez como problema computacional, pasando por la era de las máquinas emblemáticas como Deep Blue enfrentándose a Kasparov, hasta el presente con motores como Stockfish.

En este artículo vamos a entrar en uno de los conceptos fundamentales de cualquier motor o librería de ajedrez moderna: **los bitboards**.

La explicación se basa en tres pilares:

- La definición clásica descrita en [ChessProgramming Wiki](https://www.chessprogramming.org/Bitboards)
- La primera implementación que encontre del tema en swift, de la cual me basé [Sage](https://github.com/nvzqz/Sage/tree/develop)
- La implementación en la que basamos este articulo ([enlace](https://github.com/NSStudent/fischer-core/blob/main/Sources/FischerCore/Bitboard.swift))

El objetivo no es solo entender *qué es* un bitboard, sino *por qué esta aproximación es tan potente* cuando trabajamos con lógica de ajedrez.


## ¿Qué es un Bitboard?

Un **bitboard** es, conceptualmente, un número entero de 64 bits (`UInt64`) donde **cada bit representa una casilla del tablero de ajedrez**.

Como el tablero tiene exactamente 64 casillas (8×8), el encaje es perfecto:

- 1 bit = 1 casilla
- Bit a `1` → hay algo en esa casilla
- Bit a `0` → la casilla está vacía

El orden de las casillas en relación al bit queda representada en la siguiente imagen:
![bit-position](/img/squarebit.svg#center)

Por ejemplo, si quisieramos representar la posición de todos los peones blancos de esta imagen:
![pawns](/img/pawns-position.svg#center)

Indicariamos con un uno las casillas ocupadas y ordenariamos como hemos visto antes quedando el siguiente numero
![bitboardPawns](/img/bitboardPawns.svg#center)


## ¿Por qué no usar una matriz 8×8?

La forma más intuitiva de representar un tablero es algo como:

```swift
[[Piece?]] // 8x8
```

Esto funciona, pero tiene varios problemas cuando queremos **rendimiento**:

- Accesos indirectos a memoria
- Muchas condiciones (`if`, `guard`)
- Dificultad para calcular ataques de forma masiva
- Poco aprovechamiento de la CPU

Los bitboards, en cambio, permiten:

- Operaciones en **una sola instrucción**
- Uso intensivo de operaciones bit a bit (`&`, `|`, `^`, `<<`, `>>`)
- Cálculo de movimientos y ataques de forma paralela

En ajedrez, donde cada movimiento cuenta, esto marca una diferencia enorme.


## Un tablero como conjunto de bitboards

Una idea clave es que **no existe un único bitboard**, sino varios.

para una posición en un tablero necesitamos representar distintos tipos de piezas: 
- 8 peones blancos (Ya hemos visto como representarlos con un solo bitboard).
- 2 torres blancas.
- 2 caballos blancos.
- 2 alfiles blancos.
- El Rey blanco.
- La Dama Blanca
- El equivalente para las piezas negras.

Por lo que con 12 bitboards tenemos lo necesario para representar una posición de un tablero al completo.

## Nuestra implementación: `Bitboard` como tipo de dominio

En la librería no usamos directamente `UInt64` por todas partes. En su lugar, encapsulamos el concepto en un tipo dedicado:

```swift
public struct Bitboard {
    private(set) var rawValue: UInt64
}
```

Esto tiene varias ventajas:

- Tipado fuerte (no mezclamos enteros “normales” con bitboards)
- API expresiva
- Seguridad y legibilidad
- Posibilidad de añadir comportamiento específico de ajedrez

Un `Bitboard` **no es un número**, es un concepto de dominio.
 

## Operaciones bit a bit como lenguaje del ajedrez

Con los 12 bitboard definidos podemos responder preguntas como:

- ¿Qué casillas están ocupadas?
- ¿Qué casillas están libres?
- ¿Qué casillas ataca esta pieza?

La base de todo son las operaciones binarias.

### OR (`|`) – unión

Pensando en `bitboards` como el conjunto de los 12 bitboard de un posición tenemos el conjunto de las casillas ocupadas de la siguiente manera:

```swift
public var occupiedSpaces: Bitboard {
    return bitboards.reduce(0, |)
}
```

### NOT (`~`) – complemento

Una vez tenemos las casillas ocupadas es facil detectar las libres como el complemetario:

```swift
public var emptySpaces: Bitboard {
    return ~occupiedSpaces
}
```

Podemos representar todas las operadores logicos de los binarios que queramos, aqui un ejemplo:

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

## Iterar casillas: popcount y bit scan

Otro punto clave es **iterar solo sobre las casillas activas**, no sobre las 64.

En lugar de:

```swift
for square in 0..<64
```

Usamos técnicas como:

- `popcount` → número de bits activos
- extracción del bit menos significativo
- limpiar el bit una vez procesado

En nuestra implementación, el iterador del `Bitboard` hace precisamente eso: va sacando la **LS1B** (least significant set bit) con `popLSB()` hasta que el tablero queda vacío. Internamente usamos el truco clásico descrito en [BitScan](https://www.chessprogramming.org/BitScan):

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

- `lsb = x & -x` con esta operación conseguimos aislar el bit activo menos significativo (el 1 que esta mas abajo en el número binario)
- Gracias a las tablas de De Bruijn y y la función `private func index(lsb value: Bitboard) -> Int?` podemos sacar el indice para saber la casilla
- Hacemos `popLSB` para eliminar ese bit y continuar la iteración. 

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

Esto reduce muchísimo el coste computacional, especialmente en generación de movimientos, porque solo recorremos las casillas con piezas en vez de las 64.

## Conclusión

Los bitboards no son solo una optimización:  
son una **forma distinta de pensar el ajedrez desde el código**.

Al representar el tablero como datos binarios:

- El ajedrez se convierte en álgebra
- Los movimientos en operaciones
- La posición en un estado compacto y eficiente

En próximos artículos veremos cómo generar **Bitboards de ataques** para cada pieza, construir **posiciones completas** y **generación legal de jugadas** a partir de esta base.
