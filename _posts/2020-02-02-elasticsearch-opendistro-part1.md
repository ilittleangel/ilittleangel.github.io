---
layout: post
title:  "Open Distro for Elasticsearch Utils - Part I"
date:   2020-02-01 20:15:33 +0200
categories: elk elastic opendistro docker docker-compose ssl
---

### Vistazo general

Vamos a tratar de preparar un entorno de pruebas con Elasticsearch.

* Para simular un entorno lo mas real posible vamos a levantar un cluster de Elasticsearch de dos nodos con autenticación y cifrado SSL.
* Una forma muy sencilla de hacerlo es usando la distribución de [**OpenDistro for Elasticsearch**](https://opendistro.github.io/for-elasticsearch-docs/).
* Usaremos Docker Compose para levantar el stack completo y partiremos del fichero `docker-compose.yml` proporcionado en la documentacion oficial.
* Lo que haremos es configurar los certificados de cero y se los proporcionaremos a cada instancia via volumenes de docker. De esta forma podremos usarlos en otros clientes y podremos, entre otras cosas, generar otro tipo de key stores, como por ejemplo de tipo `JKS` muy tipico en clientes que usan la JVM.


### Indice

* [Generar certificados](#generar-certificados)
* [elasticsearch.yml](#elasticsearchyml)
* [kibana.yml](#kibanayml)
* [docker-compose.yml](#docker-composeyml)
* [Añadir entrada a /etc/hosts](#añadir-entrada-a-etchosts)
* [Prueba de funcionamiento](#prueba-de-funcionamiento)

### Generar certificados 

Seguiremos la documentacion oficial de Open Distro for Elasticsearch para la generacion de certificados  
[Opendistro Docs](https://opendistro.github.io/for-elasticsearch-docs/docs/security-configuration/generate-certificates/#generate-a-private-key)

#### Generamos la private key

```sh
openssl genrsa -out root-ca-key.pem 2048
```

#### Generamos el root certificate

Creamos un `root-ca.pem` que usaremos como CA para cualquier cliente de nuestro Elasticsearch Open Distro, 
ya sea un cliente de linea de comandos, en nuestro caso `curl` o para un cliente escrito en Python, etc.

> Para un cliente JVM en Scala, Java, Kotlin.. se suele usar para el handshake de la negociacion 
entre servidores un Java Key Store. En una posterior entrada del blog veremos como crear un 
`truststore.jks` a partir de nuestra `root-ca-key.pem` para poder usar con cualquier cliente JVM.

Para crear nuestro certificado `root-ca.pem`
```sh
openssl req -new -x509 -days 6552 -sha256 -key root-ca-key.pem -out root-ca.pem
```

> Los 6552 dias son porque desde hoy (11/02/2020) hasta la fecha (19/01/2038) hay 6552 dias.
Por tanto la fecha de expiracion del certificado será el dia 19/01/2038. Esta fecha es famosa por el [Y2K38 Problem](https://en.wikipedia.org/wiki/Year_2038_problem) pero puedes poner los dias que quieras.
  
Rellenamos la informacion solicitada
```
Country Name (2 letter code) []:ES
State or Province Name (full name) []:Madrid
Locality Name (eg, city) []:Madrid
Organization Name (eg, company) []:ORG
Organizational Unit Name (eg, section) []:UNIT
Common Name (eg, fully qualified host name) []:ilittleangel
Email Address []:
```

#### Generamos los node certificates

* Generaremos certificados para dos nodos de Elasticsearch (esnode1, esnode2).

* Primero generamos una nueva key temporal que usaremos para generar los certificados de ambos nodos
```sh
openssl genrsa -out node-key-temp.pem 2048
```

* Convertimos la key temporal al formato PKCS#8 para el nodo 1 y la llamaremos `esnode1-key.pem`
```sh
openssl pkcs8 -inform PEM -outform PEM -in node-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out esnode1-key.pem
```

* Creamos un CSR (certificate signing request)
```sh
openssl req -new -key esnode1-key.pem -out node.csr
```

* Rellenamos la informacion solicitada para el nodo 1. En la opcion `Common Name` hay que introducir el hostname del nodo 1 de elasticsearch, que en este caso será el que elijamos en la property `container_name` del docker-compose. (`email` y `password ` pueden quedar vacios)
```
Country Name (2 letter code) []:ES
State or Province Name (full name) []:Madrid
Locality Name (eg, city) []:Madrid
Organization Name (eg, company) []:ORG
Organizational Unit Name (eg, section) []:UNIT
Common Name (eg, fully qualified host name) []:es-opendistro-node1
Email Address []:
A challenge password []:
```

* Por último, generamos el certificado para el nodo 1
```sh
openssl x509 -req -in node.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -days 6552 -out esnode1.pem
```

* Eliminamos los ficheros que ya no usaremos
```sh
rm node-key-temp.pem
rm node.csr
rm root-ca.srl    
```

* Repetimos los pasos para generar los certificados del nodo 2


### _elasticsearch.yml_

Ahora toca editar el fichero de configuración de Elasticsearch. Para obtener este fichero puedes copiar y pegar el proporcionado aquí, el cual ya tiene la configuración correcta. Tambien se puede obtener de las siguientes formas:

* a traves del template oficial de OpenDistro en [github](https://github.com/opendistro-for-elasticsearch/security-ssl/blob/master/opendistrosecurity-ssl-config-template.yml)

* levantando un contenedor docker con Elaticsearch a partir de la imagen `amazon/opendistro-for-elasticsearch` y navegar por el filesystem local del contendor hasta el fichero para copiar y pegar el contenido
```
docker run -p 9200:9200 -p 9600:9600 -e "discovery.type=single-node" amazon/opendistro-for-elasticsearch:1.4.0
docker cp <elasticsearch-container-name>:/usr/share/elasticsearch/config/elasticsearch.yml ~/.
```

En el fichero `elasticsearch.yml` tenemos toda la configuración inicial que necesita Elasticsearch para arrancar. Entre esta configuración está la de seguridad y cifrado ssl tanto para la capa de transporte como para el API HTTP que expone el propio Elasticsearch. 

```yaml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
opendistro_security.ssl.transport.pemcert_filepath: esnode.pem
opendistro_security.ssl.transport.pemkey_filepath: esnode-key.pem
opendistro_security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
opendistro_security.ssl.transport.enforce_hostname_verification: false
opendistro_security.ssl.http.enabled: true
opendistro_security.ssl.http.pemcert_filepath: esnode.pem
opendistro_security.ssl.http.pemkey_filepath: esnode-key.pem
opendistro_security.ssl.http.pemtrustedcas_filepath: root-ca.pem
opendistro_security.allow_unsafe_democertificates: true
opendistro_security.allow_default_init_securityindex: true
opendistro_security.nodes_dn:
  - 'CN=es-opendistro-node1,OU=UNIT,O=ORG,L=Madrid,ST=Madrid,C=ES'
  - 'CN=es-opendistro-node2,OU=UNIT,O=ORG,L=Madrid,ST=Madrid,C=ES'
opendistro_security.authcz.admin_dn:
  - CN=kirk,OU=client,O=client,L=test, C=de

opendistro_security.audit.type: internal_elasticsearch
opendistro_security.enable_snapshot_restore_privilege: true
opendistro_security.check_snapshot_restore_write_privileges: true
opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
cluster.routing.allocation.disk.threshold_enabled: false
node.max_local_storage_nodes: 3
```

Tener en cuenta lo siguiente:

* Los datos introducidos a la hora de crear los certificados del nodo 1 y 2 serán los que tendremos que setear en nuestro `elasticsearch.yml` en la propiedad `opendistro_security.nodes_dn`. En esta propiedad introduciremos tantas entradas como nodos o domain names de Elasticsearch tengamos, en nuestro caso 2.
* El fichero se montará en todos los nodos del cluster de Elasticsearch y este será el mismo para todos, aunque los certificados que montaremos no. Lo que haremos es montar ficheros con el mismo nombre pero con contenido diferente. En el `docker-compose.yml` se ve como se hace eso en la parte de `volumes` de cada servicio.
  
  
  
### _kibana.yml_

El fichero `kibana.yml` podriamos dejar el que monta la distribución por defecto, ya que está activada una opcion que permite la no comprobación de certificados. Pero si queremos hacerlo correctamente, tenemos que montar el fichero con la CA que usaremos para los clientes de nuestro Elasticsearch OpenDistro, dentro del contenedor de Kibana. Y decirle a Kibana donde se encuentra este certificado `root-ca.pem` mediante la propiedad `elasticsearch.ssl.certificateAuthorities`.

```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: https://localhost:9200
elasticsearch.ssl.verificationMode: full
elasticsearch.ssl.certificateAuthorities: [ "/usr/share/kibana/config/root-ca.pem" ]
elasticsearch.username: kibanaserver
elasticsearch.password: kibanaserver
elasticsearch.requestHeadersWhitelist: ["securitytenant","Authorization"]

opendistro_security.multitenancy.enabled: true
opendistro_security.multitenancy.tenants.preferred: ["Private", "Global"]
opendistro_security.readonly_mode.roles: ["kibana_read_only"]
```
  
  
### _docker-compose.yml_

Tenemos el docker-compose oficial en la [documentacion de Open Distro](https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker/#sample-docker-compose-file).

Para nuestro ejemplo, tendríamos que añadir lo siguiente:

* el montaje de los ficheros `elasticsearch.yml` y `kibana.yml` via volumenes de docker
* el montaje de los ficheros de certificados via volúmenes de docker
* por utlimo y aunque no es estrictamente necesario, por facilitar el uso, fijar las ips que asigna docker a los nodos de Elasticsearch, mediante la propiedad de red `ipv4_address` y en la seccion de `networks` la propiedad `ipam`

```yaml
version: "3"
services:

  kibana:
    # https://opendistro.github.io/for-elasticsearch-docs/docs/troubleshoot/#java-error-during-startup
    container_name: kibana-opendistro
    image: amazon/opendistro-for-elasticsearch-kibana:1.4.0
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch1
    environment:
      ELASTICSEARCH_URL: https://es-opendistro-node1:9200
      ELASTICSEARCH_HOSTS: https://es-opendistro-node1:9200
    volumes:
      - ./opendistro/config/kibana.yml:/usr/share/kibana/config/kibana.yml
      - ./opendistro/config/root-ca.pem:/usr/share/kibana/config/root-ca.pem
    networks:
      - elastic-net

  elasticsearch1:
    container_name: es-opendistro-node1
    image: amazon/opendistro-for-elasticsearch:1.4.0
    ports:
      - 9200:9200
      - 9600:9600 # required for Performance Analyzer
    environment:
      - cluster.name=opendistro-cluster
      - node.name=es-opendistro-node1
      - discovery.seed_hosts=es-opendistro-node1,es-opendistro-node2
      - cluster.initial_master_nodes=es-opendistro-node1,es-opendistro-node2
      - bootstrap.memory_lock=true # along with the memlock settings below, disables swapping
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # minimum and maximum Java heap size, recommend setting both to 50% of system RAM
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536 # maximum number of open files for the Elasticsearch user, set to at least 65536 on modern systems
        hard: 65536
    volumes:
      - elasticsearch-data1:/usr/share/elasticsearch/data
      - ./opendistro/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./opendistro/config/esnode1.pem:/usr/share/elasticsearch/config/esnode.pem
      - ./opendistro/config/esnode1-key.pem:/usr/share/elasticsearch/config/esnode-key.pem
      - ./opendistro/config/root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
    networks:
      elastic-net:
        ipv4_address: 172.23.0.2

  elasticsearch2:
    container_name: es-opendistro-node2
    image: amazon/opendistro-for-elasticsearch:1.4.0
    environment:
      - cluster.name=opendistro-cluster
      - node.name=es-opendistro-node2
      - discovery.seed_hosts=es-opendistro-node1,es-opendistro-node2
      - cluster.initial_master_nodes=es-opendistro-node1,es-opendistro-node2
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - elasticsearch-data2:/usr/share/elasticsearch/data
      - ./opendistro/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./opendistro/config/esnode2.pem:/usr/share/elasticsearch/config/esnode.pem
      - ./opendistro/config/esnode2-key.pem:/usr/share/elasticsearch/config/esnode-key.pem
      - ./opendistro/config/root-ca.pem:/usr/share/elasticsearch/config/root-ca.pem
    networks:
      elastic-net:
        ipv4_address: 172.23.0.3

volumes:
  elasticsearch-data1:
  elasticsearch-data2:

networks:
  elastic-net:
    ipam:
      driver: default
      config:
        - subnet: 172.23.0.0/16
```


### Añadir entrada a `/etc/hosts`

Como hemos fijado las ip's que se usarán para levantar los dos nodos de elasticsearch,
podemos mapear el hostname que usaremos desde fuera de la red interna que crea docker
para enviar peticiones al elastic nodo 1. Lo haremos añadiendo la siguiente entrada
```sh
echo '172.23.0.2  es-opendistro-node1' | sudo tee -a /etc/hosts
echo '172.23.0.3  es-opendistro-node2' | sudo tee -a /etc/hosts
```

### Prueba de funcionamiento

```sh
curl --cacert root-ca.pem \
    -H "Authorization: Basic $(echo -n admin:admin | base64)" \
    -XGET "https://es-opendistro-node1:9200"
```
  
  


[Github Repo](https://github.com/ilittleangel/elasticsearch-opendistro-utils)
