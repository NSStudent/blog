---
title: "Paso a paso: las animaciones del onboarding de MindChess, de HTML a SwiftUI"
date: 2026-11-04
draft: true
layout: simple
description: "Cómo se hicieron las escenas animadas del onboarding de MindChess: prototipadas en un canvas HTML como una función pura del tiempo y portadas a un Canvas de SwiftUI con la misma función."
summary: "Una escena animada es una función draw(t). Así se prototipa en un canvas HTML con IA y se porta a SwiftUI sin usar sus animaciones, con snapshots y Reduce Motion gratis."
tags: ["swiftui", "animaciones", "canvas", "ia", "mindchess"]
showTableOfContents: true
---

<!-- Borrador 2026-11-04 en la rama drafts/kit-y-animaciones. Cotejado con design-iphone/assets/motion.js el 4 nov; fotogramas sacados del kit. Pendiente: GIF o vídeo de la escena en la app. -->

El onboarding y la pantalla de Novedades de MindChess tienen pequeñas escenas animadas: diez monedas que caen en un anillo y se funden en una insignia, un tablero de papel que se endereza y se convierte en el de la app, tres rivales de los que eliges uno y te responde 1…e5.

Todas nacieron en HTML, en el mismo kit de diseño del [artículo anterior](../1793810125083-kit-html-a-swiftui/), y todas pasaron a SwiftUI **sin usar ni una animación de SwiftUI**: ni `withAnimation`, ni `matchedGeometryEffect`, ni transiciones. Este artículo cuenta el proceso paso a paso con una de ellas, la de las monedas, que puedes tocar aquí mismo:

{{< demo src="/demos/motion-scene/" height="620" title="La escena de las monedas en el navegador: muévete por el tiempo y toca el muelle" >}}

Arrastra `t` y verás que puedes ir a cualquier instante, hacia delante o hacia atrás. Esa es toda la idea.

## Paso 1: una escena es una función del tiempo

La forma habitual de animar en SwiftUI es declarar estados y dejar que el sistema interpole entre ellos. Funciona muy bien para la interfaz, pero para una escena de cinco segundos con quince cosas moviéndose a destiempo se vuelve una cadena de `DispatchQueue.main.asyncAfter`, estados intermedios y animaciones que se pisan.

En las escenas de MindChess no hay estado. Hay una función:

```text
draw(t) → el fotograma en el segundo t
```

Todo lo que se ve se calcula a partir de `t`: dónde está cada moneda, su opacidad, el tamaño de la insignia, cuántas monedas lleva el contador. Nada depende del fotograma anterior.

Eso da tres cosas gratis:

- **Snapshots deterministas:** pintar `draw(4.4)` da siempre la misma imagen.
- **Reduce Motion:** en vez de animar, se pinta un único fotograma quieto que ya cuenta la historia.
- **Bucles sin costura:** si al final de la escena todo vuelve a opacidad cero, el segundo 0 y el último encajan.

## Paso 2: una caja fija y una gramática

Todas las escenas se dibujan en una caja lógica de **360 × 280** puntos, y luego se escalan al hueco que les dé la pantalla. Así las coordenadas del prototipo y las de Swift son las mismas, número a número.

Y todas siguen la misma gramática: **entrada → acción → resultado → reposo**. En la de las monedas:

| Fase | Tiempo | Qué pasa |
|---|---|---|
| Entrada | 0–0,25 s | Aparece el anillo de diez huecos |
| Acción | 0,35–2,15 s | Cae una moneda cada 0,2 s y el contador sube |
| Resultado | 2,75–3,7 s | El anillo se funde, la insignia gira y se gana, salen chispas |
| Reposo | 4,4 s | El fotograma quieto: la insignia y «10/10» |

El fotograma de reposo (`stillTime`) es el que se pinta con Reduce Motion y en los snapshots, así que tiene que contar la historia él solo.

Estos son cinco fotogramas de la escena del kit, pintados con `draw(t)` en cinco instantes. El último es el de reposo:

![Cinco fotogramas de la escena de las monedas: a 0,45 s el anillo vacío y la primera moneda cayendo (0/10); a 1,1 s cuatro monedas (4/10); a 2 s ocho (8/10); a 3,05 s el anillo se funde alrededor de la insignia (10/10); a 4,4 s la insignia ganada (10/10)](coins-frames.png "La misma función en cinco instantes. A 4,4 s, el fotograma quieto.")

## Paso 3: curvas en forma cerrada

Para que `draw(t)` sea pura, el movimiento no puede salir de una simulación paso a paso. Todas las curvas son **fórmulas cerradas** que se evalúan en cualquier instante. Son cuatro, y caben en una pantalla.

En JavaScript, en el prototipo:

