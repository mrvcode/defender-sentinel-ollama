# 🧠 Análisis automático de incidentes de Defender/Sentinel con IA local (Ollama)

**Laboratorio** que conecta **Microsoft Defender XDR / Microsoft Sentinel** con un modelo de **IA local (Ollama)**. Al crearse un incidente, una **Logic App** envía sus datos a tu PC vía túnel **ngrok**, recibe un análisis y lo publica como **comentario dentro del incidente**.

> ⚠️ **No es producción.** La IA se equivoca: cada comentario debe revisarlo una persona.
> 💸 **Ventaja clave:** cero coste por tokens frente a IA de pago. Todo corre en tu hardware.
> 🔒 **Ojo:** el modelo es local, pero los datos **sí pasan por ngrok**. No usar datos reales.

**Convención:** menús en español/inglés (el portal cambia según idioma). Valores entre `<...>` son tuyos (**nunca publiques los reales**).

---

## 1️⃣ Arquitectura y flujo

```
┌──────────────┐    ┌───────────────────┐    ┌─────────────────┐
│ Alerta de    │───▶│ Defender for      │───▶│ Defender XDR    │
│ prueba (PC)  │    │ Endpoint          │    │ crea incidente  │
└──────────────┘    └───────────────────┘    └────────┬────────┘
                                                       │
                                                       ▼
                                          ┌────────────────────────┐
                                          │ Regla automatización   │
                                          │ estándar (incidente    │
                                          │ creado)                │
                                          └───────────┬────────────┘
                                                      │
                                                      ▼
                                          ┌────────────────────────┐
                                          │   LOGIC APP            │
                                          │  ┌──────────────────┐  │
                                          │  │ 1. Trigger       │  │
                                          │  │ Sentinel incident│  │
                                          │  └────────┬─────────┘  │
                                          │           ▼            │
                                          │  ┌──────────────────┐  │
                                          │  │ 2. HTTP POST     │  │
                                          │  │ → ngrok          │  │
                                          │  └────────┬─────────┘  │
                                          │           ▼            │
                                          │  ┌──────────────────┐  │
                                          │  │ 3. Add comment   │  │
                                          │  │    (V3)          │  │
                                          │  └──────────────────┘  │
                                          └───────────┬────────────┘
                                                      │
                                    ┌─────────────────┘
                                    ▼
                          ┌──────────────────┐
                          │   NGROK TÚNEL    │
                          │ https://xxx.ngrok│
                          └────────┬─────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │   OLLAMA (PC)    │
                          │ localhost:11434  │
                          │   ┌──────────┐   │
                          │   │ Modelo IA│   │
                          │   └──────────┘   │
                          └──────────────────┘
```

| Pieza                             | Función                                          |
| --------------------------------- | ------------------------------------------------ |
| **Ollama**                        | Ejecuta el modelo de IA en tu PC                 |
| **ngrok**                         | Da a Azure una URL pública hacia tu Ollama local |
| **Logic App (Consumption)**       | Recibe el incidente, llama a Ollama y comenta    |
| **Regla automatización estándar** | Lanza la Logic App al crear un incidente         |

> 🔑 **Regla de oro:** una regla solo lanza playbooks cuyo *trigger* coincida. Incidente → trigger *Microsoft Sentinel incident*.

---

## 2️⃣ Requisitos

- Suscripción Azure + workspace Log Analytics con **Sentinel** activado.
- Workspace conectado a **Defender** (`security.microsoft.com`).
- Equipo de pruebas incorporado a **Defender for Endpoint**.
- PC con RAM suficiente + **WSL2** (opcional en Windows).
- Cuenta gratuita de **ngrok**.
- Usuario con permisos de creación y asignación de roles (_Owner_ en el lab).

> 💡 Base: **Windows 11 + WSL2**. Ollama y el túnel deben estar **encendidos** para que Azure llegue.

---

## 3️⃣ IA local con Ollama

