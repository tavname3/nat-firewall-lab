# Página de advertencia con nginx

## Propósito

Cuando un cliente intenta acceder a un dominio bloqueado, BIND9 resuelve ese dominio a la IP del servidor (`192.168.100.250`). nginx escucha en esa IP en los puertos 80 y 443, sirviendo una página HTML de advertencia que informa al usuario que el acceso ha sido denegado por políticas de seguridad.

---

## Flujo completo de bloqueo

```
Cliente solicita facebook.com
        │
        ▼
BIND9 resuelve facebook.com → 192.168.100.250  (zona local bloqueada)
        │
        ▼
Navegador conecta a 192.168.100.250:80
        │
        ▼
nginx sirve la página de advertencia (HTTP 200 / bloqueo visual)
```

---

## Configuración nginx sugerida

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/blocked;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Ubicación del HTML: `/var/www/blocked/index.html`

---

## Página de advertencia

La página está diseñada con un estilo de alerta de seguridad corporativa:

- Fondo oscuro (`#0f172a`) con tarjeta central bordeada en rojo
- Icono de advertencia prominente ⚠️
- Título: **"Bloqueado por Políticas de Seguridad"**
- Detalle del bloqueo: motivo, estado, fecha y código `SEC-403`
- Botón **"Regresar"** que ejecuta `history.back()`
- Pie de página: *Departamento de Seguridad Informática*

---

## Instalación

```bash
# Instalar nginx
apt install nginx

# Crear directorio para la página
mkdir -p /var/www/blocked

# Copiar el HTML
cp index.html /var/www/blocked/

# Copiar configuración del sitio
cp blocked.conf /etc/nginx/sites-available/blocked
ln -s /etc/nginx/sites-available/blocked /etc/nginx/sites-enabled/

# Deshabilitar el sitio default si interfiere
rm /etc/nginx/sites-enabled/default

# Verificar y recargar
nginx -t
systemctl reload nginx
```

---

## Verificación

```bash
# Desde un cliente en la LAN, probar que el bloqueo funciona:
curl -I http://facebook.com
# Debe responder desde 192.168.100.250

# Ver accesos en el log de nginx
tail -f /var/log/nginx/access.log
```
