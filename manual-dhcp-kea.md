# Manual de instalación de un servidor DHCP con Kea

## 1. Requisitos previos

En esta práctica he montado un servidor DHCP con Kea en la máquina maestro para que reparta IPs en la red interna, y he hecho que dns2 (el DNS esclavo de la práctica anterior) pida su IP por DHCP en lugar de tenerla fija. Las dos máquinas son Debian 13 en VirtualBox.

| Dato | Valor |
| --- | --- |
| Sistema operativo | Debian 13 (trixie) |
| Servidor DHCP | Kea 2.6.3, en el maestro (192.168.6.100) |
| Interfaz donde reparte IPs | enp0s8 (red interna 192.168.6.0/24) |
| Rango para equipos nuevos | 192.168.6.150 a 192.168.6.200 |
| Cliente | dns2, con IP reservada 192.168.6.101 |
| MAC de dns2 | 08:00:27:7d:2e:e3 |
| DNS que se reparten | 192.168.6.100 y 192.168.6.101 |
| Dominio | haven.local |

La idea de reservar la 192.168.6.101 para dns2 es que siga teniendo siempre la misma IP aunque ahora la pida por DHCP. Si cambiara de IP, se rompería todo lo del DNS esclavo, porque el maestro le pasa las zonas a esa IP.

Antes de empezar necesitas:

- Poder usar sudo en las dos máquinas.
- Que el maestro tenga IP fija en enp0s8 (192.168.6.100). Un servidor DHCP no puede pedir su propia IP por DHCP.
- Que el Adaptador 2 de las dos máquinas esté en VirtualBox en "Red interna" con el mismo nombre.

## 2. Instalar Kea en el maestro

En el maestro:

```
sudo apt update
sudo apt install -y kea-dhcp4-server
sudo kea-dhcp4 -V
```

`kea-dhcp4 -V` muestra la versión, en mi caso la 2.6.3. Hay que lanzarlo con sudo porque el programa está en /usr/sbin, que no está en el PATH del usuario normal. Sin sudo sale "command not found" aunque esté instalado.

## 3. Configurar Kea

### 3.1 Fichero de configuración

El fichero que viene por defecto es enorme y está lleno de ejemplos, así que lo guardé como copia y escribí uno nuevo:

```
sudo mv /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.orig
sudo nano /etc/kea/kea-dhcp4.conf
```

```
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "enp0s8" ]
    },

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv"
    },

    "valid-lifetime": 3600,

    "subnet4": [
        {
            "id": 1,
            "subnet": "192.168.6.0/24",
            "pools": [ { "pool": "192.168.6.150 - 192.168.6.200" } ],

            "option-data": [
                { "name": "routers", "data": "192.168.6.100" },
                { "name": "domain-name-servers", "data": "192.168.6.100, 192.168.6.101" },
                { "name": "domain-name", "data": "haven.local" }
            ],

            "reservations": [
                {
                    "hw-address": "08:00:27:7d:2e:e3",
                    "ip-address": "192.168.6.101",
                    "hostname": "dns2"
                }
            ]
        }
    ]
}
}
```

Qué hace cada parte:

- **interfaces:** Kea solo reparte IPs por enp0s8, la red interna. Por enp0s3 (el NAT de VirtualBox) no, porque ahí ya está el DHCP de VirtualBox.
- **lease-database:** dónde guarda las IPs que ha repartido. Lo guarda en un fichero CSV.
- **valid-lifetime:** cuánto dura cada IP prestada, 3600 segundos (1 hora). Antes de que caduque el cliente la renueva solo.
- **pools:** el rango de IPs para equipos que no tienen reserva.
- **routers:** la puerta de enlace. Esta línea la añadí después, en la práctica de NAT, para que dns2 salga a Internet a través del maestro. Si solo quieres el DHCP sin NAT, quítala.
- **domain-name-servers:** los DNS que se dan a los clientes, que son mis dos servidores.
- **domain-name:** el dominio, para que los clientes puedan usar nombres cortos.
- **reservations:** dns2, por su MAC, recibe siempre la 192.168.6.101.

Es formato JSON, así que hay que tener cuidado con las comas y las llaves. Cada elemento va separado por coma, pero después del último de cada bloque no va coma.

### 3.2 Permisos del fichero

Cuando probé el fichero con `sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf` me decía "Unable to open file", aunque el fichero existía. En `sudo dmesg | grep -i kea` salían líneas de `apparmor="DENIED"`: Debian trae un perfil de AppArmor para Kea, y al crear el fichero nuevo con nano se había quedado con unos permisos distintos a los del original.

Lo solucioné copiando el propietario y los permisos del fichero original:

```
sudo chown --reference=/etc/kea/kea-dhcp4.conf.orig /etc/kea/kea-dhcp4.conf
sudo chmod --reference=/etc/kea/kea-dhcp4.conf.orig /etc/kea/kea-dhcp4.conf
sudo ls -la /etc/kea/
```

Los dos ficheros tienen que salir con el mismo propietario y los mismos permisos.

### 3.3 Arrancar Kea

En vez de comprobar con `-t`, comprobé arrancando directamente el servicio, que es como funciona de verdad:

