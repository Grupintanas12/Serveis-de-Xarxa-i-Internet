# Manual de instalación de un servidor DNS esclavo (BIND9)

## 1. Requisitos previos

Para esta práctica he montado un DNS esclavo (dns2) que copia las zonas del DNS maestro que ya tenía configurado. Las dos máquinas son Debian 13 en VirtualBox. Estos son los datos que he usado:

| Dato | Valor |
| --- | --- |
| Sistema operativo | Debian 13 (trixie), BIND 9.20.29 |
| Zona directa | haven.local |
| Zona inversa | 6.168.192.in-addr.arpa |
| Nombre del maestro | ns.haven.local |
| IP del maestro (enp0s8) | 192.168.6.100/24 |
| Nombre del esclavo | ns2.haven.local (host dns2) |
| IP del esclavo (enp0s8) | 192.168.6.101/24 |
| Red interna | 192.168.6.0/24 |
| Ficheros de zona en el maestro | /etc/bind/zones/db.haven.local y /etc/bind/zones/db.6.168.192 |
| Nombre de la clave TSIG | transfer-key |

Antes de empezar necesitas:

- Poder usar sudo en las dos máquinas.
- Que el esclavo y el maestro se vean por el puerto 53, tanto TCP como UDP. Las transferencias de zona van por TCP, así que si solo tienes UDP abierto no se copia nada.
- Que las dos máquinas tengan la hora bien puesta. Esto solo importa si usas TSIG (apartado 5), porque si hay más de 5 minutos de diferencia rechaza la clave.
- El mismo esquema de red en las dos: enp0s3 en NAT por DHCP para tener Internet y enp0s8 con IP fija en la red interna 192.168.6.0/24.

## 2. Preparar el servidor maestro

Lo primero es tocar el maestro para que deje copiar las zonas al esclavo, le avise cuando haya cambios y lo tenga apuntado como servidor de nombres.

1. Edita /etc/bind/named.conf.local en el maestro y añade a las dos zonas las líneas allow-transfer, also-notify y notify:

```
// Busquedas directas
zone "haven.local" in {
    type master;
    file "/etc/bind/zones/db.haven.local";
    allow-transfer { 192.168.6.101; };
    also-notify { 192.168.6.101; };
    notify yes;
};

// Busquedas inversas
zone "6.168.192.in-addr.arpa" in {
    type master;
    file "/etc/bind/zones/db.6.168.192";
    allow-transfer { 192.168.6.101; };
    also-notify { 192.168.6.101; };
    notify yes;
};
```

Yo tengo puesto type master. En BIND 9.20 funciona igual, aunque ahora lo recomendado es poner type primary.

2. En /etc/bind/zones/db.haven.local añade el NS y el A del esclavo, debajo de los del maestro:

```
; name servers - NS records
@       IN      NS      ns.haven.local.
@       IN      NS      ns2.haven.local.

; name servers - A records
ns      IN      A       192.168.6.100
ns2     IN      A       192.168.6.101
```

Cuidado con dejar alguna línea a medias. A mí se me quedó la línea `@ IN NS` sin el `ns2.haven.local.` detrás y named-checkzone me daba "unexpected end of input" en esa línea. Con `cat -n` ves los números de línea y lo encuentras rápido.

3. En /etc/bind/zones/db.6.168.192 añade el NS y el PTR del esclavo. Aquí además tuve que corregir el NS que ya había, porque estaba puesto como `ns.` con punto al final. Eso BIND lo entiende como un nombre "ns" suelto y no como ns.haven.local. Lo curioso es que named-checkzone no da error, así que no te enteras. Tiene que quedar así:

```
; name servers - NS records
@       IN      NS      ns.haven.local.
@       IN      NS      ns2.haven.local.

; PTR Records
100     IN      PTR     ns.haven.local.
101     IN      PTR     ns2.haven.local.
```

4. Sube el número de serie del SOA en las dos zonas. Yo lo tenía en 2 y lo puse a 3. Si no lo subes, el esclavo no se entera de que ha cambiado algo. De paso, en el SOA cambié el primer nombre de `haven.local.` a `ns.haven.local.`, porque ahí tiene que ir el nombre del servidor maestro.
5. Comprueba que está todo bien y recarga:

