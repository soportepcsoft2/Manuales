# Cheatsheet: Subir archivos a GitHub

## ⚠️ Regla de oro antes de empezar

**No crees el README (ni ningún archivo) desde la web de GitHub si vas a subir un proyecto que ya tienes en tu computadora.** Si lo haces, el repositorio remoto queda con un commit que tu carpeta local no tiene, y terminas con conflictos de push innecesarios (`non-fast-forward`). Crea el repositorio **vacío**, sin README, sin `.gitignore`, sin licencia.

---

## Caso 1: Primera vez subiendo una carpeta local (repo nuevo y vacío)

Usa esto cuando: creaste el repositorio en GitHub, está vacío, y tienes archivos en tu PC que quieres subir por primera vez.

```powershell
# 1. Entra a la carpeta que quieres subir
cd C:\ruta\a\tu\carpeta

# 2. Inicializa git en esa carpeta
git init

# 3. Agrega todos los archivos al área de preparación (staging)
git add .

# 4. Crea el primer commit
git commit -m "Primer commit"

# 5. Renombra la rama a main (si no lo está ya)
git branch -M main

# 6. Conecta tu carpeta local con el repositorio remoto
#    ⚠️ IMPORTANTE: agrega tu usuario ANTES de github.com para evitar
#    conflictos de credenciales si manejas varias cuentas de GitHub
git remote add origin https://TU_USUARIO@github.com/TU_USUARIO/NOMBRE_REPO.git

# 7. Sube los archivos
git push -u origin main
```

**El detalle que no debes olvidar:** `https://TU_USUARIO@github.com/...` — no `https://github.com/...` a secas. Ese `@` con tu usuario antes del dominio hace que Git guarde la credencial separada por cuenta, en vez de pisar la credencial de otra cuenta que ya tengas guardada en Windows.

---

## Caso 2: El repositorio ya existe y tiene contenido (clonarlo)

Usa esto cuando: el repo ya existe en GitHub (con o sin archivos) y quieres traerlo a tu computadora para trabajar en él.

```powershell
# 1. Ve a la carpeta donde quieres que viva el repo (un nivel arriba de donde
#    quieres que se cree la carpeta del proyecto)
cd C:\ruta\donde\quieres\clonarlo

# 2. Clona usando tu usuario antes de github.com (mismo detalle que arriba)
git clone https://TU_USUARIO@github.com/TU_USUARIO/NOMBRE_REPO.git
```

Esto crea automáticamente una carpeta con el nombre del repo, ya conectada al remoto — no necesitas `git init` ni `git remote add`, eso ya viene incluido.

**Si la carpeta destino ya existe y tiene archivos:**
```powershell
# Opción A: clonar directamente dentro de la carpeta actual (debe estar vacía)
git clone https://TU_USUARIO@github.com/TU_USUARIO/NOMBRE_REPO.git .

# Opción B: clonar con otro nombre para no chocar
git clone https://TU_USUARIO@github.com/TU_USUARIO/NOMBRE_REPO.git NOMBRE_CARPETA_NUEVA
```

**Después de clonar, tu flujo de trabajo normal es:**
```powershell
git add .
git commit -m "Descripción del cambio"
git push
```

---

## Caso 3: Creaste un repositorio por error (o quedó anidado)

### 3a. Quitar Git de una carpeta por completo (deshacer `git init`)

Si corriste `git init` en una carpeta que no debía ser un repositorio (o en la carpeta padre equivocada), esto **no borra tus archivos**, solo quita el control de versiones:

```powershell
Remove-Item -Recurse -Force .git
```

Ejecútalo estando parado **dentro** de la carpeta que quieres "des-convertir" en repo.

### 3b. Repo anidado dentro de otro repo (el error de "se subió la carpeta pero no el contenido")

Esto pasa cuando tienes una carpeta con su propio `.git` **dentro** de otra carpeta que también es un repo. Git trata la carpeta interna como un submódulo (una cáscara vacía) en vez de subir su contenido.

**Para diagnosticarlo:**
```powershell
# Estando en la carpeta padre, revisa si tiene su propio repo
dir .git

# Revisa si la subcarpeta también tiene uno
dir NOMBRE_SUBCARPETA\.git
```

Si ambas tienen `.git`, ahí está el problema.

**Para arreglarlo**, decide cuál debe ser el repo real (normalmente la subcarpeta específica, no la carpeta padre general) y quita el `.git` sobrante:

```powershell
# Si la carpeta PADRE se convirtió en repo por accidente, quítale el .git
cd C:\ruta\a\la\carpeta\padre
Remove-Item -Recurse -Force .git
```

Después, trabaja siempre parado dentro de la subcarpeta correcta (la que sí debe ser el repo) para tus `add`/`commit`/`push`.

---

## Errores comunes y solución rápida

| Error | Causa típica | Solución |
|---|---|---|
| `repository not found` | Repo privado sin autenticación válida | Usa la URL con `TU_USUARIO@` antes de `github.com`, o revisa que el token tenga scope `repo` |
| `non-fast-forward` / `rejected` | El remoto tiene commits que tú no tienes localmente (ej. README creado desde la web) | `git pull origin main` antes de volver a hacer `push` |
| `destination path already exists` | Ya existe una carpeta con ese nombre | Usa `git clone URL .` (con punto, si está vacía) o clona con otro nombre de carpeta |
| Se sube la carpeta pero no el contenido | Repo anidado (submódulo accidental) | Ver Caso 3b arriba |
| Pide contraseña y la rechaza | Estás usando tu contraseña real de GitHub en vez de un token | Genera un Personal Access Token (Settings → Developer settings → Personal access tokens) y úsalo como contraseña |

---

## Checklist mental antes de cualquier subida

- [ ] ¿El repo remoto está vacío? → No crear README/archivos desde la web antes de tu primer push
- [ ] ¿Estoy usando `usuario@github.com` en la URL? → Evita conflictos de credenciales entre cuentas
- [ ] ¿Estoy parado en la carpeta correcta? → Verifica con `dir` o `pwd` antes de `git init` o `git add .`
- [ ] ¿Ya existe una carpeta con ese nombre? → Revisa antes de clonar para no chocar
