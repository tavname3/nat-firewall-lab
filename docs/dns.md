# DNS — BIND9 + Bloqueo de Phishing

## Descripción general

El servidor DNS cumple tres funciones principales dentro de la red del laboratorio:

1. **DNS autoritativo interno** — resuelve nombres del dominio `empresa.local` para los clientes de la LAN
2. **DNS recursivo con filtrado** — reenvía consultas externas a través de Pi-hole antes de llegar a internet
3. **Bloqueo de dominios** — intercepta dominios bloqueados (redes sociales, phishing) y redirige a una página de advertencia

---

## Topología del flujo DNS

```
Clientes (192.168.100.x / 100.50.25.x)
            │
            ▼
      BIND9 :53
   192.168.100.250
      │         │
      │         └── Zona interna → responde directamente (empresa.local)
      │         └── Dominio bloqueado → redirige a 192.168.100.250 (Nginx)
      │
      │
      ▼
  Internet
  9.9.9.9 / 1.1.1.1
```

---

## Instalación

```bash
sudo apt install bind9 bind9utils -y
```

---

## Archivos de configuración

### `/etc/bind/named.conf.options`

Configuración principal de BIND9: recursión, forwarders, logging y seguridad.

```bash
options {
    directory "/var/cache/bind";

    recursion yes;

    # Permitir consultas solo desde localhost y redes internas
    allow-recursion {
        127.0.0.1;
        192.168.100.0/24;
        100.50.25.0/24;
    };
    allow-query {
        127.0.0.1;
        192.168.100.0/24;
        100.50.25.0/24;
    };

    # Forwarders externos para resolución de dominios públicos
    forwarders {
        9.9.9.9;    # Quad9 (filtrado de malware)
        1.1.1.1;    # Cloudflare
    };

    dnssec-validation no;

    # Escuchar en todas las interfaces IPv4
    listen-on { any; };
    listen-on-v6 { none; };

    # Deshabilitar transferencias de zona (previene ataques AXFR)
    allow-transfer { none; };
};

# Logging de consultas con rotación automática
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

> **Nota sobre forwarders:** Se usan `9.9.9.9` (Quad9) y `1.1.1.1` (Cloudflare) como resolvers externos. Quad9 incluye filtrado propio de dominios maliciosos conocidos, lo que añade una capa extra de seguridad para dominios que no estén en las zonas de bloqueo locales.

---

### `/etc/bind/named.conf.local`

Define todas las zonas autoritativas del servidor.

```bash
# Zona interna principal
zone "empresa.local" {
    type master;
    file "/etc/bind/db.empresa.local";
};

# Zona reversa
zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

# Dominios bloqueados — redes sociales
zone "facebook.com" {
    type master;
    file "/etc/bind/db.blocked";
};
zone "instagram.com" {
    type master;
    file "/etc/bind/db.blocked";
};
zone "tiktok.com" {
    type master;
    file "/etc/bind/db.blocked";
};

# Dominios bloqueados — phishing simulado
zone "fakebank-login.com" {
    type master;
    file "/etc/bind/db.blocked";
};
zone "phishing-alert-test.com" {
    type master;
    file "/etc/bind/db.blocked";
};
```

---

## Archivos de zona

### `/etc/bind/db.empresa.local` — Zona forward interna

Resuelve nombres del dominio `empresa.local` a IPs de la red interna.

```bash
$TTL 604800
@   IN  SOA ns1.empresa.local. admin.empresa.local. (
            2026042901  ; Serial (formato YYYYMMDDnn)
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400 )     ; Negative Cache TTL

@       IN  NS      ns1.empresa.local.

; Registros A
ns1     IN  A       192.168.100.250
server  IN  A       192.168.100.250
files   IN  A       192.168.100.250
dns     IN  A       192.168.100.250
```

---

### `/etc/bind/db.192.168.100` — Zona reversa

Resuelve IPs de la red `192.168.100.0/24` a nombres (registros PTR).

```bash
$TTL 86400
@   IN  SOA ns1.empresa.local. admin.empresa.local. (
            2026042901
            3600
            1800
            604800
            86400 )

@       IN  NS      ns1.empresa.local.

; Registros PTR (solo el último octeto)
250     IN  PTR     server.empresa.local.
```

---

### `/etc/bind/db.blocked` — Zona de bloqueo

Todos los dominios bloqueados apuntan a este archivo. Redirige cualquier consulta a la IP del servidor, donde Nginx sirve una página de advertencia.

```bash
$TTL 86400
@   IN  SOA ns1.empresa.local. admin.empresa.local. (
        2026042901
        3600
        1800
        604800
        86400 )
    IN  NS  ns1.empresa.local.
ns1 IN  A   192.168.100.250

