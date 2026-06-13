# nat-firewall-lab

Laboratorio de red con firewall, NAT, DNS autoritativo con bloqueo de dominios y DHCP, implementado sobre Linux con nftables, BIND9 e ISC DHCP.

---

## Topología

```
                        Internet
                           │
                    [Router doméstico]
                           │
              192.168.100.0/24 (WAN)
                           │
              ┌────────────┴────────────┐
              │  enp63s0 — 192.168.100.250  │
              │       SERVIDOR              │
              │  enp32s0 — 100.50.25.1      │
              └────────────┬────────────┘
                           │
               100.50.25.0/24 (LAN)
                           │
              ┌────────────┴────────────┐
              │     Clientes del lab    │
              │   100.50.25.10 – .30    │
              └─────────────────────────┘
```

| Interfaz | Rol | Red | IP |
|----------|-----|-----|----|
| enp63s0 | WAN | 192.168.100.0/24 | 192.168.100.250 |
| enp32s0 | LAN | 100.50.25.0/24 | 100.50.25.1 |

---

## Componentes

| Componente | Tecnología | Descripción |
|------------|------------|-------------|
| Firewall + NAT | nftables | Filtrado de paquetes, forwarding y masquerade |
| DNS | BIND9 | Resolución interna y bloqueo de dominios |
| DHCP | ISC DHCP | Asignación de IPs en la red LAN |
| Página de bloqueo | nginx + HTML/CSS | Advertencia visual para dominios bloqueados |

---

## Documentación

- [Firewall y NAT con nftables](docs/nftables.md)
- [DNS con BIND9](docs/dns.md)
- [DHCP con ISC DHCP Server](docs/dhcp.md)
- [Página de advertencia con nginx](docs/nginx.md)

---

## Estructura del repositorio

```
nat-firewall-lab/
├── README.md
├── docs/
│   ├── nftables.md
│   ├── dns.md
│   ├── dhcp.md
│   └── nginx.md
└── configs/
    ├── nftables.conf
    ├── named.conf.options
    ├── named.conf.local
    ├── db.empresa.local
    ├── db.blocked
    └── dhcpd.conf
```

---

## Flujo de bloqueo DNS

```
Cliente → facebook.com
    └─► BIND9 (zona local) → resuelve a 192.168.100.250
            └─► nginx → sirve página de advertencia
```

---

## Requisitos

- Linux (Debian/Ubuntu)
- `nftables`
- `bind9`
- `isc-dhcp-server`
- `nginx`
- IP forwarding habilitado: `net.ipv4.ip_forward = 1`

## Autor
 
**Gustavo Isaias Nava Merino**  
TSU en Infraestructura de Redes Digitales  
Ingeniería en Redes Inteligentes y Ciberseguridad *(cursando)*  
Universidad Tecnológica de Puebla