```
sudo named-checkconf
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
sudo rndc reload
```

named-checkconf no dice nada si todo está bien. Los dos named-checkzone tienen que decir `loaded serial 3` y `OK`. Fíjate en que es `in-addr.arpa` con punto: yo al principio puse `in-addr-arpa` con guion.

Para comprobar que el maestro ya tiene los cambios:

```
dig @192.168.6.100 haven.local NS +short
dig @192.168.6.100 haven.local SOA +short
```

Tienen que salir ns.haven.local y ns2.haven.local, y el serial 3.

Con esto el maestro deja copiar las zonas a cualquiera que tenga la IP 192.168.6.101. Para un laboratorio vale, pero si quieres hacerlo más seguro mira el apartado 5 (TSIG).

## 3. Instalación de BIND9 en el esclavo

### 3.1 Red

En el esclavo configura /etc/network/interfaces igual que en el maestro, cambiando solo la IP interna:

```
# NAT / salida a Internet
allow-hotplug enp0s3
iface enp0s3 inet dhcp

# Red Interna
allow-hotplug enp0s8
iface enp0s8 inet static
        address 192.168.6.101
        netmask 255.255.255.0
```

Reinicia la red y comprueba que tienes la IP y que llegas al maestro:

```
sudo systemctl restart networking
ip a show enp0s8
ping -c 3 192.168.6.100
```

Si clonas la máquina del maestro para hacer el esclavo, revisa bien la IP, porque se copia todo y te puedes quedar con dos máquinas con la 192.168.6.100.

También puedes añadir los dos servidores a /etc/hosts del esclavo:

```
192.168.6.100   ns.haven.local    ns
192.168.6.101   ns2.haven.local   ns2
```

### 3.2 Paquetes

Instala BIND y las herramientas para hacer consultas:

```
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-dnsutils
```

En Debian 13 ya no hay un paquete que se llame dnsutils, ahora es bind9-dnsutils. Si pones dnsutils apt lo cambia solo.

Mira la versión y activa el servicio para que arranque solo:

```
named -v
sudo systemctl enable named
```

En Debian el servicio se llama named, pero bind9 también funciona porque es un alias. La configuración está en /etc/bind/ y los datos en /var/cache/bind/.

## 4. Configuración del esclavo

### 4.1 Opciones generales

Edita /etc/bind/named.conf.options. Así lo tengo yo:

```
acl "safeclients" {
        localhost;
        192.168.6.101;
        localnets;
};

options {
        directory "/var/cache/bind";
        recursion yes;
        allow-recursion { safeclients; };
        listen-on { 127.0.0.1; 192.168.6.101; };
        allow-transfer { none; };
        allow-query { safeclients; };
        allow-query-cache { safeclients; };

        dnssec-validation no;

        forwarders {
                1.1.1.1;
                9.9.9.9;
        };
};
```

Qué hace cada cosa:

- localnets mete toda la red 192.168.6.0/24, así que el maestro y los clientes de la red interna pueden hacer consultas.
- allow-transfer { none; } hace que el esclavo no le pase las zonas a nadie.
- Lo que no es de haven.local lo pregunta a 1.1.1.1 y 9.9.9.9 (forwarders).
- En listen-on tiene que ir la IP del esclavo. Yo había copiado la del maestro (192.168.6.100) y named arrancaba, pero no escuchaba en la red interna.

Para ver si está escuchando bien:

```
sudo ss -tulnp | grep :53
```

Tiene que salir named en 127.0.0.1:53 y en 192.168.6.101:53, en tcp y en udp.

En el log del maestro me salían errores de "network unreachable" porque BIND intentaba usar IPv6 y la máquina no tiene. Se quitan obligando a BIND a usar solo IPv4. Para eso, en /etc/default/named pon:

```
OPTIONS="-u bind -4"
```

### 4.2 Carpeta para las zonas

Crea la carpeta donde el esclavo va a guardar las copias de las zonas y haz que sea del usuario bind:

```
sudo mkdir -p /var/cache/bind/slaves
sudo chown bind:bind /var/cache/bind/slaves
```

