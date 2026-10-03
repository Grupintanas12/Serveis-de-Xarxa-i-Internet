# Manual: configuración de red e instalación de un servidor DNS en Debian (VirtualBox)
## 1. Objetivo y escenario

Preparar una máquina virtual Debian con IP estática para instalar y configurar un servidor DNS (BIND9).

Adaptador	Tipo VirtualBox	Interfaz	Configuración
Adaptador 1	NAT	enp0s3	DHCP (10.0.2.15), da salida a internet
Adaptador 2	Red interna	enp0s8	IP estática 192.168.6.100/24

## 2. Acceder como administrador

Al intentar editar el archivo de red con sudo, el usuario vboxuser no tenía permisos:

bash
sudo nano /etc/network/interfaces
vboxuser is not in the sudoers file.

Solución: entrar como root con la contraseña de root (la que se puso al instalar Debian):

bash
su -

## 3. Identificar las interfaces de red
bash
ip a

Se identifican dos interfaces: enp0s3 (NAT) y enp0s8 (red interna).




## 4. Configurar la IP estática

Se edita el archivo:

bash
nano /etc/network/interfaces

Contenido final:

source /etc/network/interfaces.d/*

"#" The loopback network interface
auto lo
iface lo inet loopback

allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
        address 192.168.6.100
        netmask 255.255.255.0



Notas:

No se añade gateway en enp0s8: la salida a internet la proporciona el NAT (enp0s3).
broadcast es opcional, el sistema lo calcula (192.168.6.255).
dns-nameservers no se pone aquí: esta máquina será el propio servidor DNS.

<img width="648" height="292" alt="interfaces_red" src="https://github.com/user-attachments/assets/96a50e7e-5baa-4ab6-a49b-dfb9a192b5ca" />

## 5. Aplicar los cambios y resolver el error
bash
systemctl restart networking


<img width="958" height="538" alt="system_status" src="https://github.com/user-attachments/assets/46839843-c9cf-4208-b310-764343f43efc" />

Y confirmar que las ips estan bien configuradas para las dos interfaces

bash
ip a

<img width="842" height="337" alt="IPa_congif" src="https://github.com/user-attachments/assets/23596cf1-dd0d-4179-800a-e16d6823040c" />


## 6. Instalar Bind9

bash
sudo apt install bind9 bind9-utils

<img width="957" height="621" alt="InstallBIND9" src="https://github.com/user-attachments/assets/62d15665-082b-437a-8b2f-b8bb785e3f23" />


## 7. Editar named.conf.local

En /etc/bind/ se edita named.conf.local para declarar la zona directa y la inversa. Los archivos de zona se guardan en un directorio zones que se crea (no es obligatorio, pero es más ordenado).

bash
sudo mkdir /etc/bind/zones

<img width="957" height="798" alt="ArchivoconfLOCAL" src="https://github.com/user-attachments/assets/646bb140-5c33-41c3-ad13-6eadd23d5944" />

Para verificar la sintaxis:

bash
named-checkconf


## 8. Crear el fichero de zona directa

bash
sudo nano /etc/bind/zones/db.haven.local

<img width="909" height="460" alt="zonalocal" src="https://github.com/user-attachments/assets/96c701bf-6ef2-4bef-83f5-754151e39af0" />


## 9. Crear el fichero de zona inversa

bash
sudo nano /etc/bind/zones/db.6.168.192

<img width="946" height="365" alt="zonainversa" src="https://github.com/user-attachments/assets/f6459162-8645-484e-8993-a6b5cf543329" />


## 10. Verificar las zonas

bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192

<img width="949" height="111" alt="Checkeozonas" src="https://github.com/user-attachments/assets/85d04a1e-9bee-40b9-9a3e-a77b76ffb80a" />


## 11. Configurar named.conf.options

Define las opciones globales del servicio.

Se añade la opción -4 en los argumentos del usuario bind (se crea solo al instalar). Evita errores de red inalcanzable por IPv6.

## 12. Reiniciar servicio y comprobar

<img width="958" height="538" alt="system_status" src="https://github.com/user-attachments/assets/5f90dcde-de4a-4a92-956c-e1a941076980" />

<img width="842" height="337" alt="IPa_congif" src="https://github.com/user-attachments/assets/3d5958a2-5528-46a9-9394-c1d191cc17e3" />

Según el status y la devolución del ipconfig, sale que el servicio está bien implementado, pero al hacer un "nslookup" no encuentra el servicio; apunta a otra dirección completamente diferente.
