# Home Assistant, domótica y medios

> (verificar) = comprobar en la doc oficial.

## Home Assistant (HA)
**Formas de instalarlo**
| Opción | Ganas | Pierdes |
|--------|-------|---------|
| **HA OS en VM de Proxmox** (recomendada) | Supervisor, add-ons, backups integrados, snapshots de VM | Ocupa una VM; el passthrough USB se hace en Proxmox |
| HA Container (Docker) | Ligero, control total | Sin add-ons ni Supervisor; más mantenimiento manual |
| HA Core en venv | Minimalista | Poco práctico; más frágil |
Los scripts de la comunidad que automatizan crear la VM son útiles, pero **revisa el script antes de ejecutarlo como root** en el host.

**Punto crítico en ESTE homelab: la red.** HA descubre dispositivos por mDNS/SSDP/broadcast, que **no cruza routers**. Si los dispositivos están en la red de casa `192.168.0.0/24` y HA vive en la aislada `192.168.10.0/24`, el descubrimiento fallará y OPNsense bloquea casa→aislada. Opciones:
1. HA en `vmbr0` (casa): fácil, pero sale del aislamiento.
2. **Dual-home** como TrueNAS (una pata en cada red, sin forwarding): descubre en casa y se gestiona desde `.10`.
3. Crear una **VLAN/segmento IoT** dedicado y aislado, con HA con pata en él (más trabajo, mejor seguridad).
4. Repetidor mDNS en OPNsense (plugin) (verificar).
Recomendar según cuántos dispositivos IoT y cuánto importe aislarlos; los IoT baratos son los menos seguros: mejor aislarlos de los PCs.

**Radios (Zigbee/Z-Wave/Thread)**: dongle USB con passthrough de USB a la VM; usar una extensión USB para alejarlo del USB3/SSD (interferencias). Zigbee2MQTT (flexible, MQTT) vs ZHA (integrado, simple). Coordinadores de red (por Ethernet/PoE) evitan el passthrough y permiten colocarlos mejor.
**Protocolos**: preferir *local-first* (Zigbee, Z-Wave, Matter/Thread, ESPHome, MQTT) frente a dispositivos que dependen de la nube del fabricante.
**Complementos habituales**: Mosquitto (MQTT), ESPHome, Node-RED (cuidado con complejidad), Grafana/InfluxDB si hay histórico real.
**Backups**: backup integrado de HA (con cifrado y copia fuera de la VM) **además** del backup de la VM (ver `backups.md`). Restaurar es parte de la prueba.
**Acceso remoto**: Tailscale (privado, recomendado) o Cloudflare Tunnel con Access; Nabu Casa es la opción de pago sencilla y financia el proyecto. No abrir el puerto 8123 al exterior.
**Fiabilidad**: automatizaciones críticas (alarma, calefacción) no deben depender de que el nodo único esté vivo ni de internet; valorar un dispositivo físico independiente para lo esencial.
**IA + HA**: integraciones con asistentes de voz locales/LLM existen; el agente de OpenClaw con acceso a HA multiplica el riesgo: dar solo los servicios mínimos y confirmar acciones físicas (cerraduras, alarma).

## Medios
**Servidor**: Jellyfin (libre, sin cuenta ni suscripción) vs Plex (más pulido, depende de cuenta/servicios de Plex) vs Emby. Recomendación por defecto: Jellyfin.
**Biblioteca**: datos en TrueNAS; compartir por **NFS** para clientes Linux (mejor rendimiento y permisos) o SMB si hay Windows. Estructura de carpetas y mismo dataset para facilitar *hardlinks* (guías TRaSH, verificar) si usas automatización de descargas.
**Transcodificación**: la GPU/iGPU Intel (Quick Sync) por passthrough o `/dev/dri` en LXC es lo eficiente; la CPU transcodifica mal en VMs. Mejor que los clientes reproduzcan en *direct play* (códecs compatibles) y evitar transcodificar.
**Otros**: Immich (fotos), Navidrome (música), Audiobookshelf (audiolibros/podcast), Kavita/Calibre-Web (libros).
**Automatización de descargas (*arr, clientes torrent/usenet)**: sé directo: úsalo solo con contenidos que tengas derecho a usar; no aconsejes pirateo. Sí puedes asesorar la arquitectura (red, VPN del cliente de descargas, aislamiento en su propia VM/red).
**Backups**: los medios reemplazables pueden tener retención baja; las fotos personales (Immich) son lo irreemplazable: prioridad 3-2-1 máxima.
**Exposición**: Jellyfin/Immich a Internet solo tras reverse proxy con HTTPS y autenticación robusta; preferir Tailscale. Cuidado con los términos de servicio de Cloudflare sobre tráfico de vídeo por el túnel gratuito (verificar).
