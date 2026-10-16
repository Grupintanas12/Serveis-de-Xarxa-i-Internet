## Configurar opciones de BIND en el esclavo

Archivo:

/etc/bind/named.conf.options
Contenido:

acl "safeclients" {
    localhost;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { safeclients; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };

    listen-on port 53 { 127.0.0.1; 192.168.1.101; };
    listen-on-v6 { none; };

    allow-transfer { none; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };
};


## Declarar zonas secundarias

Archivo:

/etc/bind/named.conf.local
Contenido completo:

zone "haven.local" {
    type secondary;
    file "/var/cache/bind/slaves/db.haven.local";
    masters { 192.168.1.100; };
};

zone "1.168.192.in-addr.arpa" {
    type secondary;
    file "/var/cache/bind/slaves/db.1.168.192";
    masters { 192.168.1.100; };
};
