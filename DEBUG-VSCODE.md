# Debug de la aplicación Spring Boot en VS Code

Guía para debuggear esta API (`articulos-api`) usando el debugger de VS Code.

> **Importante:** este proyecto lee la conexión de **variables de entorno**
> (`DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `DB_SSL_MODE`). 

---

## 1. Extensiones necesarias

En VS Code (`Ctrl+Shift+X` para abrir el panel de extensiones) instalá:

- **Extension Pack for Java** (Microsoft) — incluye el debugger de Java. **Imprescindible.**
- **Spring Boot Extension Pack** — soporte específico para Spring Boot (opcional pero recomendado).

---

## 2. Requisitos previos (la base de datos)

El debugger arranca la app y necesita una base de datos accesible. Hay dos opciones, una 
por cada configuración de debug:

### a) MySQL local

- **MySQL corriendo** en `localhost:3306`.
- La base **`articulos_db` creada**:
  ```sql
  CREATE DATABASE IF NOT EXISTS articulos_db;
  ```
- **SSL desactivado**: un MySQL local no tiene TLS, por eso la configuración local usa
  `DB_SSL_MODE=DISABLED` (el valor por defecto de la app es `VERIFY_IDENTITY`).

### b) TiDB Cloud

- Los datos de conexión del cluster (**Connect > Connect with .env**).
- No hay que modificar SSL (default `VERIFY_IDENTITY`)


---

## 3. Prueba rápida (sin configurar nada)

1. Abrí `src/main/java/com/ejemplo/articulos/ArticuloApiApplication.java`.
2. Arriba del método `main` aparece un enlace **`Run | Debug`** (un *CodeLens*).
3. Hacé click en **Debug**.

---

## 4. Debug con `launch.json`

**Importante**: No hagas commit de este archivo si tiene tus claves

`.vscode/launch.json` define **dos** configuraciones:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Debug (MySQL local)",
      "request": "launch",
      "mainClass": "com.ejemplo.articulos.ArticuloApiApplication",
      "projectName": "articulos-api",
      "console": "integratedTerminal",
      "env": {
        "DB_HOST": "localhost",
        "DB_PORT": "3306",
        "DB_DATABASE": "articulos_db",
        "DB_USERNAME": "root",
        "DB_PASSWORD": "****",
        "DB_SSL_MODE": "DISABLED"
      }
    },
    {
      "type": "java",
      "name": "Debug (TiDB Cloud)",
      "request": "launch",
      "mainClass": "com.ejemplo.articulos.ArticuloApiApplication",
      "projectName": "articulos-api",
      "console": "integratedTerminal",
      "env": {
        "DB_HOST": "gateway01.<region>.prod.aws.tidbcloud.com",
        "DB_PORT": "4000",
        "DB_DATABASE": "articulos_db",
        "DB_USERNAME": "<prefijo>.root",
        "DB_PASSWORD": "****"
      }
    }
  ]
}
```

Para usarlo:

1. Abrí el panel **Run and Debug** (`Ctrl+Shift+D`).
2. Elegí **"Debug (MySQL local)"** o **"Debug (TiDB Cloud)"**.
3. Click al ícono ▶ verde (o `F5`).
