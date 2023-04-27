## Conexion con Openvpn

Este projecto muestra el ejemplo de realizar la conexion de 2 servidores virtuales que no tiene conexion directa. La conexion entre ambos es un 3er servidor, el cual tiene conexion a nivel de red a los 2 primeros servidores. Para realizar esta conexion, se montara un tunel VPN tipo Road-Warrior con OpenVPN.

## VPN Road Warrior
Para este ejemplo, se desplegó 3 máquinas virtuales en VirtualBox con Sistema Operativo Centos 7.

Las direcciones ip de cada maquina son las siguientes.

```
server1     192.168.56.103
server2     192.168.56.102 / 172.26.0.10
server3     172.26.0.11
```

Por conveniencia, a cada uno de los servidores se les asignara los siguientes nombres

```
server1     vpn-client1
server2     vpn-server
server3     vpn-client2
```

`vpn-client1` y `vpn-server` tiene conexion por la red 192.168.56.0/24. 

![ScreenShot](/assets/ping-client1-server.png)
![ScreenShot](/assets/ping-server-client1.png)

`vpn-server` y `vpn-client2` tiene conexion por la red 172.26.0.0/24. 

![ScreenShot](/assets/ping-server-client2.png)
![ScreenShot](/assets/ping-client2-server.png)

`vpn-client1` y `vpn-client3` no tiene conexion directa, problema que se solucionara con la implementacion del tunel VPN. 

![ScreenShot](/assets/ping-client1-client2.png)


### Configurar certificados

Instalar Openvpn en los 3 servidores

`yum install openvpn -y`

![ScreenShot](/assets/install-openvpn-client1.png)

![ScreenShot](/assets/install-openvpn-server.png)

![ScreenShot](/assets/install-openvpn-client2.png)


## ==============

En `vpn-server`, activar la caracteristica `ip_fordward` para el reenvio de paquetes. Para ello agregamos la linea `net.ipv4.ip_forward = 1` en el archivo `/etc/sysctl.conf`. En caso de que ya exista la linea, solo cambiar el valor.

![ScreenShot](/assets/ip_forward.png)

Para validar que el cambio sea correcto, ejecuta el comando `sysctl -p`. Si cometiste un error, el comando imprimira una salida relacionada a un archivo no encontrado.

##### Correcto
![ScreenShot](/assets/ip_forward_validate_ok.png)

##### Incorrecto
![ScreenShot](/assets/ip_forward_validate_fail.png)


## ==============

Ahora vamos a generar los certificados en `vpn-server`. Para ello, primero instalamos el paquete `easy-rsa` 

`yum install easy-rsa -y`

![ScreenShot](/assets/install-easy-rsa-server.png)

Nos movemos al directorio `/usr/share/easy-rsa/3` y ejecutamos el comando `./easyrsa init-pki` para inicializar la infraestructura de clave publica.

![ScreenShot](/assets/easyrsa-init-pki.png)

La salida de este comando nos dice que todas las claves se van a guardar en el directorio `/usr/share/easy-rsa/3/pki`


Ahora vamos a contruir el certificado para el CA (Certificate Authority) con el comando `./easyrsa build-ca`. Nos pedirá que ingresemos una contraseña y un Valor para un campo *Common Name*. El el campo *Common Name* se puede dejar vacío con un enter

![ScreenShot](/assets/easyrsa-build-ca.png)

Se general el archivo `/usr/share/easy-rsa/3/pki/ca.crt` con el CA.

![ScreenShot](/assets/easyrsa-build-ca-file.png)


## ================  Servidor

Lo siguiente es generar una Solicitud de Certificado para `vpn-server` con el comando `./easyrsa gen-req server nopass`. La opcion nopass es para que no nos pida una contraseña cada vez que se inicie el tunel VPN en `vpn-server`. De la misma forma nos pedira un valor para el campo *Common Name*, mismo que puedes dejarlo vacio dando enter.

![ScreenShot](/assets/easyrsa-gen-req.png)

Los archivos que genera este comando son la solicitud de certificado (`/usr/share/easy-rsa/3/pki/reqs/server.req`) y la clave privada para el servidor (`/usr/share/easy-rsa/3/pki/private/server.key`)

![ScreenShot](/assets/easyrsa-gen-req-files.png)


Ahora generamos un parametro de Diffie-Hellman con el comando `openssl dhparam -out /usr/share/easy-rsa/3/pki/dh2048.pem 2048`

![ScreenShot](/assets/gen-diffie-hellman.png)


## ================  Clientes
Ahora vamos a generar una solicitud de certificado para cada uno de los clientes con el comando `./easyrsa gen-req client1 nopass`. La opcion `nopass` es para que no nos pida una contraseña cada vez que se inicie el tunel VPN en `vpn-client1`. El campo *Common Name* lo dejamos en blanco con dando un enter.

