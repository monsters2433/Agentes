# Backups: Proxmox Backup Server (PBS) y Veeam

> (verificar) = comprobar en la doc oficial; versiones y compatibilidades cambian.

## Principio rector
Un backup que no se ha restaurado no existe. Y copias en el mismo nodo, discos o ubicación **no son backup, son una comodidad**. Objetivo: 3-2-1 (3 copias, 2 medios, 1 fuera de casa), idealmente con una copia inmutable o aislada y prueba de restauración periódica.
El repo `homelab` tiene los backups como **pendiente sin documentar** y un único nodo Proxmox: es la prioridad nº1 de riesgo; señálalo cuando proceda.

## Proxmox Backup Server (PBS)
- Copias incrementales con **deduplicación** a nivel de chunk, cifrado en cliente opcional, **verify jobs** (integridad), **prune** (retención) y **garbage collection**. Para VMs usa bitmap de cambios: las siguientes copias son rápidas.
- Se integra en PVE como almacenamiento (Datacenter → Storage → PBS) y se programa con *Backup jobs*. Para contenedores y VMs también hay `proxmox-backup-client` para carpetas del host/otros equipos.
- **Dónde instalarlo** (decide con el usuario):
  | Opción | Ganas | Pierdes |
  |--------|-------|---------|
  | **Equipo/hardware aparte** (mini PC, NAS) — recomendado | Sobrevive al fallo del nodo único | Coste y un dispositivo más |
  | VM en el mismo Proxmox con disco/datastore en otro medio (p. ej. dataset de TrueNAS por NFS/iSCSI) | Barato | Si cae el nodo, pierdes PBS y las VMs a la vez; TrueNAS es VM del mismo nodo |
  | PBS alojado externo / hosting | Copia fuera de casa | Coste, ancho de banda |
- Datastore sobre ZFS o disco ext4/xfs dedicado; evitar NFS/SMB lento para el datastore principal (rendimiento y bloqueos) (verificar recomendaciones).
- **Sync jobs** a un segundo PBS remoto (offsite) por Tailscale. Ver también `Tailscale.md` para la ruta.
- Retención típica: keep-last 3, daily 7, weekly 4, monthly 6 (ajustar a espacio y RPO).
- Restauración: probar restaurando a una VM de prueba aislada; documentar el procedimiento y la clave de cifrado fuera del propio backup.
- Claves: guardar la clave de cifrado y el *fingerprint* de PBS en un gestor de contraseñas; sin la clave la copia es irrecuperable.
- Datos de aplicaciones: para bases de datos, copia coherente (snapshot de VM con guest agent, o dump); para TrueNAS, snapshots ZFS + replicación, no solo backup de la VM.
- Lo no cubierto: la config de OPNsense (exportar `config.xml`), las notas del repo (git), los secretos.

## Veeam
- Veeam Backup & Replication protege Proxmox VE mediante su integración/plug-in propio (según versión; **verificar** versión mínima y licencias). Alternativas dentro de Veeam: agentes para Linux/Windows en cada VM o equipo.
- Comparación para este homelab:
  | Criterio | PBS | Veeam |
  |----------|-----|-------|
  | Integración nativa con Proxmox | Total | Vía plug-in/agentes |
  | Coste | Gratuito (soporte opcional) | Licencias; hay edición Community con límites (verificar) |
  | Entorno mixto (Windows, VMware, nube) | Limitado | Fuerte |
  | Inmutabilidad | Datastore + permisos; no WORM nativo | Repositorio Linux *hardened* inmutable |
  | Habilidad que ya usas | Nueva | La usas en el trabajo (revisión de alertas Veeam) |
  Recomendar: si todo es Proxmox, **PBS**; Veeam si quieres unificar con el entorno profesional o necesitas inmutabilidad/agentes. Combinar ambos solo con una razón clara (más mantenimiento).
- Para revisar fallos de Veeam en el buzón de alertas existe la skill **`/revision-copias`** del repo `Agentes`; no la dupliques, remite a ella.
- Buenas prácticas Veeam: repositorio separado de la producción, regla 3-2-1-1-0 (una copia offline/inmutable, cero errores en verificación), SureBackup/pruebas de recuperación, cuentas de servicio de mínimo privilegio, hosts de Veeam fuera del dominio si es posible (verificar).

## Anti-patrones
- Backup en el mismo disco/nodo; no cifrar copias que salen de casa; no probar restauración; retención infinita sin control de espacio; olvidar la clave de cifrado; mezclar datos y backups en el mismo pool ZFS.
