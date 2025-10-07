# HOPE

Acciones para aplicar **parches de código** en GitHub con precisión. Sin editar archivos a mano. Sin clonar el repositorio.

## Qué resuelve

Los ciclos de depuración con copias completas de archivos son lentos y frágiles. Con HOPE pegas un **diff unificado** y el repositorio aplica cambios **multi-archivo y no contiguos** en un commit o en un PR. Aumenta la productividad, reduce errores humanos y estandariza el proceso.

## Características

* **Tres modos de entrada**: `inline` (input), `repo` (archivo dentro del repo) y `url` (Gist/paste crudo).
* **Aplicación 3-way**: usa contexto para merges más robustos.
* **PR automático o commit directo** a la rama por defecto.
* **Verificación opcional SHA256** del parche.
* **Limpieza del parche** si se usa `mode=repo`.
* Sin instalaciones locales. Solo GitHub Actions.

## Cómo funciona

1. Ejecutas el workflow desde **Actions → Run workflow**.
2. Proporcionas el parche (inline, archivo del repo o URL cruda).
3. El job ejecuta `git apply --index --3way --whitespace=fix`.
4. Crea un **PR** o empuja un **commit** directo según la opción elegida.

## Instalación rápida

Crea el archivo: `.github/workflows/apply-patch.yml`

````yaml
name: apply-patch
on:
  workflow_dispatch:
    inputs:
      mode:
        description: "inline | repo | url"
        required: true
        default: "inline"
        type: choice
        options: ["inline","repo","url"]
      patch:
        description: "Diff unificado (si mode=inline). SIN ```"
        required: false
        type: string
      path:
        description: "Ruta del parche en el repo (si mode=repo)"
        required: false
        type: string
      url:
        description: "URL cruda al parche (si mode=url)"
        required: false
        type: string
      sha256:
        description: "Hash opcional para verificar el parche"
        required: false
        type: string
      commit_direct:
        description: "Commit directo a la rama por defecto"
        required: true
        default: "false"
        type: choice
        options: ["false","true"]

permissions:
  contents: write
  pull-requests: write

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Resolver parche
        run: |
          set -euo pipefail
          MODE="${{ github.event.inputs.mode }}"
          case "$MODE" in
            inline)
              test -n "${{ github.event.inputs.patch }}" || { echo "Falta 'patch'"; exit 1; }
              printf "%s" "${{ github.event.inputs.patch }}" > /tmp/patch.diff
              ;;
            repo)
              test -n "${{ github.event.inputs.path }}"  || { echo "Falta 'path'"; exit 1; }
              test -f "${{ github.event.inputs.path }}" || { echo "No existe ${{
                github.event.inputs.path
              }}"; exit 1; }
              cp "${{ github.event.inputs.path }}" /tmp/patch.diff
              ;;
            url)
              test -n "${{ github.event.inputs.url }}" || { echo "Falta 'url'"; exit 1; }
              curl -fsSL "${{ github.event.inputs.url }}" -o /tmp/patch.diff
              ;;
            *) echo "mode inválido"; exit 1;;
          esac
          if [ -n "${{ github.event.inputs.sha256 }}" ]; then
            echo "${{ github.event.inputs.sha256 }}  /tmp/patch.diff" | sha256sum -c -
          fi
          sed -i 's/\r$//' /tmp/patch.diff

      - name: Detectar rama por defecto
        id: def
        run: |
          DEF=$(git remote show origin | sed -n 's/^\s*HEAD branch: //p')
          echo "def=$DEF" >> $GITHUB_OUTPUT

      - name: Aplicar parche (3-way)
        run: git apply --index --3way --whitespace=fix /tmp/patch.diff

      - name: Limpiar archivo del repo (solo mode=repo)
        if: ${{ github.event.inputs.mode == 'repo' }}
        run: git rm -f "${{ github.event.inputs.path }}" || true

      - name: PR automático
        if: ${{ github.event.inputs.commit_direct != 'true' }}
        uses: peter-evans/create-pull-request@v7
        with:
          title: "hope: apply patch"
          branch: hope/apply-${{ github.run_id }}
          commit-message: "hope: apply patch"
          delete-branch: true

      - name: Autor del commit
        if: ${{ github.event.inputs.commit_direct == 'true' }}
        run: |
          git config user.name  "${{ github.actor }}"
          git config user.email "${{ github.actor }}@users.noreply.github.com"

      - name: Commit y push directo
        if: ${{ github.event.inputs.commit_direct == 'true' }}
        env:
          TOKEN: ${{ secrets.PAT_PUSH || secrets.GITHUB_TOKEN }}
        run: |
          git commit -m "hope: apply patch"
          git push https://$TOKEN@github.com/${{ github.repository }} HEAD:${{ steps.def.outputs.def }}
