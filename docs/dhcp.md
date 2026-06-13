# DHCP con ISC DHCP Server

## Rol en el laboratorio

El servidor ISC DHCP asigna direcciones IP dinámicas a los clientes de la red LAN (`100.50.25.0/24`), proporcionándoles también el gateway y los servidores DNS del laboratorio.

---

## Configuración de la subred LAN

```
subnet 100.50.25.0 netmask 255.255.255.0 {
    range 100.50.25.10 100.50.25.30;
    option routers 100.50.25.1;
    option domain-name-servers 192.168.100.250, 8.8.8.8;
}
```

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| Red | 100.50.25.0/24 | Red LAN del laboratorio |
| Rango dinámico | 100.50.25.10 – 100.50.25.30 | Pool de IPs para clientes |
| Gateway | 100.50.25.1 | IP LAN del servidor firewall (enp32s0) |
| DNS primario | 192.168.100.250 | Servidor BIND9 interno (enp63s0) |
| DNS secundario | 8.8.8.8 | Google DNS como respaldo |
| Lease por defecto | 600 s | 10 minutos |
| Lease máximo | 7200 s | 2 horas |

> **Nota:** El DNS primario apunta a `192.168.100.250` (interfaz WAN del servidor). Esto es intencional ya que es la IP donde escucha BIND9. Los clientes LAN pueden alcanzar esa IP gracias al forwarding configurado en nftables.

---

## IPs reservadas fuera del pool DHCP

| Rango | Uso |
|-------|-----|
| 100.50.25.1 | Gateway / servidor firewall |
| 100.50.25.2 – 100.50.25.9 | Reservado para servidores estáticos |
| 100.50.25.10 – 100.50.25.30 | Pool dinámico DHCP |
| 100.50.25.31 – 100.50.25.254 | Disponible para expansión |

---

## Gestión del servicio

```bash
# Iniciar / reiniciar
systemctl start isc-dhcp-server
systemctl restart isc-dhcp-server

# Ver leases activos
cat /var/lib/dhcp/dhcpd.leases

# Ver logs en tiempo real
journalctl -u isc-dhcp-server -f

# Verificar configuración antes de aplicar
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```
