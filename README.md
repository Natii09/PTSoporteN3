# Prueba Técnica: Ingeniero de Soporte N3 - Diagnóstico de Microservicios

## Escenario
Se ha reportado una caída crítica en el portal de clientes. El equipo de Nivel 2 escaló el caso indicando que, tras el último despliegue, el servicio no responde correctamente. Tu misión es estabilizar el entorno, identificar las fallas y asegurar la comunicación entre los componentes.

## Arquitectura del Entorno
El sistema consta de tres capas corriendo en contenedores Docker:
1. **nginx-proxy**: Balanceador de carga y punto de entrada (Puerto 8080).
2. **api-service**: Lógica de negocio (Node.js).
3. **database**: Motor de base de datos (PostgreSQL).

---

## Instrucciones para el Candidato

### 1. Preparación
Asegúrate de tener instalado **Docker** y **Docker Compose** y bajar los archivos necesarios de este repositorio.

### 2. Ejecución del Incidente
Inicia el entorno ejecutando el siguiente comando en tu terminal:
```bash
docker-compose up -d
```

### 3. Entrega
1. Debe hacer un Pull request al repositorio para que el propietario del repo pueda ver los cambios.
2. Si desea puede hacer un fork del proyecto.
3. También es válido crear un repo completamente nuevo y subir allí los archivos corregidos. Debe enviar el Link de su repo a la persona que lo ha estado acompañando en el proceso de selección
4. En el archivo readme.md debe subir las notas con la explicación o justificación de los cambios efectuados para que el proyecto pueda correr. Es decir documente cómo ha resuelto el incidente y resalte las partes donde tuvo que hacer las modificaciones.

#### Plus
Agrega un healthcheck para que la API no se conecte a la base de datos sin que este servicio de Postgres esté listo.

#### ENTREGA FINAL ###### 

### HALLAZGO 1:

Durante la ejecución inicial de docker compose up falla antes de iniciar el servicio
Evidencia:  ✘ service:api-service:1            Error response from daemon: Minimum memory limit allowed is 6MB  

Error: En el archivo docker-compose.yml se configuró cn memory: 5M y es inferior al mínimo permitido por Docker

Impacto: El contenedor api-service no puede crearse, impidiendo el levantamiento completo de la plataforma.

Acción: Incrementar el límite de memoria a un valor válido o eliminar la restricción.

Corrección aplicada: Se modificó el límite de memoria del servicio api-service de 5 MB a 64 MB debido a que Docker Desktop exige un mínimo de 6 MB para crear el contenedor.


### HALLAZGO 2:

Al realizar la prueba si el portal sigue caído se accede a http://localhost:8080 y devuelve error 502 Bad Gateway

Variables de entorno del contenedor API:

DB_HOST=localhost
DB_PORT=5432


Error: El contenedor lappiz-proxy responde correctamente, pero no logra obtener una respuesta válida desde el backend lappiz-api.

El servicio api-service intenta conectarse a la base de datos usando localhost llamado al propio contenedor api-service y no el contenedor PostgreSQL.


Impacto: Los usuarios aun no pueden ver la aplicación asi este arriba. No hay conexión con la DB

Acción: Reemplazar DB_HOST=localhost por el nombre del servicio de base de datos dentro de Docker Compose por DB_HOST=database sin embargo, al verificar las variables activas del contenedor mediante: docker exec lappiz-api printenv el valor continuó siendo: localhost

La modificación aún no ha sido aplicada al contenedor en ejecución.

Acciones adicionales: Se levantaron de nuevo los contenedores: Container lappiz-db-server Started
Container lappiz-api Started
Container lappiz-proxy Started, se valida por medio del comando: docker exec lappiz-api printenv y confirma corrección aplicada:

DB_HOST=database
DB_PORT=5432


### HALLAZGO 3:

Api no responde a puerto :8080

Error: Api: target_output: 4500 Nginx:server api-service:8080

Impacto: Error 502 no reconoce la puerta de enlace, los usuarios aun no ingresan al portal.

Acción: Se ejecuto cambio en archivo nginx.conf, cambiando Nginx:server api-service:8080
 a Nginx:server api-service:4500, se detuvieron los servicios del Docker y se subieron nuevamente dando solución al error y por consecuente usuarios ya pueden ingresar con normalidad al portal


### Healthcheck:

Error: La API podía iniciar antes de que PostgreSQL estuviera completamente disponible, generando fallos de conexión durante el arranque.

Acción: Se agregó un healthcheck al servicio database:

healthcheck:
  test: ["CMD-SHELL", "pg_isready -U admin -d main_db"]
  interval: 10s
  timeout: 5s
  retries: 5
Y se configuró la dependencia:

depends_on:
  database:
    condition: service_healthy

La API inicia únicamente cuando PostgreSQL se encuentra en estado saludable (healthy).


