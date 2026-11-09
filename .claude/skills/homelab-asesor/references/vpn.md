# VPN y acceso remoto: WireGuard, Tailscale y alternativas

> (verificar) = comprobar en la doc oficial. Estado actual del homelab (repo `Tailscale.md`): **Tailscale SaaS** con dos subnet routers (Proxmox para `192.168.0.0/24`, OPNsense para `192.168.10.0/24`). No lo reemplaces sin razón.

## 1. Comparativa
| Opción | Qué es | Ganas | Pierdes |
|--------|--------|-------|---------|
| **Tailscale** (actual) | Malla WireGuard con plano de control SaaS | Cero puertos abiertos, atraviesa NAT/CGNAT, MagicDNS, ACLs, SSO | Dependes del control plane de un tercero |
| **Headscale** | Control plane de Tailscale autoalojado | Sin tercero | Mantenimiento; menos funciones; el control plane debe ser alcanzable (VPS público) |
| **NetBird** | Malla WireGuard con control plane abierto, autoalojable o SaaS | Open source completo, SSO/ACLs | Más piezas si se autoaloja |
| **ZeroTier** | Red virtual L2/L3 propia | Fácil, controlador autoalojable | Protocolo propio, no WireGuard |
| **WireGuard "puro"** (en OPNsense/Proxmox/VM) | Túnel punto a punto con claves | Mínimo, rápido, sin tercero, control total | Necesita **puerto UDP entrante**, gestionar claves/IPs a mano, no atraviesa CGNAT |
| **OpenVPN** | VPN clásica TLS | Compatibilidad, TCP/443 si el firewall lo exige | Más lento y pesado que WireGuard |
| **IPsec/IKEv2** | VPN nativa en móviles y routers | Sin app en el cliente | Configuración compleja |
| **Cloudflare Tunnel / Zero Trust** | Acceso a apps (HTTP/SSH) sin puertos | Auth, sin IP pública | No es una VPN de red completa; dependes de Cloudflare |
| **Bastión SSH / jump host** | Entrada solo por SSH con claves | Muy simple | Solo SSH/forwards, no red |

**Recomendación base:** mantener Tailscale para el día a día. Añadir **WireGuard nativo en OPNsense** como segundo camino independiente (no depende de ningún tercero) si quieres resiliencia y puedes abrir un puerto UDP; Headscale/NetBird solo si el motivo es no depender de un SaaS.

## 2. WireGuard en OPNsense (si lo montas)
Camino de un paquete en este homelab: Internet → router de la operadora (bridge) → **TP-Link** (NAT) → **OPNsense WAN `192.168.0.163`** → LAN `.10`. Hay **doble NAT**:
1. Comprobar que la IP pública no está tras **CGNAT** (si la IP WAN del TP-Link es 100.64.x.x o privada, no hay entrada posible: usar Tailscale/túnel).
2. **Port-forward UDP** (p. ej. 51820) en el TP-Link → `192.168.0.163`, y regla **WAN pass UDP 51820** en OPNsense (el WAN bloquea todo por defecto; no hay bogon/private block, ver `Red Local.md`).
3. Reservar la IP de la WAN de OPNsense por MAC (ya es pendiente del repo) o el port-forward se rompe al cambiar el DHCP.
4. DDNS si la IP pública cambia.
5. Interfaz `wg0` con red propia (p. ej. `10.10.10.0/24`), peers con **AllowedIPs** de /32 por cliente; reglas de firewall en la interfaz WireGuard (qué pueden alcanzar: solo `192.168.10.0/24` o también casa) y NAT outbound si el cliente sale a Internet.
6. Cliente: `AllowedIPs = 192.168.10.0/24` para **split tunnel** (solo el homelab) o `0.0.0.0/0` para túnel completo; `DNS = 192.168.10.1` o AdGuard; MTU 1420 por defecto, bajar si hay fragmentación; `PersistentKeepalive = 25` para móviles tras NAT.
7. Gestión de claves: una clave por dispositivo, revocar quitando el peer; claves privadas nunca en el repo.
Ventaja del diseño: tráfico WireGuard entra por su interfaz, no choca con `BLOCK LAN→.0`; razonar siempre qué reglas aplican a esa interfaz.

## 3. Monitorización de VPN
| Qué | Cómo |
|-----|------|
| Estado WireGuard | `wg show` → **latest handshake** y transferencia por peer. Con keepalive activo el handshake se renueva ~cada 2 min: si supera ~3 min (180 s), el peer está caído (sin tráfico ni keepalive no hay handshake, es normal en peers pasivos) |
| OPNsense | VPN → WireGuard → Estado; logs del servicio; exportar métricas con node_exporter/Telegraf (verificar) |
| Prometheus | `prometheus_wireguard_exporter` u otro exporter de WireGuard (verificar proyecto vigente); alerta: `time() - handshake > 180` y tráfico recibido sin cambios |
| Tailscale | `tailscale status` / consola de administración (nodos offline, rutas sin aprobar, claves por caducar: **desactivar key expiry** en subnet routers); métricas de cliente (`tailscale metrics`, verificar) |
| Disponibilidad extremo a extremo | Uptime Kuma o un job que haga ping/curl a un recurso interno **a través** de la VPN, con latido a Healthchecks.io |
| Seguridad | Alertar de peers nuevos, handshakes desde IPs inesperadas, picos de tráfico; revisar peers no usados |

## 4. Reglas de oro
- Una sola puerta de entrada documentada. Cada VPN adicional es superficie de ataque y mantenimiento: justifica cada una.
- No exponer servicios sin auth a Internet; entrar por VPN.
- Mínimo privilegio en la VPN: ACLs de Tailscale o reglas OPNsense por interfaz; los dispositivos de invitados/IoT no deben alcanzar el homelab.
- Subnet routers duplicados (Proxmox y OPNsense) cubren redes distintas; si alguno cae, esa red desaparece: monitorizarlos.
- Probar el acceso remoto **desde fuera** (datos móviles) tras cada cambio de red; tener acceso de emergencia (consola Proxmox) por si te dejas fuera.
- Documentar peers, rutas, puertos y claves (sin valores) en una nota del repo.
