# ISF Bot — Documentación de Escenarios Make
**Versión:** Junio 2026  
**Contexto:** Todos los escenarios se conectan a Salesforce NPC, Twilio WhatsApp Business, y la API de Anthropic (Claude Sonnet 4.6).

---

## NOTAS TÉCNICAS CRÍTICAS (LEER ANTES DE MODIFICAR)

- **`formatDate(now; "X")`** — Las comillas alrededor de `"X"` NUNCA se escapan como `\"X\"`. Deben ser comillas literales. Make las interpreta directamente; escaparlas rompe la expresión.
- **`escapeJSON(variable)`** — Obligatorio al inyectar cualquier texto dinámico (historial, ISF_INFO, mensaje del donante) dentro de strings JSON. Sin esto, comillas o saltos de línea en el texto rompen el JSON.
- **`newline`** — Variable especial de Make para saltos de línea (equivalente a `\n`).
- **Concatenación en Make:** usar `+` (no `&`).
- **Módulo Anthropic nativo:** no soporta `tools`. Usar HTTP module genérico.
- **Número WhatsApp producción:** `whatsapp:+19068268551`
- **Campo `Estado_ltima_op_Cerrada__c`:** sin "u" — así está en SF (error tipográfico en el campo original).
- **`Whatsapp_Liberar_en__c`:** capitalización exacta del campo en SF (lowercase 'p' y 'e') aunque en los SOQLs se escribe diferente.

---

## ESCENARIO 1 — ISF Inbound (Maitena + Genaro)

**Propósito:** Procesar mensajes entrantes de WhatsApp de donantes y ex-donantes, enrutar a Maitena (activos) o Genaro (ex-donantes), y manejar la derivación humana.

### Módulos

| # | Tipo | Descripción |
|---|---|---|
| 1 | Webhooks — Custom Webhook | Recibe el POST de Twilio con el mensaje |
| 16 | Webhooks — Webhook Response | Respuesta vacía inmediata a Twilio (evita timeout) |
| 29 | HTTP | Typing indicator a Twilio (POST a Conversations API) |
| 4 | Salesforce — Make an API Call | SOQL: busca donación activa por teléfono |
| 5 | Router | Divide: donante activo (totalSize > 0) vs no encontrado |
| [sub-router] | Router | Divide: `derivado_humano` vs flujo normal |
| 34 | Salesforce — Make an API Call | GET historial episodio (solo path derivado_humano) |
| 35 | Salesforce — Update a Record | Append mensaje donante al historial (derivado_humano) |
| 8 | HTTP — Anthropic API | Llama a Claude con system prompt de Maitena |
| 30 | Tools — Sleep | 12 segundos (simula escritura humana) |
| 17 | Twilio | Envía respuesta al donante |
| 36 | Salesforce — Update a Record | Actualiza historial y estado en SF |
| 15 | Salesforce — Create a Record | Crea tarea si hay alerta [ALERTA:...] |
| 23 | Salesforce — Search Records | SOQL: busca donación cerrada (ex-donante) |
| 24 | Router | Divide: ex-donante vs número desconocido |
| 27 | HTTP — Anthropic API | Llama a Claude con system prompt de Genaro |
| [Twilio] | Twilio | Envía respuesta Genaro |
| [SF Update] | Salesforce — Update | Actualiza historial Genaro |
| 32 | Twilio | Mensaje genérico para número desconocido |

### SOQL donaciones activas (módulo 4)
```sql
SELECT Id, ISFAR_Id_18_digitos__c,
       npe03__Contact__r.FirstName, npe03__Contact__r.LastName,
       npe03__Contact__r.MobilePhone, npe03__Contact__r.Email,
       npe03__Amount__c, Medio_de_pago__c,
       ISFAR_ultimos_4_digitos_tarjeta_cbu__c,
       npe03__Last_Payment_Date__c, npe03__Date_Established__c,
       Estado_Donante__c, WhatsApp_Estado__c, WhatsApp_Intentos__c,
       WhatsApp_Historial__c, Episodios_Rechazo__c
FROM npe03__Recurring_Donation__c
WHERE npe03__Contact__r.MobilePhone = '{{replace(1.From; "whatsapp:+54"; "")}}'
  AND npe03__Open_Ended_Status__c = 'Open'
LIMIT 1
```
**Decisión:** Se busca por teléfono extrayendo el prefijo `whatsapp:+54` del campo `From` de Twilio. `LIMIT 1` porque un donante puede tener múltiples donaciones — tomamos la más relevante.

### Lógica de derivado_humano (sub-router antes del módulo 8)
**Decisión:** Cuando `WhatsApp_Estado__c = 'derivado_humano'`, el bot NO responde. Solo se graba el mensaje del donante en el historial para que el operador humano lo vea en la plataforma. Esto evita que Maitena interrumpa conversaciones que el equipo está manejando.

