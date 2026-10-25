# 🍔 GustoBot · Arquitectura Multi-Agente, Orquestación Manager-Worker y Memoria Persistente (Checkpoints 2 & 3)

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)

---

## 📌 Descripción del Proyecto

**GustoBot** es un asistente virtual inteligente y omnicanal diseñado para la automatización de la toma de pedidos, atención de consultas y gestión de reclamos en un restaurante de comida rápida.

Este repositorio contiene la evolución técnica de los **Checkpoints 2 y 3**, migrando desde una estructura monolítica inicial hacia una **arquitectura multi-agente bajo el patrón Manager-Worker**, incorporando además protocolos de seguridad **Human-In-The-Loop (HITL)**, trazabilidad de ejecución y **memoria persistente con compresión y síntesis automática mediada por Agentes IA**.

---

## 📂 Archivos y Entregables del Repositorio

* 📄 **`Checkpoint2_marcelo_troncoso.json`**: Configuración del workflow principal (Manager) con enrutamiento inteligente, triaje mediante LLM, validación de riesgo y delegación a sub-workflows.
* 📄 **`Checkpoint3_marcelo_troncoso.json`**: Configuración avanzada que añade recuperación de historial por sesión (`Session_ID`), control de longitud de hilos conversacionales y un Agente IA de síntesis de memoria.
* 📊 **Base de Datos y Registro de Auditoría:** 
  [![Google Sheets](https://img.shields.io/badge/Ver_Planilla_Logs-Google_Sheets-34A853?style=for-the-badge&logo=googlesheets&logoColor=white)](https://docs.google.com/spreadsheets/d/1NmEYqoD3hRMmt2Ni0o4Um-sMYp7XUr-zP_bda_Aw64Y/edit?usp=drivesdk)

---

## 🏗️ Arquitectura y Componentes: Checkpoint 2 (Manager-Worker & HITL)

El flujo principal opera como un **Manager** centralizado que gestiona de forma determinista el ciclo de vida de cada solicitud:

1. **Ingesta y Mitigación de Payload ("Pasillo de la Muerte"):** Captura en tiempo real de mensajes en Slack y sanitización visual mediante nodos de edición para extraer únicamente la carga útil esencial (`mensaje`, `usuario_id`, `canal`), optimizando el uso de memoria.
2. **Router IA y Taxonomía Cerrada:** Inferencia mediante Groq para clasificar las entradas del usuario en intenciones strictly definidas del negocio:
   * `REGISTRAR_PEDIDO`
   * `CONSULTA_MENU`
   * `RECLAMO_SOPORTE`
   * `UNKNOWN`
3. **Escalamiento y Seguridad HITL (Human-in-the-Loop):** Evaluación condicional que verifica el nivel de riesgo (`risk == 'HIGH'`) o la certeza de la predicción (`confidence < 0.75`). Si se cumplen estas condiciones, el sistema desvía la ejecución y envía una alerta directa a los operadores en Slack sin invocar trabajadores automáticos.
4. **Bifurcación Determinista y Sub-Workflows (Workers):** Mediante un nodo Switch y llamadas síncronas (`Wait For Sub-Workflow Completion`), el Manager delega las tareas a lienzos especializados independientes:
   * `Worker_Registrar_Pedido`
   * `Worker_Atencion_Consultas`
   * `Worker_Atencion_Reclamos`
   * `Worker_Anfitrion`
5. **Trazabilidad y Auditoría:** Inserción automática de registros de ejecución en Google Sheets detallando marcas de tiempo, estados, workers invocados y flags de intervención humana.

---

## 🧠 Memoria Persistente y Síntesis Automática: Checkpoint 3

El **Checkpoint 3** dota al sistema de un mecanismo avanzado de retención de contexto a largo plazo:

1. **Identificación y Consulta por Sesión (`Session_ID`):** Cada interacción utiliza el ID de usuario de Slack para buscar y recuperar el historial acumulado almacenado en Google Sheets antes de procesar el mensaje entrante.
2. **Control de Umbral Conversacional:** Un nodo condicional evalúa si la longitud de la conversación supera los **5 mensajes** (`conversación > 5 mensajes`).
3. **Agente IA de Síntesis de Memoria:** Al rebasar el límite establecido, un **AI Agent** analiza el hilo reciente y actualiza de manera autónoma una celda resumen en Google Sheets estructurando:
   * **Asunto Principal del Caso**
   * **Puntos Clave** (preferencias, ítems o inconvenientes reportados)
   * **Acción Requerida o Estado Final**

---

## 🚀 Información de Autoría
* **Autor:** Marcelo Troncoso
* **Repositorio GitHub:** [matronc/n8n_avanzado](https://github.com/matronc/n8n_avanzado)
* **Entorno:** n8n Enterprise