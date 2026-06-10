# ISF Bot — Resumen General del Sistema
**Versión:** Junio 2026  
**Estado:** MVP operativo en producción

---

## QUÉ ES ESTE SISTEMA

Sistema de comunicación automatizada con donantes de **Ingeniería Sin Fronteras Argentina (ISF)** vía WhatsApp Business. Tiene dos funciones principales:

1. **Recupero de donaciones rechazadas:** cuando el débito mensual de un donante falla (tarjeta vencida, fondos insuficientes, etc.), el sistema lo contacta por WhatsApp para ayudarlo a actualizar sus datos.
2. **Gestión humana de casos complejos:** cuando el bot no puede resolver solo, deriva al equipo humano que gestiona desde una plataforma web.

---

## STACK TECNOLÓGICO

| Capa | Herramienta | Rol |
|---|---|---|
| CRM | Salesforce NPC | Fuente de verdad de donantes y donaciones |
| Orquestación | Make.com | Conecta todas las piezas, maneja lógica |
| Mensajería | Twilio WhatsApp Business | Envío/recepción de mensajes |
| IA | Claude Sonnet 4.6 (Anthropic) | Genera respuestas de Maitena y Genaro |
| Scraping | Firecrawl | Actualiza info de proyectos mensualmente |
| Tiempo real | Pusher Channels | Notificaciones push a la plataforma |
| Plataforma | HTML/JS (GitHub Pages) | Interfaz para operadores humanos |

---

## CÓMO SE COMBINAN LAS PARTES

```
SALESFORCE (fuente de verdad)
    ↕ (lectura/escritura)
MAKE.COM (orquestador central)
    ├── recibe webhooks de TWILIO (mensajes entrantes)
    ├── llama a CLAUDE API (genera respuestas)
    ├── envía mensajes vía TWILIO (respuestas + outbound)
    ├── actualiza SALESFORCE (estados, historiales)
    ├── publica eventos en PUSHER (notificaciones)
    └── expone webhooks a la PLATAFORMA WEB

PLATAFORMA WEB (GitHub Pages)
    ├── consulta Make webhooks para datos
    ├── escucha eventos de Pusher (tiempo real)
    └── operadores envían mensajes via Make webhooks

FIRECRAWL (mensual)
    └── scraping sitio ISF → Make → Custom Variable → Claude prompts
```

---

## FLUJOS PRINCIPALES

### Flujo 1: Donante recibe outbound y responde
1. Make Outbound corre diariamente → selecciona donantes con cobro rechazado (SOQL con múltiples filtros)
2. Envía template aprobado de WhatsApp via Twilio
3. Donante responde → Twilio envía webhook a Make Inbound
4. Make consulta SF por los datos del donante
5. Llama a Claude con el system prompt de Maitena + contexto del donante + historial
6. Claude genera respuesta → Make la envía por Twilio
7. Make actualiza el historial y estado en SF

### Flujo 2: Derivación a operador humano
1. En cualquier punto Maitena puede incluir `[ESTADO:derivado_humano]` en su respuesta
2. Make detecta el tag y actualiza `WhatsApp_Estado__c = 'derivado_humano'` en SF
3. El inbound en este estado ya no llama a Claude; solo graba el mensaje del donante
4. Pusher notifica a la plataforma en tiempo real
5. El operador ve la conversación en la plataforma, puede responder, derivar o cerrar

### Flujo 3: Operador gestiona desde la plataforma
- **Leer:** webhook `wl` → SOQL SF → lista de conversaciones activas
- **Responder:** webhook `ws` → Twilio send + SF update historial
- **Template:** webhook `wtr` → Twilio ContentSid dinámico + SF update
- **Tomar:** webhook `wt` → SF: estado = derivado_humano
- **Liberar:** webhook `wli` → SF: estado = null + fecha futura
- **Resolver:** webhook `wr` → SF: cerrado_positivo o cerrado_negativo
- **IA:** webhook `wa` → Claude genera sugerencia de respuesta
- **Baja:** webhook `wb` → SF: cierra donación con motivo y fecha
- **Stats bajas:** webhook `wbs` → SOQL SF → estadísticas del modal de bajas

---

## LÓGICA DE ESTADOS (WhatsApp_Estado__c)

```
null ←──────────────────── liberar_episodio (wli)
  ↓
primer_envio ←── outbound intento 1
  ↓
en_conversacion ←── donante responde template
  ↓
cerrado_positivo ←── donante retoma pago
cerrado_negativo ←── donante cancela
derivado_humano ←── derivar a humano
  ↓
[operador gestiona]
  ↓
cerrado_positivo / cerrado_negativo / null+fecha
```

**Estados y comportamiento del outbound:**
- `cerrado_negativo`, `derivado_humano`, `cerrado_positivo`, `en_conversacion` → **excluidos del outbound**
- `null` con `Liberar_En > hoy` → **excluido temporalmente**
- `null` o `primer_envio` o `en_conversacion` viejo → **elegible para outbound**

**Regla de effective_intentos:** El outbound hace hasta 3 intentos por episodio. Un "episodio" se resetea cuando el donante no respondió el mes anterior (más de 12 días en un mes diferente).

---

## BOTS

