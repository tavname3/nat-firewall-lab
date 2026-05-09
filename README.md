# 🛡️ Servidor NAT / Firewall / DNS / DHCP en Ubuntu Server 24.04

## Descripción general

Implementación de un servidor Linux multifunción que actúa como **gateway seguro** entre una red interna privada e internet. El servidor gestiona traducción de direcciones (NAT), filtrado de tráfico (Firewall), resolución de nombres (DNS) y asignación dinámica de IPs (DHCP), todo en un servidor físico con Ubuntu Server 24.04 LTS.

---

## Topología de red

```
Internet
    │
[ HomeRouter / Modem ]
  192.168.100.1
    │
    │ (enp63s0) 192.168.100.250  ← WAN del servidor
[ Ubuntu Server ]  <── Este repositorio
  (enp32s0) 100.50.25.1         ← LAN del servidor
    │
  [ Hub ]
    │
[ MainPCLab ]
  100.50.25.15
```

### Interfaces de red

| Interfaz | Rol | Dirección IP |
|---|---|---|
| `enp32s0` | LAN (red interna) | `100.50.25.1` |
| `enp63s0` | WAN (hacia router doméstico) | `192.168.100.250` |

---

## Servicios implementados

| Servicio | Herramienta | Función |
|---|---|---|
| NAT / Firewall | `nftables` | Traducción de direcciones y filtrado de tráfico |
| DNS | `bind9` | Resolución de nombres para la LAN interna |
| DHCP | `isc-dhcp-server` | Asignación dinámica de IPs a clientes de la LAN |

---

## Entorno

- **OS:** Ubuntu Server 24.04.4 LTS x86_64
- **Hardware:** Servidor físico (bare metal)
- **Firewall:** nftables (sucesor moderno de iptables)
- **DNS:** BIND9
- **DHCP:** ISC DHCP Server

---

## Configuración

### 1. Habilitar IP Forwarding

Para que el servidor pueda reenviar paquetes entre interfaces es necesario habilitar el forwarding en el kernel:

```bash
# Editar el archivo de configuración del kernel
sudo nano /etc/sysctl.conf
```

Descomentar o agregar la siguiente línea:

```
net.ipv4.ip_forward=1
```

Aplicar los cambios sin reiniciar:

```bash
sudo sysctl -p
```

Verificar que esté activo:

```bash
sysctl net.ipv4.ip_forward
# Salida esperada: net.ipv4.ip_forward = 1
```

---

### 2. nftables — NAT y Firewall

nftables es el framework moderno de filtrado de paquetes en Linux, sucesor de iptables. Trabaja con **tablas**, **cadenas** y **reglas**.

#### Conceptos clave

| Concepto | Descripción |
|---|---|
| **Table** | Contenedor de cadenas. Tipos: `filter`, `nat`, `mangle` |
| **Chain** | Conjunto de reglas. Hooks: `input`, `output`, `forward`, `prerouting`, `postrouting` |
| **Rule** | Condición + acción (`accept`, `drop`, `masquerade`, etc.) |

#### Archivo de configuración

```bash
sudo nano /etc/nftables.conf
```


```nft
#!/usr/sbin/nft -f

flush ruleset

# ─────────────────────────────────────────
# TABLA: FILTRADO DE TRÁFICO
# ─────────────────────────────────────────
table inet filter {

    chain input {
        type filter hook input priority 0; policy drop;

        # Permitir tráfico de loopback
        iif "lo" accept

        # Permitir conexiones ya establecidas y relacionadas
        ct state established,related accept

        # Permitir SSH (administración del servidor)
        tcp dport 22 accept

        # Permitir puerto 2222 (port forwarding hacia MainPCLab)
        tcp dport 2222 accept

        # Permitir ICMP IPv4 e IPv6
        icmp type echo-request accept
        icmpv6 type echo-request accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Tráfico LAN hacia Internet
        iif "enp32s0" oif "enp63s0" accept

        # Tráfico Internet hacia LAN
        iif "enp63s0" oif "enp32s0" accept

        # Conexiones ya establecidas
        ct state established,related accept
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

# ─────────────────────────────────────────
# TABLA: NAT
# ─────────────────────────────────────────
table ip nat {

    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Port forwarding: puerto 2222 → SSH de MainPCLab (100.50.25.15:22)
        iif "enp63s0" tcp dport 2222 dnat to 100.50.25.15:22
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Enmascarar tráfico saliente
        oif "enp63s0" masquerade
    }
}
```

> **Notas de implementación:**
> - El puerto `2222` en `input` es necesario para que el servidor acepte conexiones destinadas al port forwarding hacia `MainPCLab`
> - SSH está abierto en todas las interfaces; como mejora de hardening se puede restringir a la LAN con `iif "enp63s0" tcp dport 22 accept`
> - El `masquerade` sobre `enp63s0` funciona en este lab; en producción debería aplicarse sobre la interfaz WAN (`enp32s0`)


Aplicar la configuración:

```bash
sudo nft -f /etc/nftables.conf
```

Habilitar nftables al inicio del sistema:

```bash
sudo systemctl enable nftables
sudo systemctl start nftables
```

Verificar reglas activas:

```bash
sudo nft list ruleset
```

---

