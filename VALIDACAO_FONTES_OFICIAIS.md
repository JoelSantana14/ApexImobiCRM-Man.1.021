# Validação de fonte oficial — Gemini API

**Data de consulta:** 10/10/2026

A auditoria conferiu o identificador do modelo usado no backend do ApexImob (`gemini-3.8-flash`). A documentação oficial consultada lista esse identificador como modelo estável da Gemini API, atualizado em setembro de 2026; portanto, o nome configurado no código é válido na data da consulta.

Fontes oficiais do Google:

- [Gemini 3.8 Flash — página do modelo](https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash)
- [Gemini API — catálogo de modelos](https://ai.google.dev/gemini-api/docs/models)
- [Gemini API — notas de lançamento (02/09/2026)](https://ai.google.dev/gemini-api/docs/changelog)

A validação confirma o nome do modelo, mas não testa chamadas reais: este pacote não contém uma `GEMINI_API_KEY` configurada.