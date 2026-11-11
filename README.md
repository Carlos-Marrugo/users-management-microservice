# Users Management Microservice

Microservicio REST para gestionar usuarios, autenticarlos con JWT y enviar
notificaciones por correo. El proyecto usa Java 17, Spring Boot 3, Maven,
arquitectura hexagonal, JDBC/MySQL, BCrypt, Spring Security y Swagger UI.

## Requisitos

- Java 17
- Maven (se recomienda `mvnw.cmd` en Windows)
- MySQL 8+ o Docker Compose
- Docker Desktop, solo para la ejecución contenerizada

La aplicación utiliza la base de datos `crud_usuarios`. El script
[`schema.sql`](src/main/resources/schema.sql) crea las tablas y un usuario
administrador de desarrollo de forma idempotente.

## Configuración local

1. Copie `.env.example` como `.env`.
2. Genere un secreto JWT propio, por ejemplo `openssl rand -base64 32`, y
   asígnelo a `JWT_SECRET`.
3. Configure `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` y
   `DB_SSL_MODE`. Para MySQL local use `DB_SSL_MODE=DISABLED`.
4. Ejecute [`schema.sql`](src/main/resources/schema.sql) en MySQL.
5. Si necesita correo, configure las variables `SMTP_*`; sin credenciales no
   debe probarse el envío real.

Ejecute:

```powershell
.\mvnw.cmd clean test
.\mvnw.cmd clean package
```

La aplicación inicia en `http://localhost:8080`. `GET /health` solo confirma
que el servicio responde; no valida la conexión a la base de datos ni SMTP.

## Docker Compose

El archivo [`compose.yaml`](compose.yaml) inicia MySQL y la API. MySQL expone
un healthcheck y la API usa `depends_on` con `condition: service_healthy`, por
lo que no intenta iniciar antes de que la base de datos esté lista.

Antes de iniciar, defina `JWT_SECRET` en `.env` o en la sesión de terminal:

```powershell
docker compose up --build
```

La configuración de Compose usa credenciales de desarrollo explícitas
(`users_app`/`local_users_password`) y no debe reutilizarse fuera del entorno
local.

## TiDB Cloud

La aplicación mantiene el controlador MySQL y permite cambiar de MySQL local a
TiDB sin modificar código. En el panel de TiDB obtenga el endpoint, puerto,
base de datos, usuario, contraseña y el modo TLS; normalmente el puerto es
`4000` y `DB_SSL_MODE=VERIFY_IDENTITY` o el valor recomendado por el clúster.

1. Cree o seleccione la base `crud_usuarios` en TiDB.
2. Ejecute [`schema.sql`](src/main/resources/schema.sql) con el usuario de
   TiDB.
3. Configure `DB_HOST`, `DB_PORT=4000`, `DB_NAME=crud_usuarios`,
   `DB_USERNAME`, `DB_PASSWORD` y `DB_SSL_MODE`.
4. Verifique la conexión ejecutando la aplicación y consultando `/health`;
   después valide persistencia con los endpoints autenticados.

No incluya el certificado, usuario, contraseña ni el endpoint privado en Git.
La URL JDBC se construye en `DatabaseConfig` con `sslMode`, `serverTimezone`
y `allowPublicKeyRetrieval`.

## Render

[`render.yaml`](render.yaml) define un servicio Docker en la rama `main` con
healthcheck en `/health`. Render proporciona automáticamente `PORT`; no es
necesario crearla manualmente.

Complete en el panel de Render los valores reales obtenidos de TiDB:

| Variable | Valor |
| --- | --- |
| `DB_HOST` | Host del clúster TiDB |
| `DB_PORT` | Puerto TiDB, normalmente `4000` |
| `DB_NAME` | `crud_usuarios` |
| `DB_USERNAME` | Usuario TiDB |
| `DB_PASSWORD` | Contraseña TiDB |
| `DB_SSL_MODE` | Modo TLS recomendado por TiDB |
| `SMTP_HOST`, `SMTP_PORT` | Proveedor SMTP |
| `SMTP_USERNAME`, `SMTP_PASSWORD` | Credenciales SMTP |
| `SMTP_FROM_ADDRESS`, `SMTP_FROM_NAME` | Remitente autorizado |
| `JWT_SECRET` | Valor generado por Render o secreto Base64 propio de 32+ bytes |
| `JWT_EXPIRATION_SECONDS` | Segundos de validez, por ejemplo `900` |

El despliegue real, la URL asignada y la ejecución del script en TiDB requieren
acceso a las cuentas del estudiante; no se consideran verificados desde este
repositorio.

## Endpoints

- `POST /api/auth/login` — público; devuelve un token Bearer.
- `POST /api/users` — público para registrar usuarios.
- `GET /api/users` y `GET /api/users/{id}` — roles `ADMIN` o `REVIEWER`.
- `PUT /api/users/{id}` y `DELETE /api/users/{id}` — rol `ADMIN`.
- `GET /health` — público.
- Swagger UI: `/swagger-ui.html`.

Las contraseñas se almacenan con BCrypt y las credenciales no se devuelven en
las respuestas. Los tokens y secretos deben manejarse exclusivamente mediante
variables de entorno o el gestor de secretos del proveedor.

## Evidencias y estado

La carpeta [`evidencias`](evidencias) contiene instrucciones de captura para
los pasos que requieren una interfaz externa. No contiene capturas inventadas.
Las evidencias de TiDB, Render y del servicio desplegado deben agregarse
manualmente después de ejecutarlos.

No hay guías de Aiven o TiDB adicionales dentro de este repositorio; por eso
solo puede verificarse aquí la configuración versionada. Las pruebas
automatizadas se ejecutan con `.\mvnw.cmd clean test`.