No las guardes en /etc/bind. En Debian, AppArmor solo deja que named escriba en /var/cache/bind y /var/lib/bind, y si las pones en otro sitio te da "permission denied".

### 4.3 Zonas del esclavo

Edita /etc/bind/named.conf.local y pon las dos zonas como slave apuntando al maestro:

```
// Busquedas directas
zone "haven.local" in {
        type slave;
        file "/var/cache/bind/slaves/db.haven.local";
        masters { 192.168.6.100; };
};

//Busquedas inversas
zone "6.168.192.in-addr.arpa" in {
        type slave;
        file "/var/cache/bind/slaves/db.6.168.192";
        masters { 192.168.6.100; };
};
```

type slave y masters siguen funcionando en BIND 9.20, pero en el log salen como obsoletos. Las palabras nuevas son type secondary y primaries.

Los ficheros que se guardan en slaves están en formato binario, así que si haces cat no se entienden. Si los quieres en texto, añade `masterfile-format text;` dentro de cada zona.

## 5. Clave TSIG (opcional)

Esto es un extra para que las transferencias vayan con una clave compartida y el maestro solo le pase las zonas a quien tenga esa clave, en vez de fiarse solo de la IP. En el laboratorio no hace falta, pero en un entorno real sí.

1. Genera la clave en el maestro:

```
sudo tsig-keygen -a hmac-sha256 transfer-key | sudo tee /etc/bind/transfer.key
sudo chown root:bind /etc/bind/transfer.key
sudo chmod 640 /etc/bind/transfer.key
```

El fichero que sale es algo así (tu secreto será otro):

```
key "transfer-key" {
    algorithm hmac-sha256;
    secret "<secreto en base64>";
};
```

2. Copia el fichero al esclavo con scp, en la misma ruta y con los mismos permisos.
3. En el maestro, incluye la clave y cambia la IP de allow-transfer por la clave:

```
include "/etc/bind/transfer.key";

// Busquedas directas
zone "haven.local" in {
    type master;
    file "/etc/bind/zones/db.haven.local";
    allow-transfer { key "transfer-key"; };
    also-notify { 192.168.6.101; };
    notify yes;
};

// Busquedas inversas
zone "6.168.192.in-addr.arpa" in {
    type master;
    file "/etc/bind/zones/db.6.168.192";
    allow-transfer { key "transfer-key"; };
    also-notify { 192.168.6.101; };
    notify yes;
};
```

4. En el esclavo, incluye la clave y dile que la use con el maestro, en named.conf.local:

```
include "/etc/bind/transfer.key";

server 192.168.6.100 {
    keys { "transfer-key"; };
};
```

5. Recarga los dos con `sudo rndc reload`.

No subas el fichero de la clave a GitHub ni lo pases por correo. Si alguien la consigue, genera otra y cámbiala en los dos servidores.

## 6. Firewall y arranque

Si tienes firewall, abre el puerto 53 TCP y UDP en el esclavo para la red interna. Debian 13 no trae ufw instalado, así que si lo quieres usar primero haz `sudo apt install ufw` y luego:

```
sudo ufw allow from 192.168.6.0/24 to any port 53 proto udp
sudo ufw allow from 192.168.6.0/24 to any port 53 proto tcp
```

En el maestro también tiene que estar abierto el 53 TCP para el esclavo, que es por donde van las transferencias.

Comprueba la configuración y reinicia:

```
sudo named-checkconf
sudo systemctl restart named
sudo systemctl status named
```

Si named-checkconf no dice nada, está bien. Si hay un error te dice el fichero y la línea.

## 7. Comprobaciones

Sabes que funciona cuando el esclavo tiene el mismo serial que el maestro y contesta con el flag aa, que quiere decir que responde como autoritativo.

Antes de nada, asegúrate de que el maestro está encendido. Parece obvio, pero a mí me falló todo por eso: dig daba "host unreachable" y "no servers could be reached" porque tenía la máquina del maestro apagada.

Con el maestro encendido, fuerza la primera copia desde el esclavo:

```
sudo rndc retransfer haven.local
sudo rndc retransfer 6.168.192.in-addr.arpa
```

