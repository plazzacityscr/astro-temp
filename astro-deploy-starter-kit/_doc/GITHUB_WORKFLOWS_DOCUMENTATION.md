# Documentación: Archivos en .github/workflows

## Índice de Contenido

1. [deploy.yml](#1-deployyml)
2. [actions.yaml](#2-actionsyaml)
3. [deploy-agent.sh](#3-deploy-agentsh)
4. [Resumen Comparativo](#resumen-comparativo)

---

## 1. deploy.yml

**Ubicación**: `.github/workflows/deploy.yml`

### Propósito
Workflow automatizado que Compila y Despliega la aplicación Astro a **GitHub Pages** cada vez que se hace un push a la rama `main`.

### Cuándo se ejecuta
- ✅ Al hacer `push` a la rama `main`
- ✅ Al ejecutar manualmente desde la interfaz de GitHub Actions (`workflow_dispatch`)

### Permisos requeridos
```yaml
permissions:
  contents: read       # Leer contenido del repo
  pages: write         # Escribir en GitHub Pages
  id-token: write      # Generar tokens de identidad
```

### Flujo de ejecución

#### **Job: build**
Ejecuta el proceso de compilación:

| Paso | Acción | Descripción |
|------|--------|-------------|
| 1 | `Checkout` | Clona el código del repositorio |
| 2 | `Setup Node` | Instala Node.js v20 con caché de npm |
| 3 | `Install dependencies` | Ejecuta `npm ci` (instalación limpia) |
| 4 | `Build` | Compila el proyecto con `npm run build` |
| 5 | `Upload artifact` | Sube la carpeta `./dist` (compilada) como artefacto |

#### **Job: deploy**
Despliega el artefacto a GitHub Pages:

| Paso | Acción | Descripción |
|------|--------|-------------|
| 1 | `Deploy to GitHub Pages` | Usa `actions/deploy-pages@v4` para publicar el sitio |
|   | Variable de salida | Genera `steps.deployment.outputs.page_url` |

### Configuración de concurrencia
```yaml
concurrency:
  group: 'pages'
  cancel-in-progress: false  # No cancela deploys en progreso
```

### Resultado esperado
✅ El sitio compilado se publica en:
```
https://{usuario}.github.io/{nombre-del-repo}/
```

---

## 2. actions.yaml

**Ubicación**: `.github/workflows/actions.yaml`

### Propósito
Workflow de **CI (Integración Continua)** que verifica la compilación y quality checks en cada pull request y push a `main`. Valida que el código funciona en múltiples versiones de Node.js.

### Cuándo se ejecuta
- ✅ En cada `pull_request` dirigido a `main`
- ✅ En cada `push` a la rama `main`

### Flujo de ejecución

#### **Job: build**
Compila el proyecto en múltiples versiones de Node.js:

| Versión Node | Acción |
|--------------|--------|
| 18 | Compila y verifica |
| 20 | Compila y verifica |
| 22 | Compila y verifica |

**Pasos ejecutados**:
1. `Checkout` - Clona el código
2. `Setup Node` - Instala la versión específica (matriz)
3. `npm ci` - Instalación limpia de dependencias
4. `npm run build` - Compila el proyecto
5. *(Comentado: `npm test` - Pruebas unitarias)*

**Objetivo**: Asegurar que el código compila correctamente en diferentes versiones de Node.js.

#### **Job: check**
Ejecuta verificaciones de código (linting, tipos, etc.):

| Paso | Acción |
|------|--------|
| 1 | `Checkout` - Clona el código |
| 2 | `Setup Node` - Instala Node.js v22 |
| 3 | `npm ci` - Instalación limpia |
| 4 | `npm run check` - Ejecuta verificaciones (ESLint, TypeScript, etc.) |

**Objetivo**: Valida la calidad del código (sintaxis, tipos estáticos, formato).

### ¿Cuándo falla?
❌ Si el código **no compila** en cualquiera de las versiones de Node.js (18, 20, 22)  
❌ Si **falla la verificación** (`npm run check`) por errores de tipo o linting

---

## 3. deploy-agent.sh

**Ubicación**: `.github/workflows/deploy-agent.sh`

### Propósito
**Script informativo** que proporciona información sobre el estado del despliegue actual y comandos útiles para monitorear los workflows de GitHub Actions. Es una **herramienta de gestión**, no parte del workflow automatizado.

### Cuándo se usa
- 🔧 Ejecutado **manualmente** por el usuario desde la terminal
- 📊 Para obtener información del estado de despliegues
- 📋 Para acceder a enlaces y comandos útiles

### Requisitos
- ✅ **GitHub CLI** (`gh`) debe estar instalado
- ✅ **Autenticación activa** con GitHub (`gh auth login`)
- ✅ Estar dentro de un repositorio de GitHub

### Funcionalidades principales

#### 1. **Validación de autenticación**
```bash
Verifica que:
- GitHub CLI esté instalado
- Tengas sesión activa en GitHub
```

#### 2. **Enlaces útiles que muestra**
```
📁 Repositorio: https://github.com/{owner}/{repo}
🔄 GitHub Actions: https://github.com/{owner}/{repo}/actions
🌐 Sitio Desplegado: https://{owner}.github.io/{repo}/
```

#### 3. **Últimos despliegues**
Lista los 5 últimos runs del workflow con:
- ID del run
- Estado actual (success, failure, in progress)
- Conclusión

#### 4. **Comandos útiles que enseña**
```bash
Ver últimos 5 runs:
  gh run list --repo {owner/repo} --limit 5

Ver logs del último run:
  gh run view --repo {owner/repo} --log --latest

Ver un run específico:
  gh run view <RUN_ID> --repo {owner/repo} --log

Ver workflows disponibles:
  gh workflow list --repo {owner/repo}
```

#### 5. **Instrucciones para desplegar**
Muestra cómo disparar un nuevo despliegue:
```bash
git add -A
git commit -m "tus cambios"
git push
```

### Resultado esperado
📋 Información clara sobre:
- ✅ Estado del despliegue actual
- 📌 Enlaces importantes
- 📝 Últimos despliegues realizados
- 🔗 Comandos para monitorear el workflow

---

## Resumen Comparativo

| Archivo | Tipo | Propósito | Cuándo se ejecuta | Usuario |
|---------|------|----------|------------------|---------|
| **deploy.yml** | Workflow YAML | Compilación + despliegue a GitHub Pages | Al push a main / Manual | Automático |
| **actions.yaml** | Workflow YAML | CI - Compilación multi-versión + quality checks | En PullRequest y push a main | Automático |
| **deploy-agent.sh** | Script Bash | Información y monitoreo de despliegues | Manual (usuario ejecuta) | Manual |

### Flujo integrado

```
┌─────────────────────────────────────┐
│   Usuario hace: git push             │
└────────────────┬────────────────────┘
                 │
        ┌────────▼────────┐
        │ actions.yaml    │
        │ - Build (Node   │
        │   18, 20, 22)   │
        │ - Check (lint)  │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │ deploy.yml      │
        │ - Build         │
        │ - Deploy a GH   │
        │   Pages         │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │ Sitio en vivo    │
        │ https://...      │
        └─────────────────┘
        
Usuario puede monitorear con:
- deploy-agent.sh (manual)
- gh run list (CLI)
- GitHub UI (Acciones)
```

### Resumen de responsabilidades

| Workflow | Responsabilidad |
|----------|-----------------|
| **actions.yaml** | ✅ Validar que el código **funciona** y es de **calidad** |
| **deploy.yml** | ✅ Compilar y publicar el sitio en **producción** |
| **deploy-agent.sh** | ✅ Facilitar **monitoreo** y **gestión** de despliegues |

---

**Última actualización**: 31 de Marzo, 2026
