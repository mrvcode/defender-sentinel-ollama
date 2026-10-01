# 🧠 Análisis automático de incidentes de Defender/Sentinel con IA local (Ollama)

Este proyecto es un **laboratorio práctico** que conecta **Microsoft Defender XDR / Microsoft Sentinel** con un modelo de **IA que corre en tu propio PC** mediante **Ollama**. Cuando se crea un incidente en Sentinel, una **Logic App** envía los datos del incidente a tu modelo local a través de un túnel **ngrok**, recibe un análisis en texto plano y lo publica como **comentario dentro del propio incidente**.

## ¿Por qué este proyecto?

- 💸 **Cero coste por tokens**: frente a las IA de pago (OpenAI, Copilot for Security, etc.), aquí el modelo corre en tu hardware. Solo necesitas tu PC, Ollama y una cuenta gratuita de ngrok.
- 🧪 **Laboratorio y aprendizaje**: está pensado para practicar integración de Sentinel con Logic Apps, reglas de automatización, identidades administradas, permisos IAM y flujos HTTP.
- 🔒 **Privacidad del modelo (con matices)**: el modelo es local, pero los datos del incidente **sí salen por el túnel** (ngrok). Por eso el proyecto insiste en **no usar datos reales** y en tratarlo solo como entorno de pruebas.
- ⚠️ **La IA se equivoca**: el comentario es una **ayuda al analista**, nunca un veredicto. Cada salida debe revisarla una persona.

## ¿Qué vas a encontrar?

- Guía paso a paso: requisitos, Ollama, ngrok, Logic App, permisos, regla de automatización y pruebas.
- Código completo de la Logic App (trigger de Sentinel → HTTP a Ollama → comentario en el incidente).
- Ejemplos reales de fallos y limitaciones del modelo, y buenas prácticas para trabajar con IA local en un SOC.

> **Ideal para**: prácticas de laboratorio, PoCs, estudiantes de ciberseguridad y cualquiera que quiera experimentar con IA local dentro de Microsoft Sentinel sin gastar en servicios de pago.