### Body del módulo Anthropic (módulo 8)
Ver archivo `make_anthropic_body_v3.json`. Puntos clave:
- `temperature: 0.9` — Se eligió para que las respuestas sean variadas y naturales, no robóticas.
- `max_tokens: 500` — Suficiente para mensajes WhatsApp cortos. Más tokens = más costo sin beneficio.
- `escapeJSON` en ISF_INFO, historial y `1.Body` — previene errores JSON cuando el contenido tiene comillas o saltos de línea.

### Tags que procesa el módulo 36 (SF Update)
El response de Claude puede contener:
- `[ESTADO:cerrado_positivo]` → actualiza `WhatsApp_Estado__c`
- `[ESTADO:cerrado_negativo]` → ídem
- `[ESTADO:derivado_humano]` → ídem
- `[ALERTA:emails]`, `[ALERTA:voluntario]`, `[ALERTA:general]` → crea tarea en SF

---

## ESCENARIO 2 — ISF Outbound Maitena (Recupero Donantes)

**Propósito:** Contactar proactivamente donantes cuyas donaciones fueron rechazadas. Se ejecuta diariamente (cron ~9-10am).

### Módulos

| # | Tipo | Descripción |
|---|---|---|
| 1 | Salesforce — Make an API Call | SOQL producción |
| 2 | Iterator | Itera sobre cada registro |
| F4 | Filter | UltimoEnvio < 3 días O no existe |
| 3 | Tools — Set Variable | Calcula `effective_intentos` |
| F3 | Filter | `effective_intentos < 3` |
| 6 | HTTP — Twilio | Envía template con ContentSid dinámico |
| 7 | Salesforce — Update a Record | Actualiza campos post-envío |

### SOQL Outbound (módulo 1)
```sql
SELECT Id, ISFAR_Id_18_digitos__c,
       npe03__Contact__r.FirstName, npe03__Contact__r.MobilePhone,
       npe03__Amount__c, Medio_de_pago__c,
       Estado_Donante__c, npe03__Last_Payment_Date__c,
       Reactivacion__c, WhatsApp_Estado__c, WhatsApp_Intentos__c,
       Estado_ltima_op_Cerrada__c,
       WhatsApp_UltimoEnvio__c, WhatsApp_FechaInicioEpisodio__c,
       WhatsApp_Liberar_En__c, WhatsApp_Historial__c,
       WhatsApp_Historial_Archivo__c
FROM npe03__Recurring_Donation__c
WHERE
  Estado_Donante__c = 'Rechazada'
  AND npe03__Contact__r.Whatsapp_Error__c = false
  AND npe03__Open_Ended_Status__c = 'Open'
  AND No_llamar_callcenter__c = false
  AND WhatsApp_Estado__c != 'cerrado_negativo'
  AND WhatsApp_Estado__c != 'derivado_humano'
  AND WhatsApp_Estado__c != 'cerrado_positivo'
  AND WhatsApp_Estado__c != 'en_conversacion'
  AND npe03__Contact__r.MobilePhone != null
  AND (Reactivacion__c = null OR Reactivacion__c != 'Reactivado')
  AND (WhatsApp_Liberar_En__c = null OR WhatsApp_Liberar_En__c <= TODAY)
  AND npe03__Date_Established__c < THIS_MONTH
  {{if(formatDate(now; "D") >= 12; ""; "AND (npe03__Last_Payment_Date__c = null OR npe03__Last_Payment_Date__c < " + formatDate(addMonths(setDate(now; 1); -2); "YYYY-MM-DD") + ")")}}
  AND (WhatsApp_UltimoEnvio__c = null OR WhatsApp_UltimoEnvio__c < {{formatDate(addDays(now; -4); "YYYY-MM-DD")}}T00:00:00.000Z)
  AND (
    WhatsApp_FechaInicioEpisodio__c = null
    OR (
      WhatsApp_Intentos__c < 3
      AND WhatsApp_FechaInicioEpisodio__c >= {{formatDate(setDate(now; 1); "YYYY-MM-DD")}}
    )
    OR (
      WhatsApp_FechaInicioEpisodio__c < {{formatDate(setDate(now; 1); "YYYY-MM-DD")}}
      AND WhatsApp_FechaInicioEpisodio__c < {{formatDate(addDays(now; -12); "YYYY-MM-DD")}}
    )
  )
```

**Decisión historial en outbound:** `WhatsApp_Historial__c` y `WhatsApp_Historial_Archivo__c` se traen para que el SF Update al final del escenario pueda hacer append del template enviado, igual que hace `wtr`. Así la plataforma muestra el historial completo incluyendo los intentos del outbound.

**Decisión Whatsapp_Error__c:** campo booleano en Contact. Se filtra `= false` para excluir donantes cuyo número no puede recibir mensajes de WhatsApp (error 63024 u otros). Se resetea automáticamente a false en SF cuando el operador modifica el teléfono del contacto.

**Decisiones clave del SOQL:**

