# Templates de WhatsApp — Bot Lucero (Bienvenida)
**Versión:** 1.2  
**Fecha:** Junio 2026  
**Categoría Meta:** UTILITY (aprobados por Meta como Utility)  
**Idioma:** es_AR (Argentina)

---

## Notas de registro

- **Categoría:** UTILITY. Meta los aprobó como Utility porque el núcleo del mensaje es la confirmación de una transacción (la donación que se debita). Es más barato que Marketing (~$0.026 vs ~$0.06 en AR). ⚠️ Meta puede recategorizar a Marketing más adelante si considera el contenido promocional — si eso pasa, no es un error de configuración.
- **Lucero hace un solo envío proactivo:** el template de bienvenida (día 0). No hay graduación ni segundo toque. Si el donante responde, lo toma **Maitena** en la conversación (no hay inbound ni prompt separado de Lucero).
- **El template de bienvenida tiene dos variantes** según el resultado del chequeo de email en Salesforce (`EmailBouncedDate`): `_emailok` (no rebotó) y `_emailbounce` (rebotó). Make elige cuál enviar tras el sleep.
- **Header multimedia:** los templates llevan una imagen de header (miniatura del spot de ISF-Ar). El link de YouTube va igual en el body, pero como es un template NO genera preview nativo de YouTube (el preview solo aparece en mensajes de sesión); por eso se usa la imagen de header.
- **Sender:** el número de WhatsApp Business de ISF-Ar (el mismo que Maitena y Genaro).
- **Trigger Make:** Lucero día 0 — enviado unas horas después de la creación de la Recurring Donation en SF (para que el email bounce haya tenido tiempo de registrarse).

### Variables (Template 1, ambas variantes)
| Variable | Campo SF / Make | Ejemplo |
|---|---|---|
| `{{nombre}}` | `Contact.FirstName` | Sandro |
| `{{monto}}` | `npe03__Amount__c` (sin decimales) | 25000 |
| `{{medio_pago}}` | medio de pago (string construido en Make) | tarjeta Visa Crédito |
| `{{ultimos4}}` | últimos 4 dígitos de la tarjeta | 4321 |
| `{{email}}` | email registrado | sandro@gmail.com |

---

## Template 1A — `isf_bienvenida_dia0_emailok`
**ContentSid:** `HX2368ef6dfa68ed15e15aa7579c319171`

**Cuándo se usa:** Día 0 del episodio Lucero — email NO rebotado (`EmailBouncedDate` = null). Caso normal.

**Objetivo:** calidez + confirmar datos + cotejar el email registrado + presentar ISF-Ar con un video cortito + abrir conversación.

### Body

```
Hola {{nombre}} 👋

Soy Lucero, del equipo de Ingeniería Sin Fronteras Argentina. ¡Te damos la bienvenida a la comunidad!

Quería confirmarte que recibimos tu donación de ${{monto}} mensuales a través de tu {{medio_pago}} que finaliza en {{ultimos4}}. Tu apoyo nos permite llevar adelante proyectos que mejoran la vida de comunidades en situación de vulnerabilidad en todo el país. Gracias por sumarte 💙

Para que nos conozcas un poco más de cerca, te comparto este video cortito:
https://www.youtube.com/watch?v=cVMsURwWWQU

Te enviamos un email de bienvenida a {{email}}. ¿Es correcto? Cualquier duda o corrección, respondé este mensaje, estoy acá para ayudarte.
```

### Footer (opcional)
```
ISF-Ar · socios@isf-argentina.org · 11 5624-8347
```

**Sample:** `{{nombre}}=Sandro · {{monto}}=25000 · {{medio_pago}}=tarjeta Visa Crédito · {{ultimos4}}=4321 · {{email}}=sandro@gmail.com`

---

## Template 1B — `isf_bienvenida_dia0_emailbounce`
**ContentSid:** `HX50bfb54c44025cb64b6ae19b28ecbdc9`

**Cuándo se usa:** Día 0 del episodio Lucero — email rebotado (`EmailBouncedDate` ≠ null). Make detecta el bounce y envía esta variante.

**Objetivo:** igual que 1A, pero pide un email correcto en lugar de cotejar. Lucero, cuando el donante responde con el nuevo email, dispara `[ALERTA:email_rebotado]`.

### Body

```
Hola {{nombre}} 👋

Soy Lucero, del equipo de Ingeniería Sin Fronteras Argentina. ¡Te damos la bienvenida a la comunidad!

Quería confirmarte que recibimos tu donación de ${{monto}} mensuales a través de tu {{medio_pago}} que finaliza en {{ultimos4}}. Tu apoyo nos permite llevar adelante proyectos que mejoran la vida de comunidades en situación de vulnerabilidad en todo el país. Gracias por sumarte 💙

Para que nos conozcas un poco más de cerca, te comparto este video cortito:
https://www.youtube.com/watch?v=cVMsURwWWQU

Te quería comentar que intentamos enviarte el email de bienvenida a {{email}} pero nos rebotó. ¿Nos corregirías el correo así te mantenemos al tanto de todo? 🙏
```

### Footer (opcional)
```
ISF-Ar · socios@isf-argentina.org · 11 5624-8347
```

**Sample:** `{{nombre}}=Sandro · {{monto}}=25000 · {{medio_pago}}=tarjeta Visa Crédito · {{ultimos4}}=4321 · {{email}}=sandro@gmail.com`

---

## Notas de implementación

### Registro en Twilio Content Template Builder / Meta
1. Registrar las 2 variantes del template de bienvenida (1A y 1B) con **header de imagen** (miniatura del spot) + body de texto.
2. Categoría: **UTILITY** para ambas. **Language:** `es_AR`.
3. Aprobación Meta: 1–3 días.
4. Guardar el `ContentSid` (`HXxxxx…`) de cada template — se configuran en Make.

### Configuración en Make (escenario Lucero Outbound)
- **Día 0 (único envío proactivo):** tras el sleep, Make re-lee SF. Si `EmailBouncedDate` = null → envía `isf_bienvenida_dia0_emailok`. Si ≠ null → envía `isf_bienvenida_dia0_emailbounce`. Luego setea `WhatsApp_Estado__c = 'bienvenida'` y `WhatsApp_Fecha_Bienvenida__c = now`.
- **Inbound:** si el donante responde, lo toma **Maitena** (escenario 1 inbound, path donante activo). No hay prompt ni escenario de inbound de Lucero.
- No hay segundo envío programado (graduación eliminada).

### Construcción de `{{medio_pago}}` y `{{ultimos4}}`
`{{medio_pago}}` viene del campo SF `medio_de_pago_para_email__c` (ya compone "tarjeta Visa Crédito", o "CBU" si es débito bancario). El texto del body dice "tu {{medio_pago}} que finaliza en {{ultimos4}}".

| Condición SF | `{{medio_pago}}` | `{{ultimos4}}` |
|---|---|---|
| Tarjeta de crédito Visa | `tarjeta Visa Crédito` | últimos 4 de la tarjeta |
| Tarjeta de crédito Mastercard | `tarjeta Mastercard Crédito` | últimos 4 de la tarjeta |
| Tarjeta de débito | `tarjeta de débito` | últimos 4 de la tarjeta |

> ⚠️ **Caso CBU pendiente:** "que finaliza en {{ultimos4}}" no aplica a débito bancario por CBU. Si hay donantes por CBU habría que registrar una variante sin tarjeta. Por ahora se asume tarjeta.
