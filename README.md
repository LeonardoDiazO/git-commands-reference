# Git & AWS S3 — Guía de referencia técnica

> **Documentación oficial:** https://www.atlassian.com/es/git/tutorials/setting-up-a-repository/git-clone

---

## Convenciones usadas en esta guía

Antes de copiar cualquier comando, reemplaza los siguientes placeholders con tus valores reales:

| Placeholder | Descripción | Ejemplo |
|---|---|---|
| `<destino>` | Ruta donde se copiarán los archivos | `/home/usuario/proyecto/dist` |
| `<rama-base>` | Rama principal de referencia | `main`, `master`, `development` |
| `<mi-rama>` | Tu rama de trabajo actual | `fix/TICKET-123` |
| `<release>` | Rama de release o versión | `release/v1.0.0` |
| `<commit-hash>` | Identificador único de un commit | `63ccc97` |
| `<N>` | Número entero positivo | `3`, `6`, `24` |
| `<bucket>` | URI del bucket S3 | `s3://mi-proyecto` |
| `<archivo>` | Ruta relativa al archivo | `src/components/Header.js` |
| `<carpeta-local>` | Ruta local de tu proyecto compilado | `./dist`, `./public` |

---

## 1. Consulta y comparación de cambios

### Ver cambios sin confirmar en el working directory
```bash
git diff
```
**¿Qué hace?** Compara el estado actual de tus archivos contra el último commit registrado. Te muestra línea por línea qué fue agregado (en verde con `+`) y qué fue eliminado (en rojo con `-`). Es el primer comando que debes ejecutar cuando quieres revisar qué has cambiado antes de hacer un commit.

---

### Ver archivos modificados en el último commit
```bash
git diff --name-only HEAD~1 HEAD
```
**¿Qué hace?** Muestra solo los nombres de los archivos que cambiaron en el commit más reciente, sin mostrar el contenido de los cambios. `HEAD` apunta al commit actual y `HEAD~1` es el commit inmediatamente anterior. Útil cuando necesitas saber rápidamente qué archivos tocó el último commit.

---

### Ver archivos diferentes entre dos ramas
```bash
git diff --name-only <rama-base>..<release>
```
**¿Qué hace?** Compara dos ramas y lista los archivos que son diferentes entre ellas. Los dos puntos `..` le indican a Git que compare el estado final de ambas ramas. Útil para saber qué cambios tiene una rama de release respecto a la rama base antes de hacer un merge o despliegue.

---

### Comparar tu rama de trabajo contra la rama base
```bash
git diff <rama-base>...<mi-rama> --name-only
```
**¿Qué hace?** Lista todos los archivos que modificaste en tu rama respecto a la rama base. Los tres puntos `...` le indican a Git que busque el punto en común entre ambas ramas y compare desde ahí, ignorando cambios que ya existían en la base. Ideal para revisar exactamente qué archivos son parte de tu trabajo antes de abrir un Pull Request.

---

### Ver archivos modificados en los últimos `<N>` commits
```bash
git diff --name-only HEAD~<N>..HEAD
```
**¿Qué hace?** Acumula todos los archivos que cambiaron en los últimos `<N>` commits y los lista. `HEAD~<N>` retrocede `N` posiciones desde el commit actual. Por ejemplo, `HEAD~6` significa "6 commits atrás".

```bash
# Ejemplo: ver archivos de los últimos 6 commits
git diff --name-only HEAD~6..HEAD
```

---

### Ver archivos modificados en el último día en una rama
```bash
git diff --name-only $(git rev-list -n 1 --before="1 days ago" <release>)..<release>
```
**¿Qué hace?** Primero ejecuta el comando interno `git rev-list -n 1 --before="1 days ago"` para encontrar cuál era el último commit de la rama hace exactamente 24 horas, y luego compara ese punto contra el estado actual de la rama. El resultado es la lista de archivos que cambiaron durante ese período. Muy útil para revisar qué se subió a producción en el día.

---

### Ver el historial completo de un archivo (incluyendo renombrados)
```bash
git log --follow -- <archivo>
```
**¿Qué hace?** Muestra todos los commits de la historia del repositorio que tocaron ese archivo específico. El flag `--follow` es clave: hace que Git rastree el archivo incluso si fue renombrado o movido de carpeta en algún momento del pasado. Sin `--follow`, el historial se corta en el punto donde el archivo cambió de nombre. Los dos guiones `--` le indican a Git que lo que sigue es una ruta de archivo, no una rama.