- **`!= 'en_conversacion'`** — Excluido porque si el bot está en conversación activa, no tiene sentido interrumpir con un template outbound.
- **`Reactivacion__c != 'Reactivado'`** — Evita contactar a donantes que ya actualizaron sus datos aunque el rechazo no se haya resuelto en SF aún.
- **`WhatsApp_Liberar_En__c <= TODAY`** — Cuando el operador "libera un episodio" setea esta fecha para pausar el outbound. El SOQL solo incluye registros cuya fecha ya pasó.
- **`Date_Established__c < THIS_MONTH`** — Excluye donaciones dadas de alta el mes en curso; aún no tuvieron un ciclo de cobro completo.
- **`WhatsApp_UltimoEnvio__c < 4 días`** — Previene envíos duplicados. Movido al SOQL desde el filtro F4 para reducir iteraciones.
- **Condición `effective_intentos` en SOQL** — Pre-filtra registros que no pasarían F3, reduciendo iteraciones:
  - `FechaInicioEpisodio = null` → episodio nuevo, siempre pasa
  - `Intentos < 3 AND FechaInicioEpisodio >= 1ro del mes` → episodio en curso este mes con intentos disponibles
  - `FechaInicio < 1ro del mes actual AND < 12 días atrás` → mes diferente + tiempo suficiente, se trata como nuevo episodio
  - Un episodio de un mes anterior con `Intentos < 3` NO entra por la segunda rama; solo entra si además cumple la tercera (12 días), en cuyo caso se resetea el contador.
- **Condición `Last_Payment_Date`** — Si hoy es día ≥ 12: entran todos. Si es día < 12: excluye cobros del mes pasado y hace 2 meses (solo procesa cobros de hace 3+ meses o sin fecha). Umbral de 2 meses: `addMonths(setDate(now; 1); -2)` = primer día de hace 2 meses.

### effective_intentos (módulo 3 — Set Variable)
```
{{if(
  2.WhatsApp_FechaInicioEpisodio__c = null
  | (formatDate(2.WhatsApp_FechaInicioEpisodio__c; "YYYY-MM") != formatDate(now; "YYYY-MM")
     & (formatDate(now; "X") - formatDate(2.WhatsApp_FechaInicioEpisodio__c; "X")) / 86400 > 12)
; 0; 2.WhatsApp_Intentos__c)}}
```
**Explicación:** Si el episodio empezó en un mes diferente al actual Y hace más de 12 días → resetear a 0 (nuevo episodio). Esto permite que el bot retome contacto con donantes que no respondieron el mes anterior sin acumular intentos indefinidamente. El umbral de 12 días evita resetear demasiado rápido al cruzar el mes.

### Templates Twilio outbound
| effective_intentos | Content SID | Texto |
|---|---|---|
| 0 (primer contacto) | HXf642b2f6992989c36eb27e10b3081ebd | Mención del monto y medio de pago |
| 1 (segundo intento) | HX478333e9565b698a96620f39a1dbf19d | Nombre + monto + link formulario |
| 2 (tercer intento) | HXc5273876072a46f3ea8e65a9a9e87dd6 | Último intento — más directo |

**Decisión:** 3 templates distintos para no sonar repetitivo. El tercero es más directo porque es el último antes de cerrar el episodio.

---

## ESCENARIO 2B — ISF Outbound Genaro (Reactivación)

**Propósito:** Contactar proactivamente a ex-donantes (donación cerrada) para invitarlos a retomar su apoyo. Es el hermano outbound de Maitena, pero apunta a donantes que ya churnearon, no a cobros rechazados. Se ejecuta diariamente (cron ~9-10am).

**Categoría WhatsApp:** Los templates se registran en Meta como **Marketing**, no utility. El mensaje pide reactivar un aporte (re-engagement), que es marketing por definición de Meta. Forzarlo como utility arriesga recategorización y baja el quality rating del número. La diferencia de costo (≈$0,06 marketing vs ≈$0,026 utility en AR) no compensa quemar el canal.

### Módulos (clonar de Escenario 2)

| # | Tipo | Descripción |
|---|---|---|
| 1 | Salesforce — Make an API Call | SOQL ex-donantes elegibles |
| AGG | Tools — Array Aggregator | Dedup por donante (1 donación por contacto) |
| 2 | Iterator | Itera sobre cada grupo dedupeado |
| F-estado | Filter | Filtra episodios de Genaro recientes con estado vivo |
| 3 | Tools — Set Variable | Calcula `effective_intentos` (reset a 180 días) |
| F3 | Filter | `effective_intentos < 2` (solo 2 templates) |
| 6 | HTTP — Twilio | Envía template con ContentSid según intento |
| 7 | Salesforce — Update a Record | Actualiza campos post-envío |

### Cadencia

