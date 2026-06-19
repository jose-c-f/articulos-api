# API REST (Spring Boot) en Render + TiDB

Este proyecto es un ejemplo de los ajustes que se deben realizar para poder deployar 
una API REST de Spring Boot en [Render](https://render.com) usando MySQL en
[TiDB Cloud](https://tidbcloud.com).

---

## 1. Introducción

El objetivo es que **Render** construya y ejecute la app desde un contenedor Docker, y que
la base de datos sea **TiDB Cloud**.

### 1.1 Repositorio
Vamos a configurar **Render** para que lea el código de nuestra aplicación de un repositorio
público.  Para esto vamos a subir el código a **GitHub**.

### 1.2 Credenciales
Render requiere que las credenciales estén en variables
de entorno las cuales se configuran desde la consola de Render.  De esa forma en los
archivos de configuración solo quedan los nombres de las variables de entorno.

### 1.3 pom.xml
Para el build del proyecto vamos a utilizar Maven, por lo tanto debe haber un archivo 
pom.xml.  El resultado del build son todas las clases compiladas y archivos de proyecto
empaquetados en un archivo JAR (formato zip para aplicaciones Java).

### 1.4 Dockerfile
Además es necesario que el proyecto esté "dockerizado", o sea, debe contener un archivo
de Docker para el deploy y arranque de la aplicación, el archivo Dockerfile.  
Este archivo indica la imagen de sistema operativo a utilizar, el directorio de trabajo,
la versión de Java, y como Docker por defecto cierra todos los puertos en el Dockerfile se 
debe documentar qué número de puerto se va a utilizar.

### 1.5 Base de datos
Vamos a utilizar una en la nube.  Para esto vamos a configurar una base de datos en TiDB 
y en Render vamos a configurar las variables de conexión correspondientes.


---

## 2. Configuración

### 2.1 `application.properties` — configuración externalizada

Originalmente los datos de conexión estaban en el archivo de configuración properties. 
Ahora se leen de variables de entorno, con valores por defecto para desarrollo local:

```properties
server.port=${PORT:8080}
spring.datasource.url=jdbc:mysql://${DB_HOST:localhost}:${DB_PORT:3306}/${DB_DATABASE:articulos_db}?sslMode=${DB_SSL_MODE:VERIFY_IDENTITY}&enabledTLSProtocols=TLSv1.2,TLSv1.3
spring.datasource.username=${DB_USERNAME:root}
spring.datasource.password=${DB_PASSWORD:}
spring.jpa.hibernate.ddl-auto=update
```

- **`server.port=${PORT:8080}`** — Render asigna el puerto a través de la variable
  `PORT`; la app debe escuchar en ese puerto. En local cae a `8080`.
- **`DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD`** — son las
  mismas variables que TiDB Cloud te da en **Connect > Connect with .env**. La app arma
  la URL JDBC a partir de ellas, así que copiás los valores tal cual sin tener que
  construir la URL a mano.
- **`DB_SSL_MODE`** — controla el cifrado. Por defecto vale `VERIFY_IDENTITY`, que es lo
  que exige TiDB Cloud, así que en Render **no hace falta definirla**. Si alguna vez
  apuntás a un MySQL local sin SSL, poné `DB_SSL_MODE=DISABLED`.
- **`ddl-auto=update`** — Hibernate crea/actualiza la tabla `articulo`
  automáticamente en el primer arranque, así que **no hace falta crear el esquema a
  mano** en TiDB.

### 2.2 `Dockerfile` — build multi-etapa

Render construye la imagen desde este Dockerfile. Es *multi-stage*:

1. **Etapa build:** usa la imagen `maven:3.9-eclipse-temurin-17` para compilar el jar
   (`mvn package -DskipTests`). Gracias a esto **no se necesita Maven instalado**, ni
   en local ni en Render.
2. **Etapa de ejecución:** copia solo el jar a una imagen ligera
   `eclipse-temurin:17-jre-alpine` y lo arranca.

> El nombre del jar (`articulos-api-1.0.0.jar`) debe coincidir con el
> `artifactId` y `version` del `pom.xml`. Si cambias alguno, actualiza el Dockerfile.

### 2.3 `.gitignore` y `.dockerignore`

- **`.gitignore`** — evita subir `target/` y archivos de IDE al repositorio (Render
  despliega desde Git).
- **`.dockerignore`** — excluye `target/`, `.git`, etc. del contexto de build para que
  la imagen se construya más rápido.

---

## 3. Pasos para el deploy

### Paso 1 — Crear la base de datos en TiDB Cloud

1. Entrá en https://tidbcloud.com y creá un cluster **Serverless** (tiene capa
   gratuita).
2. En el cluster, hacé click en el botón **Connect** (arriba a la derecha) y elegí
   **Connect With > .env**. TiDB te muestra los valores listos para copiar:
   ```env
   DB_HOST=gateway01.<region>.prod.aws.tidbcloud.com
   DB_PORT=4000
   DB_USERNAME='<prefijo>.root'
   DB_PASSWORD='<tu-password>'
   DB_DATABASE='articulos_db'
   ```
   Anotalos: los vas a cargar tal cual en Render (Paso 4). Las comillas que muestra TiDB
   son sintaxis del archivo `.env`; en el panel de Render se pegan **sin** comillas.
3. Creá la base de datos `articulos_db`. Desde la consola SQL de TiDB (SQL Editor):
   ```sql
   CREATE DATABASE articulos_db;
   ```
   (La tabla `articulo` la creará Hibernate sola al arrancar la app.)

### Paso 2 — Subir el código a un repositorio Git

Render despliega desde GitHub/GitLab. Subí el proyecto `articulos-api` a un
repositorio público (debe incluir `Dockerfile`, `pom.xml` y `src/`).

### Paso 3 — Crear el servicio en Render

1. En Render, andá a **New > Web Service** y conectá el repositorio.
2. En **Runtime** elegí **Docker** (Render usará el `Dockerfile`).
3. En **Health Check Path** indicá `/api/articulos`.

### Paso 4 — Configurar las variables de entorno en Render

En la sección **Environment** del servicio, definí:

| Variable      | Valor |
|---------------|-------|
| `DB_HOST`     | el host de TiDB (ej. `gateway01.us-east-1.prod.aws.tidbcloud.com`) |
| `DB_PORT`     | `4000` |
| `DB_DATABASE` | `articulos_db` |
| `DB_USERNAME` | el usuario de TiDB (ej. `xxxxxxxx.root`) |
| `DB_PASSWORD` | la contraseña de TiDB |

Son los mismos valores del `.env` que copiaste en el Paso 1 (cambiando `DB_DATABASE` a
`articulos_db`). La app arma la URL JDBC con ellos automáticamente.

> **Importante:** TiDB Cloud exige conexión cifrada. El modo SSL viene por defecto en
> `VERIFY_IDENTITY` (variable `DB_SSL_MODE`), así que **no hace falta cargarlo** en
> Render. El driver MySQL valida el certificado contra los CA del sistema, que ya vienen
> incluidos en la imagen `eclipse-temurin`.

> **Health check:** asegurate de haber fijado el **Health Check Path** a
> `/api/articulos` en los ajustes del servicio (Paso 3.3). Render consulta esa ruta
> para saber si la app está viva.

### Paso 5 — Desplegar y verificar

1. Render compilará la imagen (etapa Maven → jar → imagen JRE) y arrancará el
   contenedor. El primer build tarda unos minutos.
2. Cuando el estado sea **Live**, probá la API en la URL pública que da Render:
   ```bash
   # Listar artículos (al inicio devuelve [] )
   curl https://<tu-servicio>.onrender.com/api/articulos

   # Crear un artículo
   curl -X POST https://<tu-servicio>.onrender.com/api/articulos \
        -H "Content-Type: application/json" \
        -d '{"nombre":"Teclado","precio":29.99,"imagen":"teclado.png"}'
   ```

---

## 3.bis — Probar la conexión a TiDB en local (antes de desplegar)

Conviene verificar que las credenciales de TiDB funcionan **desde tu máquina** antes
de subir nada a Render. Así separás un problema de credenciales/red de un problema de
configuración de Render.

> **Debug en VS Code:** para depurar la app con breakpoints (contra MySQL local o TiDB),
> mirá [DEBUG-VSCODE.md](DEBUG-VSCODE.md).

### Opción A — Arrancar la app apuntando a TiDB

Definí las mismas variables de entorno que vas a usar en Render y arrancá la app. En
**PowerShell** (Windows):

```powershell
$env:DB_HOST = "gateway01.<region>.prod.aws.tidbcloud.com"
$env:DB_PORT = "4000"
$env:DB_DATABASE = "articulos_db"
$env:DB_USERNAME = "xxxxxxxx.root"
$env:DB_PASSWORD = "tu-password"
mvn spring-boot:run
```

En **Linux/macOS** (bash):

```bash
export DB_HOST="gateway01.<region>.prod.aws.tidbcloud.com"
export DB_PORT="4000"
export DB_DATABASE="articulos_db"
export DB_USERNAME="xxxxxxxx.root"
export DB_PASSWORD="tu-password"
mvn spring-boot:run
```

Si arranca sin errores y en los logs aparece que Hibernate creó/validó la tabla
`articulo`, la conexión funciona. Probala en otra terminal:

```bash
curl http://localhost:8080/api/articulos
```

> Las variables de entorno solo viven en esa ventana de terminal. Al cerrarla se
> borran (no quedan guardadas en el sistema).

### Opción B — Probar solo la conexión con el cliente MySQL

Si tenés el cliente `mysql` instalado, podés comprobar la conexión sin arrancar la
app (útil para descartar problemas de credenciales o firewall). Ojo: acá el host y
puerto van separados, **sin** el prefijo `jdbc:mysql://`:

```bash
mysql -h <HOST> -P 4000 -u xxxxxxxx.root -p --ssl-mode=VERIFY_IDENTITY
```

Una vez dentro:

```sql
SHOW DATABASES;          -- debe aparecer articulos_db
USE articulos_db;
SHOW TABLES;             -- vacío hasta que la app arranque la primera vez
```

### Opción C — Probar la imagen Docker completa en local

Para reproducir exactamente lo que hará Render (build + ejecución en contenedor):

```bash
docker build -t articulos-api .
docker run --rm -p 8080:8080 \
  -e DB_HOST="gateway01.<region>.prod.aws.tidbcloud.com" \
  -e DB_PORT="4000" \
  -e DB_DATABASE="articulos_db" \
  -e DB_USERNAME="xxxxxxxx.root" \
  -e DB_PASSWORD="tu-password" \
  articulos-api
```

Si la app responde en `http://localhost:8080/api/articulos`, el despliegue en Render
debería funcionar igual.

---
