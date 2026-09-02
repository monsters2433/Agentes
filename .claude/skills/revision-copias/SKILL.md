---
name: revision-copias
description: Revisa la carpeta ALERTAS del buzón alertas-01@laberit.com en Outlook, detecta los informes de Veeam Backup & Replication con resultado Failed/Error y devuelve un resumen de los fallos junto con la tabla lista para el correo diario "Revisión Copias de seguridad". Úsalo cuando se pida revisar las copias de seguridad, las alertas de backup, los correos de Veeam o preparar el parte diario de copias.
---

# Revisión de copias de seguridad (alertas Veeam)

Revisa los informes de backup que llegan al buzón de alertas, extrae los fallos y
prepara el parte diario.

## 1. Localizar los correos

Buzón: `alertas-01@laberit.com`, carpeta `ALERTAS`.

```
outlook_email_search(
  mailboxOwnerEmail = "alertas-01@laberit.com",
  folderName        = "ALERTAS",
  afterDateTime     = <inicio del periodo a revisar>,
  order             = "newest",
  limit             = 25
)
```

Pagina con `nextOffset` / `nextCursor` hasta agotar el periodo. Si no se indica
periodo, revisa **desde las 00:00 del día anterior** (las copias corren de noche
y los informes llegan de madrugada).

### Si el buzón no es accesible

`outlook_email_search` puede responder `FORBIDDEN` o `NOT_FOUND: Default folder
Root not found`. Significa que la cuenta que ejecuta el agente no tiene **Full
Access** sobre `alertas-01@laberit.com` en Exchange — tenerlo montado en Outlook
de escritorio no basta, Graph exige el permiso a nivel de buzón.

No lo silencies ni lo trates como "no hay alertas": dilo explícitamente e indica
el remedio (un admin de Exchange debe ejecutar
`Add-MailboxPermission -Identity alertas-01@laberit.com -User alozal@laberit.com
-AccessRights FullAccess -InheritanceType All`, o crear una regla que copie las
alertas a una carpeta `ALERTAS` del buzón propio).

Alternativa mientras tanto: buscar en el buzón propio con
`outlook_email_search(query = "Backup job", ...)`.

## 2. Clasificar cada correo

Los informes los genera **Veeam Backup & Replication**. Vienen de "Alertas
Sistemas" y el asunto lleva el resultado entre corchetes:

| Resultado | Asunto de ejemplo | Banner |
|---|---|---|
| Correcto | `MSM ALVA SAG [Success] ALVA-BACKUP VMs (2 VMs)` | Verde, "Success" |
| Aviso | `... [Warning] ...` | Amarillo/naranja |
| **Fallo** | `CITRUS-[Failed] COPIA VMs (5 machines) 1 failed` | **Rojo, "Error"** |

**Solo interesan los `[Failed]` / banner rojo / `Error ≥ 1`.** Los `[Success]` se
ignoran (no se listan ni se mencionan uno a uno; como mucho, un recuento).

No te fíes solo del asunto: abre el cuerpo con `read_resource` y confirma en la
tabla de resumen que el contador **`Error`** es mayor que 0. Un job puede venir
como `[Success]` con warnings, o como `[Failed]` tras un reintento que sí acabó
bien (`Retry N` en el nombre del job).

Si hay varios reintentos del mismo job (`(Retry 1)`, `(Retry 2)`, `(Retry 3)`),
**quédate solo con el último intento** y repórtalo una vez.

## 3. Extraer los fallos

Estructura del cuerpo (HTML):

- **Cabecera**: `Backup job: <NOMBRE> (Retry N)`, creado por `DOMINIO\usuario`, y
  a la derecha el resultado y `N of M VMs processed`.
- **Tabla de resumen**: `Success` / `Warning` / `Error`, `Start time`, `End time`,
  `Duration`, `Total size`, `Data read`, `Transferred`, `Backup size`, `Dedupe`,
  `Compression`. La columna de la derecha, en los fallidos, ya trae el texto del
  error.
