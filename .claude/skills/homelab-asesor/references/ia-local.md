# IA local en el homelab

> Los datos de esta guía son orientativos y envejecen rápido. Todo lo marcado (verificar) se comprueba en la documentación oficial antes de afirmarlo.

## 1. Preguntas que cambian la recomendación
1. ¿GPU en el nodo Proxmox? Modelo y **VRAM** (decide qué modelo cabe). Si no consta en el repo, preguntar.
2. ¿CPU/RAM libre? (RAM del sistema permite offload parcial, mucho más lento.)
3. ¿Objetivo? Chat/RAG, asistente de código agéntico, embeddings, voz, visión, automatización.
4. ¿Privacidad estricta (nada sale de casa) o vale un modelo en la nube para lo difícil?
5. ¿Desde dónde se usa: PC, portátil, móvil, otros servicios? ¿Cuántos usuarios simultáneos?

## 2. Dimensionado (regla práctica)
- Memoria del modelo ≈ parámetros × bytes por peso: Q4 ≈ 0.55-0.65 GB por mil millones de parámetros, Q8 ≈ 1.1, FP16 = 2.
- Añadir **KV cache**: crece con el contexto. Los agentes de código necesitan contexto largo (32k+ como punto de partida); con poca VRAM el contexto manda más que el tamaño del modelo.
- Cabe en VRAM → rápido. Desborda a RAM/CPU → utilizable solo para tareas pequeñas.
- Modelos MoE: cargan todo en memoria pero activan pocos parámetros por token; pueden ir aceptables con offload. (verificar según el modelo concreto)

## 3. Servidores de inferencia
| Opción | Cuándo | Contras |
|--------|--------|---------|
| **Ollama** | Empezar rápido, muchos modelos, API simple | Menos control fino, rendimiento multiusuario limitado |
| **llama.cpp (`llama-server`)** | Control total, CPU/GPU mixto, GGUF | Más configuración manual |
| **vLLM** | Varios usuarios, alto throughput, GPU NVIDIA seria | Exige VRAM y GPU compatible, más pesado |
| **LM Studio / similares** | Uso en un PC de escritorio | No es un servicio de servidor |

Interfaces: Open WebUI (chat, RAG, multiusuario con login). Proxies/traductores de API: LiteLLM, claude-code-router (verificar estado actual).

## 4. Usar Claude Code / agentes sobre modelos locales
Claude Code habla el protocolo de la API de Anthropic. Hay tres rutas:

| Ruta | Cómo | Pros | Contras |
|------|------|------|---------|
| A. Servidor con API compatible con Anthropic | Variables `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL` apuntando al servidor | Sin intermediarios | Comprobar que la versión del servidor expone `/v1/messages` y soporta tool use (verificar) |
| B. Proxy traductor (LiteLLM / claude-code-router) delante de Ollama/vLLM | El proxy traduce Anthropic ↔ OpenAI | Funciona con cualquier backend | Un componente más que mantener y asegurar |
| C. Otro agente de código diseñado para modelos locales | opencode, OpenClaw, Aider, Cline/Continue (verificar estado y soporte actual de cada uno) | Pensado para backends OpenAI-compatibles, sin traducir | No es Claude Code; otro flujo y otras herramientas |

Notas críticas:
- Un modelo local **no iguala a Claude** en tareas agénticas largas: más fallos de tool use y ediciones. Recomendar modelos específicos de código y probar con una tarea pequeña antes de comprometerse.
- Contexto: el valor por defecto del servidor suele ser insuficiente; fijarlo explícitamente (p. ej. `OLLAMA_CONTEXT_LENGTH` en Ollama, verificar) y dimensionar VRAM para ello.
- Híbrido razonable: modelo local para tareas privadas/baratas, Claude para lo complejo; alternar con variables de entorno o un script de lanzamiento.
- Claude Code usa también un modelo pequeño para tareas auxiliares; si se redirige todo a un único backend, hay que fijar también esa variable (verificar nombre vigente en la doc).

## 5. Dónde alojarlo en este homelab
| Ubicación | Pros | Contras |
|-----------|------|---------|
| **VM con GPU passthrough** (IOMMU/vfio, VM exclusiva de la GPU) | Aislamiento, drivers propios, snapshots | La GPU queda dedicada a esa VM; passthrough exige BIOS/IOMMU correctos |
| **LXC con GPU compartida del host** | Varios CTs pueden compartirla, menos sobrecarga | Driver del host compartido, menos aislamiento, más delicado |
| **k3s (workers son VMs)** | Orquestación homogénea | Sin GPU en los workers no sirve; añade complejidad innecesaria para 1 servicio |
| **Otro equipo (PC con GPU)** | No toca el nodo único de producción | Otro dispositivo que mantener; hay que enrutarlo por Tailscale |

Recomendar la ubicación que **no ponga en riesgo el único nodo Proxmox** (el firewall y el NAS dependen de él): un uso intensivo de CPU/GPU/RAM puede degradar OPNsense y TrueNAS. Proponer límites de recursos y probar fuera de horas críticas.

## 6. Seguridad de la IA local
- Ollama y la mayoría de servidores **no tienen autenticación**: nunca exponer el puerto del modelo; publicar solo la interfaz con login (Open WebUI) y desactivar registro abierto.
- Acceso remoto: Tailscale (ver `red-y-plataforma.md`). Los agentes de código con permisos de shell sobre un modelo menos fiable deben ejecutarse en un sandbox/VM sin acceso a la red de casa.
- Modelos descargados de terceros: usar fuentes oficiales; los pesos no ejecutan código, pero los formatos antiguos tipo pickle sí pueden (preferir GGUF/safetensors).
