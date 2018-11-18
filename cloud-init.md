# Cloud-Init

La instalación de `cloud-init` en Debian:

```shell
sudo apt install cloud-init
```

> Una vez instalado `cloud-init`, cuando la VM arranque, intenta obtener la configuración de `cloud-init` desde algún _datasource_; si no hay ninguno configurado, la VM no arranca (se queda esperando contactar con una URL por defecto).

## NoCloud

- El volumen con sistema de ficheros `vfat` o `iso9660` debe tener la etiqueta `cidata`.
- <div style=" text-decoration: line-through;">El fichero `meta-data` debe contener la configuración del _datasource_ `ds=nocloud`</div>
  - No es necesario: durante el arranque de `cloud-init` inicialmente se buscan los _datasource_ locales, por lo que si están presentes, se usan (incluso antes de que la red esté disponible). También puede cargarse la configuración desde una URL de autoconfiguración, normalmente [http://169.254.169.254/](http://169.254.169.254).
- El fichero de configuración de usuario `user-data` contendrá la información de las acciones de personalización de la VM.
- El fichero de configuración `meta-data` contiene información sobre el nombre de la instancia, etc.

> Los ficheros de configuración son ficheros en formato YAML.

### Crear el fichero ISO

No existe ningún _cmdlet_ nativo para crear ficheros ISO en PowerShell.

El _script_ proporcionado en la TechNet `New-ISOFile` no me ha funcionado en Windows 10.

La solución ha sido usar la versión para Windows de `mkisofs` del paquete `cdrtools`, que he descargado (junto con las bibliotecas de funciones) de [Cdrtools: Win x86_64 Binaries](https://www.student.tugraz.at/thomas.plank/index_en.html).

El binario de `mkisofs` se encuentra en una carpeta llamada `cdrtools`. Desde esta carpeta, ejecutamos:

```shell
mkisofs.exe -volid cidata -joliet -rational-rock -iso-level 2 -output ..\initv3.iso ..\initimg
```

- `-volid` especifica el volumen de la ISO
- `-output` especifica la ruta y nombre del fichero ISO generado
- `-joliet` especifica extensiones para usar nombres UCS-2 en Juliet
- `-rational-rock`
- `-iso-level` 2 (especifica el nivel de ISO 9660)

`cloud-init` se ejecuta en cada arranque. Si no encuentra _datasource_ de configuración local intenta buscarlo en red, lo que bloquea el arranque. Para evitar que `cloud-init` se ejecute en cada arranque, al final de la configuración creamos el fichero  `/etc/cloud/cloud-init.disabled`. Si `cloud-init` encuentra este fichero, no se ejecuta.

#### Ficheros

- `meta-data`

Parece que no es necesario especificar nada en este fichero. He realizado una prueba con el hostname y tiene prioridad el hostname especificado en el fichero user-data.

  ```txt
  ds: nocloud
  instance-id: cloudinit-vm01
  local-hostname: cloudinit-vm01
  ```

En el fichero `meta-data` podemos especificar la configuración de una IP estática. La información recogida se guarda en `/etc/network/interfaces.d/50-cloud-init.cfg`. Si no reiniciamos el equipo, tenemos dos IPs asignadas, la obtenida por DHCP (por la configuración en `/etc/network/interfaces`) y la estática.

- `user-data`

Especificando el nombre del hostname en este fichero, se establece correctamente pero no se actualiza el fichero `/etc/hosts`, lo que genera errores indicando que no se identifica el nombre de la máquina.

> La solución pasa por usar el módulo: `manage_etc_hosts: true`, lo que actualiza el fichero `/etc/hosts`.
> En el fichero `/etc/hosts` también encontramos una entrada para el _hostname_ especificado en el fichero `meta-data`, asociado a `127.0.0.1 vm-meta-hostname.localdomain`.

  ```txt
  #cloud-config
  # TBD
  #touch /etc/cloud/cloud-init.disabled # Disable cloud-init after first boot
  ```

- `cloud-config`

  - 1 [paulgorman.org/technical](https://paulgorman.org/technical/cloud-init.txt.html)
  
    En la entrada _Cloud-init_ del blog [paulgorman.org/technical](https://paulgorman.org/technical/cloud-init.txt.html) encontramos que la configuración del _datasource_ se realiza en el fichero `/etc/cloud/cloud.cfg`.
  
    En el artículo indica que durante el arranque, antes de que la red esté disponible, `cloud-init` busca _datasources_ locales: un sistema `vfat` o `iso9660` o una URL _mágica_ que suele ser [http://169.254.169.254/meta-data/](http://169.254.169.254/meta-data/)

    También indica que los ficheros de `cloud-init` se encuentran en `/etc/cloud/` y en `/var/lib/cloud`.

  - 2 Ref: [cloud-init Configuration](https://www.suse.com/documentation/suse-caasp-1/book_caasp_deployment/data/cloud-init_configuration.html)
  
    ```txt
    #cloud-config
    datasource:
        NoCloud:
        # default seedfrom is None
        # if found, then it should contain a url with:
        #    <url>user-data and <url>meta-data
        # seedfrom: http://my.example.com/<path>/
    ```

- NoCloud [NoCloud en ReadTheDocs](https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html)
  
  En la documentación oficial sólo se indica que los ficheros que se buscan son el `user-data` y el `meta-data`. El fichero de configuración `cloud-config` no se indica.

  > De la gestión de `cloud-config` parece que se encarga el proveedor del cloud y el usuario no tiene acceso a él.

```txt
  password: p@ssw0rd
  chpasswd: { expire: False }
  ssh_pwauth: True
  ```

Usando un CD como medio de configuración, ha funcionado a la primera.

`cloud-init` se ejecuta en cada arranque, así que hay que encargarse de deshabilitarlo después de la primera configuración (porque si se elimina el CD con la configuración, se vuelve a quedar encallado durante el arranque).

> Debo revisar la documentación porque parece que podrían colocarse los ficheros de configuración en la carpeta `scripts/perboot`
>
> Tengo pendiente probar a lanzar una VM usando alguna de las _cloud images_ proporcionadas por Ubuntu con `cloud-init` ya instalado (p.ej: [https://cloud-images.ubuntu.com/bionic/current/](https://cloud-images.ubuntu.com/bionic/current/)). Se ofrecen en diferentes formatos de disco para hypervisores, entre ellos un disco VHD para Hyper-V.

Como referencia, copio el contenido de los fichero `meta-data` y `user-data` que he estado usando (son _WIP_):

- `user-data`

  ```txt
  #cloud-config
  manage_etc_hosts: true
  hostname: vm-userdata-04
  fqdn: vm-userdata-04.ameisin.local
  
  default:
  mounts:
    - [ swap, null ]
  
  # package_upgrade: true # Causes an upgrade (apt get upgrade -y)
  packages: ['figlet']
  
  timezone: "Europe/Madrid"
  
  runcmd:
    - /usr/bin/localectl set-keymap ca_ES.ISO-8859-1
  
  # users:
    # - default
    # - name: operador
      # groups: sudo
      # shell: /bin/bash
      # sudo: ['ALL=(ALL) NOPASSWD:ALL']
  #write_files:
  #  - path: /etc/cloud/cloud-init.disabled
  ```

- `meta-data`

  ```txt
  local-hostname: vm-meta-04
  instance-id: cloudimg
  network-interfaces: |
    iface eth0 inet static
    address 192.168.1.199
    network 192.168.1.0
    netmask 255.255.255.0
    broadcast 192.168.1.255
    gateway 192.168.1.1
  hostname: ci-meta-data-hostame
  ```


## 2018-11-18

El fichero de configuración de `cloud-init` está (en la máquina arrancada con cloud-init) en `/etc/cloud/cloud.cfg`. Revisando el contenido del fichero, aparece un usuario `debian` que efectivamente se encuentra en la máquina y que pertenece a los grupos indicados en el fichero de configuración.

Para poder actualizar el nombre de usuario de la máquina desplegada por cloud-init, es necesario comentar la línea `- update_hostname` en el fichero ect/cloud/cloud.cfg`, ya que en caso contrario se restablecerá al nombre original. Otra opción es eliminar `set_hostname` y `update_hostname` y cambiarlo por `set_hostname_from_dns` para que se establezca el nombre del hostname en función de lo definido en el DNS. [Ref:[Instalación y configuración de cloud-init en Linux](https://www.ibm.com/support/knowledgecenter/es/SSXK2N_1.3.1/com.ibm.powervc.standard.help.doc/powervc_install_cloudinit_hmc.html#powervc_install_cloudinit_hmc__ubuntu)]

En el fichero de configuración `/etc/cloud/cloud.cfg` se establecen _paths_ adicionales para ficheros de _cloud-init_: `/var/lib/cloud` (*cloud_dir*), `/etc/cloud/templates` (*template_dir*) y `/etc/init` (*upstart_dir*).

Para añadir más usuarios durante la configuración inicial del sistema, añadimos una sección de `users` en el fichero `user-data`. Si la primera entrada en la sección `users` es `default`, el usuario `cloud-user` se creará junto al resto de usuarios; en caso contrario, el usuario `cloud-user` no se crea.

```shell
#cloud-config
users:
  - default
  - name: foobar
    gecos: User N. Ame
    selinux-user: staff_u
    groups: users,wheel
    ssh_pwauth: True
    ssh_authorized_keys:
      - ssh-rsa AA..vz user@domain.com
...
```

> Atención , que no puede haber espacios entre los nombres de los grupos a los que pertenece el usuario.

 Si queremos ejecutar comandos en las fases *bootcmd* y/o *runcmd*, podemos añadir una sección al fichero *user-data* para cada uno [Chapter 3. Setting up cloud-init](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/installation_and_configuration_guide/setting_up_cloud_init):

```shell
...
bootcmd:
 - echo New MOTD >> /etc/motd
runcmd:
 - echo New MOTD2 >> /etc/motd
...
```

Para añadir un usuario al grupo de SUDOers:

```shell
sudo: ["ALL=(ALL) NOPASSWD:ALL"]
```

Para crear configuración de red estática hay que añadir una sección *network-interfaces* en el fichero `meta-data`. La configuración estática no reinicia la red, por lo que la configuración establecida vía DHCP sigue activa.




```
