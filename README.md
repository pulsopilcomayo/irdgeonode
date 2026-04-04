# GeoNode - Proyecto Pilcomayo (IRD)

Este repositorio contiene la configuración de infraestructura como código para el despliegue de la plataforma GeoNode del Proyecto Pilcomayo en los servidores del IRD.

## Arquitectura
El proyecto está empaquetado utilizando Docker y Docker Compose. Incluye los siguientes servicios:
* Django (Backend y panel de administración)
* Nginx (Servidor Web)
* GeoServer (Publicación de servicios WMS/WFS)
* PostgreSQL / PostGIS (Base de datos espacial)
* Redis / Celery (Gestión de tareas asíncronas)

## Requisitos Previos
* Servidor Linux (RHEL 10 o Ubuntu 24.04)
* Docker CE y Docker Compose instalados.
* Partición de datos montada en `/data`.

## Pasos para el Despliegue (Instalación)

1. **Clonar el proyecto:**
   ```bash
   cd /data
   git clone [ENLACE_DE_TU_REPOSITORIO] geonode_project
   cd geonode_project
