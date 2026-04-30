# Informe-LAB3.2-Infraestructura de Red de una Organización con VLANs
**Universidad San Francisco Xavier de Chuquisaca**

**Estudiantes:**
              Miranda Coro Angela Micaela
              Calizaya Chambi Keyda Samira
              
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)

**Docente:** Ing. Marcelo Quispe Ortega

##**Introducción:**

El presente informe documenta la implementación de una infraestructura de red empresarial basada en VLANs (Virtual Local Area Networks) utilizando máquinas virtuales en VirtualBox. A través de este laboratorio se configuró un router con Ubuntu Server 24.04 como núcleo de la red, encargado del enrutamiento inter-VLAN, el acceso a internet mediante NAT y el control de tráfico con UFW e iptables.
El objetivo principal fue demostrar cómo la segmentación de red mediante VLANs permite aplicar políticas de seguridad granulares entre departamentos, restringiendo o permitiendo el tráfico según los requisitos organizacionales, y verificar su correcto funcionamiento mediante pruebas de conectividad entre todas las VMs.


## Objetivo del Laboratorio
El objetivo de este laboratorio es que los estudiantes sean capaces de:
*   **Diseñar e implementar** una arquitectura de red empresarial con VLAN para segmentar los departamentos de una organización.
*   **Configurar un enrutador con Linux** para que gestione el enrutamiento entre VLAN, el acceso a Internet y las políticas de seguridad.
*   **Aplicar reglas de firewall (UFW)** para controlar el flujo de tráfico entre las diferentes VLAN y la red externa.
*   **Comprender y configurar** interfaces *trunk* y etiquetado de VLAN en máquinas virtuales.
*   **Demostrar el funcionamiento** de las políticas de acceso establecidas entre los diferentes departamentos.

## SECCION 1: Preparación del Envío Virtual

### Esquema de la Infraestructura
Se configuró la infraestructura utilizando **VirtualBox**, asegurando que la "Red Interna" permita el paso de tramas etiquetadas (Trunk).

| Máquina Virtual | Departamento | ID de VLAN | Subred | IP Gateway (Router) |
| :--- | :--- | :---: | :--- | :--- |
| **Router** | -- | -- | -- | 192.168.X.1 |
| **Server-DMZ1** | DMZ | 10 | 192.168.10.0/29 | 192.168.10.1 |
| **Server-DMZ2** | DMZ | 10 | 192.168.10.0/29 | 192.168.10.1 |
| **PC-TI** | TI | 20 | 192.168.20.0/29 | 192.168.20.1 |
| **PC-Ventas** | Ventas | 30 | 192.168.30.0/27 | 192.168.30.1 |
| **PC-Contabilidad**| Contabilidad | 40 | 192.168.40.0/29 | 192.168.40.1 |

IMAGEN EN PACKET TRACER

# Sección 2:

## Paso 1: Configuración de VLAN y enrutador

### En la VM Router (Ubuntu)

#### Instalar herramientas VLAN
```bash
sudo apt install vlan
```

#### Cargar módulo del kernel
```bash
sudo modprobe 8021q
```

#### Configuración de interfaces VLAN

Editar archivo:
```bash
nano /etc/netplan/50-cloud-init.yaml
```

Contenido:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      optional: true
  vlans:
    vlan10:
      link: enp0s8
      id: 10
      addresses:
        - 192.168.10.1/29
      nameservers:
        addresses: [8.8.8.8]
    vlan20:
      link: enp0s8
      id: 20
      addresses:
        - 192.168.20.1/29
      nameservers:
        addresses: [8.8.8.8]
    vlan30:
      link: enp0s8
      id: 30
      addresses:
        - 192.168.30.1/27
      nameservers:
        addresses: [8.8.8.8]
    vlan40:
      link: enp0s8
      id: 40
      addresses:
        - 192.168.40.1/29
      nameservers:
        addresses: [8.8.8.8]
```

![imagen alt](https://github.com/MicaMirandaC/Informe-LAB3.1-SIS313/blob/ebe901e699c2c3ffc9dbbdb2d0972a86668022d5/img/pas1.png)
En la imegen se puede observar que con el comando respectivo se realizo la configuracion y ademas en la imagen inferior se puede ver como se verifico dicha configuracion:
![imagen alt]()
## Enrutamiento y NAT

#### Habilitar reenvío de paquetes
```bash
sudo nano /etc/sysctl.conf
```

Asegurar:
```bash
net.ipv4.ip_forward=1
```

Aplicar cambios:
```bash
sudo sysctl -p
```

---

## Paso 2: Configuración VM Contabilidad (Alpine)

#### Instalar VLAN
```bash
apk add vlan
```

#### Configuración de red
```bash
nano /etc/network/interfaces
```

```bash
auto lo
iface lo inet loopback

auto eth0.40
iface eth0.40 inet static
    address 192.168.40.2
    netmask 255.255.255.248
    gateway 192.168.40.1
    vlan-id 40

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down
```

#### Pruebas
```bash
ping 192.168.40.1
ping google.com
```

---

## Paso 3: Configuración VM Ventas (Alpine)

#### Instalar VLAN
```bash
apk add vlan
```

#### Configuración de red
```bash
nano /etc/network/interfaces
```

```bash
auto lo
iface lo inet loopback

auto eth0.30
iface eth0.30 inet static
    address 192.168.30.2
    netmask 255.255.255.224
    gateway 192.168.30.1
    vlan-id 30

auto eth0
iface eth0 inet manual
    up ip link set $IFACE up
    down ip link set $IFACE down
```

#### Pruebas
```bash
ping 192.168.30.1
```

---

## Paso 4: Configuración de UFW en Router

#### Instalar y habilitar UFW
```bash
sudo apt install ufw
sudo ufw allow ssh
sudo ufw enable
```

---

## Reglas de tráfico entre VLAN

### Permitir TI (VLAN 20) a todos
```bash
sudo ufw route allow in on vlan20 out on vlan10
sudo ufw route allow in on vlan20 out on vlan30
sudo ufw route allow in on vlan20 out on vlan40
```

### Permitir Ventas (VLAN 30) a DMZ
```bash
sudo ufw route allow in on vlan30 out on vlan10
```

### Permitir Contabilidad (VLAN 40)
```bash
sudo ufw route allow in on vlan40 out on vlan10
sudo ufw route allow in on vlan40 out on vlan30
```

### Denegar acceso de DMZ (VLAN 10)
```bash
sudo ufw route deny in on vlan10 out on vlan20
sudo ufw route deny in on vlan10 out on vlan30
sudo ufw route deny in on vlan10 out on vlan40
```

### Denegar acceso de Ventas (VLAN 30)
```bash
sudo ufw route deny in on vlan30 out on vlan20
sudo ufw route deny in on vlan30 out on vlan40
```

### Denegar acceso de Contabilidad (VLAN 40)
```bash
sudo ufw route deny in on vlan40 out on vlan20
```

---

## Configuración acceso a Internet (NAT)

Editar:
```bash
sudo nano /etc/ufw/before.rules
```

Agregar al inicio:
```bash
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.20.0/24 -o enp0s3 -j MASQUERADE
-A POSTROUTING -s 192.168.40.0/24 -o enp0s3 -j MASQUERADE
COMMIT
```

---

## Pruebas finales

Desde VM Ventas:
```bash
ssh usuario@192.168.40.2
```

Verificar:
- ❌ Acceso a Contabilidad (bloqueado)
- ✅ Acceso a DMZ (permitido)

**##Conclusion:**
