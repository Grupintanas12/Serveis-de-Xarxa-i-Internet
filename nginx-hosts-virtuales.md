# Nginx: hosts virtuales

## Objetivo

En esta práctica monto varias webs en el mismo servidor Nginx, todas en el puerto 80. Como comparten IP y puerto, Nginx sabe cuál tiene que mostrar por el nombre de dominio que pide el navegador (la directiva `server_name`). Para que esos nombres funcionen los doy de alta en mi servidor DNS.

La práctica tiene varias partes:

1. Dos webs estáticas con dominio propio en el puerto 80.
2. Una zona protegida con usuario y contraseña.
3. Una web con PHP.
4. Una web con PHP y MySQL.

## Entorno

Uso las mismas máquinas de las prácticas anteriores:

| Máquina | IP | Qué hace en esta práctica |
| --- | --- | --- |
| Maestro (debian) | 192.168.6.100 | DNS maestro de haven.local, donde doy de alta los dominios |
| dns2 | 192.168.6.101 | DNS esclavo y servidor Nginx con las webs |

Las dos son Debian 13 en VirtualBox. Nginx en dns2 está confinado con AppArmor desde la práctica anterior, así que si algo no funciona también hay que mirar si AppArmor lo está bloqueando.

| Web | Dominio | Carpeta | Configuración |
| --- | --- | --- | --- |
| Estática 1 | ejemplo1.haven.local | /var/www/ejemplo1.haven.local | /etc/nginx/sites-available/ejemplo1.conf |
| Estática 2 | ejemplo2.haven.local | /var/www/ejemplo2.haven.local | /etc/nginx/sites-available/ejemplo2.conf |

## 1. Dar de alta los dominios en el DNS

En el maestro añadí los dos nombres a la zona haven.local, apuntando a dns2, que es donde está Nginx:

```
sudo nano /etc/bind/zones/db.haven.local
```

Subí el serial de 3 a 4 y añadí al final:

```
; hosts - A records
ejemplo1    IN      A       192.168.6.101
ejemplo2    IN      A       192.168.6.101
```

Comprobé la zona y recargué BIND:

```
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo rndc reload
```

named-checkzone dice `loaded serial 4` y `OK`. Para comprobar que los nombres resuelven pregunté al maestro y al esclavo:

```
dig @192.168.6.100 ejemplo1.haven.local +short
dig @192.168.6.101 ejemplo2.haven.local +short
```

Los dos devuelven 192.168.6.101. Como el segundo es el esclavo, eso también confirma que la zona se ha replicado sola al subir el serial.

## 2. Dos webs estáticas en el puerto 80

### 2.1 Carpetas y páginas

En dns2 creé una carpeta para cada web dentro de /var/www, cada una con su carpeta img para las imágenes, como en la guía:

```
sudo mkdir -p /var/www/ejemplo1.haven.local/img
sudo mkdir -p /var/www/ejemplo2.haven.local/img
```

Después creé una página sencilla en cada una:

```
sudo nano /var/www/ejemplo1.haven.local/index.html
```

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Ejemplo 1</title>
</head>
<body>
  <h1>Bienvenido a ejemplo1.haven.local</h1>
  <p>Esta es la primera web estatica, servida por nginx en el puerto 80.</p>
</body>
</html>
```

```
sudo nano /var/www/ejemplo2.haven.local/index.html
```

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Ejemplo 2</title>
</head>
<body>
  <h1>Bienvenido a ejemplo2.haven.local</h1>
  <p>Esta es la segunda web estatica, tambien en el puerto 80.</p>
</body>
</html>
```

Las hice distintas a propósito para que se vea claramente qué web sale con cada dominio. Comprobé la estructura con:

```
ls -R /var/www/ejemplo1.haven.local
ls -R /var/www/ejemplo2.haven.local
```

En las dos carpetas aparecen index.html y la carpeta img.

### 2.2 Configuración de los sitios

Creé un fichero de configuración para cada web en /etc/nginx/sites-available. Las dos escuchan en el mismo puerto 80 y lo único que cambia es el `server_name` y la carpeta `root`. Además le puse a cada una su propio log, para poder ver por separado las visitas y los errores de cada web.

```
sudo nano /etc/nginx/sites-available/ejemplo1.conf
```

