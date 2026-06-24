# Templates de WhatsApp — Bot Lucero (Bienvenida)
**Versión:** 1.2  
**Fecha:** Junio 2026  
**Categoría Meta:** MARKETING  
**Idioma:** es_AR (Argentina)

---

## Notas de registro

- **Categoría:** MARKETING para todos. Incluyen video institucional, bienvenida e invitación a redes → Meta los clasifica como Marketing. No registrar como Utility (los recategorizan).
- **Lucero hace un solo envío proactivo:** el template de bienvenida (día 0). No hay graduación ni segundo toque. Si el donante responde, Lucero sigue conversando normalmente (inbound) y maneja derivaciones/alertas.
- **El template de bienvenida tiene dos variantes** según el resultado del chequeo de email en Salesforce (`EmailBouncedDate`): `_emailok` (no rebotó) y `_emailbounce` (rebotó). Make elige cuál enviar tras el sleep.
- **YouTube preview:** se genera automáticamente en WhatsApp cuando la URL está sola en una línea del body. No usar header multimedia separado.
- **Sender:** el número de WhatsApp Business de ISF-Ar (el mismo que Maitena y Genaro).
- **Trigger Make:** Lucero día 0 — enviado unas horas después de la creación de la Recurring Donation en SF (para que el email bounce haya tenido tiempo de registrarse).

### Variables (Template 1, ambas variantes)
| Variable | Campo SF / Make | Ejemplo |
|---|---|---|
| `{{1}}` | `Contact.FirstName` | Sandro |
| `{{2}}` | `npe03__Amount__c` (sin decimales) | 25000 |
| `{{3}}` | medio de pago (string construido en Make) | tarjeta Visa Crédito |
| `{{4}}` | últimos 4 dígitos de la tarjeta | 4321 |
| `{{5}}` | email registrado | sandro@gmail.com |

---

## Template 1A — `isf_bienvenida_dia0_emailok`

**Cuándo se usa:** Día 0 del episodio Lucero — email NO rebotado (`EmailBouncedDate` = null). Caso normal.

**Objetivo:** calidez + confirmar datos + cotejar el email registrado + presentar ISF-Ar con un video cortito + abrir conversación.

### Body

```
Hola {{1}} 👋

Soy Lucero, del equipo de Ingeniería Sin Fronteras Argentina. ¡Te damos la bienvenida a la comunidad!

Quería confirmarte que recibimos tu donación de ${{2}} mensuales a través de tu {{3}} que finaliza en {{4}}. Tu apoyo nos permite llevar adelante proyectos que mejoran la vida de comunidades en situación de vulnerabilidad en todo el país. Gracias por sumarte 💙

Para que nos conozcas un poco más de cerca, te comparto este video cortito:
https://www.youtube.com/watch?v=cVMsURwWWQU

Te enviamos un email de bienvenida a {{5}}. ¿Es correcto? Cualquier duda o corrección, respondé este mensaje, estoy acá para ayudarte.
```

### Footer (opcional)
```
ISF-Ar · socios@isf-argentina.org · 11 5624-8347
```

**Sample:** `{{1}}=Sandro · {{2}}=25000 · {{3}}=tarjeta Visa Crédito · {{4}}=4321 · {{5}}=sandro@gmail.com`

---

## Template 1B — `isf_bienvenida_dia0_emailbounce`

**Cuándo se usa:** Día 0 del episodio Lucero — email rebotado (`EmailBouncedDate` ≠ null). Make detecta el bounce y envía esta variante.

**Objetivo:** igual que 1A, pero pide un email correcto en lugar de cotejar. Lucero, cuando el donante responde con el nuevo email, dispara `[ALERTA:email_rebotado]`.

### Body

```
Hola {{1}} 👋

Soy Lucero, del equipo de Ingeniería Sin Fronteras Argentina. ¡Te damos la bienvenida a la comunidad!

Quería confirmarte que recibimos tu donación de ${{2}} mensuales a través de tu {{3}} que finaliza en {{4}}. Tu apoyo nos permite llevar adelante proyectos que mejoran la vida de comunidades en situación de vulnerabilidad en todo el país. Gracias por sumarte 💙

Para que nos conozcas un poco más de cerca, te comparto este video cortito:
https://www.youtube.com/watch?v=cVMsURwWWQU

Te quería comentar que intentamos enviarte el email de bienvenida a {{5}} pero nos rebotó. ¿Nos corregirías el correo así te mantenemos al tanto de todo? 🙏
```

### Footer (opcional)
```
ISF-Ar · socios@isf-argentina.org · 11 5624-8347
```

**Sample:** `{{1}}=Sandro · {{2}}=25000 · {{3}}=tarjeta Visa Crédito · {{4}}=4321 · {{5}}=sandro@gmail.com`

---

## Notas de implementación

### Registro en Twilio Content Template Builder / Meta
1. Registrar las 2 variantes del template de bienvenida (1A y 1B). **Content type = Text** (sin media header — el video se previsualiza solo con la URL en el body).
2. Categoría: **MARKETING** para ambas. **Language:** `es_AR`.
3. Aprobación Meta: 1–3 días.
4. Guardar el `ContentSid` (`HXxxxx…`) de cada template — se configuran en Make.

### Configuración en Make (escenario Lucero Outbound)
- **Día 0 (único envío proactivo):** tras el sleep, Make re-lee SF. Si `EmailBouncedDate` = null → envía `isf_bienvenida_dia0_emailok`. Si ≠ null → envía `isf_bienvenida_dia0_emailbounce`. Luego setea `Bienvenida_Estado__c = 'bienvenida'`.
- **Inbound:** si el donante responde, Make llama a Claude con el prompt de Lucero, procesa tags y actualiza `Bienvenida_Estado__c` (`en_conversacion`, `derivado_humano`, `finalizado`).
- No hay segundo envío programado (graduación eliminada).

### Construcción de `{{3}}` y `{{4}}` (medio de pago)
`{{3}}` es la descripción y `{{4}}` los últimos 4 dígitos. El texto del body dice "tu {{3}} que finaliza en {{4}}".

| Condición SF | `{{3}}` | `{{4}}` |
|---|---|---|
| Tarjeta de crédito Visa | `tarjeta Visa Crédito` | últimos 4 de la tarjeta |
| Tarjeta de crédito Mastercard | `tarjeta Mastercard Crédito` | últimos 4 de la tarjeta |
| Tarjeta de débito | `tarjeta de débito` | últimos 4 de la tarjeta |

> ⚠️ **Caso CBU pendiente:** "que finaliza en {{4}}" no aplica a débito bancario por CBU. Si hay donantes por CBU hay que registrar una 4ta variante sin tarjeta (texto: "a través de débito bancario (CBU)"). Por ahora se asume tarjeta.
