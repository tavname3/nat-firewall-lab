# Firewall y NAT con nftables

## Topología de red

```
Internet
   │
[Router doméstico]
   │
   │  192.168.100.0/24
   │
[enp63s0 - WAN - 192.168.100.250]
        SERVIDOR FIREWALL
[enp32s0 - LAN - 100.50.25.1]
   │
   │  100.50.25.0/24
   │
[Clientes del laboratorio]
```

| Interfaz  | Rol  | Red               | IP del servidor   |
|-----------|------|-------------------|-------------------|
| enp63s0   | WAN  | 192.168.100.0/24  | 192.168.100.250   |
| enp32s0   | LAN  | 100.50.25.0/24    | 100.50.25.1       |

---

## Estructura de tablas

### `table inet filter`

Aplica a tráfico IPv4 e IPv6. Contiene tres cadenas:

#### `chain input` — política: DROP

Controla el tráfico destinado al propio servidor.

| Regla | Descripción |
|-------|-------------|
| `iif "lo" accept` | Permite tráfico de loopback |
| `ct state established,related accept` | Permite conexiones ya establecidas |
| `tcp dport { 22, 2222 } accept` | SSH estándar y alternativo |
| `icmp/icmpv6 echo-request accept` | Permite ping IPv4 e IPv6 |
| `iifname "enp63s0" ip saddr 192.168.100.0/24 udp/tcp dport 53 accept` | DNS desde WAN |
| `iifname "enp32s0" ip saddr 100.50.25.0/24 udp/tcp dport 53 accept` | DNS desde LAN |
| `tcp dport { 80, 443 } accept` | HTTP/HTTPS para página de advertencia nginx |

#### `chain forward` — política: DROP

Controla el tráfico que atraviesa el servidor de una red a otra.

| Regla | Descripción |
|-------|-------------|
| `iif "enp63s0" oif "enp32s0" accept` | Reenvío WAN → LAN |
| `iif "enp32s0" oif "enp63s0" accept` | Reenvío LAN → WAN |
| `ct state established,related accept` | Permite retorno de conexiones establecidas |

#### `chain output` — política: ACCEPT

Todo el tráfico generado por el servidor sale sin restricciones.

---

### `table ip nat`

Solo IPv4. Maneja la traducción de direcciones.

#### `chain prerouting` — DNAT

```
iif "enp63s0" tcp dport 2222 dnat to 100.50.25.15:22
```

Redirige el puerto 2222 entrante por WAN al host `100.50.25.15` en el puerto 22.
> **Nota:** Regla experimental para pruebas de port forwarding desde el router doméstico. No operativa en producción ya que el router no reenvía el puerto 2222 hacia este servidor.

#### `chain postrouting` — Masquerade

```
oif "enp63s0" masquerade
```

Todo el tráfico que sale por la interfaz WAN (`enp63s0`) recibe NAT de salida. Esto permite que los clientes de la LAN (`100.50.25.0/24`) naveguen usando la IP del servidor como gateway.

---

## Habilitar IP forwarding

Para que el reenvío entre interfaces funcione, se debe activar en el sistema:

```bash
# Temporal
sysctl -w net.ipv4.ip_forward=1

# Permanente
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

---

## Aplicar y verificar reglas

```bash
# Aplicar configuración
nft -f /etc/nftables.conf

# Ver reglas activas
nft list ruleset

# Habilitar al inicio
systemctl enable nftables
systemctl start nftables
```

---

## Depuración con nftrace

La config incluye `meta nftrace set 1` para tracing. Para usarlo:

```bash
nft monitor trace
```

Útil para diagnosticar si un paquete está siendo aceptado o descartado y por qué regla.
