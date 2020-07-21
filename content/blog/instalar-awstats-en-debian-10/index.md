---
title: "Instalar Awstats en Debian 10 con NGINX"
tags: ["Linux","Debian","NGINX","Awstats"]
date: 2020-07-13T10:55:18-03:00
---

**Awstats** es una muy buena herramienta si buscamos tener algunas estadísticas básicas de nuestros sitios, sin insertar `js` en nuestro html, ni código de terceros que pueda llegar a afectar la privacidad de nuestros *visitantes*. En esta guía se indica la instalación de **Awstats** sobre **Debian 10** utilizando **NGINX**.  

*Nota: Este post es un registro de la instalación que realice en el actual servidor de este blog adaptando la guia de este [post](https://luxagraf.net/src/awstats-nginx-ubuntu-1804) y este [otro](http://itman.in/en/install-awstats-multiple-nginx-sites-debian-ubuntu/).*  


![dilbert_analytics_accurate](/blog/2020/07/13/instalar-awstats-en-debian-10-con-nginx/dilbert_analytics_accurate.jpg)



## Awstats 

En primer lugar necesitamos instalar *Awstats* desde los repositorios de *Debian 10*:

```bash
apt install awstats
```


## Perl, CPAN y GeoIP

Instalamos los paquetes que nos ayudaran a construir el modulo de `geoip` para identificar fácilmente de donde provienen las conexiones a nuestro sitio.

Para trabajar en esta etapa necesitamos acceder a la consola `cpan` ejecutando en la terminal:

```bash
cpan
```

Si es la primera ves que lo ejecutamos en nuestro servidor es necesario ejecutar los siguientes comandos dentro de la consola de `cpan` para instalar módulos:

```bash
cpan[1]> make install
cpan[1]> install Bundle::CPAN
``` 

Luego instalamos *GeoIP* y cerramos la consola de `cpan`:

```bash
cpan[1]> install Geo::IP
cpan[1]> exit
``` 

## Configuración NGINX

En primer lugar es necesario establecer el formato de log que van a utilizar nuestros sitios para que awstats pueda interpretarlos. Para eso creamos el archivo `/etc/nginx/conf.d/logformat.conf`. 

```bash
log_format main     '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
```

*Nota: Si bien esta configuración la podríamos establecer en el archivo `nginx.conf`, sugiero que utilicemos la carpeta `conf.d`, la cual es incluida en las configuraciones de NGINX y mantiene nuestra configuración organizada.* 

Aplicamos la configuración: 

```bash
systemctl reload nginx
```

## Configuración de Awstats para NGINX 

Las configuraciones de **Awstats** se encuentran en el path `/etc/awstats`. Ahí encontramos dos archivos: 

* `awstats.conf`: sirve de ejemplo para crear nuesvas configuraciones para cada sitio. 
* `awstats.conf.local`: en este archivo podemos establecer configuraciones globales a todos los sitios que hayamos configurado. 

Para crear una configuración de un nuevo sitio copiamos el archivo `awstats.conf` y le damos un nuevo nombre en formato `awstats.dominio.ltd.conf`. Por ejemplo, en mi caso: `awstats.tmpjg.xyz.conf`.
Dentro del nuevo archivo de configuración modificamos los siguientes parámetros: 

```bash
# Path en donde se almacena el log.
LogFile="/var/log/nginx/dominio.com.access.log"

# Dominio de nuestro virtual host en nginx.
SiteDomain="dominio.com"

# Path en donde vamos a almacenar la información de awstats.
DirData="/var/lib/awstats/dominio.com"

# Otros subdominios que queramos incluir en el log.
HostAliases="www.dominio.com"

# El formato de log que vamos a utilizar:

LogFormat = "%host - %host_r %time1 %methodurl %code %bytesd %refererquot %uaquot %otherquot"

# Comentamos el formato pre establecido para apache que trae awstats por defecto:
# LogFormat = 1
```

Como si indica en la configuración decidí crear una carpeta especifica para almacenar los datos del dominio, pero **Awstats** no la crea automáticamente. Procedemos a crear la carpeta y darle los permisos correspondientes: 

```bash
mkdir -p /var/lib/awstats/dominio.com
chmod 644 /var/lib/awstats/dominio.com
chown www-data:www-data /var.lib.awstats/dominio.com
```

Por ultimo, dentro del archivo `awstats.conf.local` copiamos los siguientes parámetros globales: 

```bash
# Para que awstats no realice un lookup reverso de las ips
DNSLookup = 0

# Utilizar el plugin geoip
LoadPlugin="geoip GEOIP_STANDARD /usr/share/GeoIP/GeoIP.dat"
```

## Actualizar Awstats y LogRotate

Para actualizar las estadísticas de **Awstats** usamos el comando: 

```bash
/usr/share/doc/awstats/examples/awstats_updateall.pl now -awstatsprog=/usr/lib/cgi-bin/awstats.pl
```

Si bien podríamos crear un cron para que se ejecute automáticamente, recomiendo que lo agreguemos dentro de la configuración del *LogRotate* de nginx. De esta manera cada ves que rote el log, podemos ejecutar la actualización. 
Para hacer esto tenemos que agregar el comando de actualización en el `prerotate` de la configuración de *LogRotate*. Debería quedar así: 


```bash
/var/log/nginx/*.log{
        daily
        missingok
        rotate 30
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
            /usr/share/doc/awstats/examples/awstats_updateall.pl now -awstatsprog=/usr/lib/cgi-bin/awstats.pl
            if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                run-parts /etc/logrotate.d/httpd-prerotate; \
            fi \
        endscript
        postrotate
            invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
    }
```

Podemos probar la configuración ejecutando: 

```bash
logrotate -f /etc/logrotate.d/nginx
```

## Acceso web a Awstats

Por ultimo tenemos que crear un acceso web a *Awstats* para poder ver nuestras estadísticas. En mi caso elegí crear un virtual host dedicado para cada dominio de mi servidor. La configuración: 

```bash
server {
    server_name stats.dominio.com;

    root    /var/www/dominio.com/stats;
    error_log /var/log/nginx/awstats.dominio.com.log;
    access_log off;
    log_not_found off;

    location / {
        return 301 /cgi-bin/awstats.pl?config=dominio.com;
    }

    location ^~ /awstats-icon {
        alias /usr/share/awstats/icon/;
    }

    location ~ ^/cgi-bin/.*\\.(cgi|pl|py|rb) {
        auth_basic            "Admin";
        auth_basic_user_file  /var/www/tmpjg.xyz/stats/awstats.htpasswd;

        gzip off;
        include         fastcgi_params;
        fastcgi_pass    unix:/var/run/php/php7.3-fpm.sock; # change this line if necessary
        fastcgi_index   cgi-bin.php;
        fastcgi_param   SCRIPT_FILENAME    /etc/nginx/cgi-bin-awstats.php;
        fastcgi_param   SCRIPT_NAME        /cgi-bin/cgi-bin.php;
        fastcgi_param   X_SCRIPT_FILENAME  /usr/lib$fastcgi_script_name;
        fastcgi_param   X_SCRIPT_NAME      $fastcgi_script_name;
        fastcgi_param   REMOTE_USER        $remote_user;
    }
}

```

Como se ve en la configuración, vamos a necesitar php para poder interactuar con perl en nuestro servidor de manera externa. Instalamos php: 

```bash
apt install php-fpm
```

y creamos el archivo `/etc/nginx/cgi-bin-awstats.php` con el siguiente contenido: 

```php
<?php
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a file to write to
);
$newenv = $_SERVER;
$newenv["SCRIPT_FILENAME"] = $_SERVER["X_SCRIPT_FILENAME"];
$newenv["SCRIPT_NAME"] = $_SERVER["X_SCRIPT_NAME"];
if (is_executable($_SERVER["X_SCRIPT_FILENAME"])) {
   $process = proc_open($_SERVER["X_SCRIPT_FILENAME"], $descriptorspec, $pipes, NULL, $newenv);
   if (is_resource($process)) {
       fclose($pipes[0]);
       $head = fgets($pipes[1]);
       while (strcmp($head, "\n")) {
           header($head);
           $head = fgets($pipes[1]);
       }
       fpassthru($pipes[1]);
       fclose($pipes[1]);
       fclose($pipes[2]);
       $return_value = proc_close($process);
   } else {
       header("Status: 500 Internal Server Error");
       echo("Internal Server Error");
   }
} else {
   header("Status: 404 Page Not Found");
   echo("Page Not Found");
}
?> 
```

*Nota: Los detalles de esta solución para perl, nginx y cgi estan en la [wiki de arch](https://wiki.archlinux.org/index.php/AWStats#Nginx).*

Por ultimo creamos un usuario y contraseña para darle seguridad a nuestro panel de estadísticas generando un archivo .htpasswd dentro del root que configuramos para nuestra virtual host de *Awstats*:

```bash
printf "usuario:`openssl password -apr1`\n" >> /var/www/dominio.com/stats/awstats.htpasswd
```

*Nota: `usuario` y `password` deben ser reemplazados por lo que queramos que quede.*

Luego recargamos la configuración de *NGINX* nuevamente:

```bash
systemctl reload nginx
```


Listo, ya podemos acceder a http://stats.dominio.com y ver nuestras estadísticas luego de poner usuario y contraseña. 

.