### 3. BIND9 — Servidor DNS

BIND9 actúa como servidor DNS recursivo y autoritativo para la red LAN interna.

#### Instalación

```bash
sudo apt install bind9 bind9utils -y
```

#### Configuración principal

```bash
sudo nano /etc/bind/named.conf.options
```

```
options {
    directory "/var/cache/bind";

    recursion yes;

    # Permitir recursión y consultas solo desde localhost y la red WAN del servidor
    allow-recursion {
        127.0.0.1;
        192.168.100.0/24;
    };
    allow-query {
        127.0.0.1;
        192.168.100.0/24;
    };

    # Forwarders: Quad9 (con filtrado de malware) y Cloudflare como respaldo
    forwarders {
        9.9.9.9;
        1.1.1.1;
    };

    dnssec-validation no;

    # Escuchar en todas las interfaces IPv4, deshabilitar IPv6
    listen-on { any; };
    listen-on-v6 { none; };

    # Deshabilitar transferencias de zona (previene ataques de enumeración DNS)
    allow-transfer { none; };
};

# Logging de consultas DNS con rotación automática
logging {
    channel query_log {
        file "/var/log/bind_queries.log" versions 3 size 5m;
        severity info;
        print-time yes;
    };
    category queries {
        query_log;
    };
};
```

> **Nota de seguridad:** Se usa `9.9.9.9` (Quad9) como forwarder principal porque filtra dominios maliciosos conocidos por defecto, añadiendo una capa extra de protección para los clientes de la LAN. `allow-transfer { none; }` previene ataques de transferencia de zona (AXFR).

Verificar la configuración:

```bash
sudo named-checkconf
```

Reiniciar y habilitar el servicio:

```bash
sudo systemctl enable --now bind9
```

Verificar que escucha en el puerto 53:

```bash
ss -tlnup | grep 53
```

---

### 4. ISC DHCP Server — Servidor DHCP

Asigna direcciones IP dinámicas a los clientes de la red LAN interna.

#### Instalación

```bash
sudo apt install isc-dhcp-server -y
```

#### Configurar interfaz de escucha

```bash
sudo nano /etc/default/isc-dhcp-server
```

```
INTERFACESv4="enp63s0"
```

#### Configuración del servidor DHCP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;

# Subred LAN interna
subnet 100.50.25.0 netmask 255.255.255.0 {
    range 100.50.25.10 100.50.25.30;
    option routers 100.50.25.1;
    option domain-name-servers 192.168.100.250, 8.8.8.8;
}
```

> **Nota:** El rango `.10` a `.30` limita a 21 hosts simultáneos, adecuado para un entorno de laboratorio. `domain-name-servers` apunta al servidor por su interfaz WAN (`192.168.100.250`) con `8.8.8.8` como respaldo público.

Habilitar y arrancar el servicio:

```bash
sudo systemctl enable --now isc-dhcp-server
```

Verificar leases activos:

```bash
cat /var/lib/dhcp/dhcpd.leases
```

---

## Verificación del funcionamiento

### Verificar NAT activo

```bash
# Ver reglas nftables activas
sudo nft list ruleset

# Verificar forwarding activo en el kernel
sysctl net.ipv4.ip_forward
```

### Verificar conectividad desde un cliente LAN

```bash
# Ping al gateway
ping 100.50.25.1

# Ping a internet (prueba de NAT)
ping 8.8.8.8

# Prueba de resolución DNS
nslookup google.com 100.50.25.1
```

### Monitorear tráfico en tiempo real

```bash
# Ver logs del firewall
sudo journalctl -f | grep nftables

# Ver conexiones activas
sudo nft list ruleset
ss -tulnp
```

---

## Decisiones de diseño y seguridad

- **nftables sobre iptables:** Se eligió nftables por ser el estándar moderno en Linux, con mejor rendimiento, sintaxis más limpia y soporte activo a largo plazo.
- **Política por defecto DROP:** Tanto en `input` como en `forward` la política base es denegar todo. Solo se permite explícitamente lo necesario (principio de mínimo privilegio).
- **SSH restringido a LAN:** El acceso SSH al servidor solo está permitido desde la red interna (`enp63s0`), nunca desde la WAN.
- **DNS con forwarders:** BIND9 reenvía consultas externas a servidores públicos (8.8.8.8, 1.1.1.1) en lugar de resolverlas directamente, reduciendo exposición y carga.
- **DHCP autoritativo:** El servidor es el único autorizadoa asignar IPs en la subred, evitando conflictos con servidores DHCP no autorizados (rogue DHCP).

---

## Posibles mejoras futuras

- [ ] Implementar `fail2ban` para protección contra fuerza bruta en SSH
- [ ] Agregar zonas DNS locales en BIND9 para resolución de nombres internos
- [ ] Implementar logging avanzado con rotación de logs
- [ ] Configurar VPN con WireGuard para acceso remoto seguro
- [ ] Migrar de `isc-dhcp-server` a `kea-dhcp` (sucesor oficial)

---

## Autor

**Gustavo Isaias Nava Merino**  
TSU en Infraestructura de Redes Digitales  
Ingeniería en Redes Inteligentes y Ciberseguridad (cursando)  
Universidad Tecnológica de Puebla
