# Manual de instalación de un servidor DNS esclavo (BIND9)

## 1. Requisitos previos

Los datos del maestro están tomados de su configuración actual (capturas del 28/09/2026). Los del esclavo corresponden a la máquina dns2, configurada y verificada el 18/10/2026.

| Dato | Valor |
| --- | --- |
| Sistema operativo | Debian 13 (trixie), BIND 9.20.29 |
| Zona directa | haven.local |
| Zona inversa | 6.168.192.in-addr.arpa |
| Nombre del maestro | ns.haven.local |
| IP del maestro (enp0s8) | 192.168.6.100/24 |
| Nombre del esclavo | ns2.haven.local (host dns2) |
| IP del esclavo (enp0s8) | 192.168.6.101/24 |
| Red interna autorizada a consultar | 192.168.6.0/24 |
| Ficheros de zona en el maestro | /etc/bind/zones/db.haven.local y /etc/bind/zones/db.6.168.192 |
| Nombre de la clave TSIG | transfer-key |

Requisitos técnicos:

- Acceso root o sudo a ambos servidores.
- Conectividad entre esclavo y maestro en el puerto 53 TCP y UDP. Las transferencias de zona usan TCP; si solo se abre UDP, la réplica falla.
- Hora sincronizada por NTP en ambos equipos (chrony o systemd-timesyncd). TSIG rechaza mensajes con más de 5 minutos de desfase.
- Mismo esquema de red que el maestro: enp0s3 en NAT por DHCP para salida a Internet y enp0s8 con IP fija en la red interna 192.168.6.0/24.

## 2. Preparación del servidor maestro

El maestro tiene que autorizar la transferencia al esclavo, avisarle de los cambios y publicar el esclavo como servidor de nombres de la zona.

1. Edite /etc/bind/named.conf.local en el maestro y añada a las dos zonas existentes las directivas allow-transfer, also-notify y notify:

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

El maestro usa type master. BIND 9.20 lo acepta, aunque la palabra recomendada ahora es type primary; se puede cambiar sin más efecto.

2. En /etc/bind/zones/db.haven.local, añada el registro NS y el registro A del esclavo junto a los del maestro:

```
; name servers - NS records
@       IN      NS      ns.haven.local.
@       IN      NS      ns2.haven.local.

; name servers - A records
ns      IN      A       192.168.6.100
ns2     IN      A       192.168.6.101
```

3. En /etc/bind/zones/db.6.168.192, corrija el NS actual y añada el NS y el PTR del esclavo. Ahora el NS figura como ns. con punto final; BIND lo lee como un nombre ns colgado de la raíz del DNS, no como ns.haven.local. named-checkzone no lo marca como error, pero las consultas NS de la zona inversa devuelven un nombre que no resuelve. El bloque debe quedar así:

```
; name servers - NS records
@       IN      NS      ns.haven.local.
@       IN      NS      ns2.haven.local.

; PTR Records
100     IN      PTR     ns.haven.local.
101     IN      PTR     ns2.haven.local.
```

4. Incremente el número de serie del SOA en ambas zonas: ahora es 2, páselo a 3. Si el serial no sube, el esclavo no detecta el cambio. En el mismo SOA, cambie el primer nombre de haven.local. a ns.haven.local.: ese campo debe indicar el servidor maestro de la zona.
5. Compruebe la sintaxis y recargue:

```
sudo named-checkconf
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
sudo rndc reload
```

Ambos named-checkzone deben devolver loaded serial 3 y OK. En la captura de la comprobación anterior la zona inversa se escribió como 6.168.192.in-addr-arpa (guion en lugar de punto); el nombre correcto lleva punto.

En producción se recomienda sustituir la IP en allow-transfer por una clave TSIG (apartado 6). Con solo la IP, cualquier equipo que suplante esa dirección puede descargar la zona completa.

## 3. Instalación de BIND9 en el esclavo

### 3.1 Red

Configure /etc/network/interfaces con el mismo esquema que el maestro, cambiando solo la IP interna:

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

Aplique el cambio y compruebe la IP y la conectividad con el maestro:

```
sudo systemctl restart networking
ip a show enp0s8
ping -c 3 192.168.6.100
```

Añada ambos servidores a /etc/hosts del esclavo:

```
192.168.6.100   ns.haven.local    ns
192.168.6.101   ns2.haven.local   ns2
```

### 3.2 Paquetes

Instale los paquetes del servidor, las utilidades de consulta y la documentación:

```
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-dnsutils
```

