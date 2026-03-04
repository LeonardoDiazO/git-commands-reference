# Git & AWS S3 — Guía de referencia técnica

> **Documentación oficial:** https://www.atlassian.com/es/git/tutorials/setting-up-a-repository/git-clone

---

## Convenciones usadas en esta guía

| Placeholder | Descripción |
|---|---|
| `<destino>` | Ruta absoluta o relativa donde se copiarán los archivos |
| `<rama-base>` | Rama de referencia (ej: `main`, `master`, `development`) |
| `<mi-rama>` | Tu rama de trabajo (ej: `fix/TICKET-123`) |
| `<release>` | Rama de release (ej: `release/v1.0.0`) |
| `<commit-hash>` | Hash SHA del commit específico |
| `<N>` | Número entero (cantidad de commits, horas, días, etc.) |
| `<bucket>` | Nombre del bucket S3 (ej: `s3://mi-bucket`) |
| `<archivo>` | Ruta del archivo a operar |

---

## 1. Consulta y comparación de cambios

### Ver cambios sin confirmar en el working directory
```bash
git diff
```
Muestra las diferencias de todos los archivos modificados que aún no han sido commiteados.

---

### Ver archivos modificados en el último commit
```bash
git diff --name-only HEAD~1 HEAD
```
Lista únicamente los nombres de los archivos que cambiaron en el commit más reciente.

---

### Ver archivos diferentes entre dos ramas
```bash
git diff --name-only <rama-base>..<release>
```
Muestra qué archivos difieren entre dos ramas cualesquiera.

---

### Comparar tu rama de trabajo contra la rama base
```bash
git diff <rama-base>...<mi-rama> --name-only
```
Lista todos los archivos que difieren entre `<rama-base>` y `<mi-rama>`.

---

### Ver archivos modificados en los últimos `<N>` commits
```bash
git diff --name-only HEAD~<N>..HEAD
```
Ejemplo para los últimos 6 commits:
```bash
git diff --name-only HEAD~6..HEAD
```

---

### Ver archivos modificados en el último día en una rama
```bash
git diff --name-only $(git rev-list -n 1 --before="1 days ago" <release>)..<release>
```
Muestra los archivos que cambiaron durante las últimas 24 horas en la rama `<release>`.

---

### Ver el historial completo de un archivo (incluyendo renombrados)
```bash
git log --follow -- <archivo>
```
Muestra todos los commits que afectaron a `<archivo>`, incluso si fue renombrado o movido de directorio en algún punto del historial. Sin `--follow`, Git detiene el historial en el punto donde se renombró.

**Variantes útiles:**
```bash
# Mostrar también el diff de cada commit que tocó el archivo
git log --follow -p -- <archivo>

# Formato compacto: hash corto + mensaje
git log --follow --oneline -- <archivo>
```

---

### Comparar un commit específico contra su commit padre
```bash
git diff <commit-hash>^ <commit-hash> --name-only
```
Útil para inspeccionar exactamente qué cambió en un commit en particular.

---

## 2. Copia de archivos modificados

> Todos los comandos de copia usan `rsync -R` para **preservar la estructura de directorios** del proyecto.

### Copiar archivos que difieren entre tu rama y la base
```bash
for name in $(git diff <rama-base>...<mi-rama> --name-only); do
  rsync -R "$name" "<destino>"
done
```

---

### Copiar archivos modificados en el último commit
```bash
for name in $(git diff --name-only HEAD~1..HEAD); do
  rsync -R "$name" "<destino>"
done
```

---

### Copiar archivos modificados en los últimos `<N>` commits (sin duplicados)
```bash
for name in $(git diff --name-only HEAD~<N>..HEAD | sort -u); do
  rsync -R "$name" "<destino>"
done
```

---

### Copiar archivos modificados en las últimas `<N>` horas
```bash
for name in $(git diff HEAD "@{<N>.hours.ago}" --name-only); do
  rsync -R "$name" "<destino>"
done
```

---

### Copiar archivos agregados o modificados desde medianoche
```bash
for name in $(git log --since="midnight" --pretty=format: --name-only --diff-filter=AM <rama-base>); do
  rsync -R "$name" "<destino>"
done
```
El flag `--diff-filter=AM` filtra solo archivos **A**gregados y **M**odificados, excluyendo eliminados.

---

## 3. Flujo de trabajo con ramas

### Buenas prácticas antes de crear una rama

```bash
# 1. Actualizar las ramas base
git checkout <rama-base> && git pull
git checkout development && git pull

# 2. Crear tu rama de trabajo
git checkout -b <mi-rama>
```

---

### Actualizar rama local cuando hay cambios remotos pendientes
```bash
git pull --rebase --autostash
git push
```
Aplica los cambios remotos primero y luego tus commits locales encima, sin perder trabajo sin confirmar.

---

### Forzar un push de forma segura
```bash
git push --force-with-lease
```
Fuerza el push solo si nadie más ha subido cambios desde tu último `pull`. Más seguro que `--force`.

---

## 4. AWS S3 — Despliegue y sincronización

### Ver buckets disponibles
```bash
aws s3 ls
```

---

### Listar contenido de un bucket
```bash
aws s3 ls <bucket>
```

---

### Simular una sincronización (dry run — sin cambios reales)
```bash
aws s3 sync "<carpeta-local>" <bucket> --dryrun
```
Muestra qué archivos se subirían o actualizarían, sin ejecutar ningún cambio.

---

### Sincronizar carpeta local hacia S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --exact-timestamps
```
Sube y actualiza archivos comparando timestamps exactos.

---

### Sincronizar eliminando archivos sobrantes en S3
```bash
aws s3 sync "<carpeta-local>" <bucket> --delete --exact-timestamps
```
> ⚠️ **Precaución:** deja el bucket idéntico a la carpeta local, **eliminando** archivos remotos que no existan localmente.

---

### Subir un archivo puntual
```bash
aws s3 cp <archivo> <bucket>/<archivo>
```

---

### Descargar contenido de un bucket a local
```bash
aws s3 sync <bucket> <carpeta-local>
```

---

### Eliminar un archivo del bucket
```bash
aws s3 rm <bucket>/<archivo>
```

---

### Eliminar una carpeta completa del bucket
```bash
aws s3 rm <bucket>/<carpeta> --recursive
```

---

### Validar identidad AWS activa
```bash
aws sts get-caller-identity
```
Verifica la cuenta, el usuario y el ARN configurados en el perfil AWS activo.
