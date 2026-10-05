DNS 

1. Objetivo

Analizar e implementar la seguridad en el servicio DNS, estudiando cómo un protocolo legítimo como DNS puede utilizarse para ocultar y transportar información (técnica conocida como DNS tunneling o exfiltración de datos por DNS). Para ello se ejecutan dos scripts de Python, un cliente y un servidor DNS, y se captura el tráfico con Wireshark.

2. Entorno y preparación

En esta primera prueba, tanto el cliente como el servidor se ejecutan en la misma máquina virtual, comunicándose por la interfaz de loopback.

Descarga de los scripts: un script simula el servidor DNS y el otro, el cliente DNS.
Instalación de Python 3 y comprobación de que ambos scripts son compatibles con esa versión.
Instalación de Wireshark para capturar y analizar la consulta (request) y la respuesta (answer).
Problema encontrado: puerto 53 ocupado



3. Funcionamiento de los scripts
Servidor DNS
Espera la recepción de un único paquete (se puede modificar fácilmente para que escuche indefinidamente).
Interpreta la estructura del paquete DNS recibido.
Extrae el subdominio de la consulta y convierte la cadena hexadecimal a texto, obteniendo el mensaje oculto.
Responde al cliente con un registro A que cumple los mínimos exigidos por el protocolo DNS.
Cliente DNS
Realiza una consulta de tipo A a un dominio cuyo subdominio contiene los datos ocultos codificados en hexadecimal.
Tras enviar la consulta, espera la respuesta y finaliza.


<img width="1272" height="586" alt="Wireshark1" src="https://github.com/user-attachments/assets/87042dfd-0806-426f-b0de-b26784ec23bb" />
<img width="1306" height="667" alt="Wiresahrk2" src="https://github.com/user-attachments/assets/edf1337f-5604-45f7-92da-ce567d71ba21" />
<img width="593" height="188" alt="wireshark2 1" src="https://github.com/user-attachments/assets/5f7376cb-df85-4acd-b557-aacac4501a0b" />

Datos observados

Elemento	Valor
Direcciones IP	127.0.0.1 (origen y destino, interfaz de loopback)
Protocolo de transporte	UDP
Puerto del servidor	53 (DNS)
Puerto del cliente	Puerto efímero, muy por encima del 1023
Dominio consultado	secreto.com (con el subdominio hexadecimal)
IP devuelta en la respuesta	4.3.2.1


Trama 1: consulta (query)

Es una consulta DNS estándar (flags 0x0100) con una pregunta (Questions: 1) y sin registros de respuesta.

La pregunta es de tipo A, clase IN.

El campo Name contiene el subdominio hexadecimal seguido de secreto.com. Ahí viaja el mensaje oculto.



Trama 2: respuesta (response)

El servidor responde con un paquete estándar sin error (No error).

Repite la pregunta original y añade un registro de respuesta (Answer RRs: 1): secreto.com → 4.3.2.1.

El identificador de transacción (Transaction ID) coincide con el de la consulta, lo que permite al cliente asociar ambos paquetes.