- **Template 1** → día 0 (inicio de episodio)
- **Template 2** → día +7
- Sin respuesta tras el segundo → cerrar episodio. **Cooldown de ~6 meses** antes de re-entrada (vía reset de `effective_intentos` o `WhatsApp_Liberar_En__c`).
- Dos toques con una semana de aire: suave y respetuoso para alguien que ya se fue. Más que eso quema la relación.

### Campos SF

- `WhatsApp_Estado__c = 'reactivacion'` — marca el episodio outbound de Genaro (lo distingue del flujo de rechazos de Maitena).
- `WhatsApp_Intentos__c` — `0` → template 1, `1` → template 2.
- `WhatsApp_FechaInicioEpisodio__c` — base para calcular los +7 días y el cooldown de 6 meses.
- `WhatsApp_UltimoEnvio__c` — anti-duplicado (umbral de 8 días, no 4 como rechazos).

### SOQL Outbound (módulo 1)
```sql
SELECT Id, ISFAR_Id_18_digitos__c,
       npe03__Contact__r.FirstName, npe03__Contact__r.LastName,
       npe03__Contact__r.MobilePhone, npe03__Contact__c,
       npe03__Amount__c, Medio_de_pago__c,
       npe03__Date_Established__c, Fecha_de_baja__c,
       Estado_Donante__c, Estado_ltima_op_Cerrada__c,
       WhatsApp_Estado__c, WhatsApp_Intentos__c,
       WhatsApp_UltimoEnvio__c, WhatsApp_FechaInicioEpisodio__c,
       WhatsApp_Liberar_En__c, WhatsApp_Historial__c,
       WhatsApp_Historial_Archivo__c
FROM npe03__Recurring_Donation__c
WHERE
  Estado_Donante__c = 'Ex-Donante'
  AND npe03__Contact__r.Whatsapp_Error__c = false
  AND npe03__Contact__r.MobilePhone != null
  AND WhatsApp_Estado__c != 'cerrado_negativo'
  AND Fecha_de_baja__c < LAST_N_MONTHS:6
  AND (WhatsApp_Liberar_En__c = null OR WhatsApp_Liberar_En__c <= TODAY)
  AND (WhatsApp_UltimoEnvio__c = null OR WhatsApp_UltimoEnvio__c < {{formatDate(addDays(now; -8); "YYYY-MM-DD")}}T00:00:00.000Z)
  AND (
    WhatsApp_FechaInicioEpisodio__c = null
    OR WhatsApp_Intentos__c < 2
    OR WhatsApp_FechaInicioEpisodio__c < {{formatDate(addDays(now; -180); "YYYY-MM-DD")}}
  )
ORDER BY npe03__Contact__c, Fecha_de_baja__c DESC NULLS LAST
LIMIT 50
```

**Diferencias clave respecto al SOQL de Maitena:**
- **`Estado_Donante__c = 'Ex-Donante'`** — apunta a ex-donantes (no a `'Rechazada'`). Reemplaza a `npe03__Open_Ended_Status__c = 'Closed'`, que era redundante.
- **Sin `No_llamar_callcenter__c`** — ese filtro es de operaciones de cobro, no aplica a reactivación.
- **`WhatsApp_Estado__c != 'cerrado_negativo'` es el ÚNICO filtro de estado en el SOQL** — un "no" explícito de un ex-donante es doble churn (se fue y rechazó el reintento) → **permanente, no se vuelve a contactar nunca**, sin importar la antigüedad del episodio. El resto de estados (`derivado_humano`, `en_conversacion`) NO se filtran acá porque dependen de comparar dos fechas (ver filtro de Make abajo). `cerrado_positivo` no se filtra porque `Estado_Donante = 'Ex-Donante'` ya lo excluye (si reactivó, dejó de ser ex-donante).
- **`Fecha_de_baja__c < LAST_N_MONTHS:6`** — no contactar a quien se dio de baja hace menos de 6 meses. Mismo criterio que el cooldown.
- **`WhatsApp_UltimoEnvio__c < hace 8 días`** — anti-duplicado. 8 días (no 4 como Maitena) para dar holgura a la cadencia de +7.
- **`WhatsApp_Intentos__c < 2`** — solo 2 templates (Maitena usa 3).
- **Reset a 180 días** — la tercera rama del `OR` reabre el episodio recién a los 6 meses.
- **Dedup por donante** — `ORDER BY npe03__Contact__c, Fecha_de_baja__c DESC` + Array Aggregator (ver abajo) para quedarse con una sola donación por donante: la de baja más reciente.
- **Sin condición de `Last_Payment_Date`** — no aplica el ciclo de cobro mensual de Maitena.

### Dedup por donante (módulo AGG — Array Aggregator)
Un donante puede tener varias donaciones recurrentes cerradas. Queremos procesar solo la de **baja más reciente**.

- SOQL no puede traer "una fila por grupo quedándose con la más nueva" (`GROUP BY` solo da agregados). Se resuelve en Make.
- El `ORDER BY npe03__Contact__c, Fecha_de_baja__c DESC NULLS LAST` deja, dentro de cada donante, la baja más reciente primera.
- **Array Aggregator** con **Group by = `npe03__Contact__c`** agrupa por donante. El primer elemento de cada grupo es la baja más reciente.
- En los módulos siguientes se referencia `{{array[1].campo}}` (Make indexa desde 1).

### Filtro de estado (F-estado — entre Iterator y Set Variable)
SOQL no puede comparar `WhatsApp_FechaInicioEpisodio__c` contra `Fecha_de_baja__c` (dos campos de fecha). Este filtro distingue de qué "era" es el episodio:
- **`inicio <= baja`** → episodio viejo de Maitena (ocurrió antes/durante la baja). Su estado no aplica a reactivación → **dejar pasar**.
- **`inicio > baja`** → episodio de Genaro (post-baja). Si es reciente (< 180 días) y está `derivado_humano` o `en_conversacion` → todavía vivo o en cooldown → **bloquear**. Si superó los 180 días → stale → reabrir.

**El bundle PASA si CUALQUIERA de estas 4 es verdadera (OR):**

| # | Campo | Operador | Valor |
|---|---|---|---|
| 1 | `WhatsApp_FechaInicioEpisodio__c` | Does not exist | *(vacío)* |
| 2 | `WhatsApp_Estado__c` | Does not equal `derivado_humano` **AND** Does not equal `en_conversacion` | *(misma fila, dos condiciones AND)* |
| 3 | `WhatsApp_FechaInicioEpisodio__c` | Less than or equal to | `{{2.Fecha_de_baja__c}}` |
| 4 | `WhatsApp_FechaInicioEpisodio__c` | Less than | `{{formatDate(addDays(now; -180); "YYYY-MM-DD")}}` |

Si ninguna se cumple, el registro se descarta (caso a bloquear: episodio de Genaro + reciente + estado vivo).

### effective_intentos (módulo 3 — Set Variable)
```
{{if(
  2.WhatsApp_FechaInicioEpisodio__c = null
  | (formatDate(now; "X") - formatDate(2.WhatsApp_FechaInicioEpisodio__c; "X")) / 86400 > 180
; 0; 2.WhatsApp_Intentos__c)}}
```
**Explicación:** Si nunca hubo episodio, o si el último arrancó hace más de 180 días → resetear a 0 (nuevo intento de reactivación). En cualquier otro caso, conservar el contador. Simplifica la lógica de Maitena: como no hay ciclo mensual, alcanza con el umbral de 180 días.

### Templates Twilio outbound (categoría Marketing)
| effective_intentos | Día | Texto |
|---|---|---|
| 0 (primer contacto) | 0 | Reconexión cálida — saludo + invitación a charlar |
| 1 (segundo intento) | +7 | Impacto concreto + invitación a retomar |

**Template 1 — Reconexión:**
> Hola {{nombre}}, soy Genaro, del equipo de **Ingeniería Sin Fronteras Argentina**. Nos acompañaste un tiempo con tu aporte y quería saludarte y contarte en qué andamos. Tenés un minuto para que te cuente?

**Template 2 — Segundo contacto:**
> Hola {{nombre}}, te escribo desde **Ingeniería Sin Fronteras Argentina**. Gracias a personas como vos llevamos agua y obras a comunidades que de otra forma no las hubieran tenido, y queremos seguir haciéndolo. Te gustaría volver a acompañarnos? Significaría mucho para nosotros.

**Decisión de diseño:**
- Los templates son cortos, sin tono promocional ni emojis, e invitan a responder. La conversión real no la hace el template sino **Genaro en la ventana de 24h** que se abre cuando el donante responde (escenario 1 inbound, path ex-donante).
- **Variable nombrada `{{nombre}}`** (no `{{1}}`) = `FirstName`. En el módulo Twilio de Make, `ContentVariables` = `{"nombre":"{{2.npe03__Contact__r.FirstName}}"}`.
- **`Ingeniería Sin Fronteras Argentina` en negrita** — en el body del template se escribe entre asteriscos (`*...*`); WhatsApp lo renderiza en bold.
- **Saludo con coma, sin `!`** — `Hola {{nombre}},`. En el outbound en frío un `!` suena demasiado efusivo/vendedor. El `!` de admiración de Genaro se reserva para cuando ya hay reciprocidad (el donante respondió).

---

## ESCENARIO 3 — ISF_INFO Mensual (Firecrawl)

**Propósito:** Actualizar la Custom Variable de Make `var.organization.info_ISF` con información fresca del sitio de ISF Argentina: proyectos activos y monto mínimo de donación.

**Se ejecuta:** Manualmente o con Schedule el día 1 de cada mes.

### Módulos

| # | Tipo | Descripción |
|---|---|---|
| 8 | HTTP — Firecrawl | Map de `isf-argentina.org/proyectos` |
| 11 | Iterator | Itera sobre URLs encontradas |
| 9 | HTTP — Firecrawl | Scrape de cada URL de proyecto |
| [Filter] | Filter | Solo proyectos "en proceso" |
| 13 | Tools — Text Aggregator | Agrega texto de proyectos |
| 14 | HTTP — Firecrawl | Scrape de `/formularios/donar` (monto mínimo) |
| 15 | Tools — Set Variable | `escapeJSON` del contenido scrapeado |
| 16 | HTTP — Anthropic API | Formatea el contenido en bloques ISF_INFO |
| 28 | Make — Update Custom Variable | Actualiza `info_ISF` |
| 30 | Gmail | Envía ISF_INFO actualizado al equipo |

**Decisión:** Se usa `escapeJSON` en los textos scrapeados antes de inyectarlos en el body de Anthropic porque el markdown puede contener comillas y caracteres especiales.

**Decisión:** La variable `info_ISF` se guarda en Make Custom Variables (no en SF) porque no necesita persistencia por donante y se comparte entre todos los escenarios. Se referencia como `{{var.organization.info_ISF}}` en los prompts.

---

## ESCENARIOS 4-11 — Plataforma de Gestión Humana

**Propósito:** 8 webhooks que alimentan la plataforma web (GitHub Pages) para que el equipo gestione conversaciones derivadas a humano.

### SOQL del webhook `wl` (cargar conversaciones)
```sql
SELECT Id, ISFAR_Id_18_digitos__c,
       npe03__Contact__r.FirstName, npe03__Contact__r.LastName,
       npe03__Contact__r.MobilePhone,
       npe03__Contact__r.Whatsapp_error__c,
       npe03__Amount__c, Medio_de_pago__c,
       WhatsApp_Estado__c, WhatsApp_Intentos__c,
       WhatsApp_UltimoEnvio__c, WhatsApp_Liberar_En__c,
       WhatsApp_Historial_Archivo__c,
       Estado_Donante__c,
       Fecha_modificacion_Nro_Tarjeta_o_CBU__c,
       WhatsApp_FechaInicioEpisodio__c,
       npe03__Date_Established__c,
       npe03__Open_Ended_Status__c
FROM npe03__Recurring_Donation__c
WHERE (
    npe03__Open_Ended_Status__c = 'Open'
    OR Estado_Donante__c = 'Ex-Donante'
  )
  AND (
    WhatsApp_Estado__c IN ('derivado_humano','cerrado_negativo','cerrado_positivo','en_conversacion','primer_envio','reactivacion')
    OR (WhatsApp_Estado__c = null AND WhatsApp_Liberar_En__c >= TODAY)
  )
ORDER BY WhatsApp_UltimoEnvio__c DESC NULLS LAST
```
**Decisión campos de recupero:** `Estado_Donante__c`, `Fecha_modificacion_Nro_Tarjeta_o_CBU__c` y `WhatsApp_FechaInicioEpisodio__c` se agregaron para que la plataforma pueda mostrar el badge "Recuperado" en positivos donde el donante ya actualizó sus datos de pago (Estado_Donante = Normal, o fecha de modificación de tarjeta/CBU posterior al inicio del episodio).

**Decisión:** Incluye `en_conversacion` para monitoreo (ver qué hace el bot). Incluye `null + Liberar_En >= TODAY` para mostrar episodios "programados" (pausados hasta una fecha futura). `ORDER BY UltimoEnvio DESC` para mostrar la actividad más reciente primero.

**Decisión switch Rechazos/Reactivación (index.html):** El SOQL trae ambos universos en una sola query — donaciones activas (`Open`, flujo Maitena/rechazos) y ex-donantes (`Estado_Donante = 'Ex-Donante'`, flujo Genaro/reactivación). El frontend tiene un switch en la topbar (`viewMode`) que discrimina por **`npe03__Open_Ended_Status__c`**: `Open` → Rechazos, `Closed` → Reactivación. Se eligió `Open_Ended_Status` como discriminador (no `Estado_Donante`) porque es binario y semánticamente confiable. La pestaña "Iniciados" (antes "Outbound") muestra `primer_envio` en modo Rechazos y `reactivacion` en modo Reactivación; por eso ambos estados están en el `IN`. El buscador de conversaciones también filtra por tags (Normal, Recuperado, Sin entrega).

---

### Webhook `ws` — Enviar Mensaje

**Payload recibido:**
```json
{
  "donationId": "...",
  "phone": "...",
  "message": "...",
  "operatorName": "...",
  "updatedHistorial": "..."
}
```

**Módulos:**
1. Webhook
2. Twilio — Create a Message (Channel, `whatsapp:+19068268551` → `whatsapp:+54{{1.phone}}`)
3. SF GET `WhatsApp_Historial__c` del episodio actual
4. Router: `{{2.errorCode}} = 63016` (ventana 24hs vencida) → Webhook Response `{"success": false, "error": "window_expired"}` / OK → SF Update + Response `{"success": true}`
5. SF Update: `WhatsApp_Historial_Archivo__c` = `{{1.updatedHistorial}}` / `WhatsApp_Historial__c` = append con formato `[DD/MM HH:mm] Operador: mensaje`
6. Webhook Response

**Decisión:** Se usa `WhatsApp_Historial_Archivo__c` (historial completo, nunca se resetea) para que la plataforma tenga toda la historia. `WhatsApp_Historial__c` (episodio) se actualiza con append porque Maitena lo usa como contexto en el inbound. La plataforma envía el historial completo actualizado para evitar un GET adicional.

**Decisión error 63016:** Twilio retorna este código cuando se intenta enviar un mensaje libre fuera de la ventana de 24hs. El frontend detecta este error y muestra el selector de templates en lugar del input libre.

---

### Webhook `wtr` — Enviar Template de Reapertura

**Payload recibido:**
```json
{
  "donationId": "...",
  "phone": "...",
  "nombre": "...",
  "operatorName": "...",
  "templateSid": "HX...",
  "templateText": "...",
  "updatedHistorial": "..."
}
```

**Templates disponibles:**
| ID | Content SID | Label |
|---|---|---|
| t1 | HXd9ea64e75370ff621eafa57d45e36e81 | Quedó pendiente la conversación, te parece retomar? |
| t2 | HXa9784c0f8978e1ea543fb8776d976e7a | No pudimos terminar la conversación |
| t3 | HX5006ee74e3379b25fe6f36d37b32958f | No quería cerrar la conversación |

**Módulos:**
1. Webhook
2. HTTP — Twilio: ContentSid = `{{1.templateSid}}`, ContentVariables = `{"nombre": "{{1.nombre}}", "operador": "{{1.operatorName}}"}`
3. SF GET `WhatsApp_Historial__c`
4. SF Update: ambos historiales + `WhatsApp_Estado__c` = `derivado_humano`
5. Webhook Response

**Decisión tres templates:** La plataforma rota entre los 3 para no enviar el mismo mensaje repetidamente. Detecta el último enviado buscando el texto en el historial y ofrece los otros dos. La ventana de 24hs se "abre" solo cuando el donante responde al template.

---

### Webhook `wt` — Tomar Conversación

**Payload:** `{donationId}`  
**SF Update:** `WhatsApp_Estado__c → derivado_humano`, `WhatsApp_Liberar_En__c → null`

**Decisión limpiar Liberar_En:** Si el operador toma una conversación que tenía fecha de liberación programada, esa fecha ya no tiene sentido. Se limpia para que no aparezca en la tab "Programadas".

---

### Webhook `wli` — Liberar Episodio

**Payload:** `{donationId, liberarEn}`  
**SF Update:** `WhatsApp_Estado__c → null`, `WhatsApp_Liberar_En__c → fecha`

**Decisión:** Estado a null + fecha futura = el outbound SOQL excluye el registro (`Liberar_En > TODAY`). Cuando llegue la fecha, el SOQL lo vuelve a incluir y `effective_intentos` detecta el cambio de mes → episodio nuevo.

---

### Webhook `wr` — Resolver

**Payload:** `{donationId, resolution}`  
**SF Update:** `WhatsApp_Estado__c → resolution` (cerrado_positivo o cerrado_negativo)

---

### Webhook `wa` — Sugerencia IA

**Payload:** `{nombre, monto, estado, historial}`  
**Módulos:**
1. Webhook
2. Anthropic API: system = "Sos parte del equipo de donantes de ISF Argentina. Respondé de forma cálida y breve (1-2 oraciones). No uses signos de apertura (¿ ¡). Solo el texto del mensaje.", user = contexto + historial
3. Webhook Response: `{"suggestion": "{{2.content[].text}}"}`

**Decisión:** La sugerencia es a demanda (no automática) para no consumir créditos en cada mensaje. El operador la solicita presionando el botón.

---

### Webhook `wb` — Dar de Baja

**Payload:** `{donationId, motivo, observaciones}`  
**SF Update:**
- `Fecha_de_baja__c` = `{{formatDate(now; "YYYY-MM-DD")}}`
- `Motivo_de_baja__c` = `{{1.motivo}}`
- `Observaciones_baja__c` = `{{1.observaciones}}`
- `Procesador_de_pagos__c` = null
- `npe03__Open_Ended_Status__c` = `Closed`

**Motivos disponibles (picklist hardcodeada en la plataforma):**
- Falta de respuesta y débitos rechazados
- Problemas económicos
- Fuera del país
- Falta de interés
- Quería hacer un único aporte
- En respuesta a una comunicación nuestra
- Auto cese de donación

---

## PUSHER — Notificaciones en Tiempo Real

**Propósito:** Notificar a la plataforma cuando llega un mensaje nuevo de un donante derivado, sin necesidad de polling.

**Flujo:**
1. Make inbound (path `derivado_humano`) → HTTP POST a Pusher después del SF Update
2. Plataforma HTML → WebSocket suscripto al canal `isf-donantes`
3. Al recibir evento `new-message`, la plataforma consulta `wl` y actualiza el chat

**URL Pusher:** `https://api.pusherapp.com/apps/2161561/events`

**Autenticación (HMAC-SHA256 — importante):**
La firma se construye con ESPACIOS entre las partes (no newlines, aunque la documentación de Pusher lo muestra diferente):
```
HmacSHA256("POST /apps/2161561/events auth_key=...&auth_timestamp=...&...", secret)
```
Usar `sha256("POST" + " " + "/apps/2161561/events" + " " + params; secret)` en Make.

**Cluster:** `mt1` (us-east-1)  
**App ID:** 2161561

---

## CAMPOS CUSTOM EN SALESFORCE

Todos en `npe03__Recurring_Donation__c`:

| Campo API | Tipo | Descripción |
|---|---|---|
| `WhatsApp_Estado__c` | Picklist | null / primer_envio / reactivacion / en_conversacion / cerrado_positivo / cerrado_negativo / derivado_humano |
| `WhatsApp_Intentos__c` | Number | Intentos del episodio actual |
| `WhatsApp_UltimoEnvio__c` | DateTime | Timestamp último envío |
| `WhatsApp_FechaInicioEpisodio__c` | Date | Inicio del episodio actual |
| `WhatsApp_Historial__c` | Long Text | Historial del episodio (se resetea cada episodio) |
| `WhatsApp_Historial_Archivo__c` | Long Text | Historial acumulativo completo (nunca se resetea) |
| `WhatsApp_Liberar_En__c` | Date | Fecha de liberación del episodio (outbound pausado hasta esta fecha) |
| `Episodios_Rechazo__c` | Formula | Meses desde último cobro exitoso |
| `Estado_Donante__c` | Picklist | Normal / Rechazada / etc |
| `Medio_de_pago__c` | Text | Visa Crédito / CBU / etc |
| `ISFAR_ultimos_4_digitos_tarjeta_cbu__c` | Text | Últimos 4 dígitos del medio de pago |
| `ISFAR_Id_18_digitos__c` | Text | Id 18 dígitos de la donación |
| `Fecha_de_baja__c` | Date | Fecha de cierre de donación |
| `Motivo_de_baja__c` | Picklist | Motivo del cierre |
| `Observaciones_baja__c` | Long Text | Notas del cierre |
| `Procesador_de_pagos__c` | Text | Se limpia al dar de baja |
| `Reactivacion__c` | Picklist | Reactivado / Reactivación exitosa / fallida |
| `Estado_ltima_op_Cerrada__c` | Text | Sin "u" — así está en SF |

**Campos en Contact (`Contact`):**

| Campo API | Tipo | Descripción |
|---|---|---|
| `Whatsapp_error__c` | Boolean | true si el número no puede recibir mensajes de WhatsApp (ej. error 63024). Se resetea a false automáticamente cuando se modifica el teléfono en SF. |

---

## ESCENARIO — Feriados Argentina

**Propósito:** Obtener los feriados nacionales del año en curso y del año siguiente desde una API pública y guardarlos en una org variable de Make. Se ejecuta una vez al año. El escenario outbound consulta esta variable para no enviar mensajes en días feriados.

### Módulos

| # | Tipo | Descripción |
|---|---|---|
| 1 | HTTP — Make a request | GET feriados del año actual |
| 2 | HTTP — Make a request | GET feriados del año siguiente |
| 3 | Make — Update a custom variable | Guarda ambos arrays en la org variable `Feriados` |

### URLs de los requests

**Módulo 1 — Año actual:**
```
GET https://api.argentinadatos.com/v1/feriados/{{formatDate(now; "YYYY")}}
```

**Módulo 2 — Año siguiente:**
```
GET https://api.argentinadatos.com/v1/feriados/{{parseNumber(formatDate(now; "YYYY")) + 1}}
```

La API devuelve un array de objetos con campo `fecha` en formato `YYYY-MM-DD`.

### Módulo 3 — Update custom variable

| Campo | Valor |
|---|---|
| Variable in organization/team | Organization |
| Organization ID | ISF-Ar Integromat |
| Variable name | `Feriados` |
| Type | String |
| Variable value | `1. Data[]` + `4. Data[]` (array de ambos requests concatenados) |

### Filtro en escenario Outbound

Entre el módulo 1 (SOQL) y el módulo 2, agregar un filtro:

| Campo | Condición | Valor |
|---|---|---|
| Org variable `Feriados` | Does not contain | `{{formatDate(now; "YYYY-MM-DD")}}` |

Si hoy es feriado, el escenario se detiene antes de procesar cualquier donante.

### Schedule

Ejecutar el **día 1 de cada mes**. Aunque los feriados no cambian frecuentemente, la ejecución mensual cubre el caso de feriados puente o extraordinarios que el gobierno declare con poca anticipación.
