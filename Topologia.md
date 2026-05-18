# Documentación Técnica: Topología Interna de Contenedores (Entorno Docker)
## Proyecto GeoNode Pilcomayo - Infraestructura IRD Francia

---

## 1. Resumen Ejecutivo de la Arquitectura
La implementación de la plataforma **GeoNode Pilcomayo** en la infraestructura del Instituto de Investigación para el Desarrollo (IRD) en Francia ha sido diseñada bajo un paradigma de microservicios utilizando **Docker** y **Docker Compose**. 

En lugar de un despliegue monolítico tradicional, el sistema se compone de múltiples contenedores especializados y aislados que colaboran a través de redes virtuales privadas. Esta arquitectura de desacoplamiento de componentes garantiza ventajas críticas para el proyecto:
* **Portabilidad Absoluta:** Permite empaquetar el estado exacto de las dependencias, facilitando la migración transparente desde los servidores locales de la Universidad Mayor de San Andrés (UMSA) hacia la nube institucional en Francia.
* **Aislamiento de Fallos:** Si un componente secundario (como el bróker de mensajería o el obrero de tareas asíncronas) experimenta una sobrecarga, el núcleo de la base de datos espacial y el servidor de mapas continúan operando sin interrupción.
* **Escalabilidad Eficiente:** Permite asignar límites estrictos de hardware (CPU y memoria RAM) a cada contenedor de forma independiente, optimizando el rendimiento general del sistema en producción.

---

## 2. Inventario Detallado de Contenedores (Roles y Servicios)

El entorno Docker Compose del proyecto está compuesto por cinco contenedores principales, estructurados de la siguiente manera:

### A. `django4geonode_project` (Core de la Aplicación)
* **Rol Técnico:** Actúa como el núcleo administrativo y de negocio de GeoNode. Está basado en el framework Django (Python).
* **Función en el Proyecto:** Gestiona la interfaz gráfica orientada al usuario, el catálogo interactivo de metadatos (incluyendo los identificadores científicos **DOI**), el sistema de autenticación y el control de accesos por roles.
* **Puerto Interno:** `8000` (No expuesto públicamente).

### B. `geoserver` (Motor Geoespacial)
* **Rol Técnico:** Servidor de mapas e información geográfica de código abierto, ejecutado sobre una instancia optimizada de Apache Tomcat.
* **Función en el Proyecto:** Se encarga del procesamiento, renderizado y publicación de todas las capas vectoriales y ráster del Proyecto Pilcomayo. Expone los datos mediante servicios web geoespaciales estándar de la OGC (*Open Geospatial Consortium*) tales como **WMS, WFS y WCS**, permitiendo la interoperabilidad.
* **Puerto Interno:** `8080` (No expuesto públicamente).

### C. `db` (Base de Datos Espacial - PostgreSQL 15 / PostGIS 3.5)
* **Rol Técnico:** Motor de base de datos relacional robusto enriquecido con la extensión espacial PostGIS.
* **Función en el Proyecto:** Almacena de forma separada dos estructuras de datos cruciales a través de URLs de conexión independientes:
    1.  `project`: Base de datos transaccional que resguarda la configuración de Django, perfiles de usuario, metadatos y logs.
    2.  `project_data` (*Datastore*): Repositorio espacial dedicado exclusivamente a almacenar las tablas vectoriales y geometrías geográficas del Pilcomayo.
* **Puerto Interno:** `5432` (Completamente bloqueado al exterior por motivos de ciberseguridad).

### D. `rabbitmq` (Bróker de Mensajería)
* **Rol Técnico:** Gestor de colas de mensajes basado en el protocolo AMQP.
* **Función en el Proyecto:** Actúa como el intermediario de comunicación asíncrona entre el contenedor de Django y los obreros de procesamiento. Coordina el tráfico de eventos internos para evitar cuellos de botella en el servidor.
* **Puerto Interno:** `5672`.

### E. `celery` / `celery-beat` (Procesamiento Asíncrono)
* **Rol Técnico:** Obrero de tareas en segundo plano (*Background Task Worker*).
* **Función en el Proyecto:** Absorbe los procesos de alta demanda de cómputo del servidor. Se activa de forma automática cuando el administrador ejecuta comandos masivos como la sincronización de capas (`updatelayers`), el cálculo de límites geográficos (*bounding boxes*), la generación de miniaturas (*thumbnails*) o el envío de notificaciones por correo SMTP (`pilcomayo-geonode@ird.fr`).
* **Puertos Expuestos:** Ninguno (Opera internamente consumiendo la cola de RabbitMQ).

---

## 3. Topología de Redes y Seguridad (Networking Interno)

Toda la infraestructura de microservicios está interconectada a través de una red virtual privada con controlador tipo puente (*Bridge*) denominada **`geonode_network`**.

