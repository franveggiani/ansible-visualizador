# Automatización del servidor del visualizador

Este proyecto usa Ansible desde un contenedor Docker para preparar el servidor
`staging` donde se publica el visualizador.

El playbook realiza backups de la configuración existente, prepara Apache como
proxy inverso, agrega reglas de UFW y autoriza conexiones a PostgreSQL desde la
red Docker. También verifica que MariaDB tenga una cuenta `'admin'@'%'` con
`ALL PRIVILEGES ON *.*`; si la cuenta no existe, la crea. No crea bases de
datos.

## Arquitectura actual

```text
PC controladora
└── Contenedor ansible-controller
    └── SSH → servidor staging (192.168.62.47)
               ├── Apache :80
               │   └── proxy HTTP → 127.0.0.1:8088 (visualizador/Nginx)
               └── PostgreSQL :5432
                   └── acepta autenticación desde DOCKER_NETWORK_SUBNET
```

El contenedor definido en `docker-compose.yml` es solamente el controlador de
Ansible. No es el contenedor de la aplicación y no despliega el visualizador.
El playbook supone que el visualizador ya está publicado localmente en el
servidor mediante el puerto `8088`.

## Archivos del proyecto

| Archivo | Función |
| --- | --- |
| `Dockerfile` | Construye el controlador con Python 3.12, SSH y Ansible 12.x. |
| `docker-compose.yml` | Ejecuta el controlador, monta el proyecto y copia las claves SSH. |
| `.env` | Define localmente la red Docker y las conexiones a las bases de datos. |
| `.gitignore` | Evita versionar `.env` y los backups generados. |
| `ansible.cfg` | Define el inventario, opciones SSH y comportamiento general de Ansible. |
| `inventory.ini` | Declara los hosts de `staging` y `production`. |
| `playbook.yaml` | Contiene toda la configuración aplicada al servidor. |
| `templates/visualizador.conf.j2` | Genera el VirtualHost HTTP del visualizador. |
| `backups/` | Recibe en la PC los backups descargados desde el servidor. |

## Hosts e inventario

El playbook tiene configurado:

```yaml
hosts: staging
```

Por lo tanto, actualmente solo actúa sobre `192.168.62.47`, usando el usuario
`root` y `/usr/bin/python3.8` en el servidor remoto.

Aunque `inventory.ini` contiene un grupo `production`, este playbook no lo usa
y no realiza cambios sobre ese host.

## Variables principales

| Variable | Valor actual | Uso |
| --- | --- | --- |
| `nombre_aplicacion` | `visualizador` | Nombre utilizado, entre otras cosas, en los logs de Apache. |
| `dominio` | `192.168.62.47` | `ServerName` del VirtualHost. |
| `puerto_aplicacion` | `8088` | Puerto local donde Apache encuentra el visualizador. |
| `puerto_ssh` | `22` | Puerto autorizado por UFW para administrar el servidor. |
| `docker_subnet` | Entorno `DOCKER_NETWORK_SUBNET` | Red de Compose autorizada en UFW y `pg_hba.conf`. |
| `docker_gateway` | Entorno `DOCKER_NETWORK_GATEWAY` | Gateway configurado en el IPAM de Compose. |
| `postgres_version` | `12` | Versión usada para construir las rutas de configuración. |
| `postgres_cluster` | `main` | Nombre del clúster PostgreSQL existente. |
| `postgres_port` | Entorno `POSTGRES_CONNECTION_PORT`, por defecto `5432` | Puerto esperado y autorizado para PostgreSQL. |
| `postgres_connection_host` | `/var/run/postgresql` | Socket o host usado para las consultas finales. |
| `postgres_connection_user` | `postgres` | Usuario de las consultas finales. |
| `postgres_connection_password` | Entorno `POSTGRES_CONNECTION_PASSWORD` | Contraseña para autenticarse en PostgreSQL. |
| `mariadb_connection_host` | `127.0.0.1` | Host MariaDB usado para la verificación. |
| `mariadb_connection_port` | `3306` | Puerto de conexión a MariaDB. |
| `mariadb_connection_protocol` | `SOCKET` | Protocolo utilizado por el cliente MariaDB. Puede cambiarse a `TCP`. |
| `mariadb_connection_socket` | `/run/mysqld/mysqld.sock` | Socket local utilizado cuando el protocolo es `SOCKET`. |
| `mariadb_connection_user` | `root` | Usuario con permiso para consultar cuentas y grants. |
| `mariadb_connection_password` | Entorno `MARIADB_LOGIN_PASSWORD` | Contraseña de la conexión de inspección. |
| `mariadb_admin_user` | `admin` | Usuario MariaDB que se crea si no existe. |
| `mariadb_admin_host` | `%` | Origen asociado a la cuenta administrada. |
| `mariadb_admin_password` | Entorno `MARIADB_ADMIN_PASSWORD` | Contraseña aplicada solamente al crear la cuenta. |
| `apache_site_name` | `visualizador.conf` | Nombre del archivo del sitio Apache. |