```js
function spring(x, w = 14) {          // muelle amortiguado de 0 a 1
  if (x <= 0) return 0;
  const z = 0.78, wd = w * Math.sqrt(1 - z * z), e = Math.exp(-z * w * x);
  return 1 - e * (Math.cos(wd * x) + (z * w / wd) * Math.sin(wd * x));
}
const clamp = (v, a = 0, b = 1) => Math.min(b, Math.max(a, v));
function smooth(a, b, x) { const u = clamp((x - a) / (b - a)); return u * u * (3 - 2 * u); }
const lerp = (a, b, k) => a + (b - a) * k;
```

Y en Swift, en la app, línea por línea:

```swift
public enum Motion {
    /// Damped spring from 0 to 1 starting at `x = 0` (damping 0.78, `w` rad/s); 0 before it starts.
    public static func spring(_ x: Double, _ w: Double = 14) -> Double {
        guard x > 0 else { return 0 }
        let z = 0.78, wd = w * (1 - z * z).squareRoot(), e = exp(-z * w * x)
        return 1 - e * (cos(wd * x) + (z * w / wd) * sin(wd * x))
    }

    public static func clamp(_ v: Double, _ a: Double = 0, _ b: Double = 1) -> Double { min(b, max(a, v)) }

    public static func smooth(_ a: Double, _ b: Double, _ x: Double) -> Double {
        let u = clamp((x - a) / (b - a))
        return u * u * (3 - 2 * u)
    }

    public static func lerp(_ a: Double, _ b: Double, _ k: Double) -> Double { a + (b - a) * k }

    /// A spring up at `c` and back down `len` seconds later.
    public static func bump(_ t: Double, _ c: Double, _ len: Double = 0.14, _ w: Double = 30) -> Double {
        spring(t - c, w) - spring(t - c - len, w)
    }
}
```

`spring` es la respuesta de un muelle con amortiguación 0,78: sube de 0 a 1, se pasa un poco y se asienta. El parámetro `w` decide lo rápido que lo hace. `smooth` es un smoothstep entre dos instantes, perfecto para fundidos. `bump` es un muelle que sube y baja: el «pop» de la insignia al ganarse.

En la demo de arriba puedes cambiar la amortiguación y la frecuencia y ver cómo cambia la caída de las monedas.

## Paso 4: prototipar en el navegador con IA

El prototipo vive en el kit de diseño, en un archivo de escenas en JavaScript (`motion.js`). Hoy tiene catorce escenas, y la página del kit las enseña todas a la vez, en oscuro y en claro, con su duración y dónde se usa cada una:

![Página «Escenas animadas» del kit: una rejilla de escenas (Encuentra tu rival, Entrena tu táctica, Trae tus partidas, Monedas → insignia, Tu gimnasio diario…), cada una en oscuro y en claro con su nombre y duración](motion-catalog.jpg "El catálogo de escenas del kit. Con ?motion=still se pintan todas en su fotograma quieto.")

Cada escena es una función que recibe el contexto de un `<canvas>` y `t`. Escribirlas con un agente de IA funciona especialmente bien, por tres motivos:

1. **La descripción en lenguaje natural ya es casi el código.** «Diez monedas en un anillo de radio 88; cae una cada 0,2 s desde 64 pt más arriba con un muelle de frecuencia 16» se traduce casi literalmente.
2. **Se itera mirando el navegador.** Recargo, arrastro el tiempo hasta el momento que no me gusta y le digo al agente qué cambiar en ese segundo concreto.
3. **Los datos son reales.** Los colores salen de los tokens del kit, y en las escenas con tablero, de posiciones y jugadas de verdad, con la misma apariencia que la app.

Así es la escena de las monedas en el prototipo (reducida):

```js
function draw(ctx, t) {
  const cx = 180, cy = 124, r = 88, forge = 2.75;
  ctx.globalAlpha = smooth(0, 0.25, t) * (1 - smooth(5.4, 5.9, t)); // entrada y salida
  const melt = smooth(forge, forge + 0.3, t);
  for (let i = 0; i < 10; i++) {
    const a = -Math.PI / 2 + i * 2 * Math.PI / 10;
    const p = { x: cx + r * Math.cos(a), y: cy + r * Math.sin(a) };
    const ti = 0.35 + i * 0.2, k = spring(t - ti, 16);
    if (t <= ti) continue;
    const y = lerp(p.y - 64, p.y, k);            // cae desde 64 pt más arriba
    coin(ctx, p.x, y, (t - ti) * 6 * (1 - melt)); // y se funde al forjar
  }
  // …el anillo, la insignia que gira, las chispas y el contador
}
```

## Paso 5: portar a SwiftUI

Con la escena decidida, el port tiene tres piezas.

**Un protocolo para las escenas.** Cada escena dice cuánto dura, cuál es su fotograma quieto y cómo se dibuja:

