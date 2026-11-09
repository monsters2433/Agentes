# Plantilla de nota para el repo `homelab`

Imita el estilo existente (Obsidian): frontmatter YAML, títulos con emoji solo en el MOC, `[[wikilinks]]`, callouts `> [!info]`, tablas, y sección final **Relacionado**. Usa la fecha real de hoy en `actualizado`. Marca como `⚠️ por confirmar` cualquier dato no verificado (IP, ID de VM).

```markdown
---
tipo: servicio            # servicio | nodo | red | proceso
area: homelab
host: proxmox             # donde corre (opcional)
rol: <p. ej. backup | domotica | ia | medios | desarrollo>
actualizado: AAAA-MM-DD
tags: [homelab, <servicio>, <rol>]
---

# <Nombre>

<1-2 líneas: qué es y para qué sirve en el homelab.>

## Dónde corre
| Elemento | Valor |
|----------|-------|
| VM/CT (ID) | ⚠️ por confirmar |
| IP / red | `192.168.10.x` (estática, fuera del pool Kea `.100-.200`) |
| Puertos | <puerto → uso> |
| Acceso | Solo Tailscale / reverse proxy / túnel |

## Configuración relevante
<Decisiones y el POR QUÉ de cada una: valores, reglas de firewall, variables.>

## Seguridad
<Qué está expuesto, qué no, autenticación, secretos (dónde viven, sin valores).>

## Backups y restauración
<Qué se copia, dónde, retención, última prueba de restauración.>

## Pendiente
- [ ] <tareas concretas, incluidas las confirmaciones de IP/ID>

## Relacionado
- [[Homelab]] · [[Red Local]] · [[Proxmox]] · <otras>
```

## Cambios que acompañan a una nota nueva
1. Enlazarla desde `Homelab.md` (sección del componente) y, si procede, en el diagrama ASCII.
2. Añadir el host al **inventario** de `Red Local.md` §3.1 (IP, ID, notas).
3. Si abre puertos o reglas: documentarlo en las secciones de firewall/NAT de `Red Local.md`.
4. Actualizar `Pendientes / a revisar` de `Homelab.md` si introduce dudas abiertas.
5. Actualizar `actualizado:` de cada nota tocada.
Nunca incluir contraseñas, tokens, claves, IPs de Tailscale ni MACs reales (el repo es público y ofusca esos datos).