```
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server --no-pager -l
```

Si sale `active (running)`, la configuración está bien. Si falla, en las líneas de abajo del status sale el error con la línea y la columna del fichero (por ejemplo `kea-dhcp4.conf:22.35`).

A mí me falló varias veces por erratas, y cada vez el error decía exactamente qué pasaba:

| Error | Qué había hecho mal |
| --- | --- |
| `got unexpected keyword "dat"` | Había escrito `"dat"` en lugar de `"data"` |
| `Failed to parse pool definition: 192.168.6.150-192/168.6.200` | Una barra en lugar de un punto en la IP |
| `the option 'dhcp4.domain-name-server' does not exist` | Faltaba la s final de `domain-name-servers` |
| `Failed to convert string to address 'haven.local'` | Había puesto `domain-name-servers` en la línea del dominio, y Kea esperaba una IP |
| `invalid path specified: '/var/lib/kea/kea', supported path is '/var/lib/kea'` | Había escrito `/var/lib/kea/kea/leases4.csv` en lugar de `/var/lib/kea/kea-leases4.csv`. Desde la versión 2.6.3, Kea solo deja guardar ese fichero directamente en /var/lib/kea |

En el log también sale un aviso amarillo, "ConfigurationDirectory 'kea' already exists but the mode is different". No es un error: solo dice que la carpeta /etc/kea tiene permisos más cerrados de lo que systemd espera. Los avisos WARN de multi-threading también son normales.

Cuando todo está bien, en el log aparecen estas líneas:

```
DHCPSRV_CFGMGR_ADD_IFACE listening on interface enp0s8
DHCP4_CONFIG_COMPLETE DHCPv4 server has completed configuration: added IPv4 subnets: 1
```

## 4. Configurar dns2 como cliente DHCP

dns2 tenía la IP fija en /etc/network/interfaces. Para saber quién gestiona la red en dns2 miré:

```
cat /etc/network/interfaces
nmcli device status
```

En /etc/network/interfaces estaba el bloque `iface enp0s8 inet static`, y nmcli mostraba enp0s8 como "unmanaged", así que la red la lleva ifupdown (no NetworkManager) y el cambio se hace en ese fichero.

Hay que hacerlo en este orden. Si cambias el fichero antes de bajar la interfaz, ifdown se lía con la configuración vieja.

1. Bajar la interfaz:
   ```
   sudo ifdown enp0s8
   ```
2. Editar el fichero:
   ```
   sudo nano /etc/network/interfaces
   ```
   Cambiar el bloque de la red interna por este, quitando las líneas address y netmask:
   ```
   # Red Interna
   allow-hotplug enp0s8
   iface enp0s8 inet dhcp
   ```
3. Levantar la interfaz, que ahora pide la IP al maestro:
   ```
   sudo ifup enp0s8
   ```

## 5. Comprobaciones

En dns2:

```
ip a show enp0s8
```

Tiene que salir `inet 192.168.6.101/24` con la palabra `dynamic`. Eso significa que la IP viene del DHCP y que la reserva por MAC funciona.

En el maestro, el fichero de leases muestra las IPs que ha repartido Kea:

```
sudo cat /var/lib/kea/kea-leases4.csv
```

Tiene que salir una línea con la 192.168.6.101, la MAC 08:00:27:7d:2e:e3 y el nombre dns2.

Como el DNS esclavo depende de que dns2 tenga la 192.168.6.101, también conviene comprobar que sigue funcionando:

```
dig @192.168.6.101 haven.local SOA +short
```

## 6. Problemas que te puedes encontrar

| Lo que ves | Por qué pasa | Cómo se arregla |
| --- | --- | --- |
| `kea-dhcp4: command not found` | Está en /usr/sbin y no lo lanzas con sudo, o hay una errata en el nombre | `sudo kea-dhcp4 -V` |
| `Unable to open file /etc/kea/kea-dhcp4.conf` | El fichero no existe (no se guardó en nano o tiene otro nombre) o tiene permisos distintos al original | Comprobar con `sudo ls -la /etc/kea/` y copiar permisos con `--reference` |
| El servicio sale `failed` | Error en el fichero de configuración | Mirar el status con `-l`, que dice la línea exacta |
| dns2 recibe una IP de la .150 a la .200 en lugar de la .101 | La MAC de dns2 no coincide con la de la reserva | Ver la MAC con `ip a show enp0s8` y corregirla en VirtualBox o en la reserva |
| dns2 no recibe ninguna IP | El maestro está apagado o los adaptadores están en redes internas distintas | Encender el maestro primero y revisar el Adaptador 2 en VirtualBox |

El caso de la MAC me pasó al copiar las máquinas al ordenador de clase. VirtualBox le puso a dns2 una MAC nueva (08:00:27:57:f2:72), Kea no la reconoció y le dio la 192.168.6.150. Se ve en el fichero de leases del maestro. Lo arreglé devolviéndole la MAC original desde VirtualBox, con dns2 apagada: Configuración, Red, Adaptador 2, Avanzadas, Dirección MAC, y poner `0800277D2EE3`. Al arrancar volvió a recibir la 192.168.6.101.
