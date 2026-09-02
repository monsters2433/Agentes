# Agentes

Agentes para Claude Code.

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
