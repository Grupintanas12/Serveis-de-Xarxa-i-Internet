DNS 

- Dentro de una misma VM hemos simulado que un cliente le haga un request a un servidor
- Primero nos hemos descargado dos scripts de Python uno simulando el Server y el otro simulando el Cliente.
- Nos hemos tenido que instalar python3, comprobar que los scripts estaban en versión python3.
- Al ejecutar ha saltado un error de que el puerto DNS(53),por culpa del bind9.
- Nos instalamos wireshark, para analizar el Request/Answer entre el Server y el Cliente


<img width="1272" height="586" alt="Wireshark1" src="https://github.com/user-attachments/assets/87042dfd-0806-426f-b0de-b26784ec23bb" />
<img width="1306" height="667" alt="Wiresahrk2" src="https://github.com/user-attachments/assets/edf1337f-5604-45f7-92da-ce567d71ba21" />
<img width="593" height="188" alt="wireshark2 1" src="https://github.com/user-attachments/assets/5f7376cb-df85-4acd-b557-aacac4501a0b" />


DETALLES QUE PODEMOS OBSERVAR

Las Ips son la 127.0.0.1 que es la de Loopback
El puerto en los que se comunican son el 53 y un puerto muy por encima del 1023.
La pagina a la cual quiere llegar es la secreto.com con IP 4.3.2.1
