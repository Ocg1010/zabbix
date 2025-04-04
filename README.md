
# Instalación de Zabbix 7.0 en Red Hat 9.4 con PostgreSQL

### Requisitos del Sistema

La instalación de Zabbix se realizará en un sistema con las siguientes características:

- **Sistema Operativo:** Red Hat 9.4
- **Disco:** 20 GB de espacio libre
- **Memoria RAM:** 4 GB
- **CPU:** 4 vCPUs

## Requisitos Previos

Antes de proceder con la instalación de Zabbix, asegúrate de que el sistema esté:

1. **Licenciado con Red Hat:** El sistema debe contar con una licencia activa de Red Hat.
2. **Actualizado:** El sistema operativo debe estar actualizado con los últimos parches y actualizaciones disponibles.

### Para empezar con la instalación de Zabbix

Se requiere una base de datos robusta, y PostgreSQL 16 es una excelente opción. Primero debemos verificar los módulos disponibles y luego habilitar la versión correcta.

```bash
dnf module list
```
La salida debería mostrar algo similar a esto:

```
Red Hat Enterprise Linux 9 - AppStream
Name       Stream    Profiles        Summary
postgresql 12        client, server  PostgreSQL server and client module
postgresql 13        client, server  PostgreSQL server and client module
postgresql 15        client, server  PostgreSQL server and client module
postgresql 16 [e]    client, server  PostgreSQL server and client module
```
> [!NOTE]
> [e] indica que el módulo está disponible pero no habilitado por defecto. Postgresql:16 es la versión que usaremos para Zabbix.


Antes de instalarlo, debemos asegurarnos de que el módulo correcto esté habilitado.

```bash
dnf module enable postgresql:16 -y
```

## Configuración del repositorio de Zabbix
Descarga e instala el paquete RPM que configura el repositorio oficial de Zabbix 7.0 para Red Hat 9.

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
```

Limpiamos la caché de paquetes DNF para asegurar que se usen los metadatos más recientes.
```bash
dnf clean all
```


## 3. Instalación de componentes principales

```bash
dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent
```
El comando anterior instala:
 - `zabbix-server-pgsql`: Servidor Zabbix con soporte para PostgreSQL.
 - `zabbix-web-pgsql`: Frontend web para Zabbix.
 - `zabbix-apache-conf`: Configuración de Apache para Zabbix.
 - `zabbix-sql-scripts`: Scripts SQL necesarios para la base de datos.
 - `zabbix-selinux-policy`: Políticas SELinux para Zabbix.
 - `zabbix-agent`: Agente de monitoreo de Zabbix.

## Instalación y configuración de PostgreSQL
Instala el servidor de PostgreSQL.

```bash
dnf install postgresql-server -y
```
Inicializamos el cluster de base de datos de PostgreSQL.

```bash
postgresql-setup initdb
```
Habilita e inicia el servicio de PostgreSQL despues verifica su estado.

```bash
systemctl enable --now postgresql
systemctl status postgresql
```


## Creación de usuario y base de datos
Cambia al usuario 'postgres' para realizar operaciones administrativas en la base de datos.

```bash
su - postgres
``` 
Crea el usuario 'zabbix' en PostgreSQL te pedira que le asignes una contraseña.

```bash
createuser --pwprompt zabbix
```
Creación de la base de datos 'zabbix' con el usuario 'zabbix' como propietario

```bash
createdb -O zabbix zabbix
```

## Configuración de Zabbix Server

Configuración de la contraseña de la base de datos PostgreSQL en el archivo `/etc/zabbix/zabbix_server.conf`

```bash
DBPassword=password
```


## Configuración de autenticación en PostgreSQL para permitir la autenticación por contraseña

Editar `/var/lib/pgsql/data/pg_hba.conf` y cambiar:

```bash
host all all 127.0.0.1/32 ident
```
por:

```bash
host all all 127.0.0.1/32 md5
```

> [!INFO]
> Se cambia el método de autenticación de 'ident' a 'md5' para permitir autenticación por contraseña.


Reinicia PostgreSQL para aplicar los cambios de configuración.
```bash
systemctl restart postgresql.service
```

## Configuración de SELinux para habilitar la comunicación entre el frontend de Zabbix y el servidor

Con el estado SELinux habilitado en modo `Enforcing`, debe ejecutar los siguientes comandos para habilitar la comunicación:

Habilitar la comunicación entre el frontend de Zabbix y el servidor:
```
setsebool -P httpd_can_connect_zabbix on
```
Si la base de datos es accesible a través de la red (incluido 'localhost' en el caso de PostgreSQL), permitir que la interfaz de Zabbix también se conecte a la base de datos:
```
setsebool -P httpd_can_network_connect_db on
```
Reiniciar el servidor httpd
```
systemctl restart httpd
```

## Reinicio y habilitación de los servicios necesarios para Zabbix
Reinicia todos los servicios necesarios para Zabbix y habilita el inicio automático de los servicios al arrancar el sistema.

```bash
systemctl restart zabbix-server zabbix-agent httpd php-fpm
systemctl enable zabbix-server zabbix-agent httpd php-fpm
```

## Abra la página web de la interfaz de usuario de Zabbix
La URL predeterminada para la interfaz de usuario de Zabbix cuando se utiliza el servidor web httpd es http://host/zabbix
Ingrese el nombre de usuario "Admin" y la contraseña "zabbix" para iniciar sesión como superusuario de Zabbix . Tendrá acceso a todas las secciones del menú.


# Instalación y configuración de un cliente para Zabbix, contara con el Agent 2 en Red Hat 8

La instalación del cliente Zabbix se realizará en un sistema con las siguientes características:

- **Sistema Operativo:** Red Hat 8.10
- **Disco:** 5 GB de espacio 
- **Memoria RAM:** 2 GB
- **CPU:** 1 vCPUs

Antes de proceder con la instalación del cliente Zabbix, asegúrate de que el sistema esté:

1. **Licenciado con Red Hat:** El sistema debe contar con una licencia activa de Red Hat.
2. **Actualizado:** El sistema operativo debe estar actualizado con los últimos parches y actualizaciones disponibles.


### Instalar el repositorio de Zabbix
Se debe agregar el repositorio oficial de Zabbix para RHEL 8.

```
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/8/x86_64/zabbix-release-latest-7.0.el8.noarch.rpm
dnf clean all
```

### Instalar Zabbix Agent 2
```
dnf install -y zabbix-agent2
```

### Instalar plugins adicionales para el agente
Si se desea monitorear bases de datos como MongoDB, Microsoft SQL Server o PostgreSQL, se pueden instalar los siguientes paquetes:
```
dnf install -y zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
```

### Configurar Zabbix Agent 2
Editar el archivo de configuración del agente en /etc/zabbix/zabbix_agent2.conf y modificar las siguientes líneas con la dirección IP del servidor Zabbix y el hostname del agente:
```
Server=IP_DEL_SERVIDOR_ZABBIX
ServerActive=IP_DEL_SERVIDOR_ZABBIX
Hostname=NOMBRE_DEL_HOST
```

### Iniciar y habilitar el servicio Zabbix Agent 2
```
systemctl restart zabbix-agent2
systemctl enable zabbix-agent2
```
