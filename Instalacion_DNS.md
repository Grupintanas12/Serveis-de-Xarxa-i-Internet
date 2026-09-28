Manual: configuración de red e instalación de un servidor DNS en Debian (VirtualBox)
1. Objetivo y escenario

Preparar una máquina virtual Debian con IP estática para instalar y configurar un servidor DNS (BIND9).

Adaptador	Tipo VirtualBox	Interfaz	Configuración
Adaptador 1	NAT	enp0s3	DHCP (10.0.2.15), da salida a internet
Adaptador 2	Red interna	enp0s8	IP estática 192.168.6.100/24

2. Acceder como administrador

Al intentar editar el archivo de red con sudo, el usuario vboxuser no tenía permisos:

bash
sudo nano /etc/network/interfaces
vboxuser is not in the sudoers file.

Solución: entrar como root con la contraseña de root (la que se puso al instalar Debian):

bash
su -

3. Identificar las interfaces de red
bash
ip a

Se identifican dos interfaces: enp0s3 (NAT) y enp0s8 (red interna).




4. Configurar la IP estática

Se edita el archivo:

bash
nano /etc/network/interfaces

Contenido final:

source /etc/network/interfaces.d/*

# The loopback network interface
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

5. Aplicar los cambios y resolver el error
bash
systemctl restart networking


<img width="958" height="538" alt="system_status" src="https://github.com/user-attachments/assets/46839843-c9cf-4208-b310-764343f43efc" />

Y confirmar que las ips estan bien configuradas para las dos interfaces

bash
ip a

<img width="842" height="337" alt="IPa_congif" src="https://github.com/user-attachments/assets/23596cf1-dd0d-4179-800a-e16d6823040c" />