**Variantes útiles:**
```bash
# Ver los cambios exactos (diff) de cada commit que tocó el archivo
git log --follow -p -- <archivo>

# Ver el historial en formato compacto: hash corto + mensaje del commit
git log --follow --oneline -- <archivo>
```

---

### Comparar un commit específico contra su commit padre
```bash
git diff <commit-hash>^ <commit-hash> --name-only
```
**¿Qué hace?** Muestra exactamente qué archivos se modificaron en ese commit en particular. El símbolo `^` al final del hash significa "el commit anterior a este". Es útil cuando alguien te pasa un hash específico y quieres saber qué tocó sin tener que revisar toda la historia.

---

## 2. Copia de archivos modificados

> Todos los comandos de esta sección usan `rsync -R` que, a diferencia de un simple `cp`, **preserva la estructura de carpetas** del proyecto al copiar. Esto significa que si un archivo está en `src/components/Button.js`, llegará a destino como `<destino>/src/components/Button.js`.

---

### Copiar archivos que difieren entre tu rama y la base
```bash
for name in $(git diff <rama-base>...<mi-rama> --name-only); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Obtiene la lista de archivos modificados entre tu rama y la base, y los copia uno a uno al destino manteniendo la estructura de carpetas. El bucle `for` itera sobre cada archivo de la lista y `rsync -R` lo copia respetando su ruta relativa.

---

### Copiar archivos modificados en el último commit
```bash
for name in $(git diff --name-only HEAD~1..HEAD); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Toma exactamente los archivos que cambiaron en el último commit y los copia al destino. Útil para despliegues manuales donde solo necesitas enviar lo que cambió más recientemente.

---

### Copiar archivos modificados en los últimos `<N>` commits (sin duplicados)
```bash
for name in $(git diff --name-only HEAD~<N>..HEAD | sort -u); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Igual que el anterior pero acumulando `N` commits. El pipe hacia `sort -u` elimina duplicados: si un mismo archivo fue modificado en varios commits, solo se copiará una vez (su versión más reciente).

---

### Copiar archivos modificados en las últimas `<N>` horas
```bash
for name in $(git diff HEAD "@{<N>.hours.ago}" --name-only); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** En lugar de contar por commits, filtra por tiempo. `@{N.hours.ago}` es una sintaxis especial de Git para referirse al estado del repositorio hace exactamente `N` horas. Útil cuando trabajas por bloques de tiempo y quieres copiar solo lo del turno actual.

---

### Copiar archivos agregados o modificados desde medianoche
```bash
for name in $(git log --since="midnight" --pretty=format: --name-only --diff-filter=AM <rama-base>); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Usa `git log` en lugar de `git diff` para filtrar por fecha desde las 00:00 hrs del día actual. El flag `--diff-filter=AM` restringe los resultados solo a archivos **A**gregados y **M**odificados, excluyendo los eliminados (para no intentar copiar archivos que ya no existen). `--pretty=format:` elimina el encabezado de cada commit dejando solo los nombres de archivos.

---

## 3. Flujo de trabajo con ramas

### Buenas prácticas antes de crear una rama
```bash
# Paso 1: Actualizar las ramas base para partir de código actualizado
git checkout <rama-base> && git pull
git checkout development && git pull

