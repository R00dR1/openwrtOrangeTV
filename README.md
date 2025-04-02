# Configuración OpenWRT para Orange TV en Jazztel

![OpenWRT Logo](https://openwrt.org/lib/tpl/openwrt/images/logo.png)

Este repositorio contiene la configuración completa para ver Orange TV en una red Jazztel mediante un router OpenWRT, incluyendo VLANs, IGMP proxy y asignación DHCP fija para el decodificador.

## 📚 Créditos y referencias
- [RouterOSv7-Scripts de noemiabril](https://github.com/noemiabril/RouterOSv7-Scripts/tree/main)
- [Hilo en Bandaancha.eu](https://bandaancha.eu/foros/pfsense-tv-telefono-1746023)

## 📋 Tabla de contenidos
1. [Estructura de archivos](#-estructura-de-archivos-clave)
2. [Requisitos previos](#-requisitos-previos)
3. [Configuración paso a paso](#-configuración-paso-a-paso)
4. [Solución de problemas](#-solución-de-problemas)
5. [Notas importantes](#-notas-importantes)

## 📁 Estructura de archivos clave

### 1. `/etc/config/network`
```bash
# Configuración básica de interfaces
config interface 'lanTV'
    option proto 'static'
    option device 'br-TV'
    option ipaddr '192.168.40.1'
    option netmask '255.255.255.248'

# Configuración del bridge para TV
config device
    option type 'bridge'
    option name 'br-TV'
    list ports 'eth3'
    list ports 'eth4'       # Puerto para el decodificador
    list ports 'eth5.838'   # VLAN IPTV

# Configuración VLAN IPTV
config device
    option type '8021q'
    option ifname 'eth5'
    option vid '838'
    option name 'eth5.838'
```

### 2. `/etc/config/dhcp` (Configuración importante)
```bash
# Configuración DHCP para la red TV
config dhcp 'lanTV'
    option interface 'lanTV'
    option start '3'
    option limit '4'
    option leasetime '12h'
    list dhcp_option '6,87.216.1.65,87.216.1.66'  # DNS Jazztel

# Asignación fija para el decodificador
config host
    list mac 'XX:XX:XX:XX:XX:XX'  # ¡Reemplazar con MAC real!
    option ip '192.168.40.2'
    option name 'deco-iptv'
    option leasetime '1h'
```

### 3. `/etc/config/igmpproxy`
```bash
# Configuración básica IGMP
config igmpproxy
    option quickleave 1

# Interfaz downstream (LAN TV)
config phyint
    option network lanTV
    option zone lanTV
    option direction downstream
    list altnet 192.168.40.0/30

# Interfaz upstream (IPTV)
config phyint
    option network IPTV
    option zone IPTV
    option direction upstream
    list altnet 192.168.40.0/30
```

## 🔧 Requisitos previos
Instalar paquetes necesarios:
```bash
opkg update
opkg install igmpproxy
```

## 🛠 Configuración paso a paso
1. Configuración inicial
    Conectar el cable de Jazztel al puerto WAN (eth5)

    Conectar el decodificador al puerto eth4

2. Personalización obligatoria
    1. Editar `/etc/config/dhcp`:

    Reemplazar `XX:XX:XX:XX:XX:XX` con la MAC real de tu decodificador

    Verificar que la IP `192.168.40.2` esté disponible

    2. Verificar VLANs:
    ```bash
    cat /etc/config/network | grep "option vid"
    ```

3. Reiniciar servicios
    ```bash
    /etc/init.d/network restart
    /etc/init.d/dnsmasq restart
    /etc/init.d/igmpproxy restart
    ```

## 🔍 Solución de problemas
1. El decodificador no obtiene IP
    ```bash
    logread | grep dnsmasq
    ifconfig br-TV
    ```

2. Problemas con multicast
    ```bash
    tcpdump -i br-TV igmp
    ```

3. Configuraciones alternativas
    1. Para usar DNS Orange:

        ```bash
        list dhcp_option '6,62.37.228.20,62.36.225.15'  # Descomentar para DNS Orange
        ```
    1. Para usar DNS Jazztel:

        ```bash
        list dhcp_option '6,87.216.1.65,87.216.1.66'  # Descomentar para DNS Jazztel
        ```

## 📌 Notas importantes
1. Puertos físicos:

    - eth4: Exclusivo para el decodificador
    - eth5: Puerto WAN con VLAN 838

2. Rangos IP:

    - Red TV: 192.168.40.0/29
    - Deco: 192.168.40.2 fija

3. Monitorización:

    ```bash
    watch -n 1 "iptables -t mangle -nvL; echo; netstat -g"
    ```

## ⚖️ Licencia
MIT License - Basado en trabajos de la comunidad mencionada en créditos