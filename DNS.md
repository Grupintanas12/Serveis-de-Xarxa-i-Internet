DNS

¿Qué es un DNS?

Es una lista, como si de una base de datos se tratara, donde se asocia una ip a un dominio. Puede dar tanto la IP si se le pasa el dominio como el dominio si se le pasa la IP.

Para la mejora del sistema en robustez y rendimiento se consigue  a través de caching y replicación.

El servicio DNS se encuentra en la de aplicación del modelo OSI, además la peticiones DNS se hacen a través de protocolo UDP (puerto 53) ya que es más rápido y más eficaz,

Está estructurado de manera jerárquica a nivel mundial, el cual administra el espacio de los nombres de dominio. 
 
¿Qué organismo internacional coordina y asigna los parámetros a nivel global del sistema de nombres de dominio e IPs?

La ICANN(Internet Corporation for Assigned Names and Numbers) se encarga de coordinar los nombres de dominio, por ejemplo, .com, .net, .edu, etc., para que cada dirección web sea única y apunte al lugar correcto. También administran y coordinan la distribución global de IPs y los números de sistema autónomo. Autorizan y supervisan a las empresas que venden y gestionan extensiones de dominio a los usuarios y controlan los servidores raíz de Internet, los cuales actúan como la guía central que permite que los dispositivos se localicen entre sí. 



Existen 4 servidores DNS

Recursivos: 
usuario quiere visitar una página web
Servidor puede tener o no la ip de la web, si es que no
Recibe otro servidor DNS, y así hasta que un servidor tenga en caché. Y todos los anteriores servidores se van a ir guardando la ip por si mas adelante alguien quiere vovler acceder.
Raíz:
son los 13 servidores DNS principales a nivel mundial
TLD (top-level domain):
son los dominios mas comunes que existen, los cuales son publico
ejemplos: .com, .org, .net
SOA:
es el registro de autoridad de DNS, es decr nombra si tiene autoridad o no ese dominio.



¿Qué empresa u organismo gestiona (Registry) cada uno de los siguientes dominios de nivel superior (TLD)?

.es - Red.es, es una entidad pública española responsable del registro de los dominios .es
.cat - Fundació puntCAT, es la organización que patrocina y gestiona el dominio .cat
.edu - ESUCAUSE, es la organización americana que patrocina el dominio .edu
.ifp.es - no es TLD, ifp se considera un dominio de segundo nivel, ya que el TLD solo seria el .es.

¿Qué información te brinda una consulta Whois sobre un dominio?


WHOIS:
Sirver principalmente para conocer quién registra el dominio, su estado, fechas de registro y los servidores DNS que estan asociados, aunque a veces la información personal puede estar protegida. 

Define brevemente la diferencia entre el Registry de la base de datos y el Registrar (Registrador) del dominio

Un Registry es una entidad que se encarga de gestionar y regular la normativa para dominios de uno o varios TLDs, cumpliendo con la normativa de la ICANN. 
Al margen de las normas mínimas que establece la ICANN, los registros pueden fijar y aplicar unos requisitos propios para sus extensiones, como las condiciones de registro y el periodo de gracia, además de determinar el precio mínimo de sus dominios. 

Por su parte, los registros seleccionan a los registradores autorizados para comercializar nombres de dominio a los usuarios finales.
Los registradores, también denominados partners de registro, son empresas privadas que han superado un proceso de acreditación que les permite gestionar y emitir licencias de dominios bajo diferentes extensiones.
Entre sus principales responsabilidades se encuentra el cumplimiento de los protocolos establecidos por la ICANN, con el objetivo de garantizar la disponibilidad de los dominios y evitar que un mismo nombre pueda registrarse simultáneamente a través de dos o más registradores.
¿Que es DNSSEC? 