````
## Permisos y credenciales

### Workflow permissions

* Repo → **Settings → Actions → General → Workflow permissions** → **Read and write**.

### PAT para commits como tu usuario (`PAT_PUSH`)

* **Tipo**: **Personal Access Token clásico**.
* **Scopes** mínimos:

  * `repo` → permite *push* a públicos y privados.
  * `workflow` → necesario si vas a modificar archivos en `.github/workflows/*` o disparar workflows.
* Guarda el token como **`PAT_PUSH`** en **Settings → Secrets and variables → Actions → New repository secret**.

### ¿Cuándo usar `PAT_PUSH` vs `GITHUB_TOKEN`?

* **`GITHUB_TOKEN`**: válido para *push* en el **mismo repo** cuando “Workflow permissions” es “Read and write”. El autor saldrá como `github-actions[bot]`.
* **`PAT_PUSH`**: úsalo si quieres que el commit salga **a tu nombre**, si necesitas empujar a **otro repo**, o si tocas workflows y tu política exige `workflow`.

### Ramas protegidas

* Si la rama por defecto está protegida, `commit_direct=true` puede fallar. Opciones:

  * Usar **PR** (recomendado).
  * Ajustar reglas de protección para permitir *push* del bot o excepciones controladas.

## Uso

### A) Modo `inline` (rápido)

* Actions → **apply-patch** → Run workflow.
* `mode=inline`.
* Pega el diff en **patch**.
  Ejemplo de diff:

```
diff --git a/README.md b/README.md
--- a/README.md
+++ b/README.md
@@ -1,1 +1,4 @@
 # Proyecto
+Notas:
+- Cambios aplicados con HOPE
+- Soporta múltiples hunks
```

### B) Modo `url` (recomendado)

* Sube el diff a un **Gist secreto** y copia el enlace **Raw**.
* `mode=url`, `url=<raw>`.
* Opcional: añade `sha256` para validar contenido.

### C) Modo `repo`

* Guarda `patches/next.diff` en el repo.
* `mode=repo`, `path=patches/next.diff`.
* El workflow aplica y borra ese archivo en el mismo PR/commit.

## Entradas

* `mode`: `inline` | `repo` | `url`.
* `patch`: diff unificado cuando `mode=inline`.
* `path`: ruta al parche cuando `mode=repo`.
* `url`: enlace crudo cuando `mode=url`.
* `sha256`: hash opcional para verificación.
* `commit_direct`: `true` crea commit directo; `false` abre PR.

## Buenas prácticas

* Revisa en PR antes del commit directo, sobre todo en repos con reglas de protección.
* Usa `url + sha256` si el parche proviene de un tercero.
* Mantén los parches pequeños y temáticos para facilitar revisión.

## Limitaciones

* No ejecuta “razonamiento” de IA. Aplica cambios descritos por el diff.
* `GITHUB_TOKEN` solo empuja al repo actual. Para que el commit figure como tu usuario, añade un **PAT** como `PAT_PUSH`.
* Repos privados consumen minutos de Actions. En público es gratis. Con runner self-hosted no hay consumo.

## Roadmap

* Validación sintáctica opcional para lenguajes comunes.
* Comentarios automáticos con el resultado de `git apply`.
* Modo “dry-run”.

## Licencia

GNU GPLv3.