; Redirigir dominio raíz y todos sus subdominios al servidor web interno
@   IN  A   192.168.100.250
*   IN  A   192.168.100.250
```

> **¿Por qué `192.168.100.250` y no `127.0.0.1`?**
> Redirigir a `127.0.0.1` produce un error de conexión silencioso en el cliente. Al redirigir a la IP del servidor, Nginx puede servir una página de advertencia clara que informa al usuario que el dominio está bloqueado por política de seguridad.

---

## Página de advertencia con Nginx

Cuando un dominio bloqueado es consultado, el DNS resuelve a `192.168.100.250` y Nginx sirve la página de advertencia.

### Abrir el puerto 80 en nftables

```bash
sudo nft add rule inet filter input tcp dport 80 accept
sudo nft add rule inet filter input tcp dport 443 accept
sudo nft list ruleset | sudo tee /etc/nftables.conf
```

### Comportamiento según el protocolo

| Protocolo | Comportamiento |
|-----------|---------------|
| HTTP | Muestra la página de advertencia de Nginx |
| HTTPS | Firefox muestra advertencia de certificado inválido (HSTS) |

> **Nota sobre HSTS:** Sitios como Facebook, Instagram o Google tienen activado HTTP Strict Transport Security (HSTS), una política que obliga al navegador a usar HTTPS con certificado válido. Al redirigir a nuestro servidor, el navegador detecta que el certificado no corresponde al dominio y muestra una advertencia propia — lo cual es igualmente efectivo para el bloqueo, ya que el usuario no puede ignorarla fácilmente.

---

## Reglas de firewall para DNS

El servidor tiene dos interfaces de red y el tráfico DNS puede llegar por cualquiera de ellas según el origen del cliente.

| Interfaz | Red | Clientes |
|----------|-----|----------|
| `enp32s0` | `100.50.25.0/24` | MainPCLab y sus VMs |
| `enp63s0` | `192.168.100.0/24` | LapLab y otros clientes en la LAN del router |

```bash
# DNS desde la red LAN interna del servidor (enp32s0)
sudo nft add rule inet filter input \
  iifname "enp32s0" ip saddr 100.50.25.0/24 udp dport 53 accept
sudo nft add rule inet filter input \
  iifname "enp32s0" ip saddr 100.50.25.0/24 tcp dport 53 accept

# DNS desde la red del router doméstico (enp63s0)
sudo nft add rule inet filter input \
  iifname "enp63s0" ip saddr 192.168.100.0/24 udp dport 53 accept
sudo nft add rule inet filter input \
  iifname "enp63s0" ip saddr 192.168.100.0/24 tcp dport 53 accept
```

> **Lección aprendida:** Durante la implementación se detectó que el LapLab (`192.168.100.88`) y el servidor comparten la misma red física (`192.168.100.0/24`) conectada al router doméstico. Aunque lógicamente parecía tráfico LAN interno, los paquetes llegaban por `enp63s0` en lugar de `enp32s0`. El diagnóstico se realizó con `sudo nft monitor trace`, que mostró la interfaz real de entrada de cada paquete.

---

## Verificación del funcionamiento

### Desde el servidor

```bash
# Verificar que BIND9 escucha en las IPs correctas
sudo ss -tulnp | grep :53 | grep named

# Probar resolución interna
dig @127.0.0.1 server.empresa.local
dig @192.168.100.250 server.empresa.local

# Verificar sintaxis de zonas
sudo named-checkconf
sudo named-checkzone empresa.local /etc/bind/db.empresa.local
sudo named-checkzone 100.168.192.in-addr.arpa /etc/bind/db.192.168.100
```

### Desde los clientes

```bash
# Resolución de zona interna
dig @192.168.100.250 server.empresa.local
# Esperado: 192.168.100.250

# Resolución reversa
dig @192.168.100.250 -x 192.168.100.250
# Esperado: server.empresa.local

# Verificar bloqueo de dominio
dig @192.168.100.250 facebook.com
# Esperado: 192.168.100.250 (redirigido a página de advertencia)

# Verificar resolución externa (a través de Pi-hole)
dig @192.168.100.250 google.com
# Esperado: IP real de Google
```

---

## Script para agregar dominios bloqueados

Para facilitar el bloqueo de nuevos dominios sin editar manualmente `named.conf.local`:

```bash
sudo nano /usr/local/bin/block-domain.sh
```

```bash
#!/bin/bash
# Uso: sudo block-domain.sh dominio-malicioso.com

DOMAIN=$1
CONF="/etc/bind/named.conf.local"

if [ -z "$DOMAIN" ]; then
    echo "Uso: $0 <dominio>"
    exit 1
fi

if grep -q "\"$DOMAIN\"" $CONF; then
    echo " !!! $DOMAIN ya está bloqueado  !!!"
    exit 0
fi

cat >> $CONF << EOF

zone "$DOMAIN" {
    type master;
    file "/etc/bind/db.blocked";
};
EOF

named-checkconf && systemctl reload bind9
echo " $DOMAIN bloqueado exitosamente +"
```

```bash
sudo chmod +x /usr/local/bin/block-domain.sh

# Uso:
sudo block-domain.sh malicious-phishing.com
```

---

## Decisiones de diseño

- **BIND9 como punto de entrada único en :53** — centraliza el control de resolución de toda la red.
- **Forwarders externos (Quad9 + Cloudflare)** — resolución de dominios públicos con filtrado adicional de malware a nivel del resolver externo.
- **Zonas de bloqueo propias en BIND9** — permiten bloquear dominios específicos con control total y sin latencia adicional, respondiendo directamente desde el servidor local.
- **Redirección a Nginx en lugar de `127.0.0.1`** — mejora la experiencia del usuario al mostrar una página explicativa en lugar de un error de conexión genérico.
- **Reglas de firewall por interfaz** — el filtrado DNS se aplica diferenciado según la interfaz de entrada, siguiendo el principio de mínimo privilegio por segmento de red.
