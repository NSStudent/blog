---
title: "Trabajar con Packages en local"
date: 2020-07-26
defaultContentLanguage: en
draft: false
description: "Como poder modificar un package mientras esta usandose en un proyecto."
tags: ["swift", "spm", "tips", "español"]
---

## Introducción

Normalmente, cuando desarrollamos, intentamos hacer nuestro código reutilizable. Muchas cosas que hacemos en nuestros proyectos son comunes, conectarte a una API, cachear las imágenes de la red, etc.

Para poder manejar estos recursos normalmente hacemos uso de gestores de dependencias pero, para el desarrollo en iOS, no teníamos un gestor facilitado por Apple. Hasta hace bien poco la mayoría de los desarrolladores estábamos usando Cocoapods(poner link) o Carthage(poner link) para poder cubrir esta necesidad.

Con Swift Package Manager (SPM) Apple intenta dar solución a esta demanda de los desarrolladores y en la última session de la WWDC han incluido cosas que hace a SPM un posible candidato para la gestión de dependencias en producción.

Mientras estás desarrollando tu dependencia una de las necesidades es poder modificar el código en tiempo de desarrollo dentro del proyecto. Apple no deja muy claro en su documentación como hacer esto, veamos paso por paso como poderlo hacer.

## Ejemplo

El primer paso seria crear el package, para ello crea una carpeta en la ruta donde quieras hacer el ejemplo y dentro de esta carpeta ejecuta la inicialización del package.

```
mkdir PackageExample
cd PackageExample
swift package init
```

Para este tutorial vamos a crear el proyecto nuevo al que queremos incluir este package a la misma altura que la carpeta contenedora, esta seria la estructura de la carpeta actualmente.

```
Carpeta Contenedora
    - ExampleProject
    - PackageExample
```

El siguiente paso es arrastrar la carpeta del package en el proyecto

<p class="aligncenter">
    <img  alt="" src="/img/Local-package-0.png"></img>
</p>

Un detalle importante para que todo funcione es que no puedes tener abierto el package en el Xcode, de lo contrario no te mostrará la jerarquía de archivos al arrastrarlo.

<p class="aligncenter">
    <img alt="" src="/img/Local-package-1.png"></img>
</p>


El último paso que nos falta para que todo funcione es conectar la librería con el proyecto, esto se hace en la sección de `Build Phases` del proyecto en el apartado de `Link Binary With Libraries`, le damos al botón de `+` y seleccionamos la librería de nuestro package.

<p class="aligncenter">
    <img alt="" src="/img/Local-package-2.png"></img>
</p>

Con esto ya podras hacer el `import` de tu package en el proyecto y modificar el código a tu gusto.