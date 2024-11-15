# Examen-Martin-Alvarez

## Crea un repositorio para contestar todo o exame.
## Este repositorio ten que conter tódo-los ficheiros necesarios para xustifica-la túas respostas.

- Contesta as seguintes preguntas, xustificándoas, nun README.md

### 1. Explica métodos para 'abrir' unha consola/shell a un contenedor en execución.

O primero que faremos será instalar Docker, seguido de VS Code. Unha vez dentro de VS Code, no apartado de extensións, instalamos a extensión de Docker. Unha vez instalada, debería aparecer o icono de Docker (como extensión); na parte inferior temos a opción de Terminal. Esta danos ou xera unha terminal onde podemos facer numerosas cousas, runear contenedores, detelos, etc. 

Antes de poder executar un contenedor, teremos que descargar as imaxes que vaiamos utilizar (ubuntu, htdocs ou calquera imaxe). Nalgunhas teremos que crear directorios para completar o funcionamento.

- Co seguinte comando o que podemos facer é iniciar un contenedor:
```
docker run -d --name Prueba 
```

- Outro exemplo que temos é este:
```
docker run -d --name Prueba -p 8000:80 -v "$PWD"/htdocs:/usr/local/apache2/htdocs/ httpd
```

---




### 2. No contenedor anterior (en execución), qué opciones tes que ter usado ó arrincalo para poder interactuar coas súas entradas e salidas.

Principalmente as dos portos -p e -v para telos localizados.

---



### 3. Cómo sería un ficheiro docker-compose para que dous contenedores se comuniquen entre si nunha rede só deles?

Se queremos ter dous contenedores conectados a unha rede, primero, necesitamos crear con docker unha rede para ambas, temos o comando 
```
docker network create mainred
```

Coa nosa red creada, só teríamos que conectar e runear dos contenedores conectados á mesama red. O comando é o seguinte:
```
docker run -itd --name=1 --network=mainred ubuntu
docker run -itd --name=2 --network=mainred ubuntu
```
E listo, os dous contenedores (1 e 2) xa están conectados á rede mainred usando a imaxe de ubuntu. 

O que podemos facer agora é comprobar que están conectados, que poden verse entre eles, o estado da rede, e moitas cousas máis.

Crearemos este archivo dentro de un directorio que teñamos creado previamente:
```
touch docker-compose.yml
```

El archivo incluirá una configuración básica basada en las recomendaciones de la imagen:  
Con un gedit al fichero añadimos o seguinte:

```yaml
services:
  bind9: 
    image: internetsystemsconsortium/bind9:9.18
    container_name: Practica5_bind9
    ports:
      - 54:53/udp
      - 54:53/tcp
      - 127.0.0.1:953:953/tcp
    volumes:
      - ./etc/bind:/etc/bind
      - ./var/cache/bind:/var/cache/bind
    restart: always
``` 

---



### 4. Qué tes que engadir ó ficheiro anterior para que un contenedor teña unha IP fixa?

Unha manera de comprobar que teña unha IP fixa é a seguinte:

Primeiro identificaremos a IP del contenedor:  
```
docker network inspect mainred
```
Asignamos a IP da rede e listo.
"IPv4Address": "172.18.0.2/16"

---



### 5. Qué comando de consola podo usar para sabe-las ips dos contenedores anteriores?

IP ou facendo incluso o ping, se o faces con docker. Con inspeccionar a rede xa debería darnos as IPs dos nosos contenedores.

---

### 6. Cál é a funcionalidade do apartado "ports" en docker compose?

Mapea o porto que escollas da máquina anfitriona (o teu ordenador) co porto do contedor que tamén escollamos nos.

---

### 7. Para qué serve o rexistro CNAME? Pon un exemplo

A utilidade de CNAME é definir un alias para o nome canónico. Útil para suministrar nomes alternativos relacionados cos diferentes servizos nun mismo equipo.

---



### 8. Cómo podo facer para que a configuración dun contenedor DNS non se borre se creo outro contenedor?





---




### 9. Engade unha zoa tendaelectronica.int no teu docker DNS que teña:
- www á IP 172.16.0.1
- owncloud sexa un CNAME de www
- un rexistro de texto có contido "1234ASDF"

Comproba que todo funciona có comando "dig"
Mostra nos logs que o servicio funciona ben usando a saída da terminal ó levantar o compose ou có comando "docker logs [nomeContenedorOuID]"
(o apartado 9 realízase na máquina virtual)


- Primeiro, necesitamos un ficheiro chamado db.tendaelectronica.int para definir a zona DNS. O contido debe ser o seguinte:
```
$TTL 86400
@   IN  SOA ns1.tendaelectronica.int. admin.tendaelectronica.int. (
        2024111501 ; Serial
        3600       ; Refresh
        1800       ; Retry
        1209600    ; Expire
        86400 )    ; Minimum TTL

    IN  NS  ns1.tendaelectronica.int.

ns1 IN  A   172.16.0.1
www IN  A   172.16.0.1
owncloud IN CNAME www
@   IN  TXT "1234ASDF"
```

- Logo no ficheiro named.conf para configurar o servizo DNS:

options {
    directory "/var/cache/bind";
    recursion no;
    allow-query { any; };
    forwarders {};
    listen-on { any; };
};

zone "tendaelectronica.int" {
    type master;
    file "/etc/bind/zones/db.tendaelectronica.int";
};

- Preparamos o Dockerfile:
```
FROM ubuntu:20.04

run apt-get update && apt-get install -y bind9 bind9utils

COPY named.conf /etc/bind/named.conf
COPY db.tendaelectronica.int /etc/bind/zones/db.tendaelectronica.int

CMD ["named", "-g"]
```

- Temos que configurar agora o servizo no ficheiro **docker-compose.yml**:
```
version: "3.8"
services:
  dns:
    build: .
    container_name: dns-server
    ports:
      - "53:53/udp"
    networks:
      dns_network:
        ipv4_address: 172.16.0.1
    volumes:
      - ./named.conf:/etc/bind/named.conf
      - ./db.tendaelectronica.int:/etc/bind/zones/db.tendaelectronica.int

networks:
  dns_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.16.0.0/24
```

- Levantar o contedor:
```
docker-compose up -d
```

- Logs do contedor:
```
docker logs dns-server
```

- Rexistros DNS de www.tendaelectronica.int, usando DIG:
```
dig @172.16.0.1 www.tendaelectronica.int
```

- Comproba o rexistro CNAME de owncloud.tendaelectronica.int:
```
dig @172.16.0.1 owncloud.tendaelectronica.int
```

- Verifica o rexistro TXT:

```
dig @172.16.0.1 tendaelectronica.int TXT
```
### Saída esperada do dig

Para www.tendaelectronica.int:
```
;; ANSWER SECTION:
www.tendaelectronica.int.  86400  IN  A  172.16.0.1
```
Para owncloud.tendaelectronica.int:
```
;; ANSWER SECTION:
owncloud.tendaelectronica.int.  86400  IN  CNAME  www.tendaelectronica.int.
```
Para o rexistro TXT:
```
;; ANSWER SECTION:
tendaelectronica.int.  86400  IN  TXT "1234ASDF"
```

Ao revisar os logs do contedor, deberías ver algo similar a isto ao levantar o servizo:
```
starting BIND 9.16.1
running on 172.16.0.1#53
zone tendaelectronica.int/IN: loaded serial 2024111501
zone 0.in-addr.arpa/IN: loaded serial 1
```












