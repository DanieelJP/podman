# Podman installation wordpress & mysql


## 1. Instalación de Podman
En Fedora, Podman se instala utilizando el gestor de paquetes dnf con el siguiente comando:
```bash
sudo dnf install -y podman
```
Instalamos Podman usando el gestor de paquetes dnf. La opción -y acepta todas las confirmaciones automáticamente.
## 2. Verificar la instalación
Para asegurarnos de que Podman se ha instalado correctamente, podemos ejecutar:
```bash
podman --version
```
Verificamos que Podman se ha instalado correctamente mostrando la versión instalada.
Este comando muestra la versión instalada de Podman.
## 3. Descargar la imagen base de Fedora:
Usa Podman para descargar la imagen base de Fedora:
```bash
podman pull fedora
```
Esto descarga la imagen más reciente de Fedora desde el repositorio de contenedores.
## 4. Crear directorios para los contenedores
Se necesitan dos directorios separados para los archivos de configuración de los contenedores de WordPress y MySQL:
```bash
mkdir WordPress
mkdir Mysql
```
Creamos dos directorios separados para los archivos de configuración de los contenedores de WordPress y MySQL.
## 5. Crear y configurar el Containerfile para WordPress
Crea el archivo Containerfile dentro del directorio WordPress:
```bash
nano WordPress/Containerfile
```
Añade el siguiente contenido al archivo:
```bash
FROM fedora:latest
# Instalar Apache, PHP, PHP-FPM y otros paquetes necesarios para WordPress
RUN dnf -y update && \
    dnf -y install httpd php php-mysqlnd php-xml php-json php-gd php-mbstring php-fpm unzip wget
# Preparar el directorio para el socket de PHP-FPM
RUN mkdir -p /run/php-fpm && \
    chown -R apache:apache /run/php-fpm
# Descargar e instalar WordPress
RUN wget https://wordpress.org/latest.zip -O /var/www/html/wordpress.zip && \
    unzip /var/www/html/wordpress.zip -d /var/www/html/ && \
    mv /var/www/html/wordpress/* /var/www/html/ && \
    rm /var/www/html/wordpress.zip && \
    rmdir /var/www/html/wordpress
# Copiar archivo de configuración de PHP personalizado
COPY php.ini /etc/php.ini
# Abrir el puerto 80
EXPOSE 80
# Configurar y arrancar Apache y PHP-FPM, asegurando los permisos del socket
CMD ["sh", "-c", "php-fpm && chown apache:apache /run/php-fpm/www.sock && chmod 660 /run/php-fpm/www.sock && /usr/sbin/httpd -D FOREGROUND"]
```
Creamos un archivo Containerfile para definir la configuración y las instrucciones para construir la imagen de WordPress.
Explicación:

- FROM fedora:latest: Utilizamos la última imagen de Fedora como base.

- RUN dnf -y update && dnf -y install ...: Instalamos Apache, PHP y otros paquetes necesarios para WordPress.

- RUN mkdir -p /run/php-fpm && chown -R apache /run/php-fpm: Preparamos el directorio para el socket de PHP-FPM y asignamos los permisos correctos.

- RUN wget ...: Descargamos y descomprimimos WordPress en el directorio de Apache.

- COPY php.ini /etc/php.ini: Copiamos un archivo php.ini personalizado para configurar PHP.

- EXPOSE 80: Abrimos el puerto 80 para el tráfico HTTP.

- CMD ["sh", "-c", ...]: Configuramos y arrancamos Apache y PHP-FPM asegurando los permisos del socket.

## 6. Crear y configurar php.ini
Crea el archivo php.ini en el mismo directorio WordPress:
```bash
nano WordPress/php.ini
```
```bash
; Maximizar el límite de memoria para scripts PHP
memory_limit = 256M
; Aumentar el tamaño máximo de archivos subidos
upload_max_filesize = 64M
; Aumentar el tamaño máximo de datos POST que PHP aceptará
post_max_size = 64M
; Asegurar que se usen cookies para la sesión en lugar de pasar el ID de la sesión en la URL
session.use_cookies = 1
session.use_only_cookies = 1
; Establecer el manejador de errores para mostrar menos información en producción
display_errors = Off
log_errors = On
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
; Establecer zona horaria predeterminada utilizada por todas las funciones de fecha/hora
date.timezone = "Europe/Madrid"
; Aumentar el tiempo máximo de ejecución de scripts PHP
max_execution_time = 300
; Ajustes específicos para mejorar la seguridad
expose_php = Off
```
Creamos un archivo php.ini personalizado para ajustar la configuración de PHP.
Ajustamos diversas configuraciones de PHP como el límite de memoria, tamaño máximo de archivos subidos, configuración de sesión, manejo de errores y la seguridad.
## 7. Crear y configurar el Containerfile para MySQL
Crea el archivo Containerfile dentro del directorio Mysql:
```bash
nano Mysql/Containerfile
```
Añade el siguiente contenido:
```bash
# Usar la imagen base de Fedora
FROM fedora:latest
# Instalar MySQL
RUN dnf -y update && \
    dnf -y install mysql-server
# Preparar el directorio de datos
RUN mkdir -p /var/lib/mysql/ && \
    chown mysql:mysql /var/lib/mysql/
# Copiar el script de inicialización
COPY init-mysql.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/init-mysql.sh
# Exponer el puerto 3306
EXPOSE 3306
# Establecer el script de inicialización como comando por defecto
CMD ["init-mysql.sh"]
```
Creamos un archivo Containerfile para definir la configuración y las instrucciones para construir la imagen de MySQL.
Explicación:
- FROM fedora:latest: Utilizamos la última imagen de Fedora como base.

- RUN dnf -y update && dnf -y install mysql-server: Instalamos MySQL Server.