```
server {
        listen 80;
        root /var/www/ejemplo1.haven.local;
        index index.html;
        server_name ejemplo1.haven.local;

        access_log /var/log/nginx/ejemplo1.access.log;
        error_log /var/log/nginx/ejemplo1.error.log;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

```
sudo nano /etc/nginx/sites-available/ejemplo2.conf
```

```
server {
        listen 80;
        root /var/www/ejemplo2.haven.local;
        index index.html;
        server_name ejemplo2.haven.local;

        access_log /var/log/nginx/ejemplo2.access.log;
        error_log /var/log/nginx/ejemplo2.error.log;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

Qué hace cada línea:

- `listen 80`: el puerto en el que escucha. Es el mismo en las dos.
- `server_name`: el dominio de la web. Nginx mira el nombre que pide el navegador y elige el bloque `server` que coincide.
- `root`: la carpeta de donde saca los ficheros.
- `index`: el fichero que muestra si no se pide ninguno en concreto.
- `access_log` y `error_log`: los logs propios de esta web.
- `try_files $uri $uri/ =404`: busca el fichero que se pide y, si no existe, devuelve un error 404.

Para activar los sitios creé un enlace a cada fichero en /etc/nginx/sites-enabled, que es la carpeta que lee Nginx. Si quiero desactivar una web, basta con borrar su enlace.

```
sudo ln -s /etc/nginx/sites-available/ejemplo1.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/ejemplo2.conf /etc/nginx/sites-enabled/
ls -l /etc/nginx/sites-enabled/
```

Salen default, ejemplo1.conf y ejemplo2.conf, cada uno apuntando a su fichero en sites-available. El sitio default lo dejé activo: es el que responde cuando se entra por IP o por localhost.

Por último comprobé la configuración y recargué Nginx:

```
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` dice `syntax is ok` y `test is successful`.

### 2.3 Comprobación

Para probarlo hice una petición a cada dominio desde dns2:

```
curl http://ejemplo1.haven.local
curl http://ejemplo2.haven.local
curl http://localhost
```

La primera vez las dos webs nuevas me devolvieron un `403 Forbidden`, mientras que localhost seguía mostrando la página de siempre. Lo bueno es que eso ya demostraba que los hosts virtuales funcionaban: Nginx reconocía cada dominio y hasta escribía en su log propio (ejemplo1.error.log y ejemplo2.error.log). El problema era que no podía leer las carpetas.

Mirando los logs vi que era AppArmor:

```
sudo cat /var/log/nginx/ejemplo1.error.log
sudo grep DENIED /var/log/audit/audit.log | grep www | tail -2
```

En el log de Nginx salía `(13: Permission denied)` y en el de auditoría un `apparmor="DENIED"` sobre /var/www/ejemplo1.haven.local. En la práctica anterior confiné Nginx con un perfil que le dejaba leer la web con `include <abstractions/web-data>`, y esa abstracción solo cubre /var/www/html. Mis webs nuevas están fuera de esa carpeta, así que AppArmor las bloqueaba.

Lo arreglé añadiendo al perfil permiso de lectura para todo /var/www:

```
sudo nano /etc/apparmor.d/usr.sbin.nginx
```

```
  /var/www/ r,
  /var/www/** r,
```

Y recargando el perfil:

```
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

Solo le di permiso de lectura (`r`): Nginx puede servir las webs pero no modificarlas.

Después de esto, cada dominio muestra su propia web, las dos por el puerto 80:

- `http://ejemplo1.haven.local` muestra "Bienvenido a ejemplo1.haven.local".
- `http://ejemplo2.haven.local` muestra "Bienvenido a ejemplo2.haven.local".
- `http://localhost` sigue mostrando la página del sitio default.

También lo comprobé con Firefox dentro de dns2, que resuelve los nombres con mi propio DNS. Y cada web tiene sus logs por separado:

```
ls -l /var/log/nginx/
```

Aparecen ejemplo1.access.log, ejemplo1.error.log, ejemplo2.access.log y ejemplo2.error.log.

### 2.4 Verlas desde el navegador del ordenador

Para enseñar las webs desde el navegador de mi ordenador (no desde dentro de la máquina) usé el mismo camino que en la práctica de NAT: el reenvío de puertos de VirtualBox (8080 del ordenador al 80 del maestro) y el DNAT del maestro hacia dns2.

El problema es que mi ordenador no usa mi DNS, así que no sabe qué es ejemplo1.haven.local. Si escribo localhost:8080, a Nginx le llega el nombre "localhost" y muestra el sitio default. Para que le llegue el nombre de cada web añadí los dos dominios al fichero hosts de Windows (C:\Windows\System32\drivers\etc\hosts), abriendo el Bloc de notas como administrador:

```
127.0.0.1   ejemplo1.haven.local
127.0.0.1   ejemplo2.haven.local
```

Después limpié la caché de nombres con `ipconfig /flushdns` y abrí en el navegador:

```
http://ejemplo1.haven.local:8080
http://ejemplo2.haven.local:8080
```

Cada dirección muestra su web. Aunque entro por el 8080 del ordenador, a Nginx le llegan por el 80 con el nombre del dominio, y elige la web por el `server_name`.

## 3. Zona protegida

Siguiendo la guía, protegí con usuario y contraseña la carpeta /protected de ejemplo1. Todo en dns2.

Instalé apache2-utils, que trae la herramienta htpasswd para crear el fichero de usuarios:

```
sudo apt install -y apache2-utils
```

Creé el fichero con el usuario alumno. Me pidió la contraseña dos veces:

```
sudo htpasswd -Bc /etc/nginx/conf.d/.htpasswd alumno
```

- `-c` crea el fichero.
- `-B` guarda la contraseña cifrada con bcrypt.

Le puse permisos para que solo puedan leerlo root y el grupo www-data, que es el de Nginx (en la guía pone nginx:nginx, pero en Debian el usuario de Nginx es www-data):

```
sudo chown root:www-data /etc/nginx/conf.d/.htpasswd
sudo chmod 640 /etc/nginx/conf.d/.htpasswd
```

Creé la carpeta protegida con una página dentro:

```
sudo mkdir /var/www/ejemplo1.haven.local/protected
sudo nano /var/www/ejemplo1.haven.local/protected/index.html
```

Y añadí un bloque `location /protected/` en la configuración de ejemplo1:

```
server {
        listen 80;
        root /var/www/ejemplo1.haven.local;
        index index.html;
        server_name ejemplo1.haven.local;

        access_log /var/log/nginx/ejemplo1.access.log;
        error_log /var/log/nginx/ejemplo1.error.log;

        location / {
                try_files $uri $uri/ =404;
        }

        location /protected/ {
                auth_basic              "Acceso restringido";
                auth_basic_user_file    /etc/nginx/conf.d/.htpasswd;
        }
}
```

- `auth_basic`: activa la autenticación básica. El texto es el mensaje que sale en la ventana de login.
- `auth_basic_user_file`: el fichero con los usuarios y contraseñas.

La primera vez `nginx -t` me dio el error `unexpected end of file`. Era porque en la ruta del fichero había abierto comillas dobles y cerrado con una simple, así que Nginx llegaba al final del fichero sin encontrar dónde se cerraban. Le quité las comillas y ya funcionó:

```
sudo nginx -t
sudo systemctl reload nginx
```

No tuve que tocar AppArmor: el perfil de Nginx ya le deja leer todo /etc/nginx, y el fichero .htpasswd está ahí.

Para comprobarlo, primero pedí la página sin usuario:

```
curl -I http://ejemplo1.haven.local/protected/
```

```
HTTP/1.1 401 Unauthorized
Server: nginx
WWW-Authenticate: Basic realm="Acceso restringido"
```

El 401 significa que hace falta identificarse, y la cabecera WWW-Authenticate es la que hace que el navegador saque la ventana de login. Después con usuario:

```
curl -u alumno http://ejemplo1.haven.local/protected/
```

Me pidió la contraseña y me devolvió la página protegida. Con una contraseña incorrecta vuelve a dar 401.

En el navegador del ordenador, al entrar en `http://ejemplo1.haven.local:8080/protected/` sale una ventana pidiendo usuario y contraseña, y al ponerlos se ve la página "Zona protegida ejemplo1". La parte pública de ejemplo1 se sigue viendo sin contraseña.

## 4. Web con PHP

La parte de PHP la hice en la misma web ejemplo1, añadiéndole una página info.php. La parte estática y la zona protegida siguen funcionando igual.

Nginx no ejecuta PHP por sí mismo. Lo que hace es pasarle los ficheros .php a un servicio aparte, PHP-FPM, que los ejecuta y le devuelve el resultado. Se comunican por un socket, que es como un "enchufe" en forma de fichero.

### 4.1 Instalar PHP-FPM

En dns2:

```
sudo apt install -y php-fpm
php -v
ls /run/php/
```

Se instaló PHP 8.4, y en /run/php aparece el socket php8.4-fpm.sock, que es el que necesito para la configuración.

### 4.2 Permiso en AppArmor

Como Nginx está confinado con AppArmor, antes de nada amplié su perfil para que pudiera conectarse al socket de PHP-FPM. Si no, la página daría un error 502.

```
sudo nano /etc/apparmor.d/usr.sbin.nginx
```

```
  /run/php/php*-fpm.sock rw,
  unix (connect, send, receive) type=stream,
```

```
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

### 4.3 Activar PHP en ejemplo1

En /etc/nginx/sites-available/ejemplo1.conf añadí index.php a la línea index y un bloque nuevo para los ficheros .php:

```
server {
        listen 80;
        root /var/www/ejemplo1.haven.local;
        index index.php index.html;
        server_name ejemplo1.haven.local;

        access_log /var/log/nginx/ejemplo1.access.log;
        error_log /var/log/nginx/ejemplo1.error.log;

        location / {
                try_files $uri $uri/ =404;
        }

        location /protected/ {
                auth_basic              "Acceso restringido";
                auth_basic_user_file    /etc/nginx/conf.d/.htpasswd;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        }
}
```

- `location ~ \.php$`: se aplica a todo lo que acaba en .php. El `~` indica que es una expresión regular.
- `include snippets/fastcgi-php.conf`: carga la configuración estándar para pasar peticiones a PHP, que ya viene con Nginx en Debian.
- `fastcgi_pass`: el socket de PHP-FPM al que se mandan los ficheros.

Es el bloque de la guía, pero corregí dos erratas que tiene: pone `~ /.php` en lugar de `~ \.php$`, y `php8.2-fpm:sock` en lugar de `php8.4-fpm.sock`.

La primera vez `nginx -t` me dio el error `directive "location" has no opening "{"`. Era porque había escrito `\.php${` sin espacio entre el `$` y la llave, y Nginx lo leía todo como una sola palabra. Con el espacio ya funcionó:

```
sudo nginx -t
sudo systemctl reload nginx
```

### 4.4 Página de prueba

Creé /var/www/ejemplo1.haven.local/info.php:

```
<?php
echo "<h1>PHP funciona en ejemplo1</h1>";
echo "<p>Version de PHP: " . phpversion() . "</p>";
echo "<p>Fecha y hora del servidor: " . date("d/m/Y H:i:s") . "</p>";
phpinfo();
?>
```

Muestra la versión de PHP y la hora del servidor, que cambia cada vez que recargas (así se ve que la página se genera en el momento y no es un HTML fijo), y debajo la información completa de PHP con `phpinfo()`.

Al principio me daba un error 500. Para saber por qué miré el log de ejemplo1:

```
sudo tail -2 /var/log/nginx/ejemplo1.error.log
```

Ahí PHP explicaba el fallo con la línea exacta. Tenía dos erratas en el código: un `;` que faltaba al final de una línea y un `*` en lugar de `(` en `date(`. Lo curioso es que el error del `;` lo marcaba en la línea siguiente, porque PHP no se da cuenta de que falta hasta que encuentra el siguiente `echo`. Lo bueno es que el error 500 ya demostraba que Nginx y PHP-FPM se comunicaban bien, porque el mensaje venía del propio PHP.

Después de corregirlo:

```
curl -s http://ejemplo1.haven.local/info.php | head -5
```

Sale "PHP funciona en ejemplo1" con la versión y la hora. En el navegador del ordenador, `http://ejemplo1.haven.local:8080/info.php` muestra mis tres líneas y debajo la tabla de phpinfo.

## 5. Web con PHP y MySQL

La última parte también la hice en ejemplo1, con una página datos.php que lee una base de datos y muestra los datos en una tabla. En Debian se usa MariaDB, que es compatible con MySQL.

### 5.1 Instalación

En dns2 instalé MariaDB y el módulo de PHP para conectarse a ella, y reinicié PHP-FPM para que cargara el módulo:

```
sudo apt install -y mariadb-server php-mysql
sudo systemctl restart php8.4-fpm
php -m | grep -i mysql
sudo systemctl status mariadb --no-pager
```

`php -m` muestra mysqli, mysqlnd y pdo_mysql, y MariaDB sale `active (running)`.

### 5.2 Base de datos

Entré en MariaDB como administrador:

```
sudo mariadb
```

Y creé una base de datos con una tabla de alumnos, unos datos de ejemplo y un usuario solo para la web:

```
CREATE DATABASE instituto;
USE instituto;

CREATE TABLE alumnos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(50),
  ciclo VARCHAR(50)
);

INSERT INTO alumnos (nombre, ciclo) VALUES ('Ricard Forero', 'ASIX 2'), ('Raul', 'ASIX 2'), ('Eric', 'ASIX 2');

CREATE USER 'web'@'localhost' IDENTIFIED BY 'web1234';
GRANT SELECT ON instituto.* TO 'web'@'localhost';
FLUSH PRIVILEGES;

SELECT * FROM alumnos;
EXIT;
```

Al usuario web solo le di permiso de lectura (SELECT) sobre la base de datos instituto, porque la página solo tiene que mostrar datos. Así, aunque alguien consiguiera la contraseña desde la web, no podría borrar ni cambiar nada. La contraseña es solo para el laboratorio.

La primera vez el SELECT me salía vacío ("Empty set"). Había escrito `VLAUES` en lugar de `VALUES` en el INSERT, y en otro intento lo había cancelado con una coma al final, así que los datos no se llegaron a guardar.

### 5.3 Página datos.php

Creé /var/www/ejemplo1.haven.local/datos.php:

```
<?php
$conexion = new mysqli("localhost", "web", "web1234", "instituto");

if ($conexion->connect_error) {
    die("Error de conexion: " . $conexion->connect_error);
}

$resultado = $conexion->query("SELECT id, nombre, ciclo FROM alumnos");

echo "<h1>Alumnos de la base de datos</h1>";
echo "<table border='1' cellpadding='5'>";
echo "<tr><th>ID</th><th>Nombre</th><th>Ciclo</th></tr>";

while ($fila = $resultado->fetch_assoc()) {
    echo "<tr>";
    echo "<td>" . $fila["id"] . "</td>";
    echo "<td>" . $fila["nombre"] . "</td>";
    echo "<td>" . $fila["ciclo"] . "</td>";
    echo "</tr>";
}

echo "</table>";
$conexion->close();
?>
```

Qué hace:

1. Se conecta a MariaDB con el usuario web y la base de datos instituto.
2. Si la conexión falla, muestra el error y para.
3. Pide todos los alumnos con un SELECT.
4. Recorre cada fila y la escribe como una fila de una tabla HTML.
5. Cierra la conexión.

No tuve que configurar nada más en Nginx: el bloque `location ~ \.php$` que añadí para info.php ya sirve para cualquier fichero .php. Tampoco hizo falta tocar AppArmor, porque quien se conecta a MariaDB es PHP-FPM, no Nginx.

### 5.4 Comprobación

```
curl -s http://ejemplo1.haven.local/datos.php
```

Al principio la tabla salía con las filas y los ID, pero con el nombre y el ciclo vacíos. Eso quería decir que la conexión y la consulta funcionaban, y que el fallo estaba en cómo leía las columnas en el PHP: los nombres de `$fila["nombre"]` y `$fila["ciclo"]` tienen que coincidir exactamente, en minúsculas, con los de las columnas de la tabla. También borré dos filas vacías que se habían quedado de los intentos anteriores:

```
sudo mariadb -e "DELETE FROM instituto.alumnos WHERE nombre = '' OR nombre IS NULL;"
```

Después de arreglarlo, en el navegador del ordenador `http://ejemplo1.haven.local:8080/datos.php` muestra la tabla con los alumnos de la base de datos. Si añado un alumno nuevo con un INSERT y recargo la página, aparece al momento, sin tocar el PHP.
