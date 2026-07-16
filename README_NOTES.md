# ⚠️ Nota Importante sobre el Entorno y Docker

## 🧪 Validación del Código

El código y configuración proporcionados en este repositorio han sido **probados en macOS** y funcionan correctamente en dicho entorno.

---

## 🐧🪟 Compatibilidad en Linux y Windows

Aunque el entorno Docker y los scripts deberían funcionar en **Linux** y **Windows**, es posible que surjan **problemas de compatibilidad**, configuración de permisos, volúmenes, o diferencias con WSL (en Windows).
Estos inconvenientes son responsabilidad del estudiante resolver.

---

## 🛠️ Esta Es una Herramienta de Apoyo

Este entorno fue diseñado como **ayuda** para facilitar el inicio. No es obligatorio ni definitivo.

🎓 Los estudiantes están completamente capacitados para:

- Construir su **propio entorno local**
- Ejecutar **ANTLR directamente** en su sistema sin usar Docker

---

## ✅ Alternativa Recomendada

Si se presentan dificultades con Docker, se recomienda instalar ANTLR localmente y seguir la ejecución sin contenedores.

---

## 🤝 Responsabilidad

El entorno proporcionado es opcional y **no es responsabilidad del profesor** mantener compatibilidad para todos los sistemas operativos.

---

## 🪟 Problemas encontrados y solución en Windows

Al construir y correr el contenedor en Windows aparecieron dos problemas específicos del entorno. Se documentan aquí por si le pasan a alguien más.

### 1. `RUN ./python-venv.sh` falla con `not found`

**Síntoma:**

```
=> ERROR [16/20] RUN ./python-venv.sh
> [16/20] RUN ./python-venv.sh:
0.152 /bin/sh: 1: ./python-venv.sh: not found
ERROR: failed to build: ... exit code: 127
```

**Causa:** Git en Windows convierte los saltos de línea a **CRLF** al hacer checkout (`core.autocrlf=true`). Los scripts (`python-venv.sh`, `commands/antlr`, `commands/grun`) quedan con `#!/bin/sh\r`, y ese shebang con `\r` no es válido dentro del contenedor Linux, así que el sistema no encuentra el intérprete.

**Solución:** convertir los scripts a terminadores **LF** y agregar un `.gitattributes` en la raíz del laboratorio para que Git no los vuelva a convertir a CRLF:

```
* text=auto eol=lf

commands/antlr text eol=lf
commands/grun text eol=lf
python-venv.sh text eol=lf
Dockerfile text eol=lf
```

### 2. El volumen se monta vacío / `MiniLang.g4: cannot find or open file`

**Síntoma:** dentro del contenedor, `/program` aparece vacío aunque los archivos existen en el host, o `antlr` no encuentra `MiniLang.g4`.

**Causa:** depende de la terminal que uses en Windows para invocar `docker run`:

- En **Git Bash / MSYS**, `$(pwd)` se reescribe automáticamente por el traductor de rutas de MSYS antes de llegar a Docker, y el volumen termina montando algo distinto a lo esperado.
- En **PowerShell**, el problema no ocurre igual, pero hay que usar la sintaxis correcta para interpolar la ruta actual.

**Solución (PowerShell, recomendado en Windows):**

```powershell
docker build -t lab1-image .
docker run --rm -ti -v "${PWD}\program:/program" lab1-image
```

**Alternativa (si usas Git Bash):** anteponer `MSYS_NO_PATHCONV=1` para que MSYS no reescriba la ruta del volumen:

```bash
MSYS_NO_PATHCONV=1 docker run --rm -ti -v "$(pwd)/program":/program lab1-image
```
