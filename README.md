# 🍔 GustoBot · Módulo 1: Agente Base y Motor de Razonamiento ReAct

![n8n](https://img.shields.io/badge/n8n-Workflow-orange?style=for-the-badge&logo=n8n)
![Groq](https://img.shields.io/badge/Groq-LLM_Engine-purple?style=for-the-badge)
![Slack](https://img.shields.io/badge/Slack-Integration-blue?style=for-the-badge&logo=slack)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-Database-green?style=for-the-badge&logo=googlesheets)

---

## 📌 Descripción del Proyecto
**GustoBot** es un asistente virtual inteligente basado en arquitecturas agénticas para la toma de pedidos y triaje comercial en un restaurante de comida rápida. Este repositorio contiene el entregable correspondiente al **Checkpoint 1** del proyecto integrador, implementando el motor de razonamiento central en **n8n** bajo el paradigma de autonomía probabilística ReAct.

---

## 📂 Estructura de Entregables
- 📄 **`checkpoint1_marcelo_troncoso.json`**: Archivo de configuración visual exportado de n8n con el flujo completo del Agente Base.

---

## 🧠 Arquitectura y Componentes Obligatorios (Hito M1)

1. **Disparador (Trigger):**
   - **Slack Trigger:** Escucha y captura en tiempo real los mensajes desestructurados enviados por los clientes en el canal `#gestor-pedidos`.

2. **Cerebro (AI Agent Node):**
   - Configurado en modo **Tools Agent** (`@n8n/n8n-nodes-langchain.agent`).
   - Conectado a un modelo de lenguaje de alta velocidad mediante el **Groq Chat Model** (`openai/gpt-oss-20b` / Llama).
   - **Guardrail de Presupuesto:** Limitación estricta de iteraciones máximas (`maxIterations: 5`) para prevenir bucles lógicos infinitos.

3. **Constitución del Agente (System Prompt):**
   - Diseñado de forma modular (`[ROL Y ÁMBITO]` $\rightarrow$ `[OBJETIVO OPERATIVO]` $\rightarrow$ `[REGLAS DE USO DE HERRAMIENTAS]` $\rightarrow$ `[PROTOCOLO DE ESCALAMIENTO / GUARDRAILS]` $\rightarrow$ `[ESTILO]`).
   - Define explícitamente las restricciones operativas y mitiga alucinaciones corporativas.

4. **Herramienta Autónoma (Tool):**
   - **Google Sheets Tool (`Registrar_Pedido_Google_Sheets`)**: Acoplada lateralmente al agente. Cuenta con una descripción semántica orientada a evitar la instrucción huérfana y escribe estructuradamente en la planilla `Pedidos_GustoBot`.

5. **Observabilidad y Notificación (Output):**
   - **Slack Node:** Nodo final que captura el resultado de la ejecución del agente (`{{ $json.output }}`) y responde de manera integrada al canal y en el hilo correspondiente del usuario.

---

## 🚀 Vista Previa del Flujo en n8n

```text
[Chat de Slack] ──> [AI Agent (Agente GustoBot)] ──> [Respuesta a Slack]
                           │        ▲
                           │        │ (Tool Call)
                   [Groq Chat Model] [Registrar_Pedido_Google_Sheets]
