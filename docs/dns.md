# DNS con BIND9 — Resolución interna y bloqueo de dominios

## Rol del servidor DNS

El servidor actúa como **DNS recursivo autoritativo** para la red del laboratorio. Sus funciones principales son:

- Resolver nombres internos del dominio `empresa.local`
- Reenviar consultas externas a DNS públicos (Quad9 y Cloudflare)
- **Bloquear dominios** de redes sociales, phishing y typosquatting bancario mediante zonas locales que redirigen a la IP del servidor

---

## Configuración global — `named.conf.options`

### Recursión y consultas permitidas

```
allow-recursion { 127.0.0.1; 192.168.100.0/24; 100.50.25.0/24; };
allow-query     { 127.0.0.1; 192.168.100.0/24; 100.50.25.0/24; };
```

Solo responde consultas desde el propio servidor y ambas redes del laboratorio. Evita que sea usado como DNS abierto.

### Forwarders

```
forwarders { 9.9.9.9; 1.1.1.1; };
```

Reenvía resolución de dominios externos a Quad9 (privacidad y filtrado de malware) y Cloudflare como respaldo.

### Escucha

```
listen-on { any; };
listen-on-v6 { none; };
```

Escucha en todas las interfaces IPv4. IPv6 deshabilitado intencionalmente en este lab.

### Logging de consultas

Las consultas DNS se registran en `/var/log/bind_queries.log` con rotación automática (3 versiones, máx 5 MB cada una).

```bash
# Ver consultas en tiempo real
tail -f /var/log/bind_queries.log
```

---

## Zonas configuradas — `named.conf.local`

### Zona interna: `empresa.local`

Dominio privado del laboratorio. El servidor es autoritativo para esta zona.

| Hostname | IP | Descripción |
|----------|----|-------------|
| ns1.empresa.local | 192.168.100.250 | Servidor DNS / firewall |
| server.empresa.local | 192.168.100.250 | Alias del servidor principal |
| files.empresa.local | 192.168.100.250 | Servicio de archivos (alias) |
| dns.empresa.local | 192.168.100.250 | Alias DNS |

### Zona inversa: `192.168.100.0/24`

Resolución inversa (PTR) para la red WAN del servidor.

```
zone "100.168.192.in-addr.arpa" { type master; file "/etc/bind/db.192.168.100"; };
```

---

## Bloqueo de dominios

### Mecanismo

Se crean zonas autoritativas locales para cada dominio bloqueado. Cuando un cliente consulta `facebook.com`, BIND responde con la IP del servidor (`192.168.100.250`) en lugar del destino real. El servidor nginx en esa IP muestra la **página de advertencia**.

El archivo `db.blocked` es compartido por todos los dominios bloqueados:

```dns
@   IN  A   192.168.100.250
*   IN  A   192.168.100.250
```

El registro wildcard (`*`) asegura que todos los subdominios también sean bloqueados.

### Dominios bloqueados

| Categoría | Dominios |
|-----------|----------|
| Redes sociales | `facebook.com`, `instagram.com`, `tiktok.com` |
| Phishing genérico | `fakebank-login.com`, `phishing-alert-test.com`, `malicious-phishing.com` |
| Phishing / servicios | `paypal-security-check.com`, `secure-login-verify.com`, `account-suspended-alert.com` |
| Typosquatting bancario MX | `banamex-login.com`, `bbva-secure.com`, `santander-verify.com` |

---

## Verificación

```bash
# Consultar dominio interno
dig @192.168.100.250 ns1.empresa.local

# Verificar que un dominio bloqueado resuelve al servidor
dig @192.168.100.250 facebook.com
# Debe devolver 192.168.100.250

# Verificar subdominio bloqueado (wildcard)
dig @192.168.100.250 m.facebook.com

# Estado del servicio
systemctl status named
```
