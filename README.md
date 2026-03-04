# Git & AWS S3 — Guía de referencia técnica completa

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
| `<mensaje>` | Texto descriptivo del commit o stash | `"fix: corrige validación del formulario"` |

---

## Tabla de contenido

1. [Consulta y comparación de cambios](#1-consulta-y-comparación-de-cambios)
2. [Copia de archivos modificados](#2-copia-de-archivos-modificados)
3. [Flujo de trabajo con ramas](#3-flujo-de-trabajo-con-ramas)
4. [Stash — Guardado temporal de cambios](#4-stash--guardado-temporal-de-cambios)
5. [Deshacer y corregir cambios](#5-deshacer-y-corregir-cambios)
6. [Cherry-pick — Copiar commits específicos](#6-cherry-pick--copiar-commits-específicos)
7. [Blame — Rastrear autoría del código](#7-blame--rastrear-autoría-del-código)
8. [Fetch vs Pull — Diferencias clave](#8-fetch-vs-pull--diferencias-clave)
9. [Reflog — Historial de emergencia](#9-reflog--historial-de-emergencia)
10. [Resolución de conflictos](#10-resolución-de-conflictos)
11. [.gitignore — Excluir archivos del repositorio](#11-gitignore--excluir-archivos-del-repositorio)
12. [AWS S3 — Despliegue y sincronización](#12-aws-s3--despliegue-y-sincronización)

---

## 1. Consulta y comparación de cambios

### Ver cambios sin confirmar en el working directory
```bash
git diff
```
**¿Qué hace?** Compara el estado actual de tus archivos contra el último commit registrado. Muestra línea por línea qué fue agregado (en verde con `+`) y qué fue eliminado (en rojo con `-`). Es el primer comando que debes ejecutar cuando quieres revisar qué has cambiado antes de hacer un commit.

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
**¿Qué hace?** Acumula todos los archivos que cambiaron en los últimos `<N>` commits y los lista. `HEAD~<N>` retrocede `N` posiciones desde el commit actual.

```bash
# Ejemplo: ver archivos de los últimos 6 commits
git diff --name-only HEAD~6..HEAD
```

---

### Ver archivos modificados en el último día en una rama
```bash
git diff --name-only $(git rev-list -n 1 --before="1 days ago" <release>)..<release>
```
**¿Qué hace?** Primero ejecuta el comando interno `git rev-list` para encontrar cuál era el último commit de la rama hace exactamente 24 horas, y luego compara ese punto contra el estado actual. El resultado es la lista de archivos que cambiaron durante ese período. Muy útil para revisar qué se subió a producción en el día.

---

### Ver el historial completo de un archivo (incluyendo renombrados)
```bash
git log --follow -- <archivo>
```
**¿Qué hace?** Muestra todos los commits que tocaron ese archivo específico. El flag `--follow` hace que Git rastree el archivo incluso si fue renombrado o movido de carpeta. Sin `--follow`, el historial se corta en el punto donde el archivo cambió de nombre. Los dos guiones `--` le indican a Git que lo que sigue es una ruta de archivo, no una rama.

```bash
# Ver los cambios exactos (diff) de cada commit que tocó el archivo
git log --follow -p -- <archivo>

# Ver el historial en formato compacto: hash corto + mensaje
git log --follow --oneline -- <archivo>
```

---

### Comparar un commit específico contra su commit padre
```bash
git diff <commit-hash>^ <commit-hash> --name-only
```
**¿Qué hace?** Muestra exactamente qué archivos se modificaron en ese commit en particular. El símbolo `^` significa "el commit anterior a este". Útil cuando alguien te comparte un hash y quieres saber qué tocó sin revisar toda la historia.

---

## 2. Copia de archivos modificados

> Todos los comandos de esta sección usan `rsync -R` que **preserva la estructura de carpetas** del proyecto al copiar. Si un archivo está en `src/components/Button.js`, llegará al destino como `<destino>/src/components/Button.js`.

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
**¿Qué hace?** Igual que el anterior pero acumulando `N` commits. El pipe hacia `sort -u` elimina duplicados: si un mismo archivo fue modificado en varios commits, solo se copiará una vez en su versión más reciente.

---

### Copiar archivos modificados en las últimas `<N>` horas
```bash
for name in $(git diff HEAD "@{<N>.hours.ago}" --name-only); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Filtra por tiempo en lugar de por commits. `@{N.hours.ago}` es una sintaxis especial de Git para referirse al estado del repositorio hace `N` horas. Útil cuando trabajas por bloques de tiempo.

---

### Copiar archivos agregados o modificados desde medianoche
```bash
for name in $(git log --since="midnight" --pretty=format: --name-only --diff-filter=AM <rama-base>); do
  rsync -R "$name" "<destino>"
done
```
**¿Qué hace?** Filtra por fecha desde las 00:00 hrs del día actual. El flag `--diff-filter=AM` restringe los resultados solo a archivos **A**gregados y **M**odificados, excluyendo los eliminados para no intentar copiar archivos que ya no existen. `--pretty=format:` elimina encabezados de commits dejando solo los nombres de archivos.

---

## 3. Flujo de trabajo con ramas

### Buenas prácticas antes de crear una rama
```bash
# Paso 1: Actualizar las ramas base
git checkout <rama-base> && git pull
git checkout development && git pull

# Paso 2: Crear tu rama de trabajo desde la base actualizada
git checkout -b <mi-rama>
```
**¿Por qué es importante?** Si creas tu rama desde un estado desactualizado, acumularás conflictos que tendrás que resolver más adelante. Siempre hacer `pull` antes de ramificar garantiza que partes del código más reciente del equipo.

---

### Ver todas las ramas (locales y remotas)
```bash
# Solo ramas locales
git branch

# Ramas locales y remotas
git branch -a

# Solo ramas remotas
git branch -r
```
**¿Qué hace?** Lista las ramas disponibles. La rama activa aparece marcada con `*`. Con `-a` también verás las ramas que existen en el servidor remoto aunque no las tengas descargadas localmente.

---

### Cambiar de rama
```bash
git checkout <mi-rama>

# Forma moderna (Git 2.23+)
git switch <mi-rama>
```
**¿Qué hace?** Cambia tu working directory al estado de la rama indicada. Si tienes cambios sin confirmar, Git te pedirá que los guardes o confirmes antes de cambiar.

---

### Eliminar una rama
```bash
# Eliminar rama local (solo si ya fue mergeada)
git branch -d <mi-rama>

# Forzar eliminación local (aunque no haya sido mergeada)
git branch -D <mi-rama>

# Eliminar rama del servidor remoto
git push origin --delete <mi-rama>
```
**¿Qué hace?** Limpia ramas que ya no se necesitan. `-d` es seguro porque falla si la rama tiene trabajo sin mergear. `-D` lo fuerza sin verificar. Eliminar la rama remota no borra la local ni viceversa, son operaciones independientes.

---

### Actualizar rama local cuando hay cambios remotos pendientes
```bash
git pull --rebase --autostash
git push
```
**¿Qué hace?** Cuando alguien más subió cambios a tu misma rama y Git te impide hacer push, este comando resuelve el problema. `--rebase` reorganiza tus commits locales encima de los remotos en lugar de crear un merge commit innecesario. `--autostash` guarda automáticamente cualquier trabajo no commiteado, aplica el rebase y lo restaura al final.

---

### Forzar un push de forma segura
```bash
git push --force-with-lease
```
**¿Qué hace?** Fuerza el push pero con una verificación de seguridad: solo lo permite si nadie más subió cambios a esa rama desde tu último `pull`. Si alguien subió algo, el comando falla y te avisa, protegiendo el trabajo de tus compañeros. Es la alternativa segura a `--force`.

---

## 4. Stash — Guardado temporal de cambios

El stash es como una "caja de guardado temporal". Te permite dejar tus cambios en pausa sin hacer un commit, cambiar de tarea o de rama, y luego retomar exactamente donde lo dejaste.

### Guardar cambios en el stash
```bash
# Guardar con nombre descriptivo (recomendado)
git stash push -m "<mensaje>"

# Guardar rápido sin nombre
git stash
```
**¿Qué hace?** Toma todos los cambios que tienes en tu working directory (modificados y en staging), los guarda en una pila temporal y deja tu rama limpia como estaba en el último commit. Útil cuando necesitas cambiar de rama de urgencia sin haber terminado tu tarea.

---

### Ver la lista de stashes guardados
```bash
git stash list
```
**¿Qué hace?** Muestra todos los stashes guardados con su índice (`stash@{0}`, `stash@{1}`, etc.), la rama donde fueron creados y el mensaje descriptivo. El índice `0` es siempre el más reciente.

---

### Recuperar el último stash guardado
```bash
# Aplicar y eliminar el stash de la lista
git stash pop

# Aplicar sin eliminar (por si quieres aplicarlo en varias ramas)
git stash apply
```
**¿Qué hace?** Restaura los cambios guardados en tu working directory. `pop` los elimina de la pila al aplicarlos. `apply` los mantiene en la lista para poder reutilizarlos.

---

### Recuperar un stash específico
```bash
git stash pop stash@{<N>}
git stash apply stash@{<N>}
```
**¿Qué hace?** Aplica un stash específico de la lista en lugar del más reciente. El número `<N>` corresponde al índice que muestra `git stash list`.

---

### Eliminar un stash
```bash
# Eliminar un stash específico
git stash drop stash@{<N>}

# Limpiar todos los stashes
git stash clear
```
**¿Qué hace?** Elimina stashes que ya no necesitas para mantener la lista ordenada. `clear` borra todos de golpe, así que úsalo con precaución.

---

## 5. Deshacer y corregir cambios

### Corregir el último commit (mensaje o contenido)
```bash
# Corregir solo el mensaje del último commit
git commit --amend -m "<mensaje>"

# Agregar archivos olvidados al último commit (sin cambiar el mensaje)
git add <archivo>
git commit --amend --no-edit
```
**¿Qué hace?** Reescribe el último commit. Útil cuando cometiste un error en el mensaje o te olvidaste de agregar un archivo. **Importante:** solo úsalo si aún no has hecho push, porque cambia el historial y causará conflictos si otros ya tienen ese commit.

---

### Sacar un archivo del staging area (sin perder cambios)
```bash
git restore --staged <archivo>
```
**¿Qué hace?** Si hiciste `git add` por error, este comando saca el archivo del staging area pero mantiene tus cambios en el working directory. Es como deshacer el `git add` sin perder tu trabajo.

---

### Descartar cambios de un archivo (volver al último commit)
```bash
git restore <archivo>
```
**¿Qué hace?** Descarta todos los cambios sin confirmar de un archivo y lo vuelve exactamente al estado del último commit. ⚠️ Esta operación es irreversible: los cambios descartados no se pueden recuperar.

---

### Revertir un commit de forma segura
```bash
git revert <commit-hash>
```
**¿Qué hace?** Crea un nuevo commit que deshace los cambios del commit indicado, sin borrar el historial. Es la forma segura de deshacer trabajo que ya fue subido al repositorio remoto, porque no reescribe el historial sino que agrega uno nuevo. Ideal para deshacer cambios en ramas compartidas o en producción.

---

### Deshacer commits con git reset
```bash
# Deshacer commits pero conservar los cambios en staging
git reset --soft HEAD~<N>

# Deshacer commits y sacar los cambios del staging (conserva archivos)
git reset --mixed HEAD~<N>

# Deshacer commits y eliminar todos los cambios (⚠️ irreversible)
git reset --hard HEAD~<N>
```
**¿Qué hace?** Mueve el puntero `HEAD` `N` commits hacia atrás. La diferencia está en qué pasa con tus archivos:
- `--soft`: los cambios quedan listos para commitear de nuevo (en staging).
- `--mixed`: los cambios quedan en tu working directory pero no en staging.
- `--hard`: borra todo, los archivos quedan exactamente como estaban en ese commit. ⚠️ Solo úsalo en ramas locales que no hayas compartido.

---

## 6. Cherry-pick — Copiar commits específicos

### Aplicar un commit específico en tu rama actual
```bash
git cherry-pick <commit-hash>
```
**¿Qué hace?** Toma los cambios de un commit específico (de cualquier rama) y los aplica como un nuevo commit en tu rama actual. Útil cuando necesitas pasar un fix puntual de una rama a otra sin hacer un merge completo. Por ejemplo, si un bug fue corregido en `development` y necesitas ese fix urgente en `release` sin mezclar todo lo demás.

---

### Aplicar varios commits en orden
```bash
git cherry-pick <commit-hash-1> <commit-hash-2> <commit-hash-3>
```
**¿Qué hace?** Aplica varios commits en el orden indicado. Es importante mantener el orden cronológico para evitar conflictos.

---

### Cherry-pick sin crear commit automáticamente
```bash
git cherry-pick <commit-hash> --no-commit
```
**¿Qué hace?** Aplica los cambios del commit en tu working directory pero no crea el commit automáticamente. Te da la oportunidad de revisar o modificar los cambios antes de confirmarlos. Útil cuando quieres combinar cambios de varios cherry-picks en un solo commit.

---

### Continuar o abortar un cherry-pick con conflictos
```bash
# Después de resolver los conflictos manualmente
git cherry-pick --continue

# Cancelar el cherry-pick y volver al estado anterior
git cherry-pick --abort
```

---

## 7. Blame — Rastrear autoría del código

### Ver quién escribió cada línea de un archivo
```bash
git blame <archivo>
```
**¿Qué hace?** Muestra el archivo línea por línea con información adicional al lado: el hash del commit, el autor y la fecha en que esa línea fue escrita o modificada por última vez. Útil para entender el contexto de un cambio o encontrar quién introdujo un bug específico.

---

### Ver blame de un rango de líneas específico
```bash
git blame -L <linea-inicio>,<linea-fin> <archivo>

# Ejemplo: ver solo las líneas 20 a 35
git blame -L 20,35 <archivo>
```
**¿Qué hace?** Limita el resultado a un rango de líneas. Útil cuando tienes archivos grandes y solo te interesa una sección particular.

---

### Ver blame ignorando cambios de espaciado
```bash
git blame -w <archivo>
```
**¿Qué hace?** Ignora commits que solo cambiaron espacios o indentación, mostrando el autor del contenido real. Evita que una reindentación masiva aparezca como el "autor" de todo el archivo.

---

## 8. Fetch vs Pull — Diferencias clave

Esta es una de las confusiones más comunes en Git. Ambos traen información del servidor remoto, pero actúan de manera muy diferente.

### git fetch — Descargar sin aplicar
```bash
git fetch origin

# Ver qué cambió después del fetch
git diff HEAD origin/<mi-rama>
```
**¿Qué hace?** Descarga todos los cambios del servidor remoto y los guarda localmente, pero **no modifica tu rama actual ni tu working directory**. Te permite revisar qué cambió el equipo antes de decidir si integras esos cambios. Es la opción más segura cuando quieres mantenerte informado sin alterar tu trabajo.

---

### git pull — Descargar y aplicar
```bash
git pull origin <mi-rama>
```
**¿Qué hace?** Es equivalente a hacer `git fetch` seguido de `git merge`. Descarga los cambios del servidor **y los integra inmediatamente** a tu rama actual. Puede generar conflictos si tu rama y la remota divergieron.

---

### Resumen de diferencias

| Acción | `git fetch` | `git pull` |
|---|---|---|
| Descarga cambios remotos | ✅ | ✅ |
| Modifica tu rama actual | ❌ | ✅ |
| Puede generar conflictos | ❌ | ✅ |
| Cuándo usarlo | Para revisar antes de integrar | Cuando confías en los cambios remotos |

---

## 9. Reflog — Historial de emergencia

El reflog es el "historial de todo". Registra cada movimiento de `HEAD`, incluyendo resets, rebases, merges y cambios de rama. Es tu red de seguridad cuando algo sale muy mal.

### Ver el historial completo de movimientos
```bash
git reflog
```
**¿Qué hace?** Muestra una lista cronológica de todo lo que ha pasado en tu repositorio local: cada commit, reset, merge, rebase o cambio de rama. Cada entrada tiene un índice como `HEAD@{0}`, `HEAD@{1}`, etc. A diferencia de `git log`, el reflog incluye incluso commits que fueron "borrados" con `git reset --hard`.

---

### Recuperar commits perdidos después de un reset --hard
```bash
# 1. Encontrar el hash del commit que perdiste
git reflog

# 2. Volver a ese punto
git reset --hard <commit-hash>

# O crear una nueva rama desde ese punto
git checkout -b <mi-rama-recuperada> <commit-hash>
```
**¿Qué hace?** Aunque hayas hecho `git reset --hard`, los commits no se borran inmediatamente del repositorio. El reflog los registra y puedes recuperarlos usando su hash. Es el salvavidas definitivo ante un error grave.

---

### Ver el reflog de una rama específica
```bash
git reflog show <mi-rama>
```
**¿Qué hace?** Filtra el reflog mostrando solo los movimientos que afectaron a esa rama en particular.

---

## 10. Resolución de conflictos

Los conflictos ocurren cuando dos personas modificaron la misma parte de un archivo en ramas diferentes. Git no puede decidir cuál versión es la correcta, así que te pide que lo hagas tú.

### ¿Cómo se ve un conflicto?
Cuando hay un conflicto, Git marca el archivo así:
```
<<<<<<< HEAD
  // Tu versión del código
=======
  // Versión del código de la otra rama
>>>>>>> <rama-origen>
```
- Todo entre `<<<<<<< HEAD` y `=======` es tu código.
- Todo entre `=======` y `>>>>>>>` es el código entrante.

---

### Flujo para resolver un conflicto

```bash
# Paso 1: Ver qué archivos tienen conflictos
git status

# Paso 2: Abrir cada archivo conflictivo, elegir qué código conservar
# y eliminar manualmente los marcadores <<<<<<<, =======, >>>>>>>

# Paso 3: Marcar el conflicto como resuelto
git add <archivo>

# Paso 4: Finalizar el merge
git commit
```

---

### Abortar el merge y volver al estado anterior
```bash
git merge --abort
```
**¿Qué hace?** Cancela el merge en curso y devuelve tu rama al estado exacto anterior al intento de merge. Útil cuando los conflictos son demasiados o necesitas más tiempo para revisarlos.

---

### Usar la versión de tu rama (ignorar los cambios entrantes)
```bash
git checkout --ours <archivo>
git add <archivo>
```

### Usar la versión de la rama entrante (ignorar tus cambios)
```bash
git checkout --theirs <archivo>
git add <archivo>
```
**¿Qué hacen?** En lugar de mezclar manualmente, estas opciones te permiten elegir una versión completa del archivo: `--ours` mantiene tu versión y `--theirs` acepta la versión que viene del merge.

---

## 11. .gitignore — Excluir archivos del repositorio

El archivo `.gitignore` le dice a Git qué archivos o carpetas debe ignorar completamente, como dependencias instaladas, archivos de configuración local o archivos generados automáticamente.

### Crear o editar el .gitignore
Crea un archivo llamado `.gitignore` en la raíz de tu proyecto y agrega los patrones de lo que quieres ignorar:

```gitignore
# Dependencias
node_modules/
vendor/

# Archivos de entorno y configuración local
.env
.env.local
.env.*.local

# Archivos generados por el sistema operativo
.DS_Store
Thumbs.db

# Carpetas de compilación o build
dist/
build/
*.min.js

# Logs
*.log
logs/

# IDEs y editores
.vscode/
.idea/
*.sublime-workspace
```

---

### Dejar de rastrear un archivo que ya fue commiteado
```bash
# Paso 1: Agregar el archivo a .gitignore
echo "<archivo>" >> .gitignore

# Paso 2: Eliminarlo del índice de Git (sin borrarlo de tu disco)
git rm --cached <archivo>

# Paso 3: Confirmar el cambio
git commit -m "chore: remove <archivo> from tracking"
```
**¿Qué hace?** Si accidentalmente commiteaste un archivo que debería estar ignorado (como un `.env`), este flujo lo elimina del control de versiones sin borrarlo de tu computadora.

---

### Ver qué archivos están siendo ignorados
```bash
git status --ignored
```
**¿Qué hace?** Muestra todos los archivos que Git está ignorando según las reglas del `.gitignore`. Útil para verificar que tus reglas están funcionando correctamente.

---

## 12. AWS S3 — Despliegue y sincronización

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
**¿Qué hace?** Muestra los archivos y carpetas dentro del bucket, similar al comando `ls` en terminal. Útil para verificar el estado actual antes de sincronizar.

---

### Simular una sincronización (dry run — sin cambios reales)
```bash
aws s3 sync "<carpeta-local>" <bucket> --dryrun
```
**¿Qué hace?** Ejecuta toda la lógica de sincronización pero sin aplicar ningún cambio real. Muestra exactamente qué archivos se subirían, actualizarían o eliminarían. **Siempre ejecuta esto primero** antes de una sincronización real para evitar subidas accidentales.

---

### Sincronizar carpeta local hacia S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --exact-timestamps
```
**¿Qué hace?** Compara los archivos locales con los del bucket y sube únicamente los que son nuevos o fueron modificados. El flag `--exact-timestamps` usa la fecha de modificación exacta para determinar si un archivo cambió, lo que es más preciso que comparar solo por tamaño.

---

### Sincronizar eliminando archivos sobrantes en S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --delete --exact-timestamps
```
**¿Qué hace?** Igual que la sincronización normal, pero además elimina del bucket cualquier archivo que no exista en tu carpeta local. Deja el bucket como un espejo exacto de tu carpeta.

> ⚠️ **Precaución:** Si tienes archivos en S3 que no están localmente (backups, archivos generados por otros procesos), este comando los eliminará permanentemente. Usa `--dryrun` antes para revisar qué se borrará.

---

### Subir un archivo puntual
```bash
aws s3 cp <archivo> <bucket>/<archivo>
```
**¿Qué hace?** Copia un único archivo desde tu máquina hacia una ruta específica dentro del bucket. A diferencia de `sync`, no compara ni verifica nada, simplemente sube ese archivo. Útil para actualizaciones urgentes de un solo archivo.

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
**¿Qué hace?** Elimina permanentemente un archivo específico del bucket. No hay confirmación ni papelera de reciclaje, debes estar seguro de la ruta antes de ejecutarlo.

---

### Eliminar una carpeta completa del bucket
```bash
aws s3 rm <bucket>/<carpeta> --recursive
```
**¿Qué hace?** Elimina una carpeta y todo su contenido dentro del bucket. El flag `--recursive` es obligatorio para operar sobre directorios; sin él, el comando no tendrá efecto.

---

### Validar identidad AWS activa
```bash
aws sts get-caller-identity
```
**¿Qué hace?** Consulta al servicio de AWS cuál es la identidad que está usando tu terminal en este momento. Devuelve el ID de cuenta, el nombre de usuario o rol, y el ARN completo. Es el primer comando a ejecutar si tienes dudas sobre con qué cuenta o entorno (dev, staging, producción) estás trabajando.
