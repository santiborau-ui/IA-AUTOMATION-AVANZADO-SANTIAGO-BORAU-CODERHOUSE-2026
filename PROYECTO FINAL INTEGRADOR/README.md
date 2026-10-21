# Proyecto Final Integrador - Sistema Agéntico Empresarial

> **Trabajo ficticio de evaluación.** La "Clínica Predio AFA", sus pacientes, consultas, correos y el manual de procedimientos son inventados y se usan solo como práctica del curso de Automatización Avanzada con IA (Coderhouse, cohorte 2026). Nada de lo que responde el sistema es información ni consejo médico real.
>
> **Este repositorio no contiene claves de API, tokens ni contraseñas.** Solo se documentan los *nombres* de las credenciales, nunca sus valores.

**Autor:** Santiago Borau
**Informe completo:** `Borau_Santiago_ProyectoFinal_AI_Experto.pdf`

---

## 1. Qué problema resuelve

Una clínica deportiva recibe muchas consultas repetidas (cancelaciones, ausencias, derechos del paciente). Responderlas a mano consume tiempo del equipo y, si se responde mal, genera riesgo (cobros o plazos inventados, consejos médicos). El sistema:

1. Recibe la consulta y **confirma enseguida** que fue recibida (no deja esperando).
2. Responde **solo con lo que dice el manual institucional**, citando documento y sección. Si el manual no lo cubre, contesta "No sé" y avisa a una persona.
3. Un segundo modelo **revisa y califica** la respuesta antes de enviarla (AI-as-a-Judge, nota 1 a 5).
4. Si la nota es baja, la respuesta se **corrige sola** (hasta 2 reintentos) o la **decide una persona**.
5. Solo la respuesta validada se despacha (borrador de Gmail) y se guarda en el CRM y en la auditoría.
6. Si algo falla, **se avisa al equipo humano** y la consulta se puede reenviar sin duplicar.

## 2. Estructura del repositorio

```
README.md
workflows/            Los 7 flujos de n8n exportados (.json)
base-conocimiento/    Manual ficticio (versión 1.1, con aranceles) usado como única fuente del asistente (RAG)
auditoria/            Pruebas reales con la calificación del juez (.csv)
capturas/             Capturas del canvas de n8n (arquitectura)
```

## 3. Los 7 flujos (workflows)

| Flujo | Rol |
|---|---|
| `PF_Manager_Clinica_Predio_AFA` | Controlador (Manager): recibe, evita duplicados, coordina a los demás y cierra la solicitud |
| `PF_Worker_Memoria` | Lee y guarda la memoria de largo plazo de cada sesión |
| `PF_Worker_Agente_RAG` | Asistente que responde solo con fragmentos del manual y cita sus fuentes |
| `PF_Worker_Juez` | Califica la respuesta (exactitud 1 a 5) y emite el veredicto |
| `PF_Worker_Despacho` | Envía la respuesta validada y la registra en el CRM y la auditoría |
| `PF_Indexar_Manual_RAG` | Carga el manual a la base vectorial (se ejecuta una vez, o tras reiniciar n8n) |
| `PF_Alerta_Errores` | Flujo de errores global: si algo se cae, manda un aviso por Gmail |

Para importar uno: en n8n, menú del flujo, **Import from file**. Luego hay que **reasignar las credenciales** (ver sección 5) y, en el Manager, elegir los sub-flujos reales en cada nodo "Execute Workflow".

## 4. Cómo funciona (resumen)