![ScreenShot](/assets/easyrsa-gen-req-clients.png)

Para generar la solicitud de certificados para los demas clientes, cambiamos el nombre de client1 a client2, client3 ...
```
./easyrsa gen-req client2 nopass
./easyrsa gen-req client3 nopass
...
``` 

Verificamos que existan los archivos generados con las solicitudes de cada cliente (`/usr/share/easy-rsa/3/pki/reqs/*`) y las claves privadas (`/usr/share/easy-rsa/3/pki/private/*`).

![ScreenShot](/assets/easyrsa-gen-req-clients-files.png)

Lo siguiente es firmar las solicitudes de Certificados con la CA generada inicialmente. 

Primero firmamos la solicitud de Certificado del Servidor con el comando `./easyrsa sign-req server server`.
    - `sign-req`: opcion para indicar que se desea firmar el certificado
    - `server`: tipo de solicitud, en este caso es la solicitud del servidor. *client* si vas a firmar la solicitud de un cliente
    - `server`: nombre de la solicitud del certificado

El comando nos va a solicitar que confirmemos el valor *CommonName* a server, escribimos `yes` para continuar. La contraseña que se solicita es la que se configuro al emitir el CA. 

![ScreenShot](/assets/easyrsa-sign-req-server.png)

De la misma forma firmamos los certificados para los clientes coon los comandos.
```
./easyrsa sign-req server client1
./easyrsa sign-req server client2
...
```
![ScreenShot](/assets/easyrsa-sign-req-client.png)

Los certificados emitidos se encuentra en el directorio `/usr/share/easy-rsa/3/pki/issued/*`

![ScreenShot](/assets/easyrsa-sign-req-certs.png)

Ahora copiamos los archivos `/usr/share/easy-rsa/3/pki/ca.crt`, `/usr/share/easy-rsa/3/pki/dh2048.pem`, `/usr/share/easy-rsa/3/pki/issued/server.crt` y `/usr/share/easy-rsa/3/pki/private/server.key` a `/etc/openvpn`.

![ScreenShot](/assets/server-dir.png)

Preparo un directorio para cada cliente con los archivos de confiugracion que le corresponden a cada uno.
```
cd /usr/share/easy-rsa/3/
mkdir client1 client2
```

Por cada cliente, copio a su directorio recien creado el ca.crt, certificado y la clave privada generada en los pasos anteriores.
```
cd /usr/share/easy-rsa/3
cp pki/ca.crt pki/issued/client1.crt pki/private/client1.key client1/
cp pki/ca.crt pki/issued/client2.crt pki/private/client2.key client2/
```
![ScreenShot](/assets/clients-dir.png)


Genero un archivo comprimido con los archivos del directorio de cada cliente.
```
/usr/share/easy-rsa/3/client1
tar zcf ~/client1.tgz .

/usr/share/easy-rsa/3/client2
tar zcf ~/client2.tgz .
```

![ScreenShot](/assets/clients-vpn-files.png)

Copiamos los archivos comprimidos a los respectivos clientes.
```
scp ~/client1.tgz root@192.168.56.103:/etc/openvpn/
scp ~/client2.tgz root@172.26.0.11:/etc/openvpn/
```


### Configuración OpenVPN

En `vpn-server`, creamos el archivo `/etc/openvpn/server.conf`. El archivo debe contener el siguiente contenido
```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh2048.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 172.26.0.10 255.255.255.0"
keepalive 10 120
comp-lzo
user nobody
group nobody
persist-key
persist-tun
status openvpn-status.log
verb 4
```

Arrancamos el servicio de openvpn
```
systemctl start openvpn@server.service
systemctl enable openvpn@server.service
systemctl status openvpn@server.service
```

Verifico que el servicio haya arrancado correctamente.

![ScreenShot](/assets/verify-vpn-server.png)


Para configurar la conexion en cada cliente, nos vamos al servidor de cada cliente en el directorio `/etc/openvpn`. Dentro de este directorio, descomprimimos el archivo que pasamos hace unos pasos atras.
```
cd /etc/openvpn
tar xf 
tar xf client1.tgz
```

![ScreenShot](/assets/client1-openvpn-dir.png)

Creamos el archivo `/etc/openvpn/client1.conf`. El archivo debe contener el siguiente contenido
```
client
dev tun
proto udp
remote 192.168.56.102 1194
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
comp-lzo
verb 4
```

Arrancamos el servicio de openvpn
```
systemctl start openvpn@client1.service
systemctl enable openvpn@client1.service
systemctl status openvpn@client1.service
```
