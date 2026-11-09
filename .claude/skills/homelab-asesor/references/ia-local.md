# IA en el homelab: OpenClaw con Claude (y modelos locales como alternativa)

> Contexto del usuario: **el hardware es irrelevante** porque el plan es usar **Claude vía OpenClaw**, no inferencia local. El modelo corre en la nube de Anthropic; el homelab solo aloja el gateway. No preguntes por GPU/VRAM salvo que el usuario pida explícitamente modelos locales.
>
> Todo lo marcado (verificar) se comprueba en la doc oficial (`docs.openclaw.ai`, docs de Anthropic) antes de afirmarlo; en algunas sesiones ese dominio está bloqueado: dilo y pide que lo permitan.

## 1. Qué es OpenClaw (según fuentes secundarias, verificar)
Gateway de agentes **autoalojado y open source**: se conecta a un proveedor de LLM (Anthropic, OpenAI, Gemini, modelos locales) y expone el agente por canales de mensajería (Telegram, WhatsApp, Discord…) y una interfaz web, con skills y ejecución de herramientas. No ejecuta el modelo por sí mismo.

## 2. Preguntas que sí cambian el diseño
1. ¿Por qué canales lo vas a usar (móvil, Telegram, web)? ¿Solo tú o más personas?
2. ¿Qué debe poder **hacer** el agente: solo conversar, leer ficheros, ejecutar comandos, tocar el homelab/Home Assistant? Esto fija el nivel de riesgo.
3. Autenticación con Claude: **API key** (facturación por uso, límite de gasto) o login de Claude Code/suscripción (`claude-cli`). Las fuentes indican que la política sobre usar suscripción en herramientas de terceros cambió en 2026 (verificar términos vigentes antes de recomendar nada). Por defecto recomendar API key con límite de gasto.
4. ¿Presupuesto mensual tolerable? Un agente con herramientas puede gastar mucho en bucles.

## 3. Alojamiento recomendado en este homelab
| Ubicación | Ganas | Pierdes |
|-----------|-------|---------|
| **VM Linux dedicada en `vmbr1` (`192.168.10.x`) con Docker** (recomendada) | Aislada de casa por la regla `BLOCK LAN→.0/24`, snapshots, rollback trivial | Una VM más; reservar IP fuera del pool Kea |
| LXC con Docker | Ligero | Menos aislamiento para un agente con shell |
| k3s | Ya existe | Complejidad innecesaria para un servicio; el agente tendría cerca el clúster |
| Equipo personal | Cero infra | El agente alcanza tus datos personales y tu red de casa |

Razón: el agente puede ejecutar código; que viva en la red aislada limita el daño. Los recursos que necesita son modestos (las fuentes citan unos 2 GB de RAM para construir la imagen Docker, verificar).

## 4. Instalación (esquema, verificar contra la doc oficial)
- Docker Compose v2; el script de instalación oficial lanza el onboarding, pide la clave del proveedor, genera un token de gateway en `.env` y arranca el servicio. Existe también ruta manual (`docker build` + `onboard` + `docker compose up -d`).
- Proveedor Anthropic: clave `sk-ant-…` creada en la consola de Anthropic, en variable de entorno/secret, nunca en el repo.
- Acceso desde el móvil: por **Tailscale** (la ruta `192.168.10.0/24` ya la anuncia OPNsense, ver `Tailscale.md`), enlazando el gateway a la interfaz interna, no a `0.0.0.0` hacia casa.

## 5. Seguridad: lo crítico en un agente con herramientas
Un agente que lee mensajes y ejecuta herramientas es objetivo de **prompt injection**: texto malicioso en un mensaje, web o fichero puede intentar que ejecute acciones.
- Ejecutar en **VM dedicada** sin credenciales del resto del homelab (sin SSH keys a Proxmox, sin tokens de Cloudflare ni de OPNsense).
- API key **dedicada** con **límite de gasto** y rotable.
- No publicar el gateway en Internet (ni port-forward ni Cloudflare Tunnel sin auth fuerte); acceso solo por Tailscale.
- Restringir quién puede escribirle (emparejamiento/allowlist de remitentes) (verificar nombre exacto de la opción).
- Skills/plugins: revisar el código de cada skill de terceros antes de activarla; allow-list, no default-allow.
- Sandbox de ejecución (contenedor) para las herramientas; permisos mínimos; confirmación humana para acciones destructivas.
- Logs y revisión periódica del consumo.
- Snapshot de la VM antes de dar más permisos al agente.

## 6. Alternativas (compara siempre)
| Opción | Cuándo | Contras |
|--------|--------|---------|
| **OpenClaw + API de Claude** | Agente personal multicanal, autoalojado | Superficie de ataque amplia; proyecto joven y cambiante (verificar madurez) |
| **Claude Code** directo (CLI/web) | Programación y tareas de ficheros con permisos finos y soporte oficial | Sin canales de mensajería |
| **Open WebUI + API de Claude** (vía proxy/función compatible, verificar) | Chat con login multiusuario y RAG | Menos "agéntico" |
| **Modelo local** (Ollama, vLLM, llama.cpp) con opencode/OpenClaw | Privacidad total, coste cero por token | Exige GPU/VRAM; mucho menos capaz en tareas agénticas |

## 7. Modelos locales (solo si el usuario los pide)
Memoria ≈ parámetros × bytes/peso (Q4 ≈ 0.55-0.65 GB por mil millones; Q8 ≈ 1.1; FP16 = 2) + KV cache, que crece con el contexto (agentes de código: 32k+). Servidores: Ollama (rápido de montar), llama.cpp (`llama-server`, control fino), vLLM (multiusuario con GPU NVIDIA seria). Claude Code contra un backend local: variables `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL` si el servidor expone la API de Anthropic, o con proxy traductor (LiteLLM, claude-code-router) (verificar). En ese caso sí hay que preguntar por GPU y respetar que el nodo Proxmox es único.
Seguridad: Ollama no tiene autenticación; nunca exponer su puerto.