## Flujo completo del playbook

### 1. Obtención de información del servidor

Ansible ejecuta `gather_facts` antes de las tareas. Esto obtiene información del
sistema operativo y la fecha del servidor, utilizada también para identificar
los backups.

### 2. Validaciones iniciales

Antes de modificar el servidor, el playbook comprueba que:

- El sistema pertenezca a la familia Debian, por ejemplo Debian o Ubuntu.
- Exista `/etc/postgresql/12/main/postgresql.conf`.
- Exista `/etc/postgresql/12/main/pg_hba.conf`.

Si alguna condición no se cumple, la ejecución se detiene. PostgreSQL debe estar
instalado previamente y sus rutas deben coincidir con `postgres_version` y
`postgres_cluster`.

### 3. Backup previo

Antes de aplicar cambios se crea un respaldo con una marca de tiempo.

En el servidor remoto se guardan:

- `/etc/apache2/`.
- `/etc/postgresql/12/main/`.
- `/etc/ufw/`.
- La salida de `ufw status numbered` en un archivo legible.

El contenido se reúne en:

```text
/var/backups/ansible/<fecha-y-hora>/
```

Después se genera:

```text
/var/backups/ansible/config-<host>-<fecha-y-hora>.tar.gz
```

Finalmente, Ansible descarga el archivo a `/ansible/backups` dentro del
controlador. Como el proyecto está montado en `/ansible`, el archivo aparece en
la carpeta local `backups/` de este repositorio.

Cada ejecución genera un backup nuevo.

### 4. Dependencias del servidor

El playbook no administra APT ni instala paquetes. Espera que el servidor ya
tenga disponibles `apache2`, `ufw`, `mariadb-client`, `python3-pymysql`,
`python3-psycopg2` y PostgreSQL Server.

### 5. Configuración de Apache

Se habilitan los siguientes módulos:

- `proxy`.
- `proxy_http`.
- `headers`.
- `rewrite`.

Luego se procesa `templates/visualizador.conf.j2` y se genera:

```text
/etc/apache2/sites-available/visualizador.conf
```

El sitio se habilita mediante un enlace simbólico:

```text
/etc/apache2/sites-enabled/visualizador.conf
    → /etc/apache2/sites-available/visualizador.conf
```

Por último, se asegura que Apache esté iniciado y habilitado para arrancar con
el sistema.

#### Funcionamiento del VirtualHost

El VirtualHost actual escucha solamente por HTTP en el puerto `80`. Cuando una
solicitud llega con `Host: 192.168.62.47`, Apache la reenvía a:

```text
http://127.0.0.1:8088/
```

La respuesta del visualizador vuelve al cliente a través de Apache. El puerto
`8088` no necesita quedar expuesto a la red externa.

La configuración:

