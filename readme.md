# Guía para utilizar docker en entornos de produción

Esta guía se basa en los puntos revisados automaticamente por la herramienta [Docker Bench for Security](https://github.com/docker/docker-bench-security.git)

Para una mejor compresión de la herramienta se recomienda leer el [siguiente documento](https://learn.cisecurity.org/benchmarks)

Para cumplir con todos los puntos del docker bench hay que realizar una serie de configuraciones, que se cubren en este documento (aún por agregar algunos puntos).

Todos los ejemplos estan basados desde un entorno Debian y fueron probados con `Docker version 19.03.5`


## Hardening del host

Más álla de docker este punto se trata de asegurar el host en el cual se hará el despliegue. Para esto son muchas las posibilidades y depende más de los requerimientos de cada caso/proyecto. Como mínimo, debe tener un Firewall habilitado:

```bash
$ sudo ufw enable

# agregar reglas dejando solo puertos necesarios (idealmente 22 (u otro asignado a ssh) y 80/443)
$ sudo ufw allow from ip_segura to any port 22
$ sudo ufw allow 80
$ sudo ufw allow 443
```

## Partición sólo para Docker

Hay que tener una partición destinada para `uso únicamente de Docker`, ya con la partición disponible y recien instalado Docker hay que realizar la siguiente configuración:

```bash
# Detener el servicio de Docker
$ sudo service docker stop

# montar la particion a usar en /var/lib/docker para esto editar el archivo /etc/fstab agregar una linea (UUID de acuerdo a la particion) como en el eljemplo:

UUID=XXXXXXXXXXXXXXXX /var/lib/docker ext4 errors=remount-ro 0 2

# reiniciar servicio Docker
$ sudo service docker start
```

## Archivos Dockerfile
Dentro de los archivos Dockerfile para imagenes creadas de forma propia (si es el caso) hay que cumplir con las siguientes reglas:

- No usar `update` sólo. Por ejemplo:
```docker
$ RUN apt-get update # evitar esto

# En su lugar usar
$ RUN apt-get update && install -y algun-paquete
```
- Agregar `healthcheck`

Agregar elemento `HEALTHCHECK` al Dockerfile de construcción de imagen, este elemento debe contener un comando que al ejecutarse y en base a su salida indica si el estado del contenedor es `healthy` o `unhealthy`.

Ejemplo de comando para revisar si hay un servicio web corriendo en el puerto 8000:

```docker
HEALTHCHECK CMD curl --fail http://localhost:8000/ || exit 1
```

## Habilitar contenido seguro

```bash
# establecer la variable de entorno DOCKER_CONTENT_TRUST siempre antes de hacer operaciones, por ejemplo pull, push, build, etc
$ export DOCKER_CONTENT_TRUST=1

# si se desea ignorar en acciones especificas (por ejemplo al hacer un pull de images no confiables) usar el parametro: --disable-content-trust
$ docker pull --disable-content-trust imagen:version
```

## Auditar archivos y directorios Docker

Para esto se usa la herramienta [audit](https://www.linux.com/tutorials/linux-system-monitoring-and-more-auditd/)

```bash
# instalar aplicacion auditd
$ apt-get update && apt-get install auditd

# agregar configuracion en /etc/audit/rules.d/audit-docker.rules con las siguientes reglas:

  -w /usr/bin/dockerd -p rwxa -k docker
  -w /usr/bin/docker -p rwxa -k docker
  -w /etc/docker -p rwxa -k docker
  -w /lib/systemd/system/docker.service -p rwx -k docker
  -w /lib/systemd/system/docker.socket -p rwxa -k docker
  -w /etc/default/docker -p rwxa -k docker
  -w /etc/docker/daemon.json -p rwxa -k docker
  -w /usr/bin/docker-containerd -p rwxa -k docker
  -w /usr/bin/docker-runc -p rwxa -k docker
  -w /usr/bin/containerd -p rwxa -k docker
  -w /usr/sbin/runc -p rwxa -k docker
  -w /var/lib/docker -p rwxa -k docker
  -w /etc/sysconfig/docker -p rwxa -k docker

# reiniciar auditd
$ systemctl restart auditd
# comprobar que esta tomando la reglas 
$ auditctl -l # debe impirmir las reglas en el output
```

## Configuración de demonio Docker

Aqui se trata de varias configuraciones:
- Habilitar namespaces (previene escalada de privilegios)
- Restringir tráfico entre contenedores
- Enrutar logging hacia syslog
- Prevenir acceso a nuevos privilegios
- Desablitar userland-proxy (para que los contendedores puedan ver IPs de origen reales)
- Mantener contenedores vivos durante caidas del daemon

En el siguiente ejemplo se muestran las configuracion respectivas, en el mismo orden, de la lista.

```bash
$ service docker stop

# en archivo /etc/docker/daemon.json agregar las siguientes consfiguraciones:
  "userns-remap": "default",
  "icc": false,
  "log-driver": "syslog",
  "no-new-privileges": true,
  "userland-proxy": false,
  "live-restore": true

# reiniciar docker
$ service docker start
```

## Asegurar acceso TSL para API de docker
Docker puede ser administrado a traves de una conexión remota, esta tarea consiste en proteger dicha conexión habilitando la verificacion a traves de certificados TSL.

Fuentes relacionadas:
- [Protect Docker daemon socket](https://docs.docker.com/engine/security/https/)
- [Securing docker with TSL certificates](https://tech.paulcz.net/blog/secure-docker-with-tls/)
- [Enable Docker Remote API](https://gist.github.com/kekru/974e40bb1cd4b947a53cca5ba4b0bbe5)
- [Enable TCP port 2375 for external connection to Docker](https://gist.github.com/styblope/dc55e0ad2a9848f2cc3307d4819d819f)


Pasos:

```bash
# estabecer en una variable de entorno con el nombre del host (o ingresar el nombre donde sea necesario)
$ export HOST=elhostdetuvps.com

# ubicarse en el path donde se almacenaran las llaves
$ cd /etc/docker/certs.d  # recomendado

# generar pem para la llave, recomendado establecer un password
$ openssl genrsa -aes256 -out ca-key.pem 4096
# generar pem del servidor
$ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem # llenar los datos que pedirá

# generar pem de llave del servidor
$ openssl genrsa -out server-key.pem 4096
# generar certificado de servidor
$ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr

# firmar la llave publica con el ca.pem
# definir IP desde la cuales se permite Conexion (recomendado usar VPN)
$ ALLOWED_IP1=192.168.1.1
$ echo subjectAltName = DNS:$HOST,IP:$ALLOWED_IP1 >> extfile.cnf  # se puede agregar mas IPs
$ echo extendedKeyUsage = serverAuth >> extfile.cnf
$ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf

# generar llave, certificado para el cliente, firmarlos
$ openssl genrsa -out key.pem 4096
$ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
$ echo extendedKeyUsage = clientAuth > extfile-client.cnf
$ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile-client.cnf

# a este punto, opcionalmente, se pueden borrar los archivos .csr y .cnf

# por segurdad cambiar permisos
$ chmod -v 0400 ca-key.pem key.pem server-key.pem
$ chmod -v 0444 ca.pem server-cert.pem cert.pem

# activar en Docker conexion TCP y el uso del TSL
$ service docker stop
```

En el archivo /etc/docker/daemon.json agregar las reglas (usando su path real*):

> **Solo sí de verdad desea activar el acceso remoto al API**
```json
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/path/certs/ca.pem",
  "tlscert": "/path/certs/server-cert.pem",
  "tlskey": "/path/certs/server-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
```

Para que la configuración funcione crear archivo /etc/systemd/system/docker.service.d/override.conf y agregar:
```bash
  # /etc/systemd/system/docker.service.d/override.conf
  [Service]
  ExecStart=
  ExecStart=/usr/bin/dockerd
```

Finalmente recargar configuración y reiniciar docker
```bash
$ systemctl daemon-reload
$ service docker start
```

## Limitar recursos para Docker
Usando el [ulimit](https://linuxhint.com/linux_ulimit_command/) de linux hay que limitar los recursos que docker puede utilizar.

Agregar en el archivo /etc/docker/daemon.json
```json
  "default-ulimits": {
      "core": {
          "Name": "core",
          "Hard": 0,
          "Soft": 0
      },
      "memlock": {
          "Name": "memlock",
          "Hard": 512,
          "Soft": 256
      },
      "nofile": {
          "Name": "nofile",
          "Hard": 1024,
          "Soft": 1024
      }
  }
```

## Agregar plugin de autenticación para clientes
La administración de docker permite realizar todas las acciones sin restricción de permisos por usuario, para lograr estabecer politicas de permisos por acción y por usuario existen plugins y es recomendable utilizar estas politicas en un entorno de producción.

Fuentes:
- [Plugin opa-docker-authz](https://github.com/open-policy-agent/opa-docker-authz)
- [OPA Docker](https://www.openpolicyagent.org/docs/latest/docker-authorization/)

```bash
# instalar plugin opa-docker-authz
$ docker plugin install --alias opa-docker-authz --disable-content-trust \
    openpolicyagent/opa-docker-authz-v2:0.5 opa-args="-policy-file /opa/policies/authz.rego"

# Configurar su uso, en archivo /etc/docker/daemon.json
    "authorization-plugins": ["opa-docker-authz"]

# reiniciar docker
$ service docker restart

# ejemplo de regla basica para denegar todo
$ mkdir -p /etc/docker/policies
$ vim /etc/docker/policies/authz.rego

# agregar al archivo authz.rego contenido siguiente:
    package docker.authz
    allow = false

# reiniciar docker
$ service docker restart
```

## Los contenedores no deben correr con el usuario root
Al correr cualquier contenedor por default internamente su usuario es root (aunque se use el userns-remap), esto no es lo ideal sino que debe usarse un usuario diferente de root para `evitar el riesgo de escalada de privilegios` fuera del contenedor.

Para cambiar esto se usa el parametro user (debe ser un usuario ya existente dentro del contenedor) al momento de iniciar el contenedor.

Ejemplo con `run` la imagen redis cuyo usuario se llama tambien redis:
```bash
$ docker run -it --user redis --rm redis bash
```

Ejemplo en `docker-compose` con mysql cuyo usuario se llama tambien mysql:
```docker
    image: mysql
    user: "mysql"
    environment:
      ....
      ....
    healthcheck:
      ....
    volumes:
      ....
```

Recomendaciones:
- En el caso de contenedores MYSQL la primera ejecución se hace con root despues reiniciar ya con usuario mysql
- En el caso de contenedores REDIS la primera ejecucion se hace con root despues reiniciar ya con usuario redis
- En otros casos depende del contenedor, del servicio interno que corra. Tambien se puede pasar un ID generico.

## Contenedores con healthcheck
El [healthcheck](https://docs.docker.com/engine/reference/builder/#healthcheck) es una forma de saber si el estado de un contenedor es saludable o no, para esto se indica un comando que se ejecuta dentro del contenedor cada cierto tiempo y de acuerdo al resultado indica dicho estado. Idealmente este comando podria estar dado desde la construcción de la imagen (Dockerfile), pero generalmente no lo está así que hay que indicarlo en el momento de iniciar un contenedor.

Ejemplo para mysql en `docker-compose`:
```docker
healthcheck:
    test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"]
    timeout: 30s
    interval: 90s
    retries: 5
```

Ejemplo para contenedores con un servicio web (reemplazar al puerto* adecuado en cada caso):
```docker
healthcheck:
    test: ["CMD", "curl" ,"-f", "http://localhost:80", "||", "exit", "1"]
    timeout: 30s
    interval: 120s
    retries: 5
```
