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
**¿Qué hace?** En lugar de contar por commits, f