```swift
public protocol MotionScene: Sendable {
    var duration: Double { get }
    var stillTime: Double { get }
    var box: CGSize { get }       // 360 × 280 por defecto
    var speed: Double { get }     // 1 por defecto
    func draw(_ canvas: inout MotionCanvas, t: Double)
}
```

**Un lienzo que se copia.** `MotionCanvas` envuelve el `GraphicsContext` de SwiftUI, la paleta del esquema de color y unos cuantos helpers (`circle`, `fill`, `line`, `text`). Es un tipo valor, y eso sustituye al `save()` y `restore()` del canvas de HTML: para cambiar la opacidad o la transformación de una parte, se copia, se cambia la copia y se tira.

```swift
var coin = c                       // copia: guarda el estado
coin.fade(M.clamp((t - ti) * 6) * (1 - melt))
coin.circle(CGPoint(x: p.x, y: y), 10.5, accent)
// al salir de este ámbito, `c` sigue como estaba
```

**La escena, traducida línea por línea.** Con las mismas curvas y la misma caja, la escena de Swift se lee igual que la de JavaScript:

```swift
struct CoinsScene: MotionScene {
    var duration: Double { 6 }
    var stillTime: Double { 4.4 }

    func draw(_ c: inout MotionCanvas, t: Double) {
        let cx = 180.0, cy = 124.0, r = 88.0, forge = 2.75
        c.fade(M.smooth(0, 0.25, t) * (1 - M.smooth(5.4, 5.9, t)))
        let melt = M.smooth(forge, forge + 0.3, t)
        for i in 0..<10 {
            let a = -Double.pi / 2 + Double(i) * 2 * .pi / 10
            let p = CGPoint(x: cx + r * cos(a), y: cy + r * sin(a))
            let ti = 0.35 + Double(i) * 0.2, k = M.spring(t - ti, 16)
            guard t > ti else { continue }
            var coin = c
            coin.fade(M.clamp((t - ti) * 6) * (1 - melt))
            let y = M.lerp(p.y - 64, p.y, k)
            coin.circle(CGPoint(x: p.x, y: y), 10.5, accent)
        }
        // …el anillo, la insignia que gira, las chispas y el contador
    }
}
```

## Paso 6: reproducirla

Queda la vista que da tiempo a la escena. Es un `Canvas` dentro de un `TimelineView`, que escala la caja de 360 × 280 al hueco disponible y llama a `draw` con el `t` que toque:

```swift
TimelineView(.animation(minimumInterval: 1 / 60, paused: !isLive || !isOnScreen)) { timeline in
    Canvas { context, size in
        let time = reduceMotion || isResting
            ? scene.stillTime
            : liveTime(timeline.date.timeIntervalSince(start))
        draw(in: context, size: size, t: time)
    }
}
```

Aquí salieron los detalles que no se ven en el prototipo:

- **Se reproduce dos veces y se queda quieta.** Una animación en bucle infinito en un onboarding distrae y gasta batería. Después de dos vueltas, la escena se para en su `stillTime`, que está dentro de la segunda vuelta, así que no hay salto. A partir de ahí el `TimelineView` está en pausa y no se redibuja nada.
- **`speed`.** Algunas escenas se diseñaron más largas de lo que conviene en la app. En vez de rehacer los tiempos, la escena declara una velocidad y el tiempo real se multiplica por ella. Así cada escena da una vuelta en unos cinco segundos.
- **En pausa fuera de pantalla** y oculta para VoiceOver, porque es decorativa: el texto de la página ya cuenta lo mismo.
- **Con la apariencia del usuario.** Las escenas con tablero usan el tema de tablero y de piezas que la persona ha elegido en Ajustes.
- **Con la postura del dispositivo.** La escena del reloj, en el iPhone Duo, enseña el reloj en tabletop real, con la bisagra.

## Paso 7: los tests

Como el tiempo es un parámetro, un valor de entorno permite fijarlo:

```swift
public enum MotionPlayback: Equatable, Sendable {
    case live
    case still
    case time(Double)
}
```

Los snapshots del onboarding y de Novedades pintan cada escena en su fotograma quieto, en claro y oscuro, en el iPhone y en cada postura del Duo. Si alguien cambia una curva y la insignia ya no queda centrada en el segundo 4,4, el test falla.

## Lo que me llevo

- **Una función del tiempo es más fácil de razonar que una cadena de animaciones.** Y se puede ir a cualquier instante para depurarla.
- **Las fórmulas cerradas viajan bien entre lenguajes.** Las curvas de JavaScript y las de Swift son las mismas líneas con otra sintaxis.
- **Prototipar en el navegador con un agente es muy rápido** cuando puedes señalar un segundo concreto y decir qué falla.
- **Accesibilidad y tests salen solos** si el fotograma quieto cuenta la historia.

{{< promo "mindchess" >}}