- **Entrada:** Webhook `GET` en la ruta `/clinica-consulta`, con los parámetros `mensaje`, `session_id` e `id`. Responde `202` con un `codigo_recepcion` al instante; el trabajo sigue en segundo plano. Si faltan datos, responde `400`.
- **Idempotencia (sin duplicados):** la tabla `SOLICITUDES_PROCESADAS` guarda una clave `id-<id>`. Si ya está `completado` o `en_proceso` hace menos de 15 minutos, la consulta se ignora. Si quedó en `error`, se puede reintentar. Los registros de CRM y auditoría se escriben por clave (upsert), nunca se duplican.
- **RAG:** embeddings de Cohere y base vectorial en memoria de n8n. Trozos de 800 caracteres con solape de 100; se recuperan los 4 más cercanos. Cada trozo lleva metadatos `documento`, `fuente` y `version`.
- **Citas:** formato `Fuente: <documento>, sección <n>`. El sistema verifica que la sección citada exista en los fragmentos recuperados; si no coincide, fuerza la corrección.
- **Juez:** salida estructurada `exactitud_factual` (1 a 5), `estado_veredicto` (`ACEPTADO`, `CORREGIR`, `RECHAZADO`), `cita_correcta`, `critica` y `evidencia_factual` (hasta 3 afirmaciones clave, cada una con su sección del manual y si está respaldada). **Umbral 4**: menos de 4 no se acepta. Regla extra: cualquier afirmación (cobros, multas, plazos, cifras) que no esté en el manual limita la nota a 2.
- **Ruta según veredicto:** aceptar y despachar, corregir (bucle de autocorrección, máximo **2 reintentos**), o escalar a una persona por Gmail (Aprobar / Editar).
- **Memoria de largo plazo:** una fila por sesión en Airtable, que se lee antes de responder y se actualiza al final.
- **Errores:** los nodos críticos tienen una rama de error que avisa por Gmail al equipo humano y marca la solicitud como `error`. Los reintentos automáticos solo se usan en el asistente y el juez. Además, el Manager y los workers de Memoria, Asistente, Juez y Despacho usan `PF_Alerta_Errores` como flujo de error.

## 5. Credenciales y parámetros de conexión (sin valores)

**Credenciales configuradas dentro de n8n** (se cargan en *Credentials*, no viajan en los .json):

| Servicio | Para qué se usa |
|---|---|
| Groq (API) | Modelos del asistente y del juez |
| Cohere (API) | Embeddings del manual |
| Airtable (token de acceso personal) | Memoria, solicitudes procesadas y conocimiento |
| Google Sheets (OAuth2) | Auditoría y CRM |
| Gmail (OAuth2) | Borradores de despacho, avisos a personas y alertas de error |

**Parámetros del sistema:**

| Parámetro | Valor |
|---|---|
| Ruta del webhook | `GET /clinica-consulta` |
| Parámetros de entrada | `mensaje`, `session_id`, `id` |
| Modelo del asistente (Groq) | `openai/gpt-oss-20b`, temperatura 0.2 |
| Modelo del juez (Groq) | `openai/gpt-oss-120b`, temperatura 0 |
| Embeddings (Cohere) | `embed-english-v3.0` (1024 dimensiones) |
| `UMBRAL` del juez | 4 |
| `MAX_REINTENTOS` | 2 |
| Trozos del manual | 800 caracteres, solape 100 |
| Memoria vectorial | clave `clinica_predio_afa_kb_final` (en memoria; se pierde al reiniciar n8n, hay que volver a ejecutar `PF_Indexar_Manual_RAG`) |
| Despacho | Borrador de Gmail (no se envía al paciente) |

El Manager y los workers se conectan con nodos "Execute Workflow" por ID. Esos IDs son propios de cada instalación de n8n y hay que reasignarlos al importar.

## 6. Estructura de las tablas

**Airtable - `MEMORIA_SESIONES`** (una fila por sesión)

| Campo | Contenido |
|---|---|
| `Session_ID` | Clave única de la sesión |
| `Resumen Consolidado` | Resumen de lo conversado hasta ahora |
| `Estado del Caso` | Estado actual de la consulta |
| `Datos Clave` | Datos relevantes en formato JSON |
| `Cantidad_Mensajes` | Cuántos mensajes tuvo la sesión |

**Airtable - `SOLICITUDES_PROCESADAS`** (control de duplicados y estado)

