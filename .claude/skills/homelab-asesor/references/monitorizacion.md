# Monitorización y alertas

> (verificar) = comprobar en la doc oficial; nombres de plugins y exporters cambian.

## Principios
1. **Alertas antes que dashboards.** Un panel bonito que nadie mira no avisa de nada. Define qué te debe despertar y qué solo se revisa.
2. **Quien monitoriza no puede morir con lo monitorizado.** El homelab es un nodo Proxmox único: si cae, caen también Beszel o Prometheus y nadie avisa. Solución: un *dead-man's switch* externo (Healthchecks.io u otro servicio fuera de casa que alerte si **deja de recibir** latido) y/o monitor en hardware aparte (mini PC, Raspberry Pi).
3. **Pocas alertas, accionables.** Cada alerta debe tener una acción clara; el resto, a un resumen diario. Evita la fatiga de alertas.
4. **Monitoriza los backups** (que se ejecutan y que se verifican), no solo que el servicio esté arriba.
5. Empezar simple: el repo ya tiene **Beszel** (CT 106). Crecer solo cuando una necesidad concreta no la cubra.

## Herramientas (compara siempre)
| Herramienta | Para qué | Ganas | Pierdes |
|-------------|----------|-------|---------|
| **Beszel** (ya desplegado) | CPU/RAM/disco/red de hosts y contenedores | Ligero, trivial | Poco flexible, sin alertas complejas |
| **Uptime Kuma** | ¿Está vivo el servicio? HTTP, TCP, ping, DNS, push, certificados | Muy fácil, alertas a muchos canales | No hace métricas internas |
| **Prometheus + Grafana (+ Alertmanager)** | Métricas históricas, alertas con reglas | Estándar de facto, ecosistema enorme (exporters) | Curva y mantenimiento; consume recursos |
| **Netdata** | Métricas en tiempo real con cero configuración | Rápido de ver | Retención/escala; versión cloud (verificar) |
| **Zabbix / Checkmk / LibreNMS** | Infra heterogénea, SNMP, red | Completos | Pesados para un homelab pequeño |
| **Loki + Alloy/Promtail** | Logs centralizados | Se integra con Grafana | Otro componente que mantener |
| **Healthchecks.io / Gatus** | Latidos de tareas programadas (backups, cron) y estado como código | Detecta lo que **no** ocurre | Healthchecks.io es SaaS externo (también es su ventaja) |
| **ntfy / Gotify** | Notificaciones push | Simple, autoalojable | ntfy público depende de servidor externo si no se autoaloja |

Recomendación de arranque: **Beszel + Uptime Kuma + Healthchecks.io (latido externo) + ntfy**. Subir a Prometheus/Grafana solo si necesitas histórico, correlación o dashboards de Proxmox/ZFS/red.

## Qué monitorizar en ESTE homelab
| Elemento | Cómo | Señal de alerta |
|----------|------|-----------------|
| **Proxmox host** | Beszel/agente; `pve-exporter` si usas Prometheus | RAM/IO alta, disco >85 %, VM caída |
| **OPNsense** (VM 105) | Plugin `os-node_exporter` o SNMP/Telegraf (verificar); ping desde fuera | WAN caída, CPU, estado de gateway, VPN caída |
| **TrueNAS** (VM 201) | Alertas nativas + SMART + estado del pool ZFS | Pool degradado, SMART con errores, scrub fallido, espacio >80 % |
| **k3s** | Probes + `kube-prometheus-stack` (pesado) o Beszel/Uptime Kuma para empezar | Nodo NotReady, pods en CrashLoop, PVC lleno |
| **Backups** | Healthchecks.io al terminar cada job; métricas/notificaciones de PBS; `/revision-copias` para Veeam | Latido ausente, verify fallido |
| **DNS (AdGuard)** | Uptime Kuma consulta DNS | No resuelve |
| **Internet / WAN** | Ping a dos destinos + prueba DNS externa | Pérdida >X % durante N min |
| **Certificados** | Uptime Kuma (expiración) | <14 días |
| **VPN** | Ver `vpn.md` §Monitorización | Handshake caducado, nodo offline |
| **UPS** (si existe) | NUT | En batería, batería baja |

## Ruta de alertas
Evento → regla con umbral y **duración** (no alertar por picos de segundos) → canal (ntfy/Telegram/correo) → quien actúa. Dos niveles: **crítico** (push inmediato) e **informativo** (resumen diario). Probar las alertas provocándolas una vez.

## Si OpenClaw/un agente consulta la monitorización
Solo lectura (token de consulta), sin permiso de silenciar o borrar alertas.

## Anti-patrones
- Monitor solo dentro del nodo que monitoriza; dashboards sin alertas; alertar de todo; no revisar espacio de disco ni backups; Prometheus con retención infinita en un disco pequeño; exponer Grafana/Prometheus/Kuma a Internet (acceso por Tailscale).