- RUN mkdir -p /var/lib/mysql/ && chown mysql/var/lib/mysql/: Preparamos el directorio de datos de MySQL y asignamos los permisos correctos.

- COPY init-mysql.sh /usr/local/bin/ && chmod +x /usr/local/bin/init-mysql.sh: Copiamos un script de inicialización y le damos permisos de ejecución.

- EXPOSE 3306: Abrimos el puerto 3306 para el tráfico MySQL.

- CMD ["init-mysql.sh"]: Configuramos el script de inicialización como el comando por defecto.

## 8. Crear y configurar init-mysql.sh
Crea el script de inicialización dentro del directorio Mysql:
```bash
nano Mysql/init-mysql.sh
```
Añade el siguiente contenido:
```bash
#!/bin/bash
# Inicializar la base de datos si el directorio está vacío
if [ -z "$(ls -A /var/lib/mysql)" ]; then
    echo "Inicializando la base de datos..."
    mysqld --initialize --user=mysql --datadir=/var/lib/mysql/
    echo "Base de datos inicializada."
fi
# Iniciar el servidor MySQL
echo "Iniciando MySQL..."
exec mysqld --datadir='/var/lib/mysql/' --user=mysql
```
Creamos un script de inicialización para configurar y arrancar MySQL.
Explicación: 
- if [ -z "$(ls -A /var/lib/mysql)" ]; then ... fi: Inicializamos la base de datos solo si el directorio de datos está vacío.
- exec mysqld --datadir='/var/lib/mysql/' --user=mysql: Iniciamos el servidor MySQL.
## 9. Construir las imágenes de los contenedores
Construye la imagen para WordPress:
```bash
podman build -t mywordpress ./WordPress
```
Construye la imagen para MySQL:
```bash
podman build -t mymysql ./Mysql
```
Construimos las imágenes para los contenedores de WordPress y MySQL a partir de los Containerfiles respectivos.
## 10. Crear el pod y ejecutar los contenedores
Crea un pod para agrupar los contenedores y asigna los puertos:
```bash
podman pod create --name mypod -p 8080:80 -p 33060:3306
```
Ejecuta el contenedor de WordPress dentro del pod:
```bash
podman run --pod mypod -it --name mywordpress -d mywordpress
```
Ejecuta el contenedor de MySQL dentro del pod con privilegios elevados:
```bash
podman run --pod mypod -it --privileged --name mymysql -d mymysql
```
Explicación:
- podman pod create ...: Creamos un pod para agrupar los contenedores y asignamos los puertos correspondientes.
- podman run --pod mypod ...: Ejecutamos los contenedores de WordPress y MySQL dentro del pod.
## 11. Configuración inicial de MySQL
Entra al contenedor de MySQL
```bash
podman exec -it mymysql /bin/bash
```
Obtén la contraseña temporal de MySQL:
```
grep "temporary password" /var/log/mysql/mysqld.log
```
Inicia sesión en MySQL usando la contraseña temporal:
```bash
mysql -u root -p
```
Dentro de MySQL, ejecuta los siguientes comandos:
```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
CREATE DATABASE wordpress;
FLUSH PRIVILEGES;
```
Salimos:
```bash
exit
```
```bash
exit
```
Explicación:
- podman exec -it mymysql /bin/bash: Entramos al contenedor de MySQL.
- grep "temporary password" /var/log/mysql/mysqld.log: Obtenemos la contraseña temporal de MySQL.
- mysql -u root -p: Iniciamos sesión en MySQL (se ha poner la contraseña temporal generada).
- ALTER USER 'root'@'localhost' IDENTIFIED BY 'root'; Cambiamos la contraseña del usuario root.
- CREATE DATABASE wordpress; Creamos una base de datos llamada wordpress.
- FLUSH PRIVILEGES; Aplicamos los cambios de privilegios.
- exit: Salimos del contenedor de MySQL.
- exit: salimos del pod

## 12. Guardar la configuración de MySQL
Guarda el estado actual del contenedor de MySQL:
```bash
podman commit mymysql myfinalmysql
```
Guardamos el estado actual del contenedor de MySQL en una nueva imagen llamada myfinalmysql.
## 13. Configuración de WordPress
Entra al contenedor de WordPress:
```bash
podman exec -it mywordpress /bin/bash
```
Copia el archivo de configuración de ejemplo y renómbralo:
```bash
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```
Edita el archivo de configuración:
```bash
vi /var/www/html/wp-config.php
```
Ajusta las configuraciones de la base de datos en el archivo:
```bash
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'root' );
define( 'DB_PASSWORD', 'root' );
define( 'DB_HOST', '127.0.0.1' );
```
Salimos:
```bash
exit
```
Explicación:
- podman exec -it mywordpress /bin/bash: Entramos al contenedor de WordPress.
- cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php: Copiamos el archivo de configuración de ejemplo y lo renombramos.
- vi /var/www/html/wp-config.php: Editamos el archivo de configuración.
- define('',''); : Ajustamos las configuraciones de la base de datos para que WordPress pueda conectarse a MySQL.
- exit: Salimos del pod
## 14. Reiniciar los contenedores
Reinicia los contenedores para aplicar los cambios:
```bash
podman restart mymysql
podman restart mywordpress
```
Reiniciamos los contenedores para aplicar los cambios realizados en la configuración.
## 15. Acceder mediante el navegador
Ponemos esta dirección en el navegador:
```bash
http://localhost:8080
````
Accedemos a la instalación de WordPress mediante el navegador web apuntando a la dirección http://localhost:8080.

## 16. Iniciar sesión
Creas una sesión poniendo un usuario, contraseña y correo
Le das a Install
¡Y finalmente inicias sesión!


## Tutorial made by DJparreño.
