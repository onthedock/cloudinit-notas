# Private Cloud Service

> Ideas _random_

El objetivo es montar un _AWS_ privado, sobre mis propias máquinas virtuales y/o Raspberry Pi (o similar, ver Pine64).

Como servicios de infraestructura, de momento necesito un DNS (`dnsmasq` _dockerizado_). Es probable que en algún momento necesite un LDAP (`openLDAP`). También necesito un sitio donde guardar el código (_Gitea_). Para la documentación, estoy dudando sobre utilizar el Wiki integrado en Gitea (asociado a cada repositorio) o usar un Wiki dedicado (como DokuWiki). Para estas ideas _random_, quizás podría usar un blog, seguramente usando un generador de sitios estáticos (como Hugo) y creando un _pipeline_ con Jenkins/Drone para publicar sobre Nginx o Caddy.

Otras "cosas" que necesitaría sería algo como Portainer o Rancher. Por supuesto, Kubernetes para mis necesidades de aplicaciones _containerizadas_. El problema es que las aplicaciones en Kubernetes necesitan _storage_ fuera del clúster. Una solución viable sería Gluster.

Empezando por la infraestructura, mientras esté usando Hyper-V, podría usar Vagrant para crear las máquinas, pero tengo la sensación de estar matando moscas a cañonazos. Otra alternativa es usar un script con PowerShell (debe ejecutarse con permisos de administrador); tengo desarrollado una primera versión funcional (al menos para crear máquinas) con el _custom-powershell-module_.

Otra opción sería abandonar Hyper-V y montar algo como LXC (o LXD). XenServer (no me convence, porque necesita un cliente web). No me gusta VMware por su política de intentar evitar que la gente use ESX en equipos "no homologados" (escribí hace un tiempo en mi blog al respecto). AWS parece que usa KVM, pero me parece complicado (aunque sinceramente no lo he probado nunca y debería darle una oportunidad).

En cuanto al sistema operativo, hasta ahora he estado haciendo pruebas siempre con Debian/Ubuntu, aunque estoy pensando en pasar o al menos probar también con CentOS, que es algo así como el estandard en las empresas (o RHEL, pero al tener coste, usaré CentOS).

## Centrando temas

- Kubernetes - sin discusión. Para el storage, necesito algo como Gluster. Para Gluster necesitaría un DNS (puesen usarse los ficheros `/etc/host`, pero es poco _future-proof_).
- Gitea (Requiere MySQL/MariaDB) y storage (Gluster)
- dnsmasq (storage / puede ser local). dnsmasq también puede hacer de DHCP, pero para ello debería estar en una red separada de la red física (todavía no tengo claro cómo podría hacerlo. Supongo que necesitaría conectarme al _host_ donde se ejecute el hypervisor y desde ahí sí que tendría acceso a las máquinas de la red interna. El problema es cuando quiera conectar a una URL dentro de la red privada (no podré acceder desde "fuera"). Por todo esto, hasta que no tenga claro cómo hacerlo, usaré una red "bridged", accesible desde cualquier sitio (de la red de casa).

Tanto Gitea como dnsmasq se pueden ejecutar desde un contenedor, lo que significa que se pueden tener varias réplicas en K8s. La única limitación a la hora de preferir `docker-compose` a K8S hasta ahora era que el almacenamiento (que en Docker está en un volumen), en Kubernetes tenía que ser local al _pod_ o montar una carpeta de uno de los nodos y hacer algún cambalache con la afinidad para que el pod no se moviera a otro sitio (feo, muy feo). Si es posible montar una "carpeta" remota (del clúster de Gluster de manera análoga a lo que pasaría en NFS) tendríamos la capacidad de montarla independientemente del nodo donde se lance el pod.

Tengo que mirar cómo hacer que Gluster y K8S trabajen juntos... Referencias: [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/#glusterfs)

