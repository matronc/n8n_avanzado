# 🍔 GustoBot · Módulo 1: Agente Base y Motor de Razonamiento ReAct

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)

---

## 📌 Descripción del Proyecto

**GustoBot** es un asistente virtual inteligente basado en arquitecturas agénticas diseñado para la automatización de la toma de pedidos y el triaje comercial en un restaurante de comida rápida.

Este repositorio contiene la entrega técnica correspondiente al **Checkpoint 1** del proyecto integrador. Implementa un motor de razonamiento central en **n8n** bajo el paradigma de autonomía ReAct (Reasoning + Acting) conectado a canales de mensajería empresarial y persistencia de datos en la nube.

---

## 📂 Estructura de Entregables y Recursos

* 📄 **`checkpoint1_marcelo_troncoso.json`**: Archivo de configuración visual exportado de n8n que contiene la arquitectura completa del Agente Base.

* 📊 **Base de Datos de Pedidos:**
  
  [![Google Sheets](https://img.shields.io/badge/Ver_Planilla_de_Pedidos-Google_Sheets-34A853?style=for-the-badge&logo=googlesheets&logoColor=white)](https://docs.google.com/spreadsheets/d/1KIauiqRS1dviM6BVISADNyYmST6LbD9NcoHLTNf_B1c/edit?gid=0#gid=0)

---

## 🧠 Arquitectura y Componentes Obligatorios (Hito M1)

1. **Disparador (Trigger):**
   * **Slack Trigger:** Captura en tiempo real los mensajes recibidos en el canal `#gestor-pedidos` para su procesamiento asíncrono.

2. **Cerebro (AI Agent Node):**
   * Configurado en modo **Tools Agent** (`@n8n/n8n-nodes-langchain.agent`).
   * Conectado a un modelo de lenguaje de alto rendimiento a través de **Groq Chat Model** (`llama-3.1-8b-instant`).
   * **Guardrail de Presupuesto:** Configuración de iteraciones máximas (`maxIterations: 5`) para prevenir bucles lógicos y optimizar el consumo de tokens.

3. **Constitución del Agente (System Prompt):**
   * Estructura modular estricta: `[ROL Y ÁMBITO]` $\rightarrow$ `[OBJETIVO OPERATIVO]` $\rightarrow$ `[REGLAS DE USO DE HERRAMIENTAS]` $\rightarrow$ `[PROTOCOLO DE ESCALAMIENTO]` $\rightarrow$ `[ESTILO]`.
   * Limita el margen de acción del agente a la toma de pedidos y mitiga posibles alucinaciones de respuestas comerciales fuera de contexto.

4. **Herramienta Autónoma (Tool):**
   * **Google Sheets Tool (`Registrar_Pedido_Google_Sheets`)**: Herramienta conectada al Agente para la inserción estructurada de datos (cliente, teléfono, correo, detalle de pedido) directamente en la planilla en la nube.

5. **Observabilidad y Notificación (Output):**
   * **Slack Node:** Captura el resultado devuelto por el agente (`{{ $json.output }}`) y envía la confirmación de vuelta al usuario en el mismo hilo de conversación de Slack.

---

## 🚀 Vista Previa del Diagrama de Flujo en n8n

```text
[Chat de Slack] ──> [AI Agent (Agente GustoBot)] ──> [Respuesta a Slack]
                           │        ▲
                           │        │ (Tool Call)
                   [Groq Chat Model] [Registrar_Pedido_Google_Sheets]
```