1. Instala Ollama desde [ollama.com/download](https://ollama.com/download).
2. Descarga un modelo: `ollama pull <tu_modelo>`.
3. (Opcional) Alias corto: `ollama cp <tu_modelo> <renombra_tu_modelo>`.
4. Prueba local:
   ```bash
   curl http://localhost:11434/api/generate -d '{"model":"<tu_modelo>","prompt":"Di hola","stream":false}'
   ```

> Puerto por defecto: **11434**. La respuesta va en el campo `response`. Si aparece `<think>…</think>`, elimínalo antes de publicar.

---

## 4️⃣ Túnel con ngrok

1. Crea cuenta e instala el cliente.
2. Registra el token: `ngrok config add-authtoken <TU_TOKEN>`.
3. Abre el túnel: `ngrok http 11434`.
4. Endpoint → `https://<TU_DOMINIO>.ngrok-free.dev/api/generate`.
5. Prueba desde fuera. Si da **403**, usa: `ngrok http 11434 --host-header="localhost:11434"`.

> ⚠️ **Seguridad:** cualquiera con la URL puede usar tu Ollama. Los datos **pasan por ngrok**. Cierra el túnel al terminar. En plan gratuito, si el PC está apagado, la Logic App falla.
>
> 💡 Alternativa "gratis y sin terceros": montar tu propio túnel en una **VPS** con herramientas como `gotunnel`, `proxvn_tunnel` u `Octelium`.

---

## 5️⃣ Creación y permisos de la Logic App

Se crea **en Azure, no en Defender** (un playbook es una Logic App).

| Campo | Valor |
|---|---|
| Plan | **Consumption** |
| Grupo de recursos | El **mismo** que el workspace |
| Región | La misma que el workspace |
| Log Analytics | Desactivado |

Tras crearla:
1. **Activar Identidad** asignada por el sistema.
2. **Asignar roles IAM** en el grupo de recursos:

| Actor | Rol | Para qué |
|---|---|---|
| Tu usuario | Logic App Contributor + Sentinel Contributor + Owner (lab) | Crear y configurar |
| **Azure Security Insights** (servicio) | **Sentinel Automation Contributor** | Que Sentinel ejecute el playbook |
| **Identidad de la Logic App** | **Sentinel Responder** | Comentar incidentes |

3. **Verificar RBAC Unificado** en Defender → *Configuración → Permisos y roles → Administración de área de trabajo*. El workspace debe estar **DESACTIVADO** (si no, el desplegable sale vacío).

> 💡 Los permisos pueden tardar 2-3 minutos en propagarse.

---

## 6️⃣ Diseño interno de la Logic App

**Orden de bloques:**

1. **Trigger:** *Microsoft Sentinel incident* (conexión vía identidad administrada). **No renombrar**.
2. **Acción HTTP:** `POST` al endpoint de ngrok, cabecera `Content-Type: application/json`, con este body:

```json
{
  "model": "<tu_modelo>",
  "prompt": "Eres analista SOC. Responde SOLO en español y texto plano... Datos:\nTítulo: @{triggerBody()?['object']?['properties']?['title']}\nSeveridad: @{triggerBody()?['object']?['properties']?['severity']}\nDescripción: @{triggerBody()?['object']?['properties']?['description']}\nProducto: @{triggerBody()?['object']?['properties']?['additionalData']?['alertProductNames']}\nEntidades: @{triggerBody()?['object']?['properties']?['relatedEntities']}",
  "stream": false,
  "options": { "temperature": 0.2 }
}
```

3. **Acción:** *Add comment to incident (V3)* con:
```json
"message": "@{replace(replace(replace(replace(body('HTTP')?['response'], '```html',''),'```',''),'**',''),'`','')}"
```

> 🔧 `temperature: 0.2` reduce aleatoriedad. Los `replace(...)` limpian Markdown porque **Sentinel no lo interpreta**.

**Errores típicos al editar JSON:** llaves sin cerrar, coma final sobrante en `message`. Si falla el guardado, la app queda en la versión anterior.

---

## 7️⃣ Regla de automatización

> Solo las **reglas estándar** pueden lanzar Logic Apps. Las **mejoradas** no.

1. Defender → **Sentinel → Configuración → Automatización → Reglas estándar → Crear**.
2. Rellenar:

| Campo | Valor |
|---|---|
| Trigger | **Cuando se crea el incidente** |
| Área de trabajo | Tu workspace (debe estar fuera de Unified RBAC) |
| Acción | **Run playbook** → tu Logic App |
| Estado | Activo |

> ⚠️ **Sin condiciones** = se lanza con **cada** incidente nuevo. Para acotar por título usa **Contiene**, no *Es igual a*. La condición *Nombre de regla de análisis* **no sirve** para incidentes de Defender.


---

## 8️⃣ Pruebas

**Generar alerta inofensiva** en el equipo incorporado:
- **Fichero EICAR** desde [eicar.org](https://www.eicar.org).
- O el comando de prueba de Microsoft (CMD admin):
  ```shell
  powershell.exe -NoExit -ExecutionPolicy Bypass -WindowStyle Hidden $ErrorActionPreference= 'silentlycontinue';(New-Object System.Net.WebClient).DownloadFile('http://127.0.0.1/1.exe', 'C:\\test-WDATP-test\\invoice.exe');Start-Process 'C:\\test-WDATP-test\\invoice.exe'
  ```

**Antes de cada prueba:**
- Cierra (resuelve) incidentes abiertos del mismo equipo (si no, se agrupan y la regla no salta).
- Verifica que **Ollama y ngrok** estén encendidos.

**Dónde ver el resultado:**
- **Incidente → Actividades**: aparece el comentario del playbook.
- **Logic App → Run history**: tres pasos en verde (trigger → HTTP → comentario). Tarda ~40s con modelo medio.

> 💡 Para repetir pruebas sin generar nuevas alertas: lanza el playbook manualmente desde los **tres puntos** del incidente.

---

## 9️⃣ Comentario generado por la IA

1. Ve a **Incidentes**, restablece filtros (incluye severidad *Informativo*, como EICAR).
2. Abre el incidente → pestaña **Actividades**.
3. Haz clic en la actividad de tipo **Comentario** para ver el texto insertado.

> ⏱️ Al relanzar el playbook manualmente, espera ~2 minutos antes de ver el nuevo comentario.

---

## 🔟 Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Logic App no aparece en la regla | Trigger distinto | Regla de incidente + trigger *Sentinel incident* |
| Desplegable "Área de trabajo" vacío | Unified RBAC activo | Desactivarlo (ver §5) |
| No pasa nada al generar alerta | Condición no coincide o incidente agrupado | Ajustar condición (**Contiene**) y cerrar incidentes abiertos |
| HTTP falla | PC/Ollama/ngrok apagados o URL cambiada | Reiniciar túnel y actualizar URI |
| HTTP 403 | Ollama rechaza el Host | `ngrok http 11434 --host-header="localhost:11434"` |
| Timeout | Modelo demasiado grande (límite ~2 min HTTP) | Modelo más pequeño o menos datos |
| Error JSON al guardar | Llave o coma mal puesta | Revisar cierre de `options`/`body`/`inputs` |
| Comentario con `###`, `**`, ``` | Sentinel no interpreta Markdown | Usar los `replace(...)` |
| No encuentro el comentario | No está en la cabecera | Mirar pestaña **Actividades** |

---

## 1️⃣1️⃣ Buenas prácticas con IA local

**Privacidad**
- El modelo es local, pero los datos **salen por el túnel**. No envíes datos sensibles reales.
- No subas a GitHub URLs de ngrok, IDs de suscripción/tenant/workspace, equipos ni usuarios.

**Fiabilidad (casos reales de este lab)**
- Dijo "no se proporciona hash" cuando sí iban en los datos.
- Inventó el significado de EICAR y dio recomendaciones sin base.
- Marcó IPs propias como sospechosas sin motivo.
- Ignoró el formato pedido incluso con temperatura baja.

**Recomendaciones**
- Trata el comentario como **ayuda al analista**, nunca como veredicto.
- Prueba cada cambio de prompt **varias veces** (no es determinista).
- Cambia **una cosa cada vez** en el prompt.
- Envía solo los campos necesarios (JSON grande = peor respuesta).
- Filtrar entidades por tipo antes de enviarlas sería más robusto (no probado aquí).

---

## 1️⃣2️⃣ Limitaciones y mejoras pendientes

- Regla sin condición o con condición de título → ajustar según necesidad.
- Dependencia de que **PC + túnel** estén encendidos.
- Formato de salida **no garantizado**.
- Mejora: **etiqueta automática** (`IA-analizado`) vía acción *Update incident*.
- Los playbooks generados por IA (Python) y reglas mejoradas son otra vía, no usada aquí.

---

> **Ideal para:** prácticas de laboratorio, PoCs y aprendizaje de Sentinel + Logic Apps + IA local sin coste por tokens.