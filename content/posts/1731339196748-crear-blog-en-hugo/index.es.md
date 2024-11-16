---
title: "Crear un Blog con Hugo y Publicarlo en GitHub Pages"
date: 2024-11-06
defaultContentLanguage: es
draft: false
description: "Guía paso a paso para crear un blog con Hugo y publicarlo en GitHub Pages."
tags: ["hugo", "blogging", "github-pages", "español"]
---

## Introducción

Hugo es un generador de sitios estáticos rápido y flexible que permite crear y personalizar blogs de manera sencilla. Publicar tu blog en GitHub Pages es una excelente manera de alojarlo de forma gratuita y con alta disponibilidad. En esta guía, aprenderás a crear un blog con Hugo y publicarlo en GitHub Pages paso a paso.

## Prerrequisitos

Antes de comenzar, asegúrate de tener lo siguiente:
- [Git](https://git-scm.com/) instalado en tu máquina.
- [Hugo](https://gohugo.io/) instalado.
- Una cuenta en [GitHub](https://github.com/).

## Crear tu Blog con Hugo

1. **Inicializar un Nuevo Proyecto de Hugo**  
   Crea un directorio para tu blog e inicializa el proyecto:
   ```bash
   mkdir MiBlog
   cd MiBlog
   hugo new site .
   ```

2. **Elegir un Tema**  
   Busca un tema que te guste en [Hugo Themes](https://themes.gohugo.io/) y clónalo en tu proyecto:
   ```bash
   git init
   git submodule add https://github.com/<autor>/<tema>.git themes/<tema>
   echo 'theme = "<tema>"' >> config.toml
   ```

3. **Crear tu Primer Artículo**  
   Genera un archivo de contenido:
   ```bash
   hugo new posts/mi-primer-articulo.md
   ```
   Edita el archivo creado en `content/posts/mi-primer-articulo.md` y agrega contenido relevante.

4. **Previsualizar el Blog Localmente**  
   Inicia el servidor de desarrollo para ver tu blog en el navegador:
   ```bash
   hugo server
   ```
   Abre `http://localhost:1313` en tu navegador para ver tu blog.

## Publicar en GitHub Pages

1. **Crear un Repositorio en GitHub**  
   Ve a GitHub y crea un nuevo repositorio. Anota la URL del repositorio.

2. **Configurar el Repositorio Local**  
   Conecta tu proyecto a GitHub:
   ```bash
   git remote add origin https://github.com/<tu-usuario>/<repositorio>.git
   ```

3. **Configurar la Rama para Publicación**  
   Hugo genera los archivos del sitio estático en el directorio `public`. Configuraremos esta carpeta como la rama `gh-pages`:
   ```bash
   hugo
   cd public
   git init
   git remote add origin https://github.com/<tu-usuario>/<repositorio>.git
   git checkout -b gh-pages
   git add .
   git commit -m "Publicar sitio inicial"
   git push origin gh-pages --force
   ```

4. **Configurar GitHub Pages**  
   Ve a la configuración del repositorio en GitHub, selecciona **Pages**, y elige la rama `gh-pages` como fuente.

5. **Automatizar Publicaciones (Opcional)**  
   Puedes configurar GitHub Actions para automatizar la publicación cada vez que hagas un cambio:
   - Crea un archivo `.github/workflows/deploy.yml`:
     ```yaml
     name: Deploy Hugo Site

     on:
       push:
         branches:
           - main

     jobs:
       deploy:
         runs-on: ubuntu-latest
         steps:
         - uses: actions/checkout@v3
           with:
             submodules: true
         - name: Setup Hugo
           uses: peaceiris/actions-hugo@v2
           with:
             hugo-version: 'latest'
         - name: Build
           run: hugo
         - name: Deploy
           uses: peaceiris/actions-gh-pages@v3
           with:
             github_token: ${{ secrets.GITHUB_TOKEN }}
             publish_dir: ./public
     ```
   - Realiza un commit y haz un push del archivo.

## Extra: Publicar en el github-page del usuario con un Submódulo

Si quieres usar el repositorio especial `<User>.github.io` para alojar tu blog, puedes configurar la carpeta `public` como un submódulo. Este método permite que tu código fuente y los archivos generados del sitio estén separados, manteniendo el repositorio principal limpio.

### Pasos:

1. **Crear el Repositorio Especial**  
   Ve a GitHub y crea un repositorio con el nombre exacto `<User>.github.io`.

2. **Configurar el Submódulo para `public`**  
   Desde la raíz de tu proyecto Hugo, añade el repositorio como submódulo vinculado a la carpeta `public`:
   ```bash
   git submodule add -b main https://github.com/<User>/<User>.github.io.git public
   ```

3. **Generar el Sitio Estático**  
   Construye el sitio estático de Hugo, que generará los archivos en la carpeta `public`:
   ```bash
   hugo
   ```

4. **Publicar los Archivos en `<User>.github.io`**  
   Navega a la carpeta `public`, agrega y realiza un commit de los archivos generados, y luego súbelos al repositorio remoto:
   ```bash
   cd public
   git add .
   git commit -m "Publicar sitio inicial"
   git push origin main
   cd ..
   ```

5. **Guardar el Código Fuente del Blog**  
   Vuelve al repositorio principal (el directorio raíz de tu proyecto Hugo) y realiza un commit de los cambios relacionados con el submódulo:
   ```bash
   git add .
   git commit -m "Configurar submódulo para publicar en GitHub Pages"
   git push origin main
   ```

6. **Configurar GitHub Pages**  
   GitHub detectará automáticamente el contenido publicado en el repositorio `<User>.github.io` y lo servirá desde `https://<User>.github.io`.

---

### Notas:

- Cada vez que realices cambios en tu blog y generes los archivos estáticos, deberás navegar a la carpeta `public`, hacer un commit y un push para actualizar tu sitio.
- Si deseas automatizar este proceso, puedes configurar un script o usar GitHub Actions.

Esta opción es ideal si prefieres que tu blog esté alojado en el dominio principal de GitHub Pages (`<User>.github.io`) en lugar de un subdirectorio.

## Conclusión

¡Eso es todo! Ahora tienes un blog funcional en Hugo alojado gratuitamente en GitHub Pages. Puedes personalizarlo, agregar más artículos y compartir tu contenido con el mundo. Si necesitas más ayuda, consulta la documentación oficial de [Hugo](https://gohugo.io/documentation/) y [GitHub Pages](https://pages.github.com/).