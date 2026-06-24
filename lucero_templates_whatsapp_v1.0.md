# Templates de WhatsApp — Bot Lucero (Bienvenida)
**Versión:** 1.0  
**Fecha:** Junio 2026  
**Categoría Meta:** MARKETING  
**Idioma:** es (Argentina)

---

## Notas de registro

- **Categoría:** MARKETING (no Utility). Son mensajes de relación, no confirmación de transacción.
- **Variable `{{1}}`** = nombre del donante (ej. "Sandro")
- **Variable `{{2}}`** = monto de la donación (ej. "25000")
- **Variable `{{3}}`** = descripción del medio de pago (ej. "tarjeta Visa Crédito")
- **YouTube preview:** se genera automáticamente en WhatsApp cuando la URL está sola en una línea del body. No usar header multimedia separado.
- **Sender:** el número de WhatsApp Business de ISF-Ar (el mismo que Maitena y Genaro).
- **Trigger Make:** Lucero día 0 — enviado unas horas después de la creación de la Recurring Donation en SF (para que el email bounce haya tenido tiempo de registrarse).

---

## Template 1 — `isf_bienvenida_dia0`

**Cuándo se usa:** Día 0 del episodio Lucero — primer contacto al nuevo donante, horas después de la suscripción.

**Objetivo:** calidez + confirmar datos + presentar ISF-Ar con un video cortito + abrir conversación.

---

### Header (opcional)
*(sin header — el video va en el body para generar la preview de YouTube)*

### Body

```
Hola {{1}} 👋

Soy Lucero, del equipo de Ingeniería Sin Fronteras Argentina. ¡Bienvenido/a a la familia!

Quería confirmarte que recibimos tu donación de ${{2}} mensuales a través de {{3}}. Tu apoyo nos permite llevar adelante proyectos que mejoran la vida de comunidades en situación de vulnerabilidad en todo el país. Gracias por sumarte 💙

Para que nos conozcas un poco más de cerca, te comparto este video cortito:
https://www.youtube.com/watch?v=cVMsURwWWQU

¿Llegó bien el email de bienvenida que te enviamos? Respondé este mensaje si tenés alguna duda o consulta, estoy acá para ayudarte.
```

**Caracteres del body:** ~460 (límite Meta: 1024)

### Footer (opcional)
```
ISF-Ar · socios@isf-argentina.org · 11 5624-8347
```

### Botones (opcionales — quick reply)
No recomendados para el primer template. Lucero abre conversación libre.

---

### Variables Make

| Variable | Campo SF / Make | Ejemplo |
|---|---|---|
| `{{1}}` | `Contact.FirstName` | Sandro |
| `{{2}}` | `npe03__Amount__c` (formateado sin decimales) | 25000 |
| `{{3}}` | Descripción del medio de pago* | tarjeta Visa Crédito |

*`{{3}}` requiere lógica Make para construir la string. Ejemplo: si `Payment_Method__c = 'Visa'` y `Card_Type__c = 'Credito'` → "tarjeta Visa Crédito". Si CBU → "débito bancario (CBU)".

---

## Template 2 — `isf_bienvenida_graduacion`

**Cuándo se usa:** Día ~30 del episodio — cuando el primer débito se procesó correctamente y el donante completa el primer mes. Make lo dispara cuando detecta que el cobro fue exitoso o cuando se cumple la fecha programada.

**Objetivo:** celebrar el primer mes, reforzar el impacto, cerrar el episodio Lucero con gratitud. Invitar a seguir en contacto.

---

### Body

```
Hola {{1}} 🎉

¡Ya pasó tu primer mes como donante de ISF-Ar! Gracias a vos y a toda la comunidad de donantes, seguimos trabajando para transformar realidades en comunidades que más lo necesitan.

Tu aporte de ${{2}} mensuales ya está haciendo la diferencia. Si querés ver en detalle a dónde va ese apoyo, te recomendamos seguir nuestro newsletter y nuestras redes.

Por cualquier consulta, siempre podés escribirnos a socios@isf-argentina.org o al 11 5624-8347.

¡Gracias por estar! 💙
```

**Caracteres del body:** ~450

### Footer (opcional)
```
ISF-Ar · www.isf-argentina.org
```

---

### Variables Make

| Variable | Campo SF / Make | Ejemplo |
|---|---|---|
| `{{1}}` | `Contact.FirstName` | Sandro |
| `{{2}}` | `npe03__Amount__c` (formateado sin decimales) | 25000 |

---

## Notas de implementación

### Orden de aprobación Meta
1. Registrar ambos templates en Meta Business Manager (via Twilio Content Template Builder o directo en Meta).
2. Categoría: **MARKETING** para ambos.
3. El template `isf_bienvenida_dia0` tardará 1–3 días en aprobarse.
4. Guardar el `ContentSid` de Twilio para cada template — se configura en Make.

### Configuración en Make (escenario Lucero Outbound)
- **isf_bienvenida_dia0:** disparado X horas después del trigger SF. Make usa el `ContentSid` del template + variables del donante.
- **isf_bienvenida_graduacion:** disparado cuando `Bienvenida_Paso__c` llega a "graduacion" o `Bienvenida_Estado__c` pasa a "finalizado".
- Después de enviar `isf_bienvenida_graduacion` → Make actualiza `Bienvenida_Estado__c = 'finalizado'` en SF.

### Variantes de `{{3}}` (medio de pago)
Para construir la string en Make según los campos SF:

| Condición SF | String para `{{3}}` |
|---|---|
| Tarjeta de crédito Visa | `tarjeta Visa Crédito` |
| Tarjeta de crédito Mastercard | `tarjeta Mastercard Crédito` |
| Tarjeta de débito | `tarjeta de débito` |
| CBU / transferencia | `débito bancario (CBU)` |
| No especificado | `tu medio de pago registrado` |