En Debian 13 el paquete dnsutils ya no existe con ese nombre; apt lo sustituye por bind9-dnsutils, como se ve en la instalación del maestro.

En RHEL, Rocky o Alma Linux:

```
sudo dnf install -y bind bind-utils
```

Compruebe la versión y active el servicio para que arranque con el sistema:

```
named -v
sudo systemctl enable named
```

En Debian y Ubuntu el servicio se llama named (bind9 es un alias). Los ficheros de configuración están en /etc/bind/ y el directorio de trabajo en /var/cache/bind/. En RHEL la configuración está en /etc/named.conf y el directorio de trabajo en /var/named/.

## 4. Configuración del esclavo

### 4.1 Opciones generales

Edita /etc/bind/named.conf.options. El esclavo no debe permitir transferencias a terceros ni resolver recursivamente para cualquiera:

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

localnets incluye toda la red 192.168.6.0/24, por lo que el maestro y los clientes de la red interna pueden consultar. listen-on tiene que llevar la IP propia del esclavo: si se copia la del maestro (192.168.6.100), named arranca pero no escucha en la red interna. Las consultas a dominios externos se reenvían a 1.1.1.1 y 9.9.9.9; con dnssec-validation no, BIND no valida las firmas DNSSEC de esas respuestas.

Si el esclavo va a ser solo autoritativo (por ejemplo, expuesto a Internet), ponga recursion no y elimine allow-recursion.

El log del maestro muestra errores network unreachable al consultar los servidores raíz por IPv6, porque la máquina no tiene ruta IPv6 útil. Para evitarlos en el esclavo (y en el maestro), fuerce BIND a usar solo IPv4 en /etc/default/named:

```
OPTIONS="-u bind -4"
```

### 4.2 Directorio de zonas secundarias

Cree el directorio donde BIND guardará las copias de las zonas y asigne el propietario bind:

```
sudo mkdir -p /var/cache/bind/slaves
sudo chown bind:bind /var/cache/bind/slaves
```

No use /etc/bind para zonas secundarias. El perfil de AppArmor de Debian y Ubuntu solo permite a named escribir en /var/cache/bind y /var/lib/bind; si las zonas se guardan en otro sitio, la transferencia falla con permission denied. En RHEL use /var/named/slaves, que ya existe con los permisos correctos.

### 4.3 Declaración de zonas

Edita /etc/bind/named.conf.local y declare cada zona como secondary apuntando al maestro:

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

dns2 usa las palabras clave type slave y masters. BIND 9.20 las acepta, pero las registra como obsoletas; los equivalentes actuales son type secondary y primaries, y se pueden cambiar sin otro efecto.

Desde BIND 9.9 las zonas secundarias se guardan en formato binario (raw). Si necesita leerlas como texto, añada masterfile-format text; dentro de cada zona.

## 5. Clave TSIG para transferencias autenticadas

TSIG firma las transferencias con una clave compartida, de modo que el maestro solo entrega la zona a quien tenga la clave. Es opcional en laboratorio y recomendable en producción.

1. Genere la clave en el maestro:

```
sudo tsig-keygen -a hmac-sha256 transfer-key | sudo tee /etc/bind/transfer.key
sudo chown root:bind /etc/bind/transfer.key
sudo chmod 640 /etc/bind/transfer.key
```

El fichero resultante tiene este formato (el secreto será distinto):

```
key "transfer-key" {
    algorithm hmac-sha256;
    secret "<secreto en base64>";
};
```

2. Copie el fichero al esclavo por un canal seguro (scp) en la misma ruta y con los mismos permisos.
3. En el maestro, incluya la clave y sustituya la IP en allow-transfer:

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

4. En el esclavo, incluya la clave y asóciela al maestro con una sentencia server en named.conf.local:

```
include "/etc/bind/transfer.key";

server 192.168.6.100 {
    keys { "transfer-key"; };
};
```

5. Recargue ambos servidores con sudo rndc reload.

No envíe el secreto por correo ni lo guarde en repositorios. Si se compromete, genere una clave nueva y sustitúyala en los dos servidores.

## 6. Firewall y arranque

Abra el puerto 53 en TCP y UDP en el esclavo para las redes que consultan. Debian 13 no instala ufw por defecto; si no tiene otro firewall gestionado, instálelo con sudo apt install ufw y ejecute:

```
sudo ufw allow from 192.168.6.0/24 to any port 53 proto udp
sudo ufw allow from 192.168.6.0/24 to any port 53 proto tcp
```

Con firewalld (RHEL):

```
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

En el maestro, el puerto 53 TCP tiene que estar abierto hacia la IP del esclavo (transferencias) y el esclavo tiene que aceptar UDP 53 desde el maestro (NOTIFY). Si hay un firewall de red entre ambos, pida las mismas reglas.

Compruebe la configuración y arranque el servicio:

```
sudo named-checkconf
sudo systemctl restart named
sudo systemctl status named
```

named-checkconf no devuelve nada si la configuración es correcta. Si muestra un error, indica fichero y línea.

## 7. Verificación

La instalación es correcta cuando el esclavo tiene el mismo serial que el maestro y responde con el flag aa (autoritativo).

Con el maestro encendido y ya recargado, fuerce la primera transferencia desde el esclavo:

```
sudo rndc retransfer haven.local
sudo rndc retransfer 6.168.192.in-addr.arpa
```

1. Revise el log de named en el esclavo:

```
sudo journalctl -u named --since "10 minutes ago"
```

Debe aparecer una línea similar a: zone haven.local/IN: Transfer completed: 1 messages, 6 records, ... (serial 3). El número de registros depende de lo que tenga la zona; hay otra línea igual para 6.168.192.in-addr.arpa.

2. Compruebe que existen los ficheros de zona:

```
ls -l /var/cache/bind/slaves/
```

3. Compare el serial del SOA en ambos servidores:

```
dig @192.168.6.100 haven.local SOA +short
dig @192.168.6.101 haven.local SOA +short
```

El tercer campo (serial) tiene que coincidir.

4. Haga una consulta al esclavo y confirme el flag aa en la cabecera:

```
dig @192.168.6.101 ns.haven.local A
dig @192.168.6.101 -x 192.168.6.100
```

dig muestra un aviso de que .local está reservado para Multicast DNS. Es informativo y no afecta a la respuesta; en una red de producción conviene usar un dominio propio o un subdominio de uno registrado.

5. Compruebe que el esclavo no entrega la zona a terceros. Desde un equipo cualquiera de la red:

```
dig @192.168.6.101 haven.local AXFR
```

El resultado esperado es Transfer failed.

6. Pruebe la notificación. Añada un registro de prueba en el maestro, suba el serial y ejecute sudo rndc reload haven.local. En pocos segundos el esclavo debe registrar una nueva transferencia y el dig del punto 3 debe mostrar el serial nuevo en ambos.
7. Opcional: pare named en el maestro y repita el punto 4. El esclavo tiene que seguir respondiendo.

## 8. Resolución de problemas

La mayoría de los fallos se ven en sudo journalctl -u named en el esclavo. La tabla recoge los mensajes más habituales.

| Síntoma en el log del esclavo | Causa probable | Acción |
| --- | --- | --- |
| Transfer status: REFUSED | El maestro no autoriza la IP o la clave del esclavo | Revisar allow-transfer en el maestro y recargar |
| failed while receiving responses: timed out | Puerto 53 TCP bloqueado entre esclavo y maestro | Probar con nc -zv 192.168.6.100 53 y abrir el firewall |
| dumping master file: permission denied | Ruta de zona fuera de /var/cache/bind o propietario incorrecto | Mover a /var/cache/bind/slaves y chown bind:bind |
| tsig verify failure (BADTIME) | Desfase de hora entre servidores | Revisar NTP con timedatectl en ambos |
| tsig verify failure (BADSIG o BADKEY) | Clave o nombre distintos en cada servidor | Comparar los ficheros transfer.key con diff |
| Sin transferencia tras cambiar la zona | Serial no incrementado en el maestro | Subir el serial y ejecutar rndc reload |
| Respuesta sin flag aa | La zona no se ha cargado en el esclavo | Ejecutar rndc zonestatus haven.local y revisar el log |
| network unreachable resolving './DNSKEY/IN' | BIND intenta usar IPv6 y la máquina no tiene ruta IPv6 | OPTIONS="-u bind -4" en /etc/default/named y reiniciar named |
| Consulta NS de la zona inversa devuelve ns. | NS escrito sin el dominio en db.6.168.192 | Escribir ns.haven.local. completo, subir el serial y recargar (apartado 3) |
| refresh: failure trying primary; dig da host unreachable y no servers could be reached | Maestro apagado o adaptadores de VirtualBox en redes internas distintas | Encender el maestro y comprobar con ping -c 3 192.168.6.100 desde el esclavo |

Para forzar una transferencia completa desde el esclavo: sudo rndc retransfer haven.local.


No hace falta copiar los ficheros de /var/cache/bind/slaves: se regeneran con la primera transferencia.