# Paso 2: Crear tu rama de trabajo desde la base actualizada
git checkout -b <mi-rama>
```
**¿Por qué es importante?** Si creas tu rama desde un estado desactualizado, acumularás conflictos que tendrás que resolver más adelante. Siempre hacer `pull` antes de ramificar garantiza que partes del código más reciente del equipo.

---

### Actualizar rama local cuando hay cambios remotos pendientes
```bash
git pull --rebase --autostash
git push
```
**¿Qué hace?** Cuando alguien más subió cambios a tu misma rama y Git te impide hacer push, este comando resuelve el problema de forma segura. `--rebase` reorganiza tus commits locales encima de los remotos (en lugar de crear un merge commit innecesario). `--autostash` guarda automáticamente cualquier trabajo no commiteado, aplica el rebase y lo restaura al final.

---

### Forzar un push de forma segura
```bash
git push --force-with-lease
```
**¿Qué hace?** A veces necesitas reescribir el historial de tu rama (por ejemplo, después de un rebase) y el push normal es rechazado. `--force-with-lease` fuerza el push pero con una verificación de seguridad: solo lo permite si nadie más subió cambios a esa rama desde tu último `pull`. Si alguien subió algo, el comando falla y te avisa, protegiendo el trabajo de tus compañeros. Es la alternativa segura a `--force`.

---

## 4. AWS S3 — Despliegue y sincronización

### Ver buckets disponibles
```bash
aws s3 ls
```
**¿Qué hace?** Lista todos los buckets S3 a los que tiene acceso el perfil AWS configurado en tu máquina. Es el punto de partida para saber con qué entornos puedes trabajar.

---

### Listar contenido de un bucket
```bash
aws s3 ls <bucket>
```
**¿Qué hace?** Muestra los archivos y carpetas dentro del bucket especificado, similar al comando `ls` en terminal. Útil para verificar el estado actual del bucket antes de sincronizar.

---

### Simular una sincronización (dry run — sin cambios reales)
```bash
aws s3 sync "<carpeta-local>" <bucket> --dryrun
```
**¿Qué hace?** Ejecuta toda la lógica de sincronización pero sin aplicar ningún cambio real. Muestra en pantalla exactamente qué archivos se subirían, actualizarían o eliminarían. **Siempre ejecuta esto primero** antes de una sincronización real para evitar subidas accidentales.

---

### Sincronizar carpeta local hacia S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --exact-timestamps
```
**¿Qué hace?** Compara los archivos locales con los del bucket y sube únicamente los que son nuevos o fueron modificados. El flag `--exact-timestamps` usa la fecha de modificación exacta para determinar si un archivo cambió, lo que es más preciso que comparar solo por tamaño. Los archivos que ya existen y son idénticos no se transfieren.

---

### Sincronizar eliminando archivos sobrantes en S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --delete --exact-timestamps
```
**¿Qué hace?** Igual que la sincronización normal, pero además elimina del bucket cualquier archivo que no exista en tu carpeta local. Deja el bucket como un espejo exacto de tu carpeta.

> ⚠️ **Precaución:** Si tienes archivos en S3 que no están en tu carpeta local (como backups o archivos generados por otros procesos), este comando los eliminará permanentemente. Usa `--dryrun` antes para revisar qué se borrará.

---

### Subir un archivo puntual
```bash
aws s3 cp <archivo> <bucket>/<archivo>
```
**¿Qué hace?** Copia un único archivo desde tu máquina hacia una ruta específica dentro del bucket. A diferencia de `sync`, no compara ni verifica nada, simplemente sube ese archivo. Útil para actualizaciones urgentes de un solo archivo como un `index.html` o un `manifest.json`.

---

### Descargar contenido de un bucket a local
```bash
aws s3 sync <bucket> <carpeta-local>
```
**¿Qué hace?** Invierte la dirección de la sincronización: descarga los archivos del bucket hacia tu máquina. Útil para hacer backups locales o para restaurar archivos que solo existen en S3.

---

### Eliminar un archivo del bucket
```bash
aws s3 rm <bucket>/<archivo>
```
**¿Qué hace?** Elimina permanentemente un archivo específico del bucket. No hay confirmación ni papelera de reciclaje, por lo que debes estar seguro de la ruta antes de ejecutarlo.

---

### Eliminar una carpeta completa del bucket
```bash
aws s3 rm <bucket>/<carpeta> --recursive
```
**¿Qué hace?** Elimina una carpeta y todo su contenido dentro del bucket. El flag `--recursive` es obligatorio para operar sobre directorios; sin él, el comando no tendrá efecto. Úsalo con precaución.

---

### Validar identidad AWS activa
```bash
aws sts get-caller-identity
```
**¿Qué hace?** Consulta al servicio de AWS cuál es la identidad que está usando tu terminal en este momento. Devuelve el ID de cuenta, el nombre de usuario o rol, y el ARN completo. Es el primer comando a ejecutar si tienes dudas sobre con qué cuenta o entorno (dev, staging, producción) estás trabajando.