1. Mira el log de named en el esclavo:

```
sudo journalctl -u named -n 20 --no-pager -l
```

Tienen que salir líneas como `Transfer status: success` y `Transfer completed: 1 messages, 6 records, ... (serial 3)`. El número de registros depende de lo que tenga tu zona.

2. Mira que se han creado los ficheros de zona:

```
ls -l /var/cache/bind/slaves/
```

3. Compara el serial en los dos servidores:

```
dig @192.168.6.100 haven.local SOA +short
dig @192.168.6.101 haven.local SOA +short
```

El tercer número (el serial) tiene que ser el mismo en los dos. A mí me salió esto en el esclavo:

```
ns.haven.local. hostmaster.haven.local. 3 43200 900 1814400 7200
```

4. Haz unas consultas al esclavo y mira que en la línea `flags:` sale `aa`:

```
dig @192.168.6.101 ns2.haven.local
dig @192.168.6.101 -x 192.168.6.100 +short
```

La primera tiene que devolver 192.168.6.101 y la segunda ns.haven.local.

dig también saca un aviso de que .local está reservado para Multicast DNS. No pasa nada, es solo un aviso y la respuesta sale bien.

5. Comprueba que el esclavo no le pasa las zonas a cualquiera. Desde otra máquina de la red:

```
dig @192.168.6.101 haven.local AXFR
```

Tiene que salir "Transfer failed".

6. Prueba que los cambios se copian solos. Añade un registro de prueba en el maestro, sube el serial y haz `sudo rndc reload haven.local`. En unos segundos el esclavo tiene que hacer otra transferencia y el dig del punto 3 tiene que dar el serial nuevo en los dos.
7. Si quieres, para named en el maestro y repite el punto 4. El esclavo tiene que seguir contestando.

## 8. Problemas que te puedes encontrar

Casi todos los errores se ven con `sudo journalctl -u named` en el esclavo. Aquí van los más típicos, incluidos los que me pasaron a mí:

| Lo que ves | Por qué pasa | Cómo se arregla |
| --- | --- | --- |
| refresh: failure trying primary; dig da host unreachable y no servers could be reached | El maestro está apagado o los adaptadores de VirtualBox están en redes internas distintas | Enciende el maestro y prueba `ping -c 3 192.168.6.100` desde el esclavo |
| named-checkzone da "unexpected end of input" | Hay una línea a medias en el fichero de zona | Mira el fichero con `cat -n` y completa la línea que dice el error |
| Transfer status: REFUSED | El maestro no deja copiar la zona a la IP o la clave del esclavo | Revisa allow-transfer en el maestro y recarga |
| failed while receiving responses: timed out | El puerto 53 TCP está bloqueado entre las dos máquinas | Prueba `nc -zv 192.168.6.100 53` y abre el firewall |
| dumping master file: permission denied | La carpeta de zonas está fuera de /var/cache/bind o no es de bind | Usa /var/cache/bind/slaves y haz `chown bind:bind` |
| tsig verify failure (BADTIME) | Las dos máquinas tienen la hora distinta | Revisa la hora con `timedatectl` en las dos |
| tsig verify failure (BADSIG o BADKEY) | La clave o su nombre no son iguales en los dos servidores | Compara los transfer.key con `diff` |
| No se copia nada después de cambiar la zona | No has subido el serial en el maestro | Sube el serial y haz `rndc reload` |
| La respuesta no tiene el flag aa | La zona no se ha cargado en el esclavo | Haz `rndc zonestatus haven.local` y mira el log |
| network unreachable resolving './DNSKEY/IN' | BIND intenta usar IPv6 y la máquina no tiene | Pon `OPTIONS="-u bind -4"` en /etc/default/named y reinicia named |
| La consulta NS de la zona inversa devuelve ns. | El NS está escrito sin el dominio en db.6.168.192 | Pon ns.haven.local. completo, sube el serial y recarga (apartado 2) |

Si quieres forzar que el esclavo vuelva a copiar la zona entera: `sudo rndc retransfer haven.local`.

No hace falta hacer copia de los ficheros de /var/cache/bind/slaves, porque se vuelven a crear solos con la primera transferencia.
