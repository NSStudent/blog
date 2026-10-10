# NSStudent blog

Fuente de [nsstudent.dev](https://nsstudent.dev), hecho con [Hugo](https://gohugo.io) y el tema [Blowfish](https://blowfish.page).

## Escribir y publicar

```bash
./blog nuevo "Título del post"        # crea content/posts/<id>-<slug>/index.es.md en borrador
./blog nuevo "Título del post" --en   # …y también la versión en inglés (index.en.md)
./blog ver                            # http://localhost:1313 con borradores, se recarga al guardar
./blog publicar                       # comprueba, commit + push y espera a que esté online
```

Un post se publica cuando tiene `draft: false`. Las imágenes van en la carpeta del post (`featured.png` es la portada). Los posts con fecha futura se publican solos ese día, porque el despliegue se repite cada mañana.

Otros comandos: `./blog comprobar`, `./blog borradores`, `./blog estado`, `./blog actualizar-tema` (`./blog ayuda` muestra todos).

## Cómo se despliega

Cada push a `main` lanza [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml): compila con la versión de Hugo de [`.hugo-version`](.hugo-version) y publica en GitHub Pages. Las PRs solo se compilan, sin publicarse. Dependabot abre PRs cuando sale una versión nueva de Blowfish.

- `public/` y `public-dev/` son salidas locales y están ignoradas por git.
- `hugo server` (incluido el que lanza `npx blowfish-tools`) escribe en `public-dev/` gracias a [`config/development/hugo.toml`](config/development/hugo.toml).
- Para cambiar de versión de Hugo: actualiza `.hugo-version` y `brew upgrade hugo`.

## Primera vez en otra máquina

```bash
git clone --recurse-submodules git@github.com:NSStudent/blog.git
brew install hugo gh
```