### Maitena (recupero donaciones activas)
- **Activa cuando:** `npe03__Open_Ended_Status__c = 'Open'`
- **Objetivo:** que el donante actualice sus datos de pago
- **Tono:** cálido, cordial, humano — nunca informal/juvenil
- **Regla crítica:** si `WhatsApp_Intentos__c = 0`, el donante escribió espontáneamente → NO mencionar el rechazo
- **Retención:** empatía → impacto concreto → reducir monto → pausar (en ese orden)
- **Identidad:** se presenta como persona real del equipo, nunca admite ser IA
- **Cambio de monto:** se confirma en el chat sin formulario → tag `[ALERTA:cambio_monto]` → Make crea tarea para el equipo. El formulario es solo para cambio de medio de pago.
- **Aporte extra puntual vs cambio permanente:** son flujos distintos. El aporte extra (único) deriva a humano. El cambio de monto permanente se resuelve en el chat con la alerta.
- **Alertas disponibles:** `[ALERTA:emails]`, `[ALERTA:voluntario]`, `[ALERTA:cambio_monto]`, `[ALERTA:general]`

### Genaro (reactivación ex-donantes)
- **Activa cuando:** `npe03__Open_Ended_Status__c = 'Closed'`
- **Objetivo:** que el ex-donante retome su apoyo
- **Tono:** persuasivo y entusiasta, defiende la propuesta con convicción
- **Diferencia clave:** puede insistir dos veces antes de cerrar con dignidad

---

## PLATAFORMA DE GESTIÓN

**URL:** `https://adanlevy.github.io/isf-consola-bot/`  
**Configuración:** archivo `isf_config.json` cargado en la plataforma (nunca en el repositorio — contiene API keys)

**Tabs:**
- **Humano** — conversaciones donde el operador debe responder (`derivado_humano`)
- **Negativo** — cerradas negativamente (posible baja en SF)
- **Positivo** — cerradas positivamente (verificar cobro)
- **En conv.** — bot activo conversando (monitoreo)
- **Program.** — episodios pausados hasta fecha futura
- **Outbound** — primer contacto del outbound

**Funcionalidades clave:**
- Notificaciones en tiempo real (Pusher) cuando llega mensaje nuevo
- Ventana 24hs detectada automáticamente → muestra selector de templates
- 3 templates de reapertura con rotación (no ofrece el último enviado)
- Sugerencia IA a demanda (webhook `wa`)
- Dar de baja en SF con motivo y observaciones
- **Estadísticas de bajas** (botón ícono `user-minus` en topbar): modal con datos de este mes y mes anterior, desglosados en cobro antiguo (>3 meses), cobro reciente (≤3 meses) y bajas activas (motivo distinto a "Falta de respuesta y débitos rechazados"). Requiere webhook `wbs` configurado en ajustes.

---

## ARCHIVOS DEL REPOSITORIO

| Archivo | Descripción |
|---|---|
| `index.html` | Plataforma de gestión (v1.35) |
| `isf_config.json` | Template de configuración (sin credenciales reales) |
| `make_maitena_v3.4.json` | Body del módulo Anthropic para Maitena en Make |
| `isf_bot_prompt_v1_4.md` | System prompt de Maitena |
| `isf_bot_prompt_genaro_v1_1.md` | System prompt de Genaro |
| `ISF_Make_Escenarios.md` | Documentación detallada de todos los escenarios Make |
| `ISF_Resumen_General.md` | Este archivo |

---

## PENDIENTES PRIORITARIOS

1. **Notificación al operador** — cuando llega mensaje nuevo (email o Pusher — parcialmente implementado)
2. **Escenario Make para `[ALERTA:cambio_monto]`** — detectar el tag en el inbound y crear tarea en SF/Notion con los datos del donante
3. **Outbound Genaro** — escenario Make + templates + SOQL para ex-donantes
4. **Bot de bienvenida** — nuevos donantes
5. **SF Connected App** — reemplazar webhook `wl` por consulta directa a SF
6. **Rate limiting / deduplicación** — robustez para mensajes duplicados de Twilio
7. **Reset mensual automático** `cerrado_positivo` — escenario Make día 1

---

## DECISIONES DE DISEÑO IMPORTANTES

**¿Por qué Make y no código propio?**  
ISF no tiene equipo técnico permanente. Make permite mantener los flujos sin escribir código, con interfaz visual. El costo es bajo para el volumen actual.

**¿Por qué Twilio y no otra BSP?**  
360dialog cobró mínimo $59/mes. Twilio cobra $0 fijo + $0.005/mensaje. Para el volumen de ISF, Twilio es significativamente más barato.

**¿Por qué Claude y no GPT?**  
Mejor manejo del español rioplatense, mejor cumplimiento de instrucciones de tono específicas, y respuestas más naturales en contextos emocionales (retención de donantes).

**¿Por qué GitHub Pages y no servidor?**  
La plataforma es HTML/JS puro, no necesita backend propio. GitHub Pages es gratis, confiable y sin mantenimiento.

**¿Por qué historial en SF y no en Make/Claude?**  
Claude no tiene memoria entre conversaciones. El historial se guarda en SF porque es la fuente de verdad central, accesible por Make, la plataforma y el equipo de ISF directamente.

**¿Por qué dos campos de historial?**  
`WhatsApp_Historial__c` (episodio) se inyecta en el prompt de Claude — solo el contexto actual para no exceder el context window. `WhatsApp_Historial_Archivo__c` (completo) se usa en la plataforma para que el operador vea toda la historia con el donante.
