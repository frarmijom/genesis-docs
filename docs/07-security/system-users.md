# Usuarios de Sistema – Plataforma Genesis

Este documento describe los usuarios Linux creados en la plataforma Genesis, su propósito, nivel de privilegio y responsabilidades.  
La separación de usuarios es un pilar de seguridad, trazabilidad y control operacional del sistema.

---

## Principios de diseño

La plataforma Genesis sigue estos principios:

- Ningún servicio corre como `root`
- Jenkins no tiene acceso directo a credenciales de la aplicación
- Los procesos productivos tienen identidades propias
- Los accesos humanos están separados de los accesos automatizados

---

## Usuarios definidos

| Usuario | Tipo | Presente en | Propósito |
|--------|------|------------|-----------|
| `root` | Sistema | Todos | Administración del sistema operativo |
| `farmijo` | Humano | Todos | Operador y administrador |
| `jenkins_node` | Automatización | prd06, prd07, prd08 | Usuario operativo del pipeline CI/CD |
| `genesis` | Servicio | prd07, prd08 | Usuario que ejecuta la aplicación Genesis |
| `postgres` | Servicio (Docker) | prd07, prd08 | Usuario interno del contenedor PostgreSQL |

---

## root

Usuario superadministrador del sistema.

Funciones:
- Administración completa del sistema operativo
- Instalación de paquetes
- Configuración de red y systemd
- Creación de usuarios

No se utiliza para ejecutar la aplicación ni el pipeline.

---

## farmijo

Usuario humano principal.

Funciones:
- Acceso SSH interactivo
- Operación, troubleshooting y administración
- Ejecución de comandos con `sudo`

No es usado por Jenkins ni por la aplicación.

---

## jenkins_node

Usuario técnico creado para Jenkins en cada nodo (BUILD, TEST y PROD).

Funciones:
- Checkout del código desde GitHub
- Ejecución de Maven
- Copia de artefactos (SCP)
- Restart de servicios
- Ejecución de smoke tests

Privilegios:
Este usuario posee `sudo` sin contraseña para comandos específicos requeridos por el pipeline, tales como:

- `systemctl restart genesis`
- `mv /tmp/genesis.jar /opt/genesis/app/genesis.jar`
- `chown genesis:genesis /opt/genesis/app/genesis.jar`

No tiene acceso a los archivos de configuración sensibles de la aplicación.

---

## genesis

Usuario dedicado al servicio de aplicación.

Funciones:
- Ejecutar el proceso Java Spring Boot
- Acceder a los archivos de configuración y datos

Propiedades:
- No tiene login interactivo
- No posee privilegios sudo
- Es dueño de:
  - `/opt/genesis`
  - `/opt/genesis/app`
  - `/opt/genesis/config`

El archivo `application-prod.yml` tiene permisos:

