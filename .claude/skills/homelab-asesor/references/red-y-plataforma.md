# Red, acceso remoto y plataforma

## 1. Exposición de servicios: de menos a más riesgo
| Método | Cuándo | Contras |
|--------|--------|---------|
| **Tailscale** (VPN malla; ya desplegado con subnet routers en Proxmox y OPNsense) | Uso personal desde móvil/portátil | Cada dispositivo necesita el cliente |
| **Cloudflare Tunnel** (+ Cloudflare Access para autenticar) | URL pública con dominio y HTTPS, sin abrir puertos | Depende de un tercero; el servicio debe tener su propia protección |
| **Reverse proxy interno** (Nginx Proxy Manager) | HTTPS/nombres dentro de la red | No es acceso externo por sí solo |
| **Port-forward directo** | Casi nunca | Expone IP pública; doble NAT en este homelab; sin TLS/auth se convierte en objetivo |

Regla: servicios sin autenticación nativa (Ollama, Prometheus, dashboards, bases de datos) **nunca** se publican; se alcanzan por VPN.

## 2. Razonar el camino de un paquete en ESTE homelab
Antes de proponer reglas, trazar origen → destino leyendo `Red Local.md`:
- Casa `192.168.0.0/24` (gateway TP-Link `.0.1`) → OPNsense WAN `.0.163` → homelab aislado `192.168.10.0/24`. **Doble NAT**.
- Regla LAN: `BLOCK LAN net → 192.168.0.0/24` va **antes** del pase a internet (first-match). Todo host nuevo en `.10` hereda ese aislamiento.
- El tráfico de Tailscale entra por `tailscale0` y no choca con ese bloqueo.
- Un servicio nuevo en `.10`: reserva IP en Kea o estática fuera del pool `.100-.200`; actualizar el inventario de IPs.
- Para publicar algo hacia casa hay que usar Destination NAT + regla WAN en OPNsense, no solo el TP-Link.

## 3. Patrones de plataforma
- **VM vs LXC**: LXC para servicios ligeros Linux (menos RAM, arranque rápido, menos aislamiento); VM para kernels distintos, GPU passthrough, firewalls, cargas no fiables.
- **Docker simple vs k3s**: un servicio aislado → Docker Compose en una VM/CT; k3s solo si ya se gestiona ahí y se necesita su modelo (despliegues declarativos, varios nodos). No migrar por moda.
- **Almacenamiento**: TrueNAS dual-home (`.0.67`/`.10.6`) sin `ip_forward`. NFS/SMB desde `.10` usa la pata `.10.6`. Datos importantes: snapshots ZFS programados, y regla 3-2-1 (el repo marca backups como pendiente: señalarlo cuando aplique).
- **Antes de tocar nada crítico** (OPNsense, bridges, TrueNAS): snapshot de la VM en Proxmox y acceso de emergencia (consola Proxmox) comprobado. Un error en OPNsense deja sin red a todo lo aislado.
- **Punto único de fallo**: un solo nodo físico. Recomendar siempre backups fuera del nodo (PBS u otro destino) y no sobrecargar el host.
- **DNS**: AdGuard Home como DNS interno; nombres internos con reverse proxy + certificados en vez de `IP:puerto`.
- **Observabilidad**: Beszel ya cubre recursos; para más, valorar Uptime Kuma o Prometheus+Grafana según necesidad real (coste de mantenimiento).

## 4. Anti-patrones a señalar siempre
- Contenedores con `privileged` o `network_mode: host` sin necesidad.
- Contraseñas por defecto, registro abierto, `latest` sin pin en servicios críticos.
- Abrir puertos del router por comodidad.
- Dos servicios haciendo lo mismo (ej. `cloudflared` duplicado en CT y en K8s): pedir confirmación de cuál es el activo antes de añadir más.
- Documentar IPs "propuestas" como definitivas.
