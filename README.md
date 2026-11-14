# GustoBot | Flujo omnicanal de pedidos y atención (Checkpoint 6)

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![OpenAI](https://img.shields.io/badge/OpenAI-LLM_Engine-412991?style=for-the-badge&logo=openai)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Gmail](https://img.shields.io/badge/Gmail-Integration-red?style=for-the-badge&logo=gmail)
![HubSpot](https://img.shields.io/badge/HubSpot-CRM-orange?style=for-the-badge&logo=hubspot)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)
![Supabase](https://img.shields.io/badge/Supabase-Vector_Store-green?style=for-the-badge&logo=supabase)

---

## Descripción del proyecto

**GustoBot** es un asistente virtual para gestionar pedidos, consultas y reclamos de un restaurante. El **Checkpoint 6** continúa el workflow del Módulo 5: conserva la arquitectura multi-agente Manager-Worker, la atención por Slack y Gmail, HubSpot, revisión humana (HITL) y RAG documental con Supabase. La solución separa el procesamiento central de los adaptadores de canal: Slack, Gmail y Telegram normalizan sus entradas y llaman al workflow central; la ingesta documental de Drive queda en otro workflow.

## Archivos y recursos

* **`Checkpoint5_marcelo_troncoso.json`**: workflow vigente de n8n, con entradas desde Slack y Gmail, clasificación, derivación a workers, rama RAG documental e ingesta desde Google Drive a Supabase.
* **`Checkpoint6_marcelo_troncoso.json`**: evolución histórica del workflow anterior con la rama de Voice AI integrada a Slack.
* **`Checkpoint6_Final_VoiceAI_Telegram.json`**: workflow central de conversación. Recibe campos normalizados explícitos (`mensaje`, `original_input`, `usuario_id`, `canal`, `tipo_entrada` y metadatos de canal) por `Execute Workflow`; conserva el enrutamiento, workers, RAG de consulta y HITL.
* **`Checkpoint6_Slack_Channel_Module.json`**: adaptador Slack para entradas de texto/audio, transcripción y contingencias.
* **`Checkpoint6_Gmail_Channel_Module.json`**: adaptador Gmail para filtrar correos, validar remitentes y preparar el payload de conversación.
* **`Checkpoint6_Telegram_Voice_Module.json`**: adaptador de voz de Telegram. Descarga y transcribe el audio, valida la transcripción y llama al workflow central sin transferir el binario.
* **`Checkpoint6_RAG_Ingestion_Module.json`**: workflow independiente para triggers de Drive, extracción, deduplicación, aprobación de reemplazos e indexación en Supabase.
* **`Agente con Voice.json`**: ejemplo de referencia para la integración de voz.
* **`Checkpoint4_marcelo_troncoso.json`**: versión anterior del workflow principal.
* **`Supabase_document_chunks.sql`**: esquema de la tabla vectorial `document_chunks` y función `match_document_chunks`.
* **`Worker_Registrar_Pedido.json`**: worker para registrar pedidos.
* **`Worker_Atencion_Consultas.json`**: worker para responder consultas.
* **`Worker_Atencion_Reclamos.json`**: worker para atender reclamos.
* **`Worker_Anfitrion.json`**: worker de bienvenida y solicitudes generales.
* **Planillas de datos y auditoría**: `Menu_GustoBot.xlsx`, `Pedidos_GustoBot.xlsx`, `Reclamos_GustoBot.xlsx` y `Logs_Ejecucion_GustoBot.xlsx`.
* **[Pantallazos de evidencia del Checkpoint 4](https://docs.google.com/document/d/1o2TdufEyv6BqBbX8gGqfmB_-vun86KBp8qrNTTiD2_0/edit?usp=drive_link)**: capturas de la entrega, disponibles en el documento compartido.
* [Planilla de logs en Google Sheets](https://docs.google.com/spreadsheets/d/1NmEYqoD3hRMmt2Ni0o4Um-sMYp7XUr-zP_bda_Aw64Y/edit?usp=drivesdk).
Flujo completo: ![alt text](image.png)

## Arquitectura del Checkpoint 6

El Checkpoint 6 mantiene las ramas de atención descritas abajo, la recuperación documental e ingesta en Supabase, y suma el circuito de voz.

### 1. Ingreso por Slack y Gmail

* **Slack** recibe mensajes del canal configurado y los prepara en un payload mínimo para el procesamiento.
![alt text](image-1.png)

* **Gmail** consulta cada minuto los correos no leídos. Antes de procesarlos, el flujo descarta respuestas automáticas y mensajes de remitentes `no-reply`; luego normaliza remitente, asunto y cuerpo.
![alt text](image-2.png)
![alt text](image-3.png)

### 2. CRM y clasificacion

Para solicitudes recibidas por correo, el flujo busca el contacto en **HubSpot** y lo crea o actualiza según corresponda. El router IA clasifica las solicitudes en una taxonomía cerrada:

* `REGISTRAR_PEDIDO`
* `CONSULTA_MENU`
* `CONSULTA_DOCUMENTAL`
* `RECLAMO_SOPORTE`
* `UNKNOWN`
![alt text](image-4.png)
![alt text](image-5.png)

La clasificación se normaliza antes de continuar; una salida no reconocida se trata como `UNKNOWN`. El flujo incluye modelos de Groq y OpenAI para las etapas de IA.

### 3. Derivación a workers y revisión humana

El Manager deriva las solicitudes clasificadas a los sub-workflows correspondientes:

* `REGISTRAR_PEDIDO` -> `Worker_Registrar_Pedido`
* `CONSULTA_MENU` -> `Worker_Atencion_Consultas`
* `RECLAMO_SOPORTE` -> `Worker_Atencion_Reclamos`
* `UNKNOWN` y casos que requieren revisión -> camino de atención humana

![alt text](image-6.png)

Las llamadas a los workers esperan su finalización para preparar la respuesta. En Slack, los casos de revisión humana se comunican por el canal. Para Gmail, el flujo prepara un borrador para revisión; los casos que no requieren intervención pueden recibir una respuesta automática, previa validación del correo.

### 4. Memoria y auditoria

Google Sheets almacena los registros de ejecución y la memoria conversacional. El workflow recupera el historial y evalúa si supera los **5 mensajes** para activar el procesamiento analítico y actualizar el registro de memoria.

![alt text](image-7.png)

### 5. RAG e ingesta documental

Las consultas institucionales se derivan a una búsqueda RAG conectada a la tabla `document_chunks` de Supabase. La recuperación devuelve hasta cuatro fragmentos (Top-K = 4) ordenados por similitud, sin un umbral fijo que pueda excluir consultas válidas. Si no se recupera contexto, el flujo responde directamente que no hay información disponible; las respuestas generadas se restringen a los fragmentos. Las consultas sobre arrepentimiento se amplían con términos legales relacionados para distinguirlas de reembolsos por fallas. Los documentos nuevos o modificados en la carpeta de Google Drive se descargan, se procesan como PDF, se fragmentan en bloques de 1000 caracteres con solapamiento de 200 y se indexan con embeddings de OpenAI. Antes de indexar, la ingesta busca la combinación exacta de `drive_file_id` y `content_hash` en los metadatos para omitir archivos ya cargados; los registros históricos sin esas claves no pueden detectarse mediante esta regla. La limpieza de fragmentos anteriores se ejecuta después de insertar la nueva versión. Las respuestas automáticas se envían por el canal de origen: Slack responde al canal del mensaje y Gmail al remitente del correo.

Ejecuta `Supabase_document_chunks.sql` en Supabase antes de usar esta rama. Configura las credenciales de Google Drive, Supabase y OpenAI en n8n.

### 6. Voice AI final por Telegram

Para la preentrega final se utiliza un bot oficial de Telegram creado con BotFather. El workflow importable es [`Checkpoint6_Final_VoiceAI_Telegram.json`](./Checkpoint6_Final_VoiceAI_Telegram.json). Parte del workflow del Módulo 5/Checkpoint 6, conserva RAG, workers y las ramas existentes. La transcripción de Telegram entra en `Entrada multicanal`, atraviesa el enrutamiento central, RAG, workers y HITL, y solo la respuesta final vuelve a la rama TTS de Telegram: Telegram Trigger, IF de nota de voz, descarga binaria, Whisper, IF de contingencia, AI Agent central en modo Tools Agent, validación de 200 caracteres, ElevenLabs y Telegram Send Audio.

El nodo de Whisper recibe obligatoriamente el binario `data` y el idioma `es`. La rama falsa del IF posterior a Whisper cubre transcripciones vacías; la salida de error de Whisper, del agente, del validador y de ElevenLabs converge en un fallback de texto. El AI Agent está marcado como `toolsAgent` y tiene conectada la herramienta `Telegram Voice - Herramienta validar salida`. La respuesta se valida nuevamente antes de generar audio, por lo que el límite de 200 caracteres no depende solo del prompt.

**System Prompt exacto del AI Agent central:**

> Sos el agente central de enrutamiento de voz de GustoBot. Elegí la ruta de atención según la transcripción y el contexto. No respondas al cliente ni registres pedidos: la ruta seleccionada pasa por los controles de riesgo, la aprobación humana y el worker correspondiente. Los workers y la preparación posterior deben mantener las respuestas de voz en un máximo de 200 caracteres, sin enlaces, viñetas ni emojis.
>
> Usá exactamente una vez la herramienta Seleccionar ruta voz. Su entrada debe ser un string JSON con intent, confidence y risk. Intent permitido: REGISTRAR_PEDIDO, CONSULTA_MENU, CONSULTA_DOCUMENTAL, RECLAMO_SOPORTE o UNKNOWN. Confidence es un número entre 0 y 1. Para entradas de audio, toda respuesta final debe limitarse a 200 caracteres y no incluir enlaces, viñetas ni emojis.

ElevenLabs usa `eleven_multilingual_v2`, idioma `es`, Stability `0.65` y Clarity/Similarity `0.80`. Telegram envía el binario generado con la operación `Send Audio` y la propiedad `data`.

**Privacidad y descarte:** `saveDataSuccessExecution` y `saveDataErrorExecution` están en `none`, el progreso no se guarda y los nodos `Privacy - Descartar...` eliminan el objeto binario antes de continuar. No hay nodos de base de datos, Drive ni planilla en este workflow. El audio solo permanece durante la ejecución y se descarta al terminar; la retención del mensaje original en Telegram y la configuración de almacenamiento de la instancia deben verificarse por separado.

### Antecedente del checkpoint 6: Voice AI por Slack

El **Slack Trigger** del checkpoint histórico conserva la entrada de mensajes y también detecta archivos compartidos, descargándolos como binarios. Una condición IF distingue archivos de audio de mensajes de texto: los textos siguen por la ruta existente; los audios se mapean a `data` y se transcriben con **OpenAI Whisper** en idioma `es`. La entrega final no usa esta ruta: se dejó como antecedente para evitar confundir Slack con el canal solicitado.

Para audio, después de entrar por `Entrada multicanal` y recuperar el contexto, **AI Agent central - Enrutamiento de voz** selecciona una sola ruta mediante su herramienta **Seleccionar ruta voz**. La clasificación validada pasa por el control de riesgo y aprobación humana existentes antes del Switch y los workers; el agente no ejecuta directamente los workers ni redacta la respuesta final. Las consultas documentales conservan su ruta RAG. Para una entrada Telegram, la aprobación HITL se solicita y espera en Telegram, no en Slack; Slack queda reservado para entradas Slack. La respuesta final de un mensaje Telegram/audio se redacta, valida, convierte con ElevenLabs y se envía por Telegram. Si el agente falla o no usa exactamente una vez la herramienta, se deriva a contingencia y no se ejecuta ningún worker.

Las respuestas de los workers y RAG reciben instrucciones condicionales para solicitudes de audio. La cadena **Redactar respuesta natural para voz** resume la respuesta en español de Chile, prioriza preguntas y confirmaciones pendientes y escribe cifras y unidades en palabras; por ejemplo, `CLP 1.500` pasa a «mil quinientos pesos chilenos». Si el primer resumen supera 200 caracteres, se solicita una sola vez otro más breve a partir de la respuesta original. Luego se valida que el texto no esté vacío, no supere 200 caracteres Unicode, no incluya cifras, enlaces, formato ni emojis, y conserve al menos una pregunta si el original la contiene. La validación no garantiza fidelidad semántica ni exactitud de cantidades: deben verificarse en pruebas reales. Si falla la redacción o la validación, se envía la respuesta original por escrito en el hilo, sin limitarla a 200 caracteres. **ElevenLabs** usa el modelo *Eleven Multilingual v2*, voz institucional y ajustes iniciales de Stability 0.65 y Clarity (similarity) 0.80. Slack publica el archivo de audio en el hilo original y su texto como comentario. Si falla la generación o carga de audio, se publica el texto de voz preparado.

**System Message para respuestas de voz:**

> Si el mensaje del usuario indica "Canal de respuesta: audio (Slack).", responde en español claro con un máximo de 200 caracteres. No uses enlaces, viñetas ni emojis. Redacta una respuesta breve y natural para escuchar. Para mensajes de texto, conserva las reglas de salida existentes.

**Evaluación de viabilidad:** la voz es viable de forma condicionada para consultas breves y frecuentes que se hacen con las manos ocupadas; reduce la fricción de entrada y limita la carga auditiva con respuestas cortas. No conviene para ambientes ruidosos, información sensible que no deba oírse en público ni respuestas extensas. Antes de escalar, comparar el costo de STT/TTS y la tasa de transcripción correcta con el tiempo de atención manual ahorrado. Las condiciones de contingencia evitan seguir con transcripciones vacías o errores de transcripción.

**Privacidad:** el workflow desactiva el guardado de ejecuciones exitosas, fallidas y manuales, así como el guardado del progreso, y elimina los binarios de los items que continúan por los nodos de limpieza. Esto afecta a todo el workflow y limita el diagnóstico en el historial. No garantiza la destrucción inmediata: los pasos de aprobación que esperan una respuesta pueden requerir persistir datos de la ejecución. Verifica los ajustes efectivos de la instancia y su almacenamiento binario; en n8n autohospedado, `N8N_DEFAULT_BINARY_DATA_MODE=default` utiliza memoria para los binarios, pero no elimina los datos que puedan persistirse en ejecuciones en espera. Revisa también la retención de Slack y de los proveedores de voz. Slack conserva los archivos según la configuración del espacio de trabajo. Se conserva el registro textual ya existente en Google Sheets. La eliminación segura de datos ya almacenados requiere un procedimiento de retención específico; estos ajustes no borran ejecuciones anteriores.

**Configuración de Slack:** el Slack App debe suscribirse al evento de archivos compartidos en el canal configurado y tener los scopes `files:read` y `files:write`, además de los permisos de mensajes existentes. La credencial de n8n debe poder leer y publicar archivos en ese canal. Si se agregan scopes, reinstala la app en el workspace y actualiza la credencial si corresponde. Un error 403 al descargar puede indicar permisos insuficientes o falta de acceso al archivo; se debe verificar con la configuración real de Slack.

## Importar el workflow

Importa primero `Checkpoint6_Final_VoiceAI_Telegram.json`; después importa los módulos `Checkpoint6_Slack_Channel_Module.json`, `Checkpoint6_Gmail_Channel_Module.json` y `Checkpoint6_Telegram_Voice_Module.json`. En cada módulo, confirma que `Call - Core Conversation Workflow` apunte al workflow central importado y que tenga mapeados los campos explícitos del contrato; no uses un campo único llamado `payload`. Si n8n asigna un ID diferente al importar, vuelve a seleccionar el workflow central en ese nodo. Importa también `Checkpoint6_RAG_Ingestion_Module.json` y los cuatro workflows de worker requeridos por el workflow central.

Configura las credenciales necesarias para Telegram, Slack, Gmail, HubSpot, Google Sheets, Google Drive, Supabase, OpenAI y ElevenLabs en los workflows que las usan. Verifica las conexiones y permisos de cada integración antes de activar los workflows. Activa por separado los tres adaptadores de canal y el workflow de ingesta RAG cuando sus credenciales y suscripciones estén verificadas.

## Autor

* **Autor:** Marcelo Troncoso
* **Repositorio GitHub:** [matronc/n8n_avanzado](https://github.com/matronc/n8n_avanzado)
* **Entorno:** n8n Enterprise