- Deshabilita el uso de Apache como proxy abierto.
- Conserva el encabezado `Host` original.
- Informa al backend que el protocolo externo es HTTP y el puerto es 80.
- Escribe logs en `visualizador_access.log` y `visualizador_error.log` dentro
  del directorio de logs de Apache.

Si cambia la plantilla, Ansible conserva un backup del archivo anterior antes
de reemplazarlo.

### 6. Reglas de UFW

El playbook agrega las siguientes autorizaciones:

| Puerto | Origen | Finalidad |
| --- | --- | --- |
| `22/tcp` | Cualquier origen | Administración SSH. |
| `80/tcp` | Cualquier origen | Acceso HTTP al visualizador. |
| `443/tcp` | Cualquier origen | Reservado para acceso HTTPS. |
| `5432/tcp` | `DOCKER_NETWORK_SUBNET` | PostgreSQL desde la red Docker. |

No abre el puerto `8088`, porque Apache accede al visualizador mediante
`127.0.0.1`.

Las tareas que establecen políticas predeterminadas y habilitan UFW están
comentadas. En consecuencia, el playbook agrega reglas, pero no activa el
firewall si estaba deshabilitado.

### 7. Configuración de `pg_hba.conf`

El playbook agrega o mantiene una regla equivalente a:

```text
host    all    all    <DOCKER_NETWORK_SUBNET>    scram-sha-256
```

Esto significa:

- `host`: la regla aplica a conexiones TCP/IP.
- Primer `all`: permite intentar la conexión a cualquier base existente.
- Segundo `all`: permite intentar la conexión con cualquier usuario existente.
- `<DOCKER_NETWORK_SUBNET>`: limita el origen a la subred definida en `.env`.
- `scram-sha-256`: exige autenticación con contraseña SCRAM.

La regla no crea bases, usuarios ni contraseñas. Solamente determina desde qué
red se permite intentar la autenticación.

La modificación genera un backup de `pg_hba.conf` y solicita una recarga de
PostgreSQL.

Las tareas para configurar `listen_addresses = '*'` y el puerto dentro de
`postgresql.conf` están comentadas. Por eso, para que los contenedores puedan
conectarse, PostgreSQL ya debe estar escuchando en una interfaz accesible desde
la red Docker. Modificar únicamente `pg_hba.conf` no alcanza si PostgreSQL está
escuchando solo en `localhost`.

### 8. Aplicación de handlers

Antes de las verificaciones finales, el playbook fuerza la ejecución de los
handlers pendientes:

- Para Apache, primero ejecuta `apache2ctl configtest`. Solo si la sintaxis es
  válida recarga el servicio.
- Para cambios en `pg_hba.conf`, recarga PostgreSQL.

Existe también un handler para reiniciar PostgreSQL, pero las tareas que lo
utilizarían están comentadas actualmente.

### 9. Verificaciones finales

Después de aplicar los cambios se comprueba:

- Que la configuración de Apache pase `apache2ctl configtest`.
- Que Apache responda localmente en `http://127.0.0.1/`.
- El valor efectivo de `listen_addresses` en PostgreSQL.
- El puerto efectivo de PostgreSQL.
- Se crea la cuenta `'admin'@'%'` con privilegios globales si no existe.
- Que esa cuenta tenga `ALL PRIVILEGES ON *.*`, es decir, acceso total global.
- El estado final de UFW.

Al terminar, Ansible muestra un resumen con las rutas de backup y los valores
obtenidos.

La prueba HTTP actual confirma que Apache responde, pero no garantiza por sí
sola que la respuesta provenga del VirtualHost del visualizador, porque consulta
`127.0.0.1` y no envía el `Host` configurado para el sitio.

## Estado actual de HTTPS

HTTPS todavía no está configurado. El puerto `443` está autorizado en UFW, pero
el proyecto no realiza aún estas acciones:

- Habilitar el módulo `ssl` de Apache.
- Instalar un certificado y su clave privada.
- Crear un VirtualHost en el puerto `443`.
- Redirigir las solicitudes HTTP hacia HTTPS.

Como el sitio usa actualmente una IP privada, un certificado debería ser
emitido para esa IP por una CA interna, ser autofirmado para pruebas, o se
debería asignar un nombre DNS interno con su certificado correspondiente.

## Lo que el playbook no hace

- No despliega ni inicia el contenedor del visualizador.
- No actualiza la caché de APT ni instala paquetes del sistema.
- No instala PostgreSQL Server.
- No crea bases de datos, usuarios ni contraseñas de PostgreSQL.
- No reemplaza la contraseña de una cuenta MariaDB ya existente.
- No corrige los privilegios de una cuenta MariaDB existente: si no son
  globales, la validación final detiene la ejecución.
- No configura actualmente `listen_addresses` ni cambia el puerto de PostgreSQL.
- No habilita UFW si el servicio está deshabilitado.
- No configura HTTPS ni certificados.
- No deshabilita el sitio predeterminado `000-default` de Apache.
- No ejecuta cambios sobre el grupo `production`.

## Ejecución

### Construir e iniciar el controlador

Desde PowerShell, en la carpeta del proyecto:

```powershell
docker compose up -d --build
```

El proyecto se monta dentro del contenedor en `/ansible`. Las claves de
`$env:USERPROFILE\.ssh` se montan en modo lectura y luego se copian dentro del
contenedor con permisos compatibles con SSH.

### Verificar conectividad

```powershell
docker compose exec ansible ansible staging -m ping
```

### Validar la sintaxis sin aplicar cambios

```powershell
docker compose exec ansible ansible-playbook --syntax-check playbook.yaml
```

### Ejecutar el playbook

Completá en `.env` las contraseñas de PostgreSQL y MariaDB:

```dotenv
DOCKER_NETWORK_SUBNET=172.23.0.0/16
DOCKER_NETWORK_GATEWAY=172.23.0.1
POSTGRES_CONNECTION_PASSWORD=clave-del-usuario-postgres
MARIADB_LOGIN_PASSWORD=clave-del-usuario-root
MARIADB_ADMIN_PASSWORD=clave-para-el-nuevo-admin
```

Docker Compose lee `.env` automáticamente. Creá o recreá el controlador después
de modificarlo y ejecutá el playbook:

```powershell
docker compose up -d --build --force-recreate
docker compose exec ansible ansible-playbook playbook.yaml
```

El host, puerto y usuario pueden reemplazarse para una ejecución con variables
extra, por ejemplo:

```powershell
docker compose exec ansible ansible-playbook playbook.yaml `
  -e mariadb_connection_host=192.168.62.50 `
  -e mariadb_connection_port=3306 `
  -e mariadb_connection_user=root
```

## Consideraciones de seguridad

- `host_key_checking` está deshabilitado, por lo que Ansible no valida la
  identidad SSH del servidor contra `known_hosts`.
- El acceso a `staging` se realiza como `root`.
- Las contraseñas de MariaDB se leen desde `.env`, que está excluido por
  `.gitignore`, y las tareas que las utilizan tienen `no_log`.
- La regla de PostgreSQL permite todas las bases y usuarios desde la red Docker,
  pero sigue exigiendo credenciales SCRAM válidas.
- Los backups pueden contener configuración sensible y se crean con permisos
  restrictivos en el servidor. La copia local también debe protegerse.
- Antes de habilitar UFW conviene verificar que la regla y el puerto SSH sean
  correctos para evitar perder el acceso remoto.

## Idempotencia

Las tareas de paquetes, módulos, plantillas, enlaces, servicios, reglas de UFW y
`pg_hba.conf` están declaradas para no repetir cambios cuando el estado ya es el
esperado. Los backups son la excepción intencional: cada ejecución crea uno
nuevo usando la fecha y hora obtenidas al inicio.
