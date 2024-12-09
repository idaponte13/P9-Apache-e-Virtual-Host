# P9-Apache-e-Virtual-Host

Neste punto comentarei os arquivos configurados para levar a cabo esta práctica:


**Docker-compose.yml**

Aqui configuraremos o yaml como nos casos das anteriores prácticas. Có servidor, o dns e a rede.

**web**

Nesta parte definimos a imaxe, o nome do servidor, o porto que empregaremos, os volumes que máis adiante configuramos e a red

```
web:
    image: php:7.4-apache
    container_name: apache_server
    ports:
      - "80:80"
    volumes:
      - ./www:/var/www
      - ./confApache:/etc/apache2
    networks:
      apache_red:
        ipv4_address: 172.39.4.2
```  

**dns**

```
dns:
    container_name: dns_server
    image: ubuntu/bind9
    ports:
      - "57:53"
    volumes:
      - ./confDNS/conf:/etc/bind
      - ./confDNS/zonas:/var/lib/bind
    networks:
      apache_red:
        ipv4_address: 172.39.4.3
```
**rede**
```
networks:
  apache_red:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.39.0.0/16
```
**confApache**

Esta carpeta podemola descargar do repositorio de damian ou descargando apache, onde so debemos engadir dous ficheiros na carpeta "sites-available" pero o importante é na carpeta "sites-enable". Este dous ficheiros son os mismos para ambas carpeta. Tamén, no caso de querer empregar varios portos, debemos engadir no ficheiro "ports.conf" a liña listen 8000. Os ficheiros que engadiremos as carpetas son os seguintes:

**fabulasmaravillosas.conf**

Neste arquivo debemos chamar ao porto configurado anteriormente, no meu caso será o porto 80, tamén diremos quen vai ser o admin, un correo xeralmente, o nome e o alias da nosa páxina, que será o que buscaremos no navegador. Por último a ruta onde temos o noso index.html.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost  #admin 
    ServerName fabulasmaravillosas.asircastelao.int  #nome da paxina
    ServerAlias www.fabulasmaravillosas.asircastelao.int #alias da paxina
    DocumentRoot /var/www/fabulasmaravillosas #ruta onde temos o documentoo index.html
</VirtualHost>
```

**fabulasoscuras.conf**

Faremos o mesmo que na páxina anterior.

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost #admin 
    ServerName fabulasoscuras.asircastelao.int #nome da paxina
    ServerAlias www.fabulasoscuras.asircastelao.int #alias da paxina 
    DocumentRoot /var/www/fabulasoscuras #ruta onde temos o documento index.html
</VirtualHost>
```

**confDNS**

Nesta carpeta configuraremos as zonas e a propia configuración destas.

**Zonas**

Nesta carpeta configuraremos la resolución de nomes da nosa zona directa. Dentro deste documento, primero debemos empregar o servidor de nomes principal, no meu caso ns.asircastelao.int, tamén declaramos un rexistro NS que indica o servidor de nomes autoritativos para o dominio, ademáis asociamos o nome do servidor á dirección IP do dns, e asociamos as páxinas webs á dirección IP.

```
$TTL    604800  
@       IN      SOA     ns.asircastelao.int. some.email.address.com. ( ; Indica o servidor de nomes principal e un correo de contacto.
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative 

@       IN      NS      ns.asircastelao.int. #nome autoritativo
ns 	    IN 	    A 	    172.39.4.3  #asociamos IP ao nome do servidor
fabulasoscuras       IN      A       172.39.4.2  #páxina asociada a IP
fabulasmaravillosas     IN      A       172.39.4.2  # páxina asociada a IP
```

**conf**

Dividiremos esta carpeta en 3 partes.

O primerio de todo é o `named.conf.local`, onde debemos chamar ao arquivo da zona.

```
zone "asircastelao.int" {
    type master;
    file "/var/lib/bind/db.asircastelao.int";
    allow-query {
    	any;
    	};
};
```

O seguinte documento é `named.conf.options`, aqui configuraremos as opcións xenerales que se aplican a todo o servidor DNS.
```
options {
    directory "/var/cache/bind";
    recursion yes;                      # Permitir la resolución recursiva
    allow-query { any; };               # Permitir consultas desde cualquier IP
    dnssec-validation no;
    forwarders {
        8.8.8.8;                        # Google DNS
        1.1.1.1;                        # Cloudflare DNS
    };
    listen-on { any; };                 # Escuchar en todas las interfaces
    listen-on-v6 { any; };
};
```
Por último `named.conf` onde simplemente enlazamos os dous documentos anteriores.
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```
Adicionalmente, está o arquivo chamado `named.conf.default-zones` onde teño definido as zoas inversas.
```
// prime the server with knowledge of the root servers
zone "." {
	type hint;
	file "/usr/share/dns/root.hints";
};

// be authoritative for the localhost forward and reverse zones, and for
// broadcast zones as per RFC 1912

zone "localhost" {
	type master;
	file "/etc/bind/db.local";
};

zone "127.in-addr.arpa" {
	type master;
	file "/etc/bind/db.127";
};

zone "0.in-addr.arpa" {
	type master;
	file "/etc/bind/db.0";
};

zone "255.in-addr.arpa" {
	type master;
	file "/etc/bind/db.255";
};
```
**www**

Nesta carpeta teremos outras dúas carpetas, chamadas exactemente como as páxinas e dentro de cada carpeta, debemos ter os documentos index.html de cada páxina web.

Uha vez configurado todo debemos realizar cambios no documento `sudo nano /etc/systemd/resolved.conf` onde debemos de buscar a liña do DNS e descomentala e añadir a dirección IP do noso DNS que configuramos co correspondete porto. Isto podemos comprobalo no .yml. 

```DNS=172.39.4.3#57```

Para aplicar os cambios realizados debemos executar o seguinte comando ```systemctl restart systemd-resolved```.

Por ultimo accederemos a configuración de rede cableada da nosa maquina (arriba a dereita) e desactivaremos, no apartado de IPv4, o DNS automático.