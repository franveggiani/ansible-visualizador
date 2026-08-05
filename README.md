# Automatización de infraestructura con Ansible y Docker

Proyecto práctico basado en un caso real para automatizar la preparación de un servidor Linux utilizado como entorno de staging de una aplicación web.

El nodo controlador de Ansible se ejecuta dentro de Docker y administra el servidor remoto mediante SSH. El playbook realiza validaciones previas, genera backups de configuraciones críticas, configura Apache como reverse proxy, administra reglas de UFW, autoriza conexiones de PostgreSQL desde una red Docker y ejecuta verificaciones posteriores.

## Objetivos

- Reducir cambios manuales sobre el servidor.
- Mantener un entorno de control reproducible.
- Generar backups antes de modificar configuraciones críticas.
- Validar servicios y archivos antes y después de aplicar cambios.
- Diseñar tareas idempotentes siempre que el proceso lo permita.

## Arquitectura

```text
PC controladora
└── Contenedor ansible-controller
    └── SSH → servidor staging
               ├── Apache :80
               │   └── reverse proxy → 127.0.0.1:8088
               ├── PostgreSQL :5432
               ├── MariaDB :3306
               └── UFW
```

El contenedor definido en `docker-compose.yml` contiene solamente el controlador de Ansible. No despliega ni inicia la aplicación.

## Funcionalidades

### Validaciones previas

- Verifica que el servidor pertenezca a la familia Debian/Ubuntu.
- Comprueba la existencia de `pg_hba.conf`.
- Detiene la ejecución si faltan precondiciones obligatorias.

### Backups

Antes de aplicar cambios, el playbook respalda:

- `/etc/apache2/`
- La configuración del clúster PostgreSQL.
- `/etc/ufw/`
- El estado numerado de UFW.

Los archivos se comprimen en el servidor y luego se descargan al directorio local `backups/`.

Cada ejecución genera un backup nuevo con marca de tiempo. Por ese motivo, esta parte es intencionalmente no idempotente.

### Apache HTTP Server

- Habilita `proxy`, `proxy_http`, `headers` y `rewrite`.
- Genera el VirtualHost desde `templates/visualizador.conf.j2`.
- Configura Apache como reverse proxy hacia la aplicación local.
- Ejecuta `apache2ctl configtest` antes de recargar el servicio.

### UFW

Administra reglas para:

- SSH.
- HTTP.
- HTTPS.
- PostgreSQL, restringido a la subred Docker configurada.

El puerto interno de la aplicación no se expone porque Apache accede al backend mediante `127.0.0.1`.

### PostgreSQL

- Agrega una regla en `pg_hba.conf` para permitir conexiones desde la red Docker.
- Recarga PostgreSQL cuando cambia la configuración.
- Verifica la conectividad y consulta los valores efectivos de puerto y `listen_addresses`.

La regla de acceso no crea bases de datos, usuarios ni contraseñas.

### MariaDB

El playbook verifica la existencia de una cuenta administrativa y puede crearla si no existe.

La configuración actual permite definir mediante variables el nombre de usuario, el host y la contraseña. El caso original utiliza privilegios globales porque responde a un requerimiento concreto del entorno interno.

Para un entorno productivo nuevo se recomienda aplicar mínimo privilegio:

- Restringir el host a una subred o dirección específica.
- Otorgar permisos solamente sobre la base necesaria.
- Evitar usuarios genéricos como `admin`.
- Gestionar secretos con Ansible Vault o un gestor de secretos externo.

## Estructura

```text
.
├── .env.example
├── .gitignore
├── Dockerfile
├── README.md
├── ansible.cfg
├── docker-compose.yml
├── inventory.example.ini
├── playbook.yaml
└── templates/
    └── visualizador.conf.j2
```

Los archivos reales `.env`, `inventory.ini` y `backups/` están excluidos del repositorio.

## Configuración

### 1. Crear los archivos locales

```bash
cp .env.example .env
cp inventory.example.ini inventory.ini
```

Editá ambos archivos con los datos del entorno que vas a administrar.

### 2. Variables principales

```dotenv
APP_DOMAIN=staging.example.internal
APP_PORT=8088
DOCKER_NETWORK_SUBNET=172.23.0.0/16
DOCKER_NETWORK_GATEWAY=172.23.0.1
POSTGRES_CONNECTION_PORT=5432
POSTGRES_CONNECTION_PASSWORD=change-me
MARIADB_LOGIN_PASSWORD=change-me
MARIADB_ADMIN_PASSWORD=change-me
```

No versiones `.env` ni el inventario real.

## Ejecución

### Construir el controlador

```powershell
docker compose up -d --build
```

### Verificar conectividad

```powershell
docker compose exec ansible ansible staging -m ping
```

### Validar sintaxis

```powershell
docker compose exec ansible ansible-playbook --syntax-check playbook.yaml
```

### Ejecutar el playbook

```powershell
docker compose exec ansible ansible-playbook playbook.yaml
```

## Idempotencia

Las tareas de módulos, plantillas, enlaces, servicios, reglas de UFW y `pg_hba.conf` están declaradas para no repetir cambios cuando el servidor ya se encuentra en el estado esperado.

Los backups constituyen la excepción: cada ejecución crea uno nuevo para conservar un punto de recuperación previo.

## Consideraciones de seguridad

- Las credenciales se leen desde `.env`, excluido mediante `.gitignore`.
- Las tareas que manipulan contraseñas utilizan `no_log`.
- Los backups pueden contener información sensible y deben protegerse.
- El inventario real no se versiona.
- `host_key_checking` está deshabilitado para simplificar el laboratorio; en producción debería habilitarse y utilizar `known_hosts`.
- El acceso remoto no debería realizarse como `root` en un entorno nuevo. Es preferible un usuario dedicado con `sudo` controlado.
- Antes de habilitar UFW se debe confirmar que la regla SSH sea correcta para evitar perder acceso remoto.

## Alcance y limitaciones

El proyecto no:

- Despliega la aplicación.
- Instala PostgreSQL o MariaDB Server.
- Crea bases de datos PostgreSQL.
- Configura certificados HTTPS.
- Activa UFW si estaba deshabilitado.
- Gestiona alta disponibilidad.
- Sustituye una solución completa de gestión de secretos.

## Tecnologías

- Ansible
- Docker y Docker Compose
- Linux
- Apache HTTP Server
- UFW
- PostgreSQL
- MariaDB
- Bash y SSH
