# Agentes

Agentes para Claude Code.

## `homelab-asesor`

Asesor **crítico** de tu homelab (repo `monsters2433/homelab`): lo lee en vivo,
pregunta cuando falta un dato que cambia la recomendación, compara los caminos
posibles con ventajas/riesgos y recomienda uno justificando el porqué para *tu*
red. Discrepa cuando una petición es insegura o frágil. Cubre apps internas,
red/OPNsense/Tailscale, Proxmox, K8s, TrueNAS, backups e **IA local** (Ollama,
vLLM, llama.cpp; Claude Code, opencode u OpenClaw sobre modelos locales).

Solo asesora y propone: no ejecuta cambios ni toca el repo `homelab`.

Se invoca con `/homelab-asesor` o pidiendo opinión sobre cualquier cambio del
homelab. Conocimiento de apoyo en `references/` (IA local; red y plataforma).

### `homelab-auditor` (subagente)

Auditor de solo lectura (`.claude/agents/homelab-auditor.md`) para revisiones
largas: contradicciones entre notas, pendientes, servicios duplicados, backups y
contraste de un plan con la topología antes de ejecutarlo.

## `revision-copias`

Revisa la carpeta `ALERTAS` del buzón `alertas-01@laberit.com`, detecta los
informes de Veeam Backup & Replication con resultado `[Failed]` y devuelve un
resumen de los fallos junto con la tabla lista para el parte diario
"Revisión Copias de seguridad".

Se invoca con `/revision-copias`, o simplemente pidiendo "revisa las copias de
seguridad".

Por defecto **solo reporta**: no crea ni envía correos. El borrador se genera
únicamente si se pide de forma explícita.

### Requisito de acceso

La cuenta que ejecuta el agente necesita permiso **Full Access** sobre
`alertas-01@laberit.com`. Tener el buzón montado en Outlook de escritorio no es
suficiente: Microsoft Graph exige el permiso a nivel de buzón.

```powershell
Add-MailboxPermission -Identity alertas-01@laberit.com `
  -User alozal@laberit.com -AccessRights FullAccess -InheritanceType All
```

Alternativa: una regla que copie las alertas a una carpeta `ALERTAS` del buzón
propio.
