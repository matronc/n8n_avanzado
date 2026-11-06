# GustoBot | Flujo omnicanal de pedidos y atención (Checkpoint 5)

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![OpenAI](https://img.shields.io/badge/OpenAI-LLM_Engine-412991?style=for-the-badge&logo=openai)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Gmail](https://img.shields.io/badge/Gmail-Integration-red?style=for-the-badge&logo=gmail)
![HubSpot](https://img.shields.io/badge/HubSpot-CRM-orange?style=for-the-badge&logo=hubspot)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)
![Google Drive](https://img.shields.io/badge/Google_Drive-Documents-yellow?style=for-the-badge&logo=googledrive)
![Supabase](https://img.shields.io/badge/Supabase-Vector_Store-3FCF8E?style=for-the-badge&logo=supabase)

---

## Descripción

**GustoBot** es un asistente virtual para gestionar pedidos, consultas y reclamos de un restaurante. El **Checkpoint 5** amplía el flujo multi-agente Manager-Worker del Checkpoint 4 con respuestas documentales basadas en RAG, ingesta y actualización de documentos desde Google Drive, y aprobaciones humanas (HITL) en Slack.

## Archivos y recursos

* **`Checkpoint5_marcelo_troncoso.json`**: workflow principal de n8n. Integra Slack, Gmail, HubSpot, Google Sheets, Google Drive y Supabase.
* **Workers**: `Worker_Registrar_Pedido.json`, `Worker_Atencion_Consultas.json`, `Worker_Atencion_Reclamos.json` y `Worker_Anfitrion.json`.
* **Planillas de datos y auditoría**: `Menu_GustoBot.xlsx`, `Pedidos_GustoBot.xlsx`, `Reclamos_GustoBot.xlsx` y `Logs_Ejecucion_GustoBot.xlsx`.
* **[Planilla de logs en Google Sheets](https://docs.google.com/spreadsheets/d/1NmEYqoD3hRMmt2Ni0o4Um-sMYp7XUr-zP_bda_Aw64Y/edit?usp=drivesdk)**.
* **[Evidencia visual del Checkpoint 4](https://docs.google.com/document/d/1o2TdufEyv6BqBbX8gGqfmB_-vun86KBp8qrNTTiD2_0/edit?usp=drive_link)**: referencia de la versión anterior; las capturas locales `image*.png` también corresponden a esa arquitectura y no muestran las nuevas funciones documentales del Checkpoint 5.

## Arquitectura del Checkpoint 5

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

## Importar y configurar

1. Importa `Checkpoint5_marcelo_troncoso.json` y los workers `Worker_Registrar_Pedido.json`, `Worker_Atencion_Consultas.json`, `Worker_Atencion_Reclamos.json` y `Worker_Anfitrion.json` en n8n.
2. Configura las credenciales y permisos de Slack, Gmail, HubSpot, Google Sheets, Google Drive, Supabase y los modelos de IA (OpenAI y Groq).
3. Configura la carpeta de Drive que observará la ingesta y prepara en Supabase la tabla `document_chunks` y la búsqueda vectorial requerida por el workflow.
4. Revisa las conexiones, IDs y valores propios del entorno en los nodos antes de activar los workflows. Ejecuta pruebas controladas para validar las integraciones y las rutas de aprobación.

## Autor

* **Autor:** Marcelo Troncoso
* **Repositorio GitHub:** [matronc/n8n_avanzado](https://github.com/matronc/n8n_avanzado)
* **Entorno:** n8n Enterprise
