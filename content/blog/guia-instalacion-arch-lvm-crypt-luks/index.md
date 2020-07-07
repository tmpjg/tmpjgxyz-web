---
title: "Guia: Instalación de Arch LVM Crypt LUKS"
date: 2020-07-06T13:00:00-03:00
tags: ["Linux", "Arch", "LVM", "LUKS", "Crypt","GRUB", "Gnome", "Xfce", "Kde"]
---

A esta altura, si llegaste a este post, doy por entendido que sabes lo que es [Arch Linux](https://www.archlinux.org/) y seguramente ya cruzaste *150* guias que explican como instalarlo. Por eso, mi intención en esta guia, es poder estudiar y comprender lo que hacemos, modificamos e instalamos en cada uno de los pasos que solemos `copiar-pegar` de las guias.

Ademas, esta guia considera la necesidad de cifrar nuestros discos con **luks** dentro de volúmenes **LVM** utilizando un disco *ssd*. En mi opinión, la velocidad de cpu/ram y discos de hoy en día hace que no haya pretextos para no proteger el acceso físico a nuestra información. Ademas, nos obliga a controlar nuestros backups en caso de que la caguemos. 

*Nota: Esta guia esta pensada para realizar la instalación sobre Hardware *UEFI*.*

Empecemos... 

## Inicio

Lo primero es generar un usb desde el que podamos iniciar el sistema. En este [link](/corregir-este-link) hay una guia de como hacerlo desde linux con el comando `dd`. Si no tenemos esta herramienta sugiero utilizar [balena-etcher](https://www.balena.io/etcher/). 
Cuando tengamos listo nuestro usb, podemos iniciar la instalación.

### Distribución del teclado

Para asegurarnos de no equivocarnos en ningún comando/password es recomendable configurar la distribución de nuestro teclado. En mi caso, es *Español(LatinoAmerica)* y para eso utilizamos el comando `loadkeys`. Si queremos listar la lista de distribuciones disponibles podemos ejecutar `ls /usr/share/kbd/keymaps/**/*.map.gz`. En mi caso es `la-latin1`

```bash
loadkeys la-latin1
```

### Conexión a Internet

Para poder realizar la instalación es necesario una conexión a Internet con la que se descargaran los paquetes necesarios. 

#### wifi

Si necesitamos realizar la instalación utilizando wifi, sugiero utilizar la herramienta [`iwctl`](https://wiki.archlinux.org/index.php/Iwd#iwctl).
Ambas herramientas nos facilitan un menú sencillo para listar las redes disponibles y conectarnos. 

```bash
iwctl
```

Luego, en la consola de *iwd* listamos las interfaces disponibles (en mi caso es `wlan0`):

```bash
[iwd]# device list
```
Buscamos y listamos las redes disponibles:

```bash
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
```

Nos conectamos a la red: 

```bash
[iwd]# station wlan0 connect NOMBRE_DE_LA_RED_WIFI
```

Escribimos nuestra password y salimos de la consola *iwd*:

```bash
[iwd]# exit
```
*Nota: En mi caso, al querer conectar a un red, recibía un error al querer crear el perfil de conexión. Para poder configurar la red fue necesario desactivar la interfaz y dejar que la herramienta de conexión la active. Para esto tuve que ejecutar `ip set wlan0 down`.* 

*Nota II: Sugiero antes de continuar la instalación que probemos nuestra conexión a Internet con un `ping kernel.org`. Si nos conectamos a la red wifi pero seguimos sin Internet, es posible que la interfaz no haya solicitado una ip al router. Esto podemos verificarlo ejecutando `ip a`. Si necesitamos forzar la solicitud de una ip podemos ejecutar `dhcpcd`.*

#### Ethernet

Nunca tuve inconvenientes con el kernel actual para utilizar un puerto ethernet, pero en caso de tener problemas sugiero buscar el modelo de nuestra placa y revisar la [Wiki](https://wiki.archlinux.org/) de Arch. 

### Elegir mirror 

Todos los gestores de paquetes utilizan uno o varios *mirrors* que son repositorios en donde se encuentran los paquetes que instalamos. En este caso es recomendable que revisemos la lista de repositorios que se encuentra en `/etc/pacman.d/mirrorlist` y elijamos la que consideremos mas que sera mas rápida por su ubicacion al momento de descargar paquetes. Para elegirla es necesario que la copiemos y peguemos la url al principio de la lista, de esta manera, al momento de descargar un paquete siempre intentara con el primer repositorio en la lista, en caso de fallar, continuara al siguiente.  

Para editar este archivo como el resto de los que es necesario editar podemos utilizar la herramienta `vim` o `nano`. (Si no tenemos experiencia con `vim` sugiero ir directamente por `nano` que funciona como un editor de texto simple).

```bash
nano /etc/pacman.d/mirrorlist
```

### Actualizar paquetes de instalación y Pacman

En este paso ejecutamos utilizamos por primera ves [pacman](https://wiki.archlinux.org/index.php/Pacman_(Espa%C3%B1ol)), el gestor de paquetes de Arch. Este es el momento para probar la velocidad de descarga y actualizar los repositorios y herramientas que utilizaremos para realizar la instalación. 

```bash 
pacman -Syy
``` 

El parámetro `-S` se utiliza para sincronizar los paquetes que tenemos instalado, el `y` se utiliza para descargar la base de datos con la lista de paquetes y sus versiones, y al utilizarlo dos veces forzamos que esto se haga aunque la base este actualizada. 

## Particionado (EFI, FS, LUKS, LVM, Swap)

En este paso definimos como queremos dividir nuestro disco para instalar el sistema. Es necesario tener en cuenta que en un sistema cifrado, el directorio `/boot` no puede estar cifrado ya que en el mismo se encuentra toda la información necesaria para iniciar nuestro Sistema Operativo. Ademas si utilizamos un sistema con *UEFI*, el directorio `/boot/EFI` tiene que estar en una partición *FAT32*.

*Nota Swap: [Swap](https://es.wikipedia.org/wiki/Espacio_de_intercambio) es un espacio en donde nuestro Sistema Operativo copiara paginas de memoria RAM a nuestro Disco en caso de que sea necesario. Con respecto a la cantidad de espacio asignada y el método de creación hay muchas discusiones, por lo que sugiero investigar en base al uso que se le va a dar en nuestro equipo. Al momento de realizar esta guia yo utilizo una laptop que suelo poner en suspención muy seguido y con bastante RAM ocupada, por lo que elegí que su tamaño sea de la misma cantidad de RAM que tengo (16GB)*

*Nota root: Entiendo que mucha gente prefiere separar su partición raíz `/` de la de su `/home`, pero en mi caso no veo lo veo necesario.(Ademas, nuevamente me obliga a estar atento a mis backups por si la cago)*

Este es un esquema simple de lo que vamos a preparar en el particionado de disco. `nvme0n1` es mi disco, y `nvme0n1p1`, `nvme0n1p2`, `nvme0n1p3` son las particiones. Dentro de la partición 3 (`nvme0n1p3`) configuraremos un lvm con un grupo de volúmenes que contendrá los volúmenes lógicos que se utilizaran como *swap* y *root* (`/`).

```bash
nvme0n1 (Disco)         
├─nvme0n1p1 (/boot/EFI) [fat32]
├─nvme0n1p2 (/boot) [ext4]   
└─nvme0n1p3 (LVM) [luks]          
  └─lvgroup0              
      ├─lvgroup0-lv_swap (Swap)
      └─lvgroup0-lv_root (/) [ext4]
(uso) [formato]
```

### fdisk 

Para realizar el particionado prefiero [*fdisk*](https://www.tldp.org/HOWTO/Partition/fdisk_partitioning.html), es la herramienta que yo conozco. Si preferimos utilizar alguna mas sencilla o particionar el disco antes de iniciar la instalación no hay problema. 
El ssd de esta instalación tiene un tamaño de 500GB y se el sistema lo llama nvme0n1. Para verificar los discos disponibles podemos ejecutar `fdisk -l`. 

1. Seleccionamos el disco a particionar
    ``` bash
    fdisk /dev/nvme0n1
    ``` 
2. Creamos una nueva tabla de parcion GPT 
    ```bash 
    Command (m for help): g 
    ```
3. Creamos la primer partición para `/boot/EFI`
    ```bash
    Command (m for help): n
    ```
    Dejamos el valor de `Partition number` y `First cylinder` por defecto apretando *Enter*. Al momento de solicitarnos el ultimo cilindro de la particion (`Last cylinder or +size or +sizeM or +sizeK`) escribimos `+500M` (500Mb).
    Por defecto va a crear el tipo de particion en modo *Linux filesystem*, pero esta particion necesita ser tipo `EFI`. Para cambiar el tipo de particion ejecutamos `t`
    ```bash
    Command (m for help): t
    ```
    Elegimos el nro de particion *1* y luego el tipo de particion *EFI* con el numero `1`. 
   
   (*Nota: Si necesitamos listar los tipos de particiones y sus numeros podemos ejcutar `l`*)

4. Creamos la segunda partición para `/boot` de la misma manera que creamos la partición anterior, pero dejamos el tipo de partición en `Linux filesystem`. 

5. Creamos la tercera y ultima partición que contendrá el *LVM*. Seguimos los mismos pasos que antes, pero al momento de solicitarnos el cilindro (`Last cylinder or +size or +sizeM or +sizeK`) apretamos `Enter` para seleccionar el ultimo y utilizar el espacio restante. 
En este caso cambiamos el tipo de partición a `LVM` usando la opción `t` y el numero `30`. 

6. Guardamos los cambios y los aplicamos en el disco con la opción `w`.

### crypt LUKS

Antes de crear el LVM, vamos a cifrar la tercera partición con *LUKS*:

```bash 
cryptsetup luksFormat /dev/nvme0n1p3 
```
*NOTA: Escribimos YES (en mayúsculas y escribimos la password de nuestra partición cifrada)*

Y luego "abrimos" la partición cifrada para poder utilizarla: 

```bash
cryptsetup open --type luks /dev/nvme0n1p3 lvm
```

### LVM

Antes de formatear vamos a crear el *LVM* que estará cifrado y contendrá la *swap* y nuestro *filesystem*.

Primero tenemos que inicializar nuestro `Physical Volume` (*PV*) con el *dataalignment* para un *SSD*. 

```bash
pvcreate --dataalignment 1m /dev/mapper/lvm
```

Luego creamos un grupo de volúmenes con nombre `lvgroup0` dentro de nuestro *PV*:

```bash
vgreate lvgroup0 /dev/mapper/lvm 
```

Por ultimo creamos el volumen lógico para la *swap* (16GB) con nombre `lv_swap`:

```bash
lvcreate -L 16G lvgroup0 -n lv_swap
```
y el volumen lógico para la raíz (`/`) con nombre `lv_root` utilizando la opción `100%FREE` para indicarle que utilice el resto del espacio disponible:

```bash
lvcreate -l 100%FREE lvgroup0 -n lv_root
```

### Formateo

Ahora vamos a darle formato a las particiones que creamos. La especificación *UEFI* espera que la partición `/boot/EFI` sea *FAT32* ([info](https://wiki.archlinux.org/index.php/EFI_system_partition_(Espa%C3%B1ol))) por lo que ejecutamos: 

```bash
mkfs.fat -F32 /dev/nvme0n1p1
``` 

Luego formateamos la partición para `/boot`: 

```bash
mkfs.ext4 /dev/nvme0n1p1
```

Y por ultimo formateamos el volumen lógico para la raíz `/`:

```bash
mkfs.ext4 /dev/lvgroup0/lv_root
```

### Swap

Por ultimo crearemos y activaremos la *swap* con los siguientes comandos: 

```bash
mkswap /dev/lvgroup0/lv_swap
swapon /dev/lvgroup0/lvswap
```

## Instalación 

A esta altura ya podemos comenzar la instalación en si misma. Para esto es necesario montar todas las particiones que creamos en `/mnt` para instalar los diferentes componentes del sistema y sus configuraciones. 

### Montaje

Primero montamos la raíz (`/`): 

```bash 
mount /dev/lvgroup0/lv_root /mnt
```

Creamos un directorio para `/boot` y lo montamos: 

```bash
mkdir /mnt/boot
mount /dev/nvme0n1p2 /mnt/boot
```

*Nota: Todavía no es necesario montar la partición que preparamos para `/boot/EFI`*

### Fstab

El archivo [*fstab*](https://wiki.archlinux.org/index.php/Fstab_(Espa%C3%B1ol)) le indica al sistema que particiones o sistemas de archivos deben ser montados en su arranque. 

Para generar un archivo fstab en base a lo que montamos en el `/mnt` y con la configuración de la swap, creamos el directorio `/etc` y utilizamos la herramienta `genfstab`: 

```bash 
mkdir /mnt/etc
genfstab -U -p /mnt > /mnt/etc/fstab
```

### pacstrap 

[Pacstrap](https://jlk.fjfi.cvut.cz/arch/manpages/man/extra/arch-install-scripts/pacstrap.8.en) es una herramienta que nos permite instalar paquetes en un nuevo directorio raíz (`/mnt`). Lo utilizaremos para instalar el paquete [`base`](https://www.archlinux.org/packages/core/any/base/) que es un meta-paquete (conjunto de paquetes) que contiene la base de un sistema Arch. 
Recomiendo tambien instalar el paquete [`base-devel`](https://www.archlinux.org/groups/x86_64/base-devel/) que contiene herramientas necesarias para compilar muchos paquetes que posiblemente necesitemos. 


```bash 
pacstrap -i /mnt base base-devel
```

### arch-chroot

La herramienta `arch-chroot` nos sirve para ejecutar un [`chroot`](https://wiki.archlinux.org/index.php/Chroot#Using_arch-chroot) montando algunas otras particiones que son necesarias para poder cambiar nuestra raíz (actualmente en el usb que montamos) a `/mnt` y de esta manera poder trabajar como si estuviésemos usando la nueva instalación. 
*Nota: `arch-chroot` puede ser de mucha utilidad si rompemos o instalamos mal nuestro sistema para volver a acceder a este punto (luego de montar las particiones) y reparar/corregir el problema que se presente.* 

```bash
arch-chroot /mnt 
```

### Instalación de paquetes básicos

#### Editor de texto

Si o si vamos a necesitar un editor de texto para modificar las configuraciones de nuestro sistema en este punto. Como comente antes, yo utilizo *vim* pero te recomiendo utilizar el que te sea mas cómodo. 

```bash
pacman -S vim
```

#### Kernel Linux

Sugiero instalar el kernel Linux mas reciente y ademas su versión LTS (*LONG TIME SUPPORT*) para que este ultimo nos sirva de backup si hay algún problema con el kernel mas nuevo al momento de actualizar nuestro sistema. 
Ademas sugiero la instalación de los paquetes [*headers*](https://www.archlinux.org/packages/core/x86_64/linux-headers/) y el paquete [`linux-firmware`](://www.archlinux.org/packages/core/any/linux-firmware/) para asegurarnos que no tendremos problemas con controladores propietarios de nuestro hardware.

*Nota: Si sabemos que nuestro sistema soporta completamente drivers libres que se incluyen en el kernel de linux, sugiero no instalar `linux-firmware` y apoyar a los desarrolladores que trabajaron en los drivers que utilizamos.*

```bash
pacman -S linux-lts linux-lts-headers linux linux-headers
```

#### Network 

Para poder utilizar nuestros dispositivos de red y controlarlos necesitamos la herramienta [`networkmanager`](https://wiki.archlinux.org/index.php/NetworkManager_(Espa%C3%B1ol))  y si ademas vamos a utilizar wifi es necesario instalar los paquetes [`wpa_supplicant`](https://wiki.archlinux.org/index.php/Wpa_supplicant) (*autenticacion wep y wpa*), [`wireless_tools`](https://www.archlinux.org/packages/core/x86_64/wireless_tools/)(*kit de herramientas para controlar nuestro wifi*), [`iwd`](https://www.archlinux.org/packages/community/x86_64/iwd/)(*administra perfiles de conexión*). 
También recomiendo instalar el paquete [`dialog`](https://www.archlinux.org/packages/core/x86_64/dialog/) que sirve para generar cuadros de dialogo en la terminal(muy útil si necesitamos conectarnos a la red wifi cuando rompamos la interfaz).

*Nota: Aunque nuestro equipo no tenga wifi, sugiero instalar estos paquetes de todos modos. No interfieren en el rendimiento de nuestro sistema y pueden ser de utilidad si en algún momento queremos utilizar una placa wifi.*

```bash
pacman -S networkmanager #if wifi wpa_supplicant wireless_tools netctl dialog
```

Por ultimo activamos el `networkmanager` para que se inicie junto a nuestro sistema: 

```bash 
systemctl enable NetworkManager
```

#### Usuario y password 

Es importante que establezcamos una contraseña para el usuario `root`, pero ademas crear el usuario que vamos a utilizar nosotros y darle acceso `sudo`. 

Para cambiar la contraseña del usuario `root` utilizamos la herramienta `passwd`: 

```bash
passwd root
```

Luego creamos nuestro usuario con la herramienta `useradd` con el parámetro `-m` para que cree nuestra carpeta en `/home`, `-g` para configurar el grupo primario `users`, `-G` para agregarlo en el grupo `wheel` (*grupo de sudo*) y por ultimo el nombre de usuario:

```bash
useradd -m -g users -G wheel usuario
```

Por ultimo establecemos el password de nuestro nuevo usuario: 

```bash 
passwd usuario
```

Por ultimo tenemos que activar el comando `sudo` para el grupo `wheel`. Para esto editamos el archivo `/etc/sudoers` y descomentamos la linea (82) que contiene `%wheel ALL=(ALL) ALL`.

*Nota: Si el archivo `/etc/sudoers` no existe, es posible que tengamos que instalar sudo:*
```bash
pacman -S sudo
```

#### Hostname y Hosts

Para configurar el nombre nuestro equipo solamente es necesario escribirlo en el archivo `/etc/hostname`: 

```bash
echo ${NOMBRE_DE_NUESTRO_EQUIPO} > /etc/hostname
```

Ademas es importante crear la configuración básica de nuestro [archivo hosts](https://es.wikipedia.org/wiki/Archivo_hosts) creando y editando el archivo `/etc/hosts` con la siguiente información: 

```bash
# Static table lookup for hostnames.
# See hosts(5) for details.

127.0.0.1       localhost
::1             localhost
127.0.1.1       ${NOMBRE_DE_NUESTRO_EQUIPO}

``` 

*Nota: ${NOMBRE_DE_NUESTRO_EQUIPO} debe ser reemplazado por el nombre que queramos configurar para nuestro equipo.*

#### locales

Los (locales)[https://wiki.archlinux.org/index.php/Locale_(Espa%C3%B1ol)] definen el juego de caracteres y configuraciones regionales en base a nuestra región. 
Primero tenemos que descomentar (borrando el `#`) el *locale* que corresponda a nuestra región en el archivo `/etc/locale.gen`. Luego ejecutamos: 

```bash 
locale-gen
```


### mkinitcpio (initramfs)

[`mkinitcpio`](https://wiki.archlinux.org/index.php/Mkinitcpio_(Espa%C3%B1ol)#Introducci%C3%B3n) es un script bash que crea un [*initramfs*](https://es.wikipedia.org/wiki/Initrd) moderno que se encarga de preparar un pequeño sistema en memoria para poder manejar el inicio del sistema. 
Antes de generar la configuración de *mkinitcpio* tenemos que agregar algunos *HOOKS* para que este mini-sistema pueda manejar sistemas cifrados luks y lvm.
Editamos el archivo `/etc/mkinitcpio.conf` y en la linea descomentada de *HOOKS* (52), agregamos `encrypt` y `lvm2`: 

`HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)`

Luego instalamos el manejador de lvm [`lvm2`](https://www.archlinux.org/packages/core/x86_64/lvm2/):

```bash
pacman -S lvm2
```

Por ultimo, generamos la configuración para los kernels que tengamos instalados: 

```bash
mkinitcpio -p linux-lts  
mkinitcpio -p linux
```

#### GRUB

[Grub](https://es.wikipedia.org/wiki/GNU_GRUB) es la herramienta que nos ayudara a elegir nuestro sistema y kernel. Con el podemos pasarle el mando a nuestro kernel para que inicie el sistema y ademas nos da la posibilidad de tener múltiples sistemas operativos en nuestro equipo.

Antes de continuar tenemos que instalar `grub`, `efibootmgr`(herramienta para modificar el boot manager de EFI), `dosfstools`(herramientas de DOS), `os-prober` (herramienta para detectar otros sistemas en los discos) y `mtools` (herramientas para acceder a discos MS-DOS)

```bash
pacman -S grub efibootmgr dosfstools os-prober mtools 
```

Una ves que tengamos las herramientas instaladas tenemos que configurar los [parámetros](https://wiki.archlinux.org/index.php/Kernel_parameters_(Espa%C3%B1ol)) para que este listo para poder levantar nuestra partición cifrada y su lvm. Para eso editamos el archivo `/etc/default/grub` y en la linea (6) que contiene las opciones de `GRUB_CMDLINE_LINUX_DEFAULT`, entre `loglevel=3` y `quiet` el parámetro `cryptdevice=/dev/nvme0n1p3:lvgroup0:allow-discards`
El resultado de ambos cambios deberia ser:

`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=/dev/nvme0n1p3:lvgroup0:allow-discards quiet"`

Dentro del parámetro `cryptdevice` declaramos cual es la unidad cifrada que kernel tiene que abrir en el arranque y configuramos el [soporte discard/TRIM](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Specialties_(Espa%C3%B1ol)#Soporte_Discard/TRIM_para_unidades_de_estado_s%C3%B3lido_(SSD)) para ssd. 


Ademas debemos descomentar la linea (13) que habilita el inicio de unidades cifradas con *LUKS*:

`GRUB_ENABLE_CRYPTDISK=y`

En este paso vamos a usar la herramienta [`grub-install`](https://jlk.fjfi.cvut.cz/arch/manpages/man/grub-install.8) para instala Grub, indicando nuestra la arquitectura de nuestra plataforma y contemplando el uso de UEFI.
Antes es necesario montar la particion para EFI creando el directorio `/boot/EFI`:

```bash
mkdir /boot/EFI
mount /dev/nvme0n1p1 /boot/EFI
```

Luego ejecutamos la instalación: 

```bash
grub-install --target=x86_64-efi --botloader-id=grub_uefi --recheck
```

Y por ultimo generamos el archivo de configuración que utilizara nuestro Grub al iniciar: 

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

A partir de este punto, nuestro sistema Arch ya esta listo para iniciar, pero vamos a continuar instalando algunos paquetes extras que ayudaran al rendimiento de nuestro equipo y nos facilitaran una interfaz gráfica. 


#### Microcode

Es recomendable instalar los microcodigos que correspondan a nuestro procesador para asegurarnos un correcto funcionamiento del mismo. 

Intel: 

```bash
pacman -S intel-ucode
```

Amd:

```bash
pacman -S amd-ucode
```

#### Drivers de Video

Es recomendable instalar los drivers de video que mejor funcionen con nuestro hardware. Para comprobar nuestro hardware de video podemos ejecutar:

```bash
lshw -c video 
```
##### Amd Intel

Para placas Amd o Intel sugiero `mesa`: 

```bash
pacman -S mesa 
```
##### Nvidia

Para placas Nvidia: 

```bash 
pacman -S nvidia-lts nvidia nvidia-utils
```

*Nota: se agregan los drivers lts para mayor seguridad.* 

#### DE 

Obviamente sugiero instalar nuestro DE de preferencia. Voy a ejemplificar con lo que para mi son los 3 pilares de linux en el escritorio. En los 3 casos tenmos que instalar [`xorg`](https://wiki.gentoo.org/wiki/Xorg/Guide/es#.C2.BFQu.C3.A9_es_el_Servidor_X_Window.3F) que nos permite gestionar interfaces gráficas de usuario, un [gestor de pantalla o sesión](https://wiki.archlinux.org/index.php/Display_manager_(Espa%C3%B1ol)) y por ultimo nuestro entorno de escritorio. 

*Nota: En todos los casos es importante agregar nuestro gestor de pantalla o sesión al inicio de nuestro sistema con `systemctl enable ${GESTOR_ELEGIDO}`*

En los tres casos vamos a necesitar instalar xorg: 

```bash
pacman -S xorg
```

##### Gnome

```bash
pacman -S gdm gnome
systemctl enable gdm.service
```

##### Kde

```bash
pacman -S plasma plasma-wayland-session kde-applications 
systemctl enable sddm.service
```
##### Xfce

```bash
pacman -S lightdm xfce4 xfce4-goodies
systemctl enable lightdm.service
```

### Fin de la instlaacion 

Para finalizar la instalación ejecutamos el comando `exit` y luego `shutdown now`, retiramos nuestra unidad USB y al iniciar el equipo nuestro Arch ya debería ser funcional. 

## Referencias: 

[How to Install Arch Linux [Step by Step Guide]](https://itsfoss.com/install-arch-linux/)

[Arch Linux Full Installation Walkthrough (On LVM with Encryption)](https://www.youtube.com/watch?v=Lq4cbp5AOZM)

[TRIM](https://es.wikipedia.org/wiki/TRIM)

[lvm para torpes (parte 2)](https://blog.inittab.org/administracion-sistemas/lvm-para-torpes-ii/)
