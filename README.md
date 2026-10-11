# Infraestructura 1: FortiGate con DMZ y segmentación por VLAN

**Video demostrativo:** [Ver en YouTube(https://youtu.be/vi289vaGnQ4)

Autor: Jorge Emmanuel Sime Matos | Matrícula: 2025-0763
Instituto Tecnológico de las Américas (ITLA), Seguridad Informática | Asignatura: Seguridad de Redes

## Propósito del laboratorio

Este laboratorio implementa una red segmentada protegida por un FortiGate configurado por interfaz gráfica. La red de usuarios (VLAN 10 y VLAN 20) queda separada de una DMZ que aloja tres servidores: Sistema de Caja, Sistema de Inventario y un servidor de base de datos.

Las políticas de firewall cumplen los siguientes objetivos:

- Evitar que el tráfico de la DMZ llegue a la red LAN de usuarios.
- Impedir que la DMZ tenga acceso abierto a Internet, permitiendo únicamente el repositorio de actualizaciones del sistema operativo de los servidores.
- Permitir el acceso SSH a los servidores solamente desde la VLAN 20.
- Restringir a la VLAN 10 el acceso al Sistema de Inventario, mostrando una página de violación de política y registrando el evento.

## Topología

La topología se construyó en GNS3 con un FortiGate 7.0.9, dos switches, dos clientes webterm (uno por VLAN) y tres servidores Alpine Linux. El FortiGate sale a Internet a través de un nodo NAT.

![Topología en GNS3](docs/imagenes/01-topologia-gns3.png)

```mermaid
graph TD
  NET[Internet / NAT] --- FG[FortiGate 7.0.9]
  FG ---|port2 trunk VLAN 10 y 20| SW1[SW1]
  FG ---|port3 DMZ VLAN 30| SW2[SW2]
  SW1 --- W1[webterm-1 VLAN 10]
  SW1 --- W2[webterm-2 VLAN 20]
  SW2 --- CAJA[Caja 10.7.64.2]
  SW2 --- INV[Inventario 10.7.64.3]
  SW2 --- DB[DB 10.7.64.4]
```

## Direccionamiento IP

El direccionamiento se calculó a partir de los dígitos 07 y 63 de la matrícula (2025-0763).

| Segmento | Red | Gateway | Observación |
|---|---|---|---|
| VLAN 10 Usuarios | 10.7.63.0/25 | 10.7.63.1 | DHCP, subinterfaz de port2 |
| VLAN 20 Usuarios | 10.7.63.128/25 | 10.7.63.129 | DHCP, subinterfaz de port2 |
| DMZ (VLAN 30) | 10.7.64.0/28 | 10.7.64.1 | Servidores, port3 |
| WAN | 192.168.42.0/24 | 192.168.42.1 | port1 por DHCP del NAT |

| Servidor | IP | Servicio |
|---|---|---|
| Sistema de Caja | 10.7.64.2 | Web (HTTP) |
| Sistema de Inventario | 10.7.64.3 | Web (HTTP) |
| DB Server | 10.7.64.4 | MariaDB (3306) |

## Switches

SW1 conecta los usuarios y SW2 la DMZ. Son routers c2691 con el módulo de switch NM-16ESW emulado en GNS3. En ambos, los puertos sin uso están en la VLAN 999 (BLACKHOLE) y apagados. Las VLAN no aparecen en el running-config porque este módulo las guarda en su base de datos de VLAN; su estado se muestra en las capturas.

El puerto fa1/0 de SW1 es un trunk 802.1Q hacia el FortiGate. El módulo emulado no aplicó la restricción de VLAN permitidas ni la VLAN nativa en el trunk.

![VLAN de SW1](docs/imagenes/06-sw1-vlans.png)
![Trunk de SW1](docs/imagenes/07-sw1-trunk.png)
![VLAN de SW2](docs/imagenes/08-sw2-vlans.png)

Configuraciones completas: [SW1](configs/SW1-running-config.txt) y [SW2](configs/SW2-running-config.txt).

## FortiGate

La configuración se realizó por GUI. La única excepción fue la creación inicial de la subinterfaz VLAN10 por consola, necesaria para abrir la GUI por primera vez desde la red de usuarios. El acceso a la GUI se hizo por HTTP desde la VLAN 10.

Respaldo de la configuración: [FortiGate-backup.conf](configs/FortiGate-backup.conf).

### Interfaces, rutas y DNS

port1 obtiene dirección por DHCP del NAT. port2 funciona como trunk con las subinterfaces VLAN10 y VLAN20, ambas con servidor DHCP. port3 es la interfaz DMZ. El FortiGate ofrece servicio DNS recursivo en VLAN10, VLAN20 y DMZ, y los servidores de la DMZ lo usan como DNS para resolver el repositorio de actualizaciones.

![Interfaces](docs/imagenes/02-interfaces-fortigate.png)
![Rutas](docs/imagenes/10-rutas-estaticas.png)
![DNS por interfaz](docs/imagenes/03-dns-servicio-interfaces.png)
![DNS](docs/imagenes/04-dns-settings.png)

### Objetos de dirección

| Objeto | Tipo | Valor |
|---|---|---|
| VLAN10-NET | Subred | 10.7.63.0/255.255.255.128 |
| VLAN20-NET | Subred | 10.7.63.128/255.255.255.128 |
| DMZ-NET | Subred | 10.7.64.0/255.255.255.240 |
| SRV-CAJA | Subred | 10.7.64.2/32 |
| SRV-INVENTARIO | Subred | 10.7.64.3/32 |
| SRV-DB | Subred | 10.7.64.4/32 |
| FQDN-ALPINE | FQDN | dl-cdn.alpinelinux.org |
| UPDATE-ENDPOINTS | Grupo | Contiene FQDN-ALPINE |

### Perfil Web Filter

El perfil `BLOQUEO-INVENTARIO` usa un filtro de URL estático que bloquea la dirección 10.7.64.3. Se aplica únicamente en la política 3, de modo que el usuario de la VLAN 10 que intenta abrir el Sistema de Inventario recibe la página de violación de política y el evento queda registrado.

### Políticas de firewall

El FortiGate evalúa las políticas de arriba hacia abajo. Todo tráfico no listado cae en la denegación implícita, por lo que la DMZ no tiene acceso abierto a Internet.

| # | Nombre | Origen | Destino | Servicio | Acción |
|---|---|---|---|---|---|
| 1 | V20-SSH-DMZ | VLAN20 | DMZ-NET | SSH | Aceptar |
| 2 | V10-DENY-SSH | VLAN10 | DMZ-NET | SSH | Denegar con log |
| 3 | V10-INV-BLOCK | VLAN10 | SRV-INVENTARIO | HTTP | Aceptar con Web Filter (bloquea) |
| 4 | V10-DENY-INV | VLAN10 | SRV-INVENTARIO | ALL | Denegar con log |
| 5 | V10-WEB-CAJA | VLAN10 | SRV-CAJA | HTTP, HTTPS | Aceptar |
| 6 | V20-WEB-DMZ | VLAN20 | Caja, Inventario | HTTP, HTTPS | Aceptar |
| 7 | DMZ-DENY-LAN | DMZ | VLAN10, VLAN20 | ALL | Denegar con log |
| 8 | DMZ-UPDATES | DMZ-NET | UPDATE-ENDPOINTS | HTTP, HTTPS | Aceptar con NAT |
| 9 | LAN-INTERNET | VLAN10, VLAN20 | all | ALL | Aceptar con NAT |

![Políticas](docs/imagenes/05-politicas-lista.png)

## Servidores

Los tres servidores son Alpine Linux con red estática, gateway 10.7.64.1 y el FortiGate como DNS. Los comandos de instalación están en [scripts/servidores-setup.sh](scripts/servidores-setup.sh).

## Pruebas realizadas

### La DMZ no tiene acceso abierto a Internet

Desde el servidor de la DMZ, el ping hacia la LAN (10.7.63.10) y hacia 8.8.8.8 falla con 100 % de pérdida, mientras que `apk update` funciona porque dl-cdn.alpinelinux.org está permitido por la política DMZ-UPDATES. Una descarga HTTP hacia otro sitio falla por tiempo de espera.

![Pings y apk update desde la DMZ](docs/imagenes/11-dmz-ping-apk.png)
![Descarga a un sitio no autorizado](docs/imagenes/09-wget-timeout-dmz.png)

### Bloqueo de la VLAN 10 al Sistema de Inventario

El registro de tráfico del FortiGate muestra los intentos de la VLAN 10 hacia 10.7.64.3 atendidos por la política V10-INV-BLOCK (3) con el resultado "Deny: policy violation". Las dos primeras entradas permitidas son de pruebas anteriores a activar el perfil Web Filter.

![Registro de bloqueo](docs/imagenes/13-log-bloqueo-inventario.png)

### Servidor de base de datos

![MariaDB](docs/imagenes/12-mysql-db.png)

## Estructura del repositorio

```
README.md
docs/imagenes/
configs/FortiGate-backup.conf
configs/SW1-running-config.txt
configs/SW2-running-config.txt
scripts/servidores-setup.sh
```
# Infraestructura 2: FortiGate + VPN IPsec + Jump Server (RemoteApp)

Laboratorio en **GNS3** que conecta un sitio de usuarios (router Cisco) con un sitio de servidores protegido por un **FortiGate**, mediante una **VPN IPsec site-to-site**. Los usuarios solo pueden llegar a un **Jump Server (Windows Server con RemoteApp)**, y desde ahí acceden de forma controlada al **Web Server "Sistema de Caja"**.

![Topología general](docs/img/01-topologia.png)

---

## Tabla de contenido

1. [Objetivos](#1-objetivos)
2. [Requisitos](#2-requisitos)
3. [Dispositivos](#3-dispositivos)
4. [Direccionamiento](#4-direccionamiento)
5. [Conexiones](#5-conexiones)
6. [Configuración del ISP](#6-configuración-del-isp)
7. [Configuración de R2 (sitio de usuarios)](#7-configuración-de-r2-sitio-de-usuarios)
8. [VPN IPsec](#8-vpn-ipsec)
9. [FortiGate: red, objetos y políticas](#9-fortigate-red-objetos-y-políticas)
10. [Web Server (Alpine)](#10-web-server-sistema-de-caja-alpine)
11. [Jump Server (Windows Server + RemoteApp)](#11-jump-server-windows-server--remoteapp)
12. [Conexión de la VM a GNS3 (VMnet2)](#12-conexión-de-la-vm-a-gns3-vmnet2)
13. [Pruebas de validación](#13-pruebas-de-validación)
14. [Problemas comunes](#14-problemas-comunes)

---

## 1. Objetivos

- Establecer un túnel **IPsec** entre R2 (Cisco) y el FortiGate.
- Limitar la VPN para que **solo dé acceso al Jump Server**, con dos capas: ACL del túnel en R2 y política en el FortiGate.
- Denegar **SSH** al usuario sin privilegios.
- Publicar aplicaciones mediante **RemoteApp / RD Web Client** con asignación por usuario.
- Restringir al Jump Server a **HTTPS, RDP y SSH** hacia el Web Server. Todo lo demás cae en la denegación implícita.

## 2. Requisitos

- GNS3 con las imágenes: FortiGate 7.0.9, Cisco c2691, Ethernet switch, webterm y Alpine Linux.
- Una VM de **Windows Server 2019 o 2022** (sirve la ISO de evaluación) en VMware. RemoteApp y RemoteApp Web Client son funciones de Windows Server (Servicios de Escritorio Remoto), por eso no puede reemplazarse por un equipo Linux.
- Nodo **Cloud** de GNS3 para conectar la VM a la topología.

> No se usan VPCS, c7200 ni el Switch vIOS.

## 3. Dispositivos

| Dispositivo | Cant. | Función |
|---|---|---|
| FortiGate 7.0.9 | 1 | Firewall del sitio de servidores, VPN y políticas (todo por GUI) |
| c2691 | 2 | ISP (IPs públicas) y R2 (equipo Cisco del sitio de usuarios) |
| Ethernet switch | 1 | VLAN 10 de los usuarios |
| webterm | 2 | Usuario sin privilegios y usuario con privilegios |
| Alpine Linux | 1 | Web Server: Sistema de Caja |
| Cloud | 1 | Conecta la VM de Windows Server (Jump Server) |
| NAT | 1 | Temporal, solo para instalar paquetes en el Web Server |

## 4. Direccionamiento

Se usan los dígitos de matrícula **07** y **63**.

| Segmento | Red | Direcciones |
|---|---|---|
| ISP a FortiGate | 200.7.63.0/30 | ISP .1, FortiGate port1 .2 |
| ISP a R2 | 200.7.63.4/30 | ISP .5, R2 fa0/0 .6 |
| Usuarios (VLAN 10) | 10.7.63.0/25 | Gateway R2 .1, sin privilegios .10, con privilegios .11 |
| LAN del Jump Server | 10.7.64.0/29 | FortiGate port2 .1, Jump Server .2 |
| LAN del Web Server | 10.7.64.8/29 | FortiGate port3 .9, Web Server .10 |

## 5. Conexiones

| Desde | Hacia |
|---|---|
| ISP fa0/0 | FortiGate port1 |
| ISP fa0/1 | R2 fa0/0 |
| R2 fa0/1 | Ethernet switch, puerto 0 |
| Switch, puerto 1 | webterm-1 (sin privilegios) |
| Switch, puerto 2 | webterm-2 (con privilegios) |
| FortiGate port2 | Cloud (VM del Jump Server) |
| FortiGate port3 | Alpine Web Server |

## 6. Configuración del ISP

Consola del c2691:

```
enable
conf t
hostname ISP
no ip domain-lookup
interface fa0/0
 description HACIA-FORTIGATE
 ip address 200.7.63.1 255.255.255.252
 no shutdown
interface fa0/1
 description HACIA-R2
 ip address 200.7.63.5 255.255.255.252
 no shutdown
end
wr
```

## 7. Configuración de R2 (sitio de usuarios)

### Switch Ethernet (clic derecho > Configure)

- Puerto 0: tipo **dot1q**, VLAN nativa **999**.
- Puertos 1 y 2: tipo **access**, VLAN **10**.

### Reservas DHCP

En cada webterm ejecuta `ip link show eth0` y toma la MAC. Para la MAC `02:42:9c:b5:10:00` el identificador queda `0102.429c.b510.00` (prefijo `01` más la MAC en grupos de cuatro).

```
enable
conf t
hostname R2
no ip domain-lookup
interface fa0/0
 description WAN-HACIA-ISP
 ip address 200.7.63.6 255.255.255.252
 no shutdown
interface fa0/1
 no ip address
 no shutdown
interface fa0/1.10
 description USUARIOS-VLAN10
 encapsulation dot1Q 10
 ip address 10.7.63.1 255.255.255.128
ip route 0.0.0.0 0.0.0.0 200.7.63.5
ip dhcp excluded-address 10.7.63.1 10.7.63.29
ip dhcp pool USUARIOS
 network 10.7.63.0 255.255.255.128
 default-router 10.7.63.1
ip dhcp pool USER-SINPRIV
 host 10.7.63.10 255.255.255.128
 client-identifier 01<MAC-DE-WEBTERM-1>
 default-router 10.7.63.1
ip dhcp pool USER-PRIV
 host 10.7.63.11 255.255.255.128
 client-identifier 01<MAC-DE-WEBTERM-2>
 default-router 10.7.63.1
end
wr
```

En cada webterm, **Edit config**:

```
auto eth0
iface eth0 inet dhcp
```

Reinicia el nodo. Si la reserva no funciona, `show ip dhcp binding` muestra qué IP recibió cada uno; esas son las que se usan en los objetos del FortiGate.


## 8. VPN IPsec

### Lado Cisco (R2)

```
conf t
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
crypto isakmp key <PRE-SHARED-KEY> address 200.7.63.2
crypto ipsec transform-set TS-FG esp-aes 256 esp-sha-hmac
 mode tunnel
ip access-list extended VPN-A-JUMP
 permit ip 10.7.63.0 0.0.0.127 host 10.7.64.2
crypto map CM-FG 10 ipsec-isakmp
 set peer 200.7.63.2
 set transform-set TS-FG
 match address VPN-A-JUMP
interface fa0/0
 crypto map CM-FG
end
wr
```

La ACL limita el túnel a un solo destino, el Jump Server (`10.7.64.2`). Es la **primera capa** de la restricción "la VPN solo da acceso al Jump Server".

### Lado FortiGate (por GUI)

**VPN > IPsec Tunnels > Create New > Custom**

| Parámetro | Valor |
|---|---|
| Nombre | VPN-R2 |
| Remote Gateway | Static IP 200.7.63.6, interfaz port1 |
| NAT Traversal | Desactivado |
| Autenticación | Pre-shared Key, IKE v1, modo Main |
| Fase 1 | AES256, SHA1, DH grupo 2 (solo ese), lifetime 86400 |
| Fase 2 | AES256, SHA1, PFS desactivado, lifetime 3600, Auto-negotiate activo |
| Selectores fase 2 | Local `10.7.64.2/32`, Remote `10.7.63.0/25` (espejo de la ACL de R2) |

**Ruta hacia los usuarios** (Network > Static Routes): destino `10.7.63.0/25`, interfaz `VPN-R2`.

![Configuración del túnel en FortiGate](docs/img/03-fortigate-tunel.png)

## 9. FortiGate: red, objetos y políticas

### Acceso inicial por consola (una sola vez)

Entra con `admin`, define una contraseña nueva y pega:

```
config system interface
edit "port2"
set mode static
set ip 10.7.64.1 255.255.255.248
set allowaccess ping https http
next
end
```

Luego abre `https://10.7.64.1` desde el navegador del Jump Server (o `http://10.7.64.1` si el certificado da problemas).

> Mientras la VM de Windows no esté lista, puedes usar un Alpine temporal en port2 con IP `10.7.64.2/29` y probar la VPN con ping.

### Interfaces (Network > Interfaces)

| Interfaz | Configuración |
|---|---|
| port1 | Role WAN, `200.7.63.2/255.255.255.252`, acceso PING |
| port2 | Alias JUMP, `10.7.64.1/255.255.255.248` |
| port3 | Alias WEB, `10.7.64.9/255.255.255.248`, solo PING |

**Ruta por defecto:** `0.0.0.0/0` por port1, gateway `200.7.63.1`.

![Interfaces del FortiGate](docs/img/04-fortigate-interfaces.png)

### Objetos (Policy & Objects > Addresses)

| Objeto | Valor |
|---|---|
| LAN-CLIENTES | 10.7.63.0/25 |
| USER-SINPRIV | 10.7.63.10/32 |
| USER-PRIV | 10.7.63.11/32 |
| SRV-JUMP | 10.7.64.2/32 |
| SRV-WEB | 10.7.64.10/32 |

### Políticas (en este orden, con *Log Allowed Traffic* activo)

| # | Nombre | Entrada | Salida | Origen | Destino | Servicio | Acción |
|---|---|---|---|---|---|---|---|
| 1 | DENY-SSH-SINPRIV | VPN-R2 | JUMP, WEB | USER-SINPRIV | SRV-JUMP, SRV-WEB | SSH | Denegar |
| 2 | VPN-A-JUMP | VPN-R2 | JUMP | LAN-CLIENTES | SRV-JUMP | HTTPS, PING | Aceptar |
| 3 | JUMP-A-WEB | JUMP | WEB | SRV-JUMP | SRV-WEB | HTTPS, RDP, SSH | Aceptar |

- **Política 1:** restricción explícita de SSH para el usuario sin privilegios.
- **Política 2:** segunda capa de "la VPN solo llega al Jump Server".
- **Política 3:** limita al Jump Server a esos tres servicios del Web Server.
- Todo lo demás cae en la **denegación implícita**.

![Políticas del FortiGate](docs/img/05-fortigate-politicas.png)

## 10. Web Server: Sistema de Caja (Alpine)

**Edit config** del nodo:

```
auto eth0
iface eth0 inet static
  address 10.7.64.10
  netmask 255.255.255.248
  gateway 10.7.64.9
```

Para instalar paquetes, agrega un segundo adaptador conectado a un nodo NAT y ejecuta `udhcpc -i eth1`. Al terminar, quita ese enlace.

```
apk add nginx openssh openssl
ssh-keygen -A
passwd
sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd
mkdir -p /etc/nginx/ssl /run/nginx
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/caja.key -out /etc/nginx/ssl/caja.crt \
  -subj "/CN=caja.lab.local"
cat > /etc/nginx/http.d/default.conf <<'EOF'
server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/caja.crt;
    ssl_certificate_key /etc/nginx/ssl/caja.key;
    root /var/www/localhost/htdocs;
    index index.html;
}
EOF
echo "<h1>Sistema de Caja</h1>" > /var/www/localhost/htdocs/index.html
nginx
```

**Listener RDP de demostración** (Alpine no tiene un RDP real; esto solo demuestra que el puerto 3389 está permitido por la política):

```
nc -lk -p 3389 &
```

> Para un RDP real habría que instalar `xrdp` con un entorno gráfico, que es pesado.

## 11. Jump Server (Windows Server + RemoteApp)

**Red:** IP `10.7.64.2/255.255.255.248`, gateway `10.7.64.1`, nombre `JUMP`. Se deja un adaptador NAT adicional para descargar PuTTY, el navegador y los módulos de PowerShell.

**Dominio:** el asistente de Servicios de Escritorio Remoto normalmente exige dominio. Para el laboratorio, instala **AD DS** en la misma VM y promuévela a controlador de dominio (`lab.local`).

**Usuarios:** `usuario.web` (sin privilegios) y `usuario.admin` (con privilegios).

**RDS:** Server Manager > Add Roles > Remote Desktop Services > *Quick Start* de escritorio basado en sesión. Agrega también el rol **RD Gateway**, porque el Web Client lo necesita.

**Certificado:** Configure deployment > Certificates, crea uno autofirmado con nombre `jump.lab.local`, aplícalo a todos los roles y expórtalo como `.cer`.

**Web Client** (PowerShell como administrador):

```powershell
Install-Module -Name RDWebClientManagement
Install-RDWebClientPackage
Import-RDWebClientBrokerCert .\jump.cer
Publish-RDWebClientPackage -Type Production -Latest
```

**Aplicaciones RemoteApp** (Edit properties > User Assignment):

| Aplicación | Usuarios con acceso |
|---|---|
| Web (navegador con parámetro `https://10.7.64.10`) | usuario.web y usuario.admin |
| PuTTY | solo usuario.admin |
| Conexión a Escritorio Remoto (mstsc) | solo usuario.admin |


**Acceso desde los usuarios:** en cada webterm agrega `10.7.64.2 jump.lab.local` a `/etc/hosts` y abre `https://jump.lab.local/RDWeb/webclient/index.html`. Acepta la excepción del certificado.

## 12. Conexión de la VM a GNS3 (VMnet2)

La VM necesita una red **VMnet2** dedicada para unirse a la topología a través del nodo Cloud.

### En VMware

1. Abre el **Virtual Network Editor** como administrador (*Cambiar configuración*).
2. Agrega o selecciona **VMnet2** con tipo **Solo host**.
3. Desmarca *Usar el servicio DHCP local*.
4. Marca *Conectar un adaptador virtual de host a esta red*.
5. En la VM, asigna un adaptador de red a VMnet2 con *Conectado* y *Conectar al encender*.

### En el host

Verifica en `ncpa.cpl` que existe **VMware Network Adapter VMnet2** y está habilitado. No uses las tarjetas físicas (Ethernet/Wi-Fi) en el Cloud para no mezclar el laboratorio con tu red real.

### En GNS3

Cloud > Configure > Ethernet interfaces, con *Show special Ethernet interfaces* marcado: agrega **VMware Network Adapter VMnet2**, elimina cualquier otra (por ejemplo VMnet1), Apply y OK. Confirma que el cable de **FortiGate port2** quede en la interfaz VMnet2.


### En Windows (PowerShell)

Identifica el adaptador nuevo con `Get-NetAdapter` (por ejemplo `Ethernet2`) y pasa la IP:

```powershell
Get-NetAdapter
Remove-NetIPAddress -InterfaceAlias Ethernet0 -Confirm:$false
Remove-NetRoute -InterfaceAlias Ethernet0 -DestinationPrefix 0.0.0.0/0 -Confirm:$false
Disable-NetAdapter -Name Ethernet0 -Confirm:$false
New-NetIPAddress -InterfaceAlias Ethernet2 -IPAddress 10.7.64.2 -PrefixLength 29 -DefaultGateway 10.7.64.1
Set-NetIPInterface -InterfaceAlias Ethernet2 -InterfaceMetric 50
```

> Cambia los nombres de interfaz según lo que muestre `Get-NetAdapter`. Si estás conectado por escritorio remoto al adaptador que vas a deshabilitar, reconéctate por la IP del adaptador NAT. El adaptador NAT se mantiene activo para descargar módulos.

Prueba la conectividad:

```powershell
ipconfig
ping 10.7.64.1
Test-NetConnection 10.7.64.1 -Port 443
```

En el FortiGate: `execute ping 10.7.64.2`. Debe responder sin pérdidas y el puerto 443 debe dar `TcpTestSucceeded : True`.

## 13. Pruebas de validación

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | `show crypto isakmp sa` y `show crypto ipsec sa` en R2; Dashboard > Network > IPsec en FortiGate | Túnel activo |
| 2 | Iniciar sesión en RD Web con cada usuario | `usuario.web` ve solo Web; `usuario.admin` ve Web, PuTTY y RDP |
| 3 | Usuario sin privilegios intenta `ssh` al Jump y al Web Server | Falla; el deny aparece en el log de la política 1 |
| 4 | Desde un usuario: `ping 10.7.64.10` y `ping 10.7.64.2` | El primero falla; el segundo responde |
| 5 | Desde el Jump Server: PuTTY por SSH al Web Server, navegador a `https://10.7.64.10`, puerto 3389 | Todo funciona |
| 6 | Desde el Jump Server: `http://10.7.64.10` (puerto 80) y ping al Web Server | Fallan (denegación implícita) |


## 14. Problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| VMnet2 no aparece en la lista del Cloud | La red no se creó con adaptador de host | Revisa el Virtual Network Editor; cierra y reabre las propiedades del Cloud o GNS3 |
| La reserva DHCP no asigna la IP esperada | Identificador de cliente mal formado | Revisa la MAC con `ip link show eth0` y usa `show ip dhcp binding` |
| La GUI del FortiGate no abre por HTTPS | Problema de certificado | Usa `http://10.7.64.1` |
| El túnel no levanta | Parámetros distintos entre ambos lados | Compara fase 1/2, PSK y selectores (deben ser espejo de la ACL) |
| El Web Client de RemoteApp falla | Falta RD Gateway o el certificado no coincide | Instala el rol y reimporta el `.cer` con `jump.lab.local` |

---

## Video

[Laboratorio académico (ITLA). Las credenciales y claves mostradas son solo de laboratorio; cámbialas en cualquier entorno real.
](https://youtu.be/Iiyzxy719LI)
