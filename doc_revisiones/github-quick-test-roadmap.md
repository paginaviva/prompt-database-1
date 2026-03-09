# Quick Deployment Roadmap - GitHub Testing

**Objetivo:** Probar el proyecto en funcionamiento en menos de 1 hora usando GitHub.

---

## 1. Preparación del Repositorio (10 minutos)

### Archivos requeridos

**Crear `.env.example`:**
```bash
DATABASE_URL="file:./dev.db"
NODE_ENV=development
NEXT_PUBLIC_BASE_PATH=/prompt-database
```

**Verificar `.gitignore`:**
```
/node_modules
/.next
/data
*.db
.env
```

### Commit inicial
```bash
git add .
git commit -m "Prepare for GitHub testing"
git push origin main
```

---

## 2. GitHub Codespaces - Entorno Temporal de Prueba (20 minutos)

GitHub Codespaces permite ejecutar el proyecto en un contenedor temporal accesible desde el navegador.

### Paso 1: Habilitar Codespaces

1. Ir al repositorio en GitHub
2. Click en **"Code"** (botón verde)
3. Pestaña **"Codespaces"**
4. Click **"Create codespace on main"**

### Paso 2: Configurar Codespace

GitHub detecta automáticamente Next.js. Verificar que `.devcontainer/devcontainer.json` exista o crearlo:

```json
{
  "name": "Prompt Database",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "forwardPorts": [3000],
  "postCreateCommand": "npm install && npm run db:generate"
}
```

### Paso 3: Iniciar la Aplicación

En la terminal del Codespace:

```bash
# Instalar dependencias
npm install

# Generar Prisma Client
npm run db:generate

# Iniciar servidor de desarrollo
npm run dev
```

### Paso 4: Acceder a la Aplicación

1. En Codespaces, ir a pestaña **"Ports"**
2. Click en el link de **port 3000**
3. La aplicación se abre en el navegador

**URL temporal:** `https://<codespace-id>-3000.app.github.dev/prompt-database`

---

## 3. GitHub Actions - Build Verification (15 minutos)

Para verificar que el proyecto compila correctamente:

### Crear `.github/workflows/test-build.yml`:

```yaml
name: Test Build

on: [workflow_dispatch]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate Prisma Client
        run: npm run db:generate
      
      - name: Build
        run: npm run build
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: .next/standalone
          retention-days: 1
```

### Ejecutar el workflow:

1. Ir a **Actions** en el repositorio
2. Seleccionar **"Test Build"**
3. Click **"Run workflow"**
4. Esperar a que complete (5-10 minutos)
5. Descargar artifact para verificar el build

---

## 4. Verificación Rápida (10 minutos)

### Checklist de funcionalidades

| Test | Cómo verificar | Resultado |
|------|----------------|-----------|
| **Página carga** | Abrir URL de Codespaces | ☐ |
| **Lista de prompts** | Navegar a `/prompts` | ☐ |
| **Crear prompt** | Click "New Prompt", guardar | ☐ |
| **Buscar** | Usar barra de búsqueda | ☐ |
| **API funciona** | `curl <URL>/api/prompts` | ☐ |

### Verificar logs

En Codespaces, terminal:
```bash
# Ver logs del servidor
docker-compose logs -f app
```

### Verificar base de datos

```bash
# Chequear que SQLite se creó
ls -la dev.db

# Ver tablas
npx prisma studio
```

---

## 5. Limitaciones de esta Prueba

| Limitación | Impacto |
|------------|---------|
| **Codespaces se apaga** | Después de 30 min de inactividad |
| **Datos temporales** | Se pierden al cerrar Codespace |
| **Acceso limitado** | Solo quien tiene el link de Codespace |
| **Recursos limitados** | 2 cores, 4GB RAM (free tier) |

---

## Resumen - Tiempo Total: ~45 minutos

| Paso | Tiempo |
|------|--------|
| Preparar repositorio | 10 min |
| Configurar Codespaces | 5 min |
| Iniciar aplicación | 10 min |
| Crear GitHub Actions | 10 min |
| Verificación | 10 min |
| **Total** | **45 min** |

---

## Enlaces Útiles

- **Codespaces:** https://github.com/features/codespaces
- **GitHub Actions:** https://github.com/features/actions
- **Documentación Next.js:** https://nextjs.org/docs

---

**Fin del documento**
