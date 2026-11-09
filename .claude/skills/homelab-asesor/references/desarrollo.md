# Programación, web y aplicaciones (para el homelab)

> Actúa como revisor técnico senior: antes de escribir código, valida si hace falta. (verificar) = comprobar en doc oficial.

## 1. Antes de construir
1. ¿Existe ya una solución autoalojada madura? Si cubre ≥80 % de lo necesario, recomendar usarla y no desarrollar (menos mantenimiento y menos superficie de ataque).
2. Usuarios, datos sensibles, disponibilidad esperada y si es público o solo privado. Eso decide arquitectura y exposición.
3. Quién lo mantendrá y cuánto tiempo. La complejidad del stack debe corresponder a eso.

## 2. Elegir tipo de aplicación
| Tipo | Cuándo | Contras |
|------|--------|---------|
| **Web app** (responsive) | Acceso desde cualquier dispositivo, despliegue único | Sin APIs nativas profundas |
| **PWA** | Instalable en móvil, offline básico, sin tiendas | Límites en iOS |
| **Cross-platform** (Flutter, React Native) | App móvil real con una base de código | Curva y tooling |
| **Nativa** | Máximo rendimiento/integración | Doble coste |
| **CLI / script / automatización** | Tareas internas | Sin UI |
| **Bot** (Telegram/Discord) | Interacción por chat | Depende de plataforma externa |

## 3. Stack (da opciones, no impongas)
- **Frontend**: Astro (contenido/sitios), Next.js/React (apps complejas), Svelte/Vue (alternativas ligeras), HTML+HTMX (mínimo mantenimiento).
- **Backend**: Python (FastAPI/Django), Node/TypeScript (Express/NestJS/Fastify), Go (binarios sencillos y eficientes), .NET (entorno Microsoft).
- **Datos**: SQLite (una instancia, muy simple), PostgreSQL (por defecto cuando crece), Redis solo con necesidad real.
- **Auth**: delegar en un IdP autoalojado (Authentik, Authelia, Keycloak) y OIDC en vez de reinventar login.
Regla: la tecnología que el usuario ya domina gana, salvo razón fuerte.

## 4. Despliegue en este homelab
| Camino | Cuándo | Contras |
|--------|--------|---------|
| **Docker Compose en VM/CT en `.10`** | La mayoría de apps | Despliegue manual, sin HA |
| k3s | Ya lo opera y quiere despliegue declarativo/escalado | Más complejidad; no migrar por moda |
| Plataforma gestionada (p. ej. AgentUI, hosting) | Quiere olvidarse de infra | Dependencia externa y datos fuera de casa |
| Frontend estático en CDN/Pages | Sitio sin backend | Sin lógica servidor |
Exposición: ver `red-y-plataforma.md` (Tailscale para privado, Cloudflare Tunnel + Access para público; el servicio debe llevar su propia auth). Reverse proxy: Nginx Proxy Manager ya existe (ojo con el `cloudflared` duplicado).

## 5. Calidad y operación
- Git para todo (incluidas notas e infra como código); CI con Forgejo/Gitea Actions o GitHub Actions; imágenes con versión fijada, no `latest`.
- Secretos en variables de entorno/secret manager, nunca en el repo; `.env` fuera de git.
- Pruebas automáticas en lo crítico; migraciones de BD versionadas; **backup de la BD y prueba de restauración** (ver `backups.md`).
- Observabilidad mínima: logs estructurados, healthcheck, alerta si cae (Beszel/Uptime Kuma).
- Seguridad (OWASP): validar entradas, consultas parametrizadas, CSRF/CORS correctos, cabeceras de seguridad, dependencias actualizadas (Dependabot/Renovate), HTTPS siempre, límites de tamaño/velocidad.
- Rendimiento web: Core Web Vitals, imágenes optimizadas, caché; SEO técnico en sitios públicos.
- Accesibilidad básica (contraste, teclado, etiquetas).

## 6. Cuando el agente ayuda a programar
Entrega el plan primero (qué ficheros, qué cambia, cómo se prueba); código pequeño y verificable; explica decisiones y alternativas descartadas; señala deuda técnica y riesgos de seguridad. No generes grandes bloques sin plan de pruebas.