| Campo | Contenido |
|---|---|
| `clave_idempotencia` | `id-<id>`, clave única de la solicitud |
| `estado` | `en_proceso`, `completado` o `error` |
| `ruta_final` | `ACEPTAR`, `ESCALAR_APROBADO` o `ERROR` |
| otros | campos de seguimiento de la solicitud (por ejemplo `traceId` y `session_id`) |

**Airtable - `CONOCIMIENTO_CLINICA`:** tabla de apoyo de módulos anteriores. Se le agregó la fila de aranceles de consultas particulares (ficticios), que también está en la sección 4.3 del manual.

**Google Sheets - `Auditoria_Juez_M8`**

- Pestaña `auditoria final` (una fila por intento): `fecha`, `traceId`, `clave_auditoria` (traceId-intento), `session_id`, `pregunta`, `respuesta_agente`, `fuentes_citadas`, `cita_correcta`, `exactitud_factual`, `estado_veredicto`, `critica`, `intentos`, `ruta_final`, `tokens_entrada`, `tokens_salida`, `costo_usd`, `version_prompt`, `veredicto_json`, `evidencia_factual`.
- Pestaña `CRM_resultados` (una fila por consulta): `traceId`, `fecha`, `session_id`, `pregunta`, `respuesta_final`, `exactitud_factual`, `ruta_final_registrada`, `aprobado_por`, `fuentes_citadas`, `respondio_no_se`, `estado_despacho`, `canal`, `costo_usd`.

## 7. Pruebas realizadas

**Auditoría oficial (12 de octubre, 10 consultas):** respuesta normal con cita, "No sé" ante una pregunta fuera del manual, rechazo de consulta médica, premisa falsa, intento de inyección de instrucciones, y un caso donde el juez puso nota 2, el sistema corrigió y la segunda versión llegó a 5. Nota media 4,6 en el primer intento y 4,9 final; 8 de 10 resueltas sin persona (las otras 2 se escalaron a propósito para probar el semáforo).

**Pruebas posteriores (21 de octubre, se informan aparte):** 11 consultas llegaron al juez; 8 se aceptaron al primer intento y 3 quedaron sin cerrar porque el tope de uso del modelo gratuito cortó la ejecución. Incluyen la prueba de que el campo `evidencia_factual` marca como no respaldada una tarifa inventada, y la corrección del manual para que el asistente conteste los aranceles (versión 1.1).

**Prueba de estrés:** de 14 consultas enviadas con unos 12 segundos de diferencia, 9 fallaron por el tope de 8.000 tokens por minuto del plan gratuito de Groq. Quedaron registradas como `error` y se pueden reenviar.

El archivo CSV de la carpeta `auditoria/` tiene las filas reales de la hoja `auditoria final`. También se probaron (y quedaron registradas en n8n, no en el CSV): duplicado ignorado, escalada a una persona con aprobación, y una caída simulada de API con alerta por correo.

**Costo medido:** alrededor de USD 0,001 por consulta (unos 5.000 a 6.200 tokens de entrada y 130 a 280 de salida), con precios equivalentes a GPT-4o-mini.

## 8. Limitaciones conocidas

- Los modelos gratuitos de Groq tienen un límite de 8.000 tokens por minuto: alcanza para más o menos una consulta por minuto. Por eso el asistente y el juez usan modelos distintos y, en producción, haría falta un plan de pago.
- Las conexiones con Google (Sheets y Gmail) vencen y hay que volver a iniciar sesión en n8n.
- El control de citas por código no reconoce secciones cuando el fragmento recuperado es una tabla, y un "No sé" no se puede verificar si el buscador no trajo el fragmento correcto.
- La base vectorial en memoria se vacía al reiniciar n8n.
- El cargador de documentos no guarda número de página: las citas son por documento y sección.
- En una prueba, un juez más chico dejó pasar un cobro inventado; se reforzó la regla de verificación y se usó un juez más fuerte (ver informe).
- Algunas pruebas de la auditoría se hicieron forzando condiciones (umbral alterado a propósito) para ver las otras rutas.
