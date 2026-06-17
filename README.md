Install custom wireguard tunnel in Ubuntu 24.04.

Instalar tunnel wireguard sobre linux Ubuntu 24.04 (Gestor de DNS predeterminado systemd-resolved) 
Escenario del problema: Instalar wireguard en este escenario con el script wg-quick up, a veces reporta error, pues no se reconoce el dispositivo de red a implementar, configurado a partir del fichero de configuración generado por Mullvad:
¿Què sucedió? Repitamos los pasos recomendado por MullVad
Paso 1: Log in as root
rootprofile@rootprofile-MYDEVICE:~$ sudo --login [sudo] contraseña para rootprofile: #**************
Paso 2: Se toma el control como root y nos ubicamos en el directorio /etc/wireguard/
root@rootprofile-MYDEVICE:~# rootprofile-MYDEVICEcd /etc/wireguard/ 
root@rootprofile-MYDEVICE:/etc/wireguard#
Paso 3: Ejecutamos el script wg-quick up para crear el dispositivo wireguard a partir de su fichero de configuración
root@rootprofile-MYDEVICE:/etc/wireguard# sudo WG_QUICK_USERSPACE=no WG_QUICK_UP_SUDO=1 wg-quick up es-mad-wg-201 
[#] ip link add es-mad-wg-201 type wireguard 
[#] wg setconf es-mad-wg-201 /dev/fd/63 
[#] ip -4 address add 10.65.183.36/32 dev es-mad-wg-201 
[#] ip -6 address add fc00:bbbb:bbbb:bb01::2:b723/128 dev es-mad-wg-201 
[#] ip link set mtu 1420 up dev es-mad-wg-201
Y al ejecutar el comando «resolvconf» del paquete gestor deb «systemd-resolved (255.4-1ubuntu8.11)»
aparece el error "No such device" y a la salida el script wg-quick up borra la interfaz registrada
(en este es-mad-wg-201 desde su fichero homólogo de configuración)
[#] resolvconf -a tun.es-mad-wg-201 -m 0 -x Failed to resolve interface "tun.es-mad-wg-201": No such device [#] ip link delete dev es-mad-wg-201
Causa probable: Este error específico viene de resolvconf (alias de  resolvectl) intentando acceder al dispositivo "tun.es-mad-wg-201" pero wireguard creó el dispositivo "es-mad-wg-201" o en la resolución del dominio o el registro del dns server de Mullvad (100.64.0.55). La herramiente resolvconf es un alias de compatibilidad que es proveido por el kernel de linux para permitir la compatibilidad de comandos nativos de diferentes versiones de linux con la herramienta resolvectl del gestor systemd-resolved de Ubuntu 24.04, creando fallos durante la ejecución del comando ya que varias opciones no se implementan en Ubuntu 24.04 ya que varios paquetes quedaron depreciados al eliminarse la herramienta de resolución de dominios Openresolv de las versiones anteriores de linux, por eso este script (wg-quick up de wireguard) falla en Ubuntu 24.04 donde el gestor predeterminado de resolución de DNS es systemd-resolved y no Openresolv.

RESOLUCIÒN:
Recomendación: No usar el script wg-quick up y en su lugar realizar el registro y el lanzamiento de la interfaz de red del tunel vpn de wireguard manualmente usando resolvectl directamente en lugar de resolvconf

"Configuración, registro y activación manual del tunel vpn wireguard en Ubuntu 24.04 (Noble numbat)".
Paso 1: log as root
rootprofile@rootprofile-MYDEVICE:~$ sudo --login [sudo] contraseña para rootprofile: #**************
Previamente se mueve, desde la carpeta de descarga, el fichero de configuración de Mullvad (descargado y generado desde la pagina (https://mullvad.net/)) en la sesión del usuario autenticado correspondiente, a la carpeta /etc/wireguard. En este ejemplo usamos el fichero de configuración es-mad-wg-201.conf:
root@rootprofile-MYDEVICE:~# mv es-mad-wg-201.conf /etc/wireguard

Paso 2: Otorgar los permisos correctos al fichero de cofiguración de manera que solo root pueda leerlo:
root@rootprofile-MYDEVICE:~# sudo chown root:root -R /etc/wireguard && sudo chmod 600 -R /etc/wireguard

Paso 3: Cambiar al directorio /etc/wireguard
root@rootprofile-MYDEVICE:~# cd /etc/wireguard 
root@rootprofile-MYDEVICE:/etc/wireguard#

Paso 4: Se edita el fichero de configuración y se comentan las líneas Address y DNS para evitar posibles errores de sintasis con el comando setconf.
root@rootprofile-MYDEVICE:/etc/wireguard#vim es-mad-wg-201.conf 
#################################################
[Interface]
Device: Rapid Koala
PrivateKey = yHnY3g/lK3k4f2fXcGefJc3PnQTfHsbpI8REvRuZ+lE=
# Address = 10.65.183.36/32,fc00:bbbb:bbbb:bb01::2:b723/128
# DNS = 100.64.0.55
[Peer] 
PublicKey = LyO4Xs1eV8JwFr63a1FRnKboQn2Tu/oeMzHhbr7Y6GU= 
AllowedIPs = 0.0.0.0/0,::0/0 
Endpoint = 146.70.128.194:51820 

"es-mad-wg-201.conf" 10L, 298B
#################################################

Paso 5: Otorgar los permisos correctos al fichero de cofiguración de manera que solo root pueda leerlo (si no se hizo anteriormente)
root@rootprofile-MYDEVICE:/etc/wireguard# chmod 600 /etc/wireguard/es-mad-wg-201.conf

Paso 6: Ahora podemos adicionar manualmente la interfaz de wireguard con ip-link:
root@rootprofile-MYDEVICE:/etc/wireguard# ip link add dev es-mad-wg-201 type wireguard

Paso 7: Y luego se configura la interfaz pura wireguard adicionada con su archivo de configuración:
root@rootprofile-MYDEVICE:/etc/wireguard# wg setconf es-mad-wg-201 es-mad-wg-201.conf

Paso 8: Posteriormente se asignan las IP (IPv4/IPv6) a la interfaz de las configuraciones Address comentadas en el archivo de configuración:
root@rootprofile-MYDEVICE:/etc/wireguard# ip -4 address add 10.65.183.36/32 dev es-mad-wg-201 
root@rootprofile-MYDEVICE:/etc/wireguard# ip -6 address add fc00:bbbb:bbbb:bb01::2:b723/128 dev es-mad-wg-201

Paso 9: Ya se puede activar la interfaz con el comando ip link set up
root@rootprofile-MYDEVICE:/etc/wireguard# ip link set mtu 1420 up dev es-mad-wg-201

Paso 10: Finalmente se registra y reconfigura el DNS resolver de la interfaz (Mullvad) para que resuelva las solicitudes de tráfico de nombres de dominio, direcciones IPv4/IPv6, servicios y recursos DNS:
root@rootprofile-MYDEVICE:/etc/wireguard# resolvectl dns es-mad-wg-201 100.64.0.55 
root@rootprofile-MYDEVICE:/etc/wireguard# resolvectl domain es-mad-wg-201 "~."

Paso 11: Verificar la configuración y las key de la interfaz y del peer con wg show o wg showconf
root@rootprofile-MYDEVICE:/etc/wireguard# wg show 
interface: es-mad-wg-201 
public key: GjW+VmtbOWaCsVwAoYM+k96mOzEbsXojh7Xv636Hgxg= 
private key: (hidden) listening port: 50336
peer: LyO4Xs1eV8JwFr63a1FRnKboQn2Tu/oeMzHhbr7Y6GU= 
endpoint: 146.70.128.194:51820 
allowed ips: 0.0.0.0/0, ::/0 
latest handshake: 1 minute, 26 seconds ago 
transfer: 6.07 KiB received, 3.38 KiB sent

Paso 12: Confirmar DNS de la interfaz (debe coincidir con el DNS de Mullvad 100.64.0.55)
root@rootprofile-MYDEVICE:/etc/wireguard# resolvectl dns es-mad-wg-201 
Link 5 (es-mad-wg-201): 100.64.0.55

Paso 13: Confirmar las IP (IPv4/IPv6) asignadas a la interfaz:
root@rootprofile-MYDEVICE:/etc/wireguard# ip addr show es-mad-wg-201
5: es-mad-wg-201: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000 link/none inet 10.65.183.36/32 scope global es-mad-wg-201 valid_lft forever preferred_lft forever inet6 fc00:bbbb:bbbb:bb01::2:b723/128 scope global valid_lft forever preferred_lft forever

Paso 14: Verificar la IP de Mullvad
root@rootprofile-MYDEVICE:/etc/wireguard# curl https://am.i.mullvad.net/ip

Paso 15: Verificar si hay conección establecida con Mullvad
root@rootprofile-MYDEVICE:/etc/wireguard# curl https://am.i.mullvad.net/connected

Paso 16: Verificar si hay fugas de IPv4
root@rootprofile-MYDEVICE:/etc/wireguard# curl https://api.ipify.org

Paso 17: Verificar si hay fugas de DNS
root@rootprofile-MYDEVICE:/etc/wireguard# curl https://am.i.mullvad.net/dns-leak

Paso 18: Listamos todas las interfaces de red activas:
root@rootprofile-MYDEVICE:/etc/wireguard# ip link show 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 
2: enp7s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000 link/ether 8c:73:6e:b0:17:16 brd ff:ff:ff:ff:ff:ff 
3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DORMANT group default qlen 1000 link/ether 90:00:4e:54:e2:5c brd ff:ff:ff:ff:ff:ff 
5: es-mad-wg-201: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000 link/none

O simplemente, listamos solamente la interfaz de wireguard
root@rootprofile-MYDEVICE:/etc/wireguard# ip link show es-mad-wg-201 
5: es-mad-wg-201: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000 link/none

Paso 19: Verificar si hay tráfico por la interfaz wg (es este caso es-mad-wg-201)
root@rootprofile-MYDEVICE:/etc/wireguard# ip route show

Paso 20: Verificar si la interfaz wg es la predeterminada (default route dev) (es en este caso la interfaz es-mad-wg-201)
root@rootprofile-MYDEVICE:/etc/wireguard# ip route show table all | grep -i default

Paso 21: Y, por ùltimo, comprobamos las tablas de enrutamiento
root@rootprofile-MYDEVICE:/etc/wireguard# ip rule show

Si la interfaz no es la predeterminada entonces, por método agregativo una vez levantada, se obliga a ser la ruta predeterminada:
Paso 1: Primero se borra la ruta predeterminada existente
root@rootprofile-MYDEVICE:/etc/wireguard# ip route del default 2>/dev/null

Paso 2: Luego se designa la interfaz de wireguard como la predeterminada y probamos nuevamente los pasos anteriores
root@rootprofile-MYDEVICE:/etc/wireguard# ip route add default dev es-mad-wg-201

Paso 3: Comprobamos resolver un dominio conocido para probar la interfaz
root@rootprofile-MYDEVICE:/etc/wireguard# dig google.com
; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> google.com 
;; global options: +cmd 
;; Got answer: 
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24977 
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION: 
; EDNS: version: 0, flags:; udp: 65494 
;; QUESTION SECTION: 
;google.com.			IN	A
;; ANSWER SECTION: 
google.com.		86	IN	A	74.125.206.100 
google.com.		86	IN	A	74.125.206.102 
google.com.		86	IN	A	74.125.206.139 
google.com.		86	IN	A	74.125.206.101 
google.com.		86	IN	A	74.125.206.113 
google.com.		86	IN	A	74.125.206.138
;; Query time: 16 msec 
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP) 
;; WHEN: Mon Dec 15 10:28:35 CST 2025 
;; MSG SIZE  rcvd: 135

Otro ejemplo:
root@rootprofile-MYDEVICE:/etc/wireguard# nslookup mullvad.net 
Server:		127.0.0.53 
Address:	127.0.0.53#53
Non-authoritative answer: 
Name:	mullvad.net 
Address: 45.83.223.209

Paso 4: Probar tráfico a través de la interfaz:
root@rootprofile-MYDEVICE:/etc/wireguard# ping -I es-mad-wg-201 1.1.1.1
PING 1.1.1.1 (1.1.1.1) from 10.65.183.36 es-mad-wg-201: 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=59 time=42.7 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=59 time=14.8 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=59 time=33.7 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=59 time=9.67 ms
64 bytes from 1.1.1.1: icmp_seq=5 ttl=59 time=18.0 ms
^C
--- 1.1.1.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4007ms
rtt min/avg/max/mdev = 9.666/23.771/42.718/12.422 ms

Paso 5: Finalmente comprobamos las interfaces activas y su resolución por el gestor de dns systemd-resolved de Ubuntu 24.04:
root@rootprofile-MYDEVICE:/etc/wireguard# resolvectl status 
Global Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported resolv.conf mode: stub
Link 2 (enp7s0) Current Scopes: none Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Link 3 (wlp3s0) Current Scopes: DNS Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported Current DNS Server: 192.168.10.1 DNS Servers: 192.168.10.1 DNS Domain: lan
Link 5 (es-mad-wg-201) Current Scopes: DNS Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported Current DNS Server: 100.64.0.55 DNS Servers: 100.64.0.55 DNS Domain: ~.

En caso de ocurrir algún error de resolución:

Paso 1: Ver logs del mismo:
root@rootprofile-MYDEVICE:/etc/wireguard# journalctl -xe -u systemd-resolved | tail -20

Paso 2: Detener la interfaz de red de wireguard
root@rootprofile-MYDEVICE:/etc/wireguard# ip link set mtu 1420 down dev se-got-wg-001

Paso 3: Borrar la interfaz de wireguard
root@rootprofile-MYDEVICE:/etc/wireguard# ip link delete dev se-got-wg-001

Paso 4: Ver Tabla de ruteo previa del equipo para comparar luego de restaurar a la interfaz local
root@rootprofile-MYDEVICE:/etc/wireguard# ip route show table main

Paso 5: Una vez resuelto el problema, restaurar la interfaz de wireguard repitiendo los paso del 1 al 21.





Paso 4: Copiar Tabla de ruteo previa del equipo para comparar luego de restaurar a la interfaz local
root@rootprofile-MYDEVICE:/etc/wireguard# ip route show table main 
