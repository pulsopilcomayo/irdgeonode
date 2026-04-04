# Guía de Mantenimiento y Comandos Básicos: GeoNode + Docker

Esta guía está diseñada para que cualquier administrador del Proyecto Pilcomayo pueda gestionar, reiniciar o solucionar problemas básicos en la plataforma GeoNode alojada en el servidor del IRD, sin necesidad de ser un experto avanzado en servidores.

## 1. Ubicación del Proyecto
Todos los comandos de Docker Compose deben ejecutarse desde el directorio raíz donde se encuentra el archivo `docker-compose.yml` y el `.env`.

Para ingresar al directorio correcto desde la terminal del servidor, ejecuta:
```bash
cd /data/geonode_project/pilcomayo
````
## 2. Gestión del Sistema (Comandos Docker Compose)
Docker agrupa todos los componentes de GeoNode (Base de datos, Servidor Web, GeoServer) en contenedores. Estos son los comandos básicos para controlarlos:

Ver el estado de la plataforma
Para comprobar si los servicios están encendidos y funcionando sin errores:

```bash
docker compose ps
````
(Deberías ver una lista de contenedores con el estado Up o healthy).

Apagar toda la plataforma
Si necesitas hacer un mantenimiento profundo o cambiar configuraciones estructurales:

```bash
docker compose down
````
(Nota: Esto apaga el sistema, pero no borra los datos guardados en la base de datos ni los mapas).

Encender la plataforma
Para levantar todos los servicios en segundo plano:

```bash
docker compose up -d
````

Reiniciar un servicio específico
A veces, si se cambia un parámetro en GeoServer o Django, solo necesitas reiniciar ese componente en lugar de apagar todo.
Para reiniciar solo el panel web (Django):

```bash
docker compose restart django
````
Para reiniciar solo el publicador de mapas (GeoServer):

```bash
docker compose restart geoserver
````
Leer los registros (Logs)
Si la página web muestra un error o no carga, los logs te dirán exactamente qué está fallando en tiempo real:

```bash
# Ver los logs de toda la plataforma
docker compose logs -f

# Ver los logs de un componente específico (ej. django o nginx)
docker compose logs -f django
````
(Presiona Ctrl + C para salir de la vista de logs).

## 3. Administración Básica de GeoNode
GeoNode utiliza Django en su núcleo. A veces es necesario enviar comandos directamente a la plataforma para gestionar usuarios o sincronizar capas.

Crear un nuevo usuario Superadministrador
Si pierdes el acceso o necesitas crear un nuevo perfil con control total:

```bash
docker compose exec django python manage.py createsuperuser
````
(El sistema te pedirá un nombre de usuario, correo electrónico y contraseña).

Sincronizar capas de GeoServer
Si subiste mapas directamente por GeoServer y no aparecen en el catálogo web de GeoNode, puedes forzar la sincronización:

```bash
docker compose exec django python manage.py updatelayers
````
## 4. Modificación de Variables de Entorno (.env)
El archivo .env contiene las contraseñas, configuraciones de seguridad y nombres de dominio del proyecto. Está oculto por seguridad.

Para editarlo en el servidor, usa:

```bash
nano .env
````
Realiza los cambios necesarios (por ejemplo, cambiar la variable ALLOWED_HOSTS cuando se asigne un dominio público).

Guarda los cambios con Ctrl+O, presiona Enter y sal del editor con Ctrl+X.

Muy importante: Para que los cambios surtan efecto, debes reiniciar la plataforma ejecutando:

```bash
docker compose down
docker compose up -d
````
