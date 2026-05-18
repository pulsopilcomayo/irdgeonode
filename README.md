# 🌍 GeoNode - Proyecto Pilcomayo (IRD)

Repositorio oficial de la configuración de **Infraestructura como Código (IaC)** para el despliegue de la plataforma GeoNode del Proyecto Pilcomayo. Este sistema gestiona, publica y permite el intercambio de datos geoespaciales, asegurando la interoperabilidad y el almacenamiento persistente en los servidores del Institut de Recherche pour le Développement (IRD) en Francia.

🚀 **Plataforma en Producción:** [https://irdpilcomayo.geovisor.org/](https://irdpilcomayo.geovisor.org/)

---

## 🏗️ Arquitectura del Sistema y Topología de Contenedores

El proyecto está orquestado completamente mediante **Docker** y **Docker Compose**, garantizando portabilidad y aislamiento. La topología interna opera a través de una red virtual privada con controlador tipo puente (`geonode_network`), manteniendo los puertos críticos cerrados al exterior por motivos de ciberseguridad.

### 🗺️ Diagrama de Red y Flujo de Peticiones

```text
[ Cliente Externo / QGIS ] 
          │
          ▼ (HTTPS - Puerto 443)
┌────────────────────────────────────────────────────────┐
│  Capa de Seguridad Externa: Nginx / HAProxy (IRD)      │
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


## 🧩 Inventario de Microservicios
django4geonode_project (Core): Actúa como el backend y panel de administración (Puerto interno: 8000). Gestiona la interfaz web, el catálogo de metadatos (incluyendo asignación de DOI) y la autenticación de usuarios.

geoserver (Motor Espacial): Encargado de procesar, renderizar y publicar las capas vectoriales y ráster mediante servicios estándar OGC (WMS, WFS, WCS) (Puerto interno: 8080).

db (PostgreSQL 15 / PostGIS 3.5): Motor de base de datos relacional y espacial. Resguarda la configuración de Django (project) y el repositorio de geometrías espaciales (project_data). (Puerto interno: 5432, completamente aislado del exterior).

rabbitmq: Bróker de mensajería (AMQP) intermediario para el encolamiento y coordinación de eventos internos (Puerto interno: 5672).

celery / celery-beat: Obrero asíncrono que absorbe y procesa tareas pesadas en segundo plano, como la actualización de capas, envío de correos SMTP y generación de miniaturas.


## 💾 Modelo de Persistencia (Volúmenes)
Los contenedores son efímeros. Para garantizar que no haya pérdida de información ante reinicios, los datos críticos se almacenan en el disco físico del servidor:

Datos Espaciales (data_dir): Almacena configuraciones de estilos cartográficos (SLD) y pirámides ráster.

Base de Datos: Persistencia del clúster de PostgreSQL para asegurar la integridad de las tablas migradas desde la UMSA.

Archivos Estáticos y Subidas (statics): Centraliza documentos multimedia anexados y elementos compilados de la interfaz.

---

# ⚙️ Requisitos Previos
Para desplegar este entorno en un servidor nuevo, se requiere:

Sistema Operativo Linux (RHEL Enterprise u Ubuntu 24.04).

Docker Engine y Docker Compose instalados (Versión API >= 1.24).

Partición de almacenamiento físico montada (ej. /mnt/volumes/statics/ para persistencia de datos).

Accesos de red habilitados (Firewall configurado para tráfico TCP Outbound y puertos 80/443).

## 🚀 Guía Rápida de Despliegue
1. Clonar el repositorio
Ingresa al directorio raíz de despliegue en tu servidor y clona el proyecto:

```Bash
cd /data
git clone [https://github.com/pulsopilcomayo/irdgeonode.git](https://github.com/pulsopilcomayo/irdgeonode.git) geonode_project
cd geonode_project
```

2. Configuración de Variables de Entorno
El sistema requiere credenciales de seguridad y rutas absolutas que no se versionan por seguridad.

Crea una copia del archivo de ejemplo:

```Bash
cp .env.sample .env
```

3. Construcción y Despliegue
Una vez configurado el entorno, levanta los microservicios en segundo plano:

```Bash
docker compose up -d
```

Verifica que todos los contenedores estén operando correctamente:

```Bash
docker compose ps
```

## 🛠️ Mantenimiento y Soporte
Para la gestión diaria del servidor y administración básico, por favor revisa el manual anexo:
[📄 Guía de Mantenimiento y Comandos Básicos: GeoNode + Docker](https://github.com/pulsopilcomayo/irdgeonode/blob/main/Guia-de-mantenimiento.md)

Contacto Técnico
Para soporte relacionado con la administración de contenedores, recuperación de accesos o escalamiento de recursos, contactar a la administración técnica del sistema en: pilcomayo-geonode@ird.fr.
