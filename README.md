# GustoBot | Flujo omnicanal de pedidos y atención (Checkpoint 6)

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![OpenAI](https://img.shields.io/badge/OpenAI-LLM_Engine-412991?style=for-the-badge&logo=openai)
![ElevenLabs](https://img.shields.io/badge/ElevenLabs-Voice_AI-black?style=for-the-badge)
![Voice AI](https://img.shields.io/badge/Voice_AI-Audio-purple?style=for-the-badge)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Gmail](https://img.shields.io/badge/Gmail-Integration-red?style=for-the-badge&logo=gmail)
![HubSpot](https://img.shields.io/badge/HubSpot-CRM-orange?style=for-the-badge&logo=hubspot)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)
![Google Drive](https://img.shields.io/badge/Google_Drive-Documents-yellow?style=for-the-badge&logo=googledrive)
![Supabase](https://img.shields.io/badge/Supabase-Vector_Store-3FCF8E?style=for-the-badge&logo=supabase)

---

## Descripción

**GustoBot** es un asistente virtual para gestionar pedidos, consultas y reclamos de un restaurante. El **Checkpoint 6**, como preentrega de Ecosistemas Multimedia de Audio, incorpora Voice AI al flujo omnicanal: recibe notas de voz por Slack, las transcribe, procesa la solicitud con la arquitectura existente y responde con audio generado, manteniendo una alternativa de respuesta por texto. También conserva las funciones de RAG documental, ingesta desde Google Drive y aprobación humana (HITL) en Slack incorporadas en checkpoints anteriores.

## Archivos y recursos

* **`Checkpoint6_marcelo_troncoso.json`**: workflow principal de n8n. Integra Slack, Gmail, HubSpot, Google Sheets, Google Drive, Supabase, OpenAI Whisper y ElevenLabs.
* **Workers**: `Worker_Registrar_Pedido.json`, `Worker_Atencion_Consultas.json`, `Worker_Atencion_Reclamos.json` y `Worker_Anfitrion.json`.
* **Planillas de datos y auditoría**: `Menu_GustoBot.xlsx`, `Pedidos_GustoBot.xlsx`, `Reclamos_GustoBot.xlsx` y `Logs_Ejecucion_GustoBot.xlsx`.
* **[Planilla de logs en Google Sheets](https://docs.google.com/spreadsheets/d/1NmEYqoD3hRMmt2Ni0o4Um-sMYp7XUr-zP_bda_Aw64Y/edit?usp=drivesdk)**.
* **[Evidencia visual del Checkpoint 4](https://docs.google.com/document/d/1o2TdufEyv6BqBbX8gGqfmB_-vun86KBp8qrNTTiD2_0/edit?usp=drive_link)**: referencia de una versión anterior; las capturas locales `image*.png` no muestran las funciones documentales del Checkpoint 5 ni el módulo de Voice AI del Checkpoint 6.

## Pre entrega Ecosistemas Multimedia de Audio (Voice AI)

Esta preentrega corresponde al **Checkpoint 6** e integra interacción por voz en Slack con el flujo de atención de GustoBot.

## Arquitectura del Checkpoint 6

### 1. Entrada multicanal y CRM

* **Slack** recibe mensajes y los normaliza antes de enviarlos al Manager.
* **Gmail** consulta cada minuto los correos no leídos. El flujo descarta respuestas automáticas y remitentes `no-reply`, y normaliza remitente, asunto y cuerpo.
* Para solicitudes por correo, **HubSpot** busca el contacto y lo crea o actualiza según corresponda.

### 2. Clasificación y workers

El router normaliza la clasificación y deriva cada solicitud:

* `REGISTRAR_PEDIDO` → `Worker_Registrar_Pedido`
* `CONSULTA_MENU` → `Worker_Atencion_Consultas`
* `RECLAMO_SOPORTE` → `Worker_Atencion_Reclamos`
* `CONSULTA_DOCUMENTAL` → búsqueda RAG en la base documental de Supabase
* `UNKNOWN` y solicitudes que requieren intervención → atención humana

Los workers se ejecutan como sub-workflows. El flujo registra las ejecuciones y mantiene memoria conversacional en Google Sheets.

### 3. Consultas documentales (RAG)

Las preguntas sobre políticas, procedimientos y preguntas frecuentes se responden con contexto recuperado desde **Supabase**, en lugar de depender únicamente del conocimiento del modelo. La búsqueda devuelve hasta cuatro fragmentos relevantes; si no encuentra contexto, el flujo utiliza una respuesta alternativa.

### 4. Ingesta y actualización de documentos

La rama de ingesta del workflow principal observa documentos nuevos o modificados en una carpeta de **Google Drive**, descarga cada archivo, calcula su hash SHA-256, extrae y fragmenta el texto y guarda sus embeddings en la tabla `document_chunks` de Supabase. Los duplicados exactos se agrupan para evitar reingestas innecesarias.

Si un documento cargado se parece semánticamente a una versión ya indexada (similitud de **0,88 o superior**), el flujo solicita aprobación por Slack antes de reemplazarlo. Si se aprueba, indexa la nueva versión y elimina los fragmentos de la anterior identificada; si se rechaza, conserva el índice actual.

### 5. Revisión humana y respuestas

Las solicitudes que requieren intervención se canalizan mediante **HITL en Slack**, con el estado de la decisión registrado en Google Sheets. Para Gmail, el flujo puede preparar un borrador para revisión o enviar una respuesta automática según el resultado y las validaciones del correo.

El flujo también conserva el procesamiento analítico de la memoria conversacional: cuando el historial supera los **5 mensajes**, actualiza el resumen y los datos del cliente en Google Sheets.

### 6. Voice AI por Slack

* **Entrada de audio:** el flujo identifica archivos de audio recibidos en Slack, extrae el binario y usa **OpenAI Whisper** para transcribirlos en español. La transcripción se incorpora a la misma ruta de clasificación y atención que los mensajes de texto.
* **Respuesta hablada:** para solicitudes de audio, GustoBot redacta una respuesta oral breve en español y limita su extensión a un máximo de **200 caracteres**. **ElevenLabs** genera el audio y Slack lo publica como archivo en el hilo original, con el texto como comentario.
* **Alternativas ante errores:** si la transcripción no es válida o falla la generación/publicación del audio, el flujo intenta entregar una respuesta por texto. Los archivos de Slack no compatibles se ignoran.
* **Privacidad:** el workflow desactiva el guardado de ejecuciones exitosas, fallidas, manuales y del progreso, y elimina los binarios de los items posteriores a su procesamiento. Esto no controla la retención de Slack ni de los proveedores externos; revisa las políticas de almacenamiento y retención de cada servicio.

## Importar y configurar

1. Importa `Checkpoint6_marcelo_troncoso.json` y los workers `Worker_Registrar_Pedido.json`, `Worker_Atencion_Consultas.json`, `Worker_Atencion_Reclamos.json` y `Worker_Anfitrion.json` en n8n.
2. Configura las credenciales y permisos de Slack, Gmail, HubSpot, Google Sheets, Google Drive, Supabase, OpenAI y ElevenLabs.
3. Configura la carpeta de Drive que observará la ingesta y prepara en Supabase la tabla `document_chunks` y la búsqueda vectorial requerida por el workflow.
4. Verifica que Slack permita recibir y descargar los archivos de audio, y que la instancia de n8n tenga configurado el almacenamiento binario. Revisa también los límites y costos de Whisper y ElevenLabs.
5. Revisa las conexiones, IDs y valores propios del entorno en los nodos antes de activar los workflows. Ejecuta pruebas controladas para validar las integraciones, las respuestas de audio y texto, y las rutas de aprobación.

## Autor

* **Autor:** Marcelo Troncoso
* **Repositorio GitHub:** [matronc/n8n_avanzado](https://github.com/matronc/n8n_avanzado)
* **Entorno:** n8n Enterprise