DNSSEC es una característica del Sistema de nombres de dominio (DNS) que utiliza la autenticación criptográfica para verificar que los registros DNS devueltos en una consulta DNS provienen de un servidor de nombres autorizado y no se alteran en el camino. 
DNSSEC intenta resolver la falta de autenticidad e integridad en las resoluciones DNS, un problema que permite a los atacantes falsificar las respuestas y redirigir a los usuarios a sitios web maliciosos sin que se den cuenta. 
Rendimiento DNS:
Ve a la web de https://www.grc.com/dns/benchmark.htm


<img width="717" height="456" alt="image12" src="https://github.com/user-attachments/assets/78b9ef04-ec73-4116-ba10-646da073bc96" />
<img width="581" height="461" alt="image5" src="https://github.com/user-attachments/assets/850cbfc4-3fcb-4952-bb1f-95f22b9b40bd" />


Los 3 servidores DNS más rápidos:

4.2.2.3 - Level 3 Parent, LLC - Louisiana, se dedica a la prestación de servicios de telecomunicaciones por línea fija wireline, redes basadas en IP, fibra óptica y soluciones de conectividad empresarial.

1.0.0.1	- Cloudflare, se dedica a ofrecer infraestructura, seguridad y optimización de rendimiento para sitios web, aplicaciones y redes en Internet

1.1.1.1 - Cloudflare










Configuración y Caché

¿Cómo puedes ver mediante consola (CLI) qué servidores DNS tienes asignados actualmente en Windows y en Linux?


Para ver los servidores DNS asignados en Windows mediante la línea de comandos, “ipconfig /all”.
Para ver los servidores DNS asignados en Linux mediante la línea de comandos, “cat /etc/resolv.conf”.

Cambia la configuración de red de tu equipo principal (Windows o Linux) poniendo como DNS primario y secundario los que obtuviste en el Benchmark de la Fase 1. Muestra captura del cambio

<img width="717" height="456" alt="image12" src="https://github.com/user-attachments/assets/8def2df7-a393-4d67-ac87-343730819f80" />

 
¿En qué menú de tu dispositivo móvil (Android/iOS) podrías forzar el uso de unos DNS específicos para tu conexión Wi-Fi?

Tanto en Android como en iOS, puedes forzar el uso de unos DNS específicos para tu conexión Wi-Fi desde el menú de Configuración (o Ajustes), entrando en el apartado de la red Wi-Fi y editando los parámetros de la red concreta a la que estás conectado.




Gestión de la caché DNS (ipconfig / resolvectl):

Utilizando tu terminal de Windows (ipconfig /displaydns) o Linux (resolvectl statistics o similar):
Muestra una captura de pantalla de algunas direcciones almacenadas en la caché de tu equipo


<img width="497" height="239" alt="image7" src="https://github.com/user-attachments/assets/69df4f1c-cd6c-46b0-8c6f-469ba8e20c5f" />

<img width="543" height="336" alt="image14" src="https://github.com/user-attachments/assets/7e4e84fa-9c98-4da6-b9bd-55646b16f562" />



Vacía la caché de tu equipo (ipconfig /flushdns o resolvectl flush-caches). Explica para qué es útil esta acción en el día a día de un administrador de sistemas.

<img width="517" height="120" alt="image2" src="https://github.com/user-attachments/assets/08b25b1c-2f10-4a9c-96f9-be643774f94b" />


Resolución de problemas de conectividad: Cuando un usuario o un servidor no puede acceder a un sitio web o a un servicio interno, vaciar la caché asegura que el equipo solicite la dirección IP directamente al servidor DNS, descartando que el problema provenga de un dato local corrupto u obsoleto.

Administración -Troubleshooting con DIG y CLI [3p]

En un entorno profesional, especialmente servidores Linux, la herramienta nslookup se considera obsoleta, siendo dig - Domain Information Groper -el estándar de la industria.

1.Consultas específicas de registros (Usa dig en Linux o WSL):
Documenta con capturas de pantalla y explica el resultado de ejecutar las siguientes consultas sobre el dominio aliexpress.com u otro de tu elección:

○A:dig aliexpress.com

<img width="637" height="509" alt="image4" src="https://github.com/user-attachments/assets/6dec9370-9d27-456e-8faa-9aa9031805ae" />

○Short: dig +short aliexpress.com -
¿Por qué es útil este formato en scripts de Bash?

<img width="661" height="473" alt="image3" src="https://github.com/user-attachments/assets/7bd78152-8318-402d-ad35-31f09f86b982" />



○MX: dig MX aliexpress.com - Identifica el campo de Prioridad/Preference de los servidores de correo.


○NS: dig NS aliexpress.com ¿Cuáles son los servidores que tienen la autoridad sobre las zonas de este dominio?

<img width="820" height="416" alt="image1" src="https://github.com/user-attachments/assets/aba96969-8586-4975-a9aa-de59ef311600" />

















Autoridad y Caché (TTL):



<img width="540" height="298" alt="image9" src="https://github.com/user-attachments/assets/c9c1d6b0-576e-4c3d-8826-468dfc4f50c9" />
<img width="1101" height="521" alt="image8" src="https://github.com/user-attachments/assets/03cc5b85-ff04-444d-a776-4d9fccd57e96" />

<img width="538" height="318" alt="image15" src="https://github.com/user-attachments/assets/8873e9b2-d5c2-4b2e-8e41-f76c85582b3d" />








3.Trazabilidad Completa (Trace)


<img width="1101" height="521" alt="image8" src="https://github.com/user-attachments/assets/d43a0e0e-a4aa-416c-bce7-d7ab5c7c759b" />




<img width="1095" height="575" alt="image11" src="https://github.com/user-attachments/assets/fe70bd8b-14b9-469b-8fab-34c9b1a2185b" />










Analisi de Tráfico de Red

<img width="750" height="575" alt="image13" src="https://github.com/user-attachments/assets/d090f7aa-4d5f-40ff-b02a-83ca0ce2deb3" />


<img width="746" height="427" alt="image6" src="https://github.com/user-attachments/assets/e0a60e67-1d5a-494a-93c5-e708b0edd0e1" />



Capa de Transporte: ¿Qué protocolo se utiliza (TCP o UDP)? ¿Por qué DNS utiliza estem protocolo por defecto en lugar del otro?
UDP porque es más rápido que TCP, ya que no tiene repsuesta de confirmacion.
Puertos: Identifica el puerto de origen (dinámico) del cliente y el puerto de destino (conocido) del servidor.
Puerto de origen: 64775 Puerto de destino 53
Identificador: Expande la sección Domain Name System ¿Qué identificador de transacción(Transaction ID) vincula la respuesta del servidor con la petición de tu cliente?
La petición para MX google.com tiene la Transaction ID 0x0002. La respuesta correspondiente tambien muestra 0x0002, por lo que este identificadpr vincula ambas.


Flags: En el paquete de Respuesta, despliega la sección Flags. Busca la opción Authoritative Answer. ¿Está a 0 o a 1? ¿Qué significa esto?
Esta a 0, significa que la respuesta no procede directamente de un servidor DNS autoritativo para google.com
Respuestas (Answers): Despliega el bloque de respuestas. ¿Qué servidor de correo de Google tiene la prioridad (preference) más alta (el número más bajo)?


https://www.grc.com/dns/benchmark.htm

https://www.icann.org/resources/pages/what-2012-02-25-es

https://www.ionos.es/digitalguide/servidores/know-how/iana-que-es-y-cual-es-su-funcion/

https://www.cloudflare.com/es-es/learning/dns/dns-records/dns-soa-record/

https://ayuda.hostalia.com/hc/es/articles/360010530557--Qu%C3%A9-es-WHOIS

https://www.escueladeinternet.com/que-diferencia-hay-entre-registro-y-registrador-de-dominios/