```
[ Cliente Externo / QGIS ] 
          │
          ▼ (HTTPS - Puerto 443)
┌────────────────────────────────────────────────────────┐
│  Capa de Seguridad Externa: Nginx / HAProxy (IRD)     │
└───────────────────┬────────────────────────────────────┘
                    │
                    ▼ (Red Interna Aislada: geonode_network)
   ┌────────────────┼────────────────┬────────────────┐
   │                │                │                │
   ▼ (Puerto 8000)  ▼ (Puerto 8080)  ▼ (Puerto 5432)  ▼ (Puerto 5672)
┌──────────┐     ┌────────────┐   ┌──────────┐     ┌──────────┐
│  Django  │◄───►│ GeoServer  │◄─►│ PostGIS  │     │ RabbitMQ │
└──────────┘     └────────────┘   └──────────┘     └──────────┘
     ▲                                                  ▲
     │                                                  │
     └───────────────────► [ Celery ] ──────────────────┘
```

### Características de la Red de Datos:
1.  **Resolución de Nombres por DNS Interno:** Docker provee un servidor DNS embebido. Los contenedores no necesitan conocer las direcciones IP dinámicas de sus vecinos; se comunican directamente utilizando el nombre del servicio como host (ej. Django localiza la base de datos apuntando a `db:5432`).
2.  **Aislamiento y Mitigación de Riesgos:** Ningún puerto crítico de los contenedores (`5432`, `8000`, `8080`, `5672`) está abierto al internet público. Esto impide ataques de fuerza bruta directos contra la base de datos o consolas administrativas de backend.
3.  **Proxy Inverso y Cifrado SSL:** El único punto de entrada autorizado hacia la topología es la capa de Nginx / HAProxy controlada por el firewall perimetral del IRD. Este componente recibe las peticiones en el puerto público seguro `443` (HTTPS), valida el certificado digital TLS/SSL del dominio `irdpilcomayo.geovisor.org` y redistribuye internamente el tráfico de forma segura.

---

## 4. Modelo de Persistencia y Almacenamiento (Volúmenes)

Por diseño de arquitectura, los contenedores Docker son efímeros (si un contenedor se detiene o se actualiza, sus archivos internos se destruyen). Para garantizar la persistencia absoluta de los datos geográficos e institucionales del Proyecto Pilcomayo, se han implementado mapas de volúmenes persistentes vinculados directamente al almacenamiento físico del servidor en Francia:

* **Persistencia de Capas y Datos Crudos (PostgreSQL):** Vincula las carpetas del clúster de base de datos a un directorio permanente en el host. Asegura que los usuarios creados y las tablas vectoriales migradas sobrevivan a reinicios o mantenimientos del sistema.
* **Directorio de Datos de GeoServer (`data_dir`):** Almacena de forma física los archivos de configuración de estilos cartográficos (SLD), workspaces institucionales, conexiones a almacenes de datos (*stores*) y pirámides de imágenes ráster.
* **Volumen de Archivos Estáticos y Almacenados (`statics`):** Centraliza los archivos estáticos compilados de la interfaz web (`/mnt/volumes/statics/static/`) y los documentos multimedia anexados o metadatos de mapas subidos por los investigadores (`/mnt/volumes/statics/uploaded/`).

---

## 5. Flujo de Interoperabilidad y Trazabilidad de Peticiones

Para comprender la interacción de la topología en un escenario real (por ejemplo, un investigador externo consultando un mapa desde **QGIS** o un usuario navegando en la plataforma web), el flujo técnico de la información sigue estrictamente la siguiente ruta:

1.  **Petición Inicial:** El cliente realiza una solicitud WMS o accede al catálogo ingresando a `https://irdpilcomayo.geovisor.org`.
2.  **Validación de Capa de Red:** El Firewall perimetral del IRD intercepta la petición, descifra el canal seguro HTTPS y delega la solicitud a la red interna de Docker a través de los puertos de enlace locales.
3.  **Enrutamiento Inteligente:**
    * Si el usuario solicita interactuar con el catálogo, editar usuarios o revisar metadatos científicos, la petición es absorbida por el contenedor `django4geonode_project` (Puerto `8000`).
    * Si la petición proviene de un software SIG de escritorio (como **QGIS**) solicitando el renderizado de mapas vectoriales, el flujo se redirige a `geoserver` (Puerto `8080`).
4.  **Consulta de Datos:** GeoServer procesa la solicitud cartográfica realizando una consulta interna de alta velocidad al contenedor de la base de datos `db` (Puerto `5432`), extrayendo las geometrías y atributos almacenados en el *datastore* (`project_data`).
5.  **Respuesta Consolidada:** GeoServer renderiza los datos en un formato estándar de imagen (PNG/JPEG) o vector (GeoJSON/GML) y lo devuelve a través del proxy inverso al usuario final de forma transparente y eficiente.