- **Tabla `Details`**: una fila por máquina con `Name`, `Status`, `Start time`,
  `End time`, `Size`, `Read`, `Transferred`, `Duration`, `Details`.

De cada fila con `Status` = `Error` (o `Failed`) toma:

- **Nombre de la máquina** (`Name`), p. ej. `SERVERFRUIT`.
- **Texto íntegro de `Details`**, sin resumir ni reformular. Es lo que va en la
  columna ERROR del parte. Conserva los códigos (`[0x800423f4]`), los nombres de
  writer (`[SqlServerWriter]`), los GUID y los `Processing finished with errors
  at <fecha>`. Mantén los saltos de línea entre errores distintos de la misma
  máquina.

Ejemplo real de lo que hay que capturar tal cual:

```
Processing SERVERFRUIT Error: Unfreeze error: [Backup job failed. Cannot create a
shadow copy of the volumes containing writer's data. A VSS critical writer has
failed. Writer name: [SqlServerWriter]. Class ID: [{a65faa63-5ea8-4ebc-9dbd-a0c4db26912a}].
Instance ID: [{2885bd9a-42db-4dd4-a623-e9ba555c2b0d}]. Writer's state:
[VSS_WS_FAILED_AT_PREPARE_SNAPSHOT]. Error code: [0x800423f4].]
```

## 4. Identificar el cliente

El prefijo del asunto identifica al cliente, pero en el parte se escribe el
nombre comercial completo. Equivalencias conocidas:

| Prefijo en el asunto | CLIENTE en el parte | SW BACKUP |
|---|---|---|
| `CITRUS` | CITRUSLANDIA | VEEAM |
| `MSM ALVA` | Alva MSM | VEEAM / SyncBackPro |
| `TMD` | TMD Friction | VEEAM |
| `CAXTON` | CAXTON | VMware |
| `ITALISPAÑA` | ITALISPAÑA | Backup HyperV |

Si aparece un prefijo que no esté en la tabla, **no lo inventes**: usa el prefijo
tal cual y avisa de que falta la equivalencia para que se añada aquí.

## 5. Entregar el resultado

Devuelve el resumen **en el chat**. Por defecto **no** crees ni envíes ningún
correo — solo si se pide explícitamente.

El resumen lleva:

1. Periodo revisado y cuántos informes se han mirado (`N revisados, M con
   errores, K correctos`).
2. Por cada fallo: **cliente**, **job**, **máquina**, **hora** y el **texto del
   error**.
3. Si algún error ya salía en el parte del día anterior, márcalo como
   `Este error se repite` (compáralo buscando los envíos previos con
   `outlook_email_search(query = "Revisión Copias de seguridad", ...)`).
4. Si no hay ningún `[Failed]`, dilo en una línea: todas las copias correctas.

### Tabla lista para el parte

Añade al final la tabla HTML con el formato exacto del correo diario, para poder
pegarla o convertirla en borrador sin retocar:

| CLIENTE | SW BACKUP | ERROR | OBSERVACIONES |
|---|---|---|---|
| CITRUSLANDIA | VEEAM | *(texto íntegro del error)* | *(vacío, o "Este error se repite")* |

Una fila por cliente afectado. La columna OBSERVACIONES se deja vacía salvo que
el error sea recurrente.

### Si se pide preparar el correo

Solo entonces, con `outlook_create_draft`:

- **De**: `alozal@laberit.com` (la cuenta que ejecuta).
- **Para**: `smann@laberit.com`, `vmoreno@laberit.com`, `cfuertes@laberit.com`,
  `xpuig@laberit.com`, `dtorralba@laberit.com`.
- **Asunto**: `Revisión Copias de seguridad DD/MM/AAAA`.
- **Cuerpo**: `bodyType = "html"`, la tabla anterior con bordes
  (`border:1px solid #ababab`), fuente Calibri 11pt, cabeceras en negrita y
  centradas.
- **Sin firma**: la añade el servidor al enviar.

Deja el borrador y da el `webLink`. **Nunca lo envíes** sin que se pida.
