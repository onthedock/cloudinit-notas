# Cloud-Init

La instalación de `cloud-init` en Debian:

```shell
sudo apt install cloud-init
```

> Una vez instalado `cloud-init`, cuando la VM arranque, intenta obtener la configuración de `cloud-init` desde algún _datasource_; si no hay ninguno configurado, la VM no arranca (se queda esperando contactar con una URL de AWS por defecto).

## NoCloud

- El volumen con sistema de ficheros `vfat` o `iso9660` debe tener la etiqueta `cidata`.
- el fichero `meta-data` debe contener la configuración del _datasource_ `ds=nocloud`
- el fichero de configuración de usuario `user-data` contendrá la información de las acciones de personalización de la VM.

### Crear un floppy virtual

En PowerShell, se puede crear un virtual floppy mediante:

```powershell
    New-VHD -Path <RutaAlFicheroDelVirtualFloppy>
    Set-VmFloppyDiskDrive -VMName <nombreVM> -Path <RutaAlFicheroDelVirtualFloppy>
    Copy-VMFile -SourcePath <nombreFichero> -DestinationPath <RutaAlFicheroEnVM> -FileSourceHost (* Pendiente de testear: primero hay que formatear el disco)
```

#### Formatear el floppy

```shell
    # Como root
    mkfs.vfat -n cidata          # Formateamos el disco
    mkdir /mnt/floppy            # Creamos el punto de montaje
    mount /dev/fs0 /mnt/floppy   # Montamos el disco
```

### Crear el fichero ISO

#### Ficheros

En la documentación oficial no queda claro cómo configurar los _datasource_ que usará `cloud-init`; sólo se indica cómo pasar esta configuración como parámetro del kernel durante el arranque, pero no cómo incluirlo en un fichero de configuración.

En función de la fuente consultada, esta información se indica en un fichero u otro.

`cloud-init` parece estar centrado únicamente en que la configuración se pase a través del `user-data` usando un proveedor _cloud_ y no se describe cómo configurar la aplicación en otros escenarios de _auto-consumo_ donde no hay un proveedor externo (OpenStack no cuenta).

- `meta-data`

  ```txt
  ds: nocloud
  instance-id: cloudinit-vm01
  local-hostname: cloudinit-vm01
  ```

- `user-data`

  ```txt
  #cloud-config
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