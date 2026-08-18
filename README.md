# Léo Fagundes — Portfólio de Automação e Agentes de IA

De Product Manager para AI Product Engineer: os projetos abaixo documentam minha transição — construindo, quebrando e corrigindo automações e agentes de IA reais, aplicados a um negócio real (uma clínica de estética), usando n8n, LLMs (DeepSeek, Gemini) e integrações via WhatsApp, Google Sheets e Google Calendar.

Cada pasta abaixo é um projeto com case study completo: contexto do problema, decisões técnicas (com alternativas descartadas e o porquê), obstáculos reais enfrentados durante a construção, arquitetura final e aprendizados.

---

## Projetos

### 🤖 [Teka — Agente de Atendimento e Agendamento via WhatsApp](./teka/)
Agente conversacional com tool calling (LangChain + DeepSeek V4 Flash) que atende clientes de uma clínica de estética via WhatsApp: verifica disponibilidade, cria/altera/cancela agendamentos no Google Calendar, cadastra clientes novas e escala casos sensíveis para a proprietária. Inclui transcrição de áudio (Groq Whisper) e memória conversacional com Redis.
**Stack:** n8n · Evolution API · DeepSeek V4 Flash · Google Calendar/Sheets · Redis · Groq Whisper

### 📊 [Tize — Assistente de Gestão Operacional](./tize/)
Segunda aplicação da mesma arquitetura de agente, voltada para a gestão interna do negócio: briefing diário automático (contas a pagar, estoque, agenda) e um agente conversacional que responde perguntas e executa ações administrativas via WhatsApp.
**Stack:** n8n · Evolution API · DeepSeek V4 Flash · Google Calendar/Sheets · Redis

### 💰 [Automação de Fluxo de Caixa via Telegram](./cash-flow-telegram/)
Sistema de registro financeiro por mensagem de texto: o usuário lança uma movimentação de caixa no Telegram e o dado é estruturado, validado e salvo automaticamente em planilha, com sincronização para Excel. Projeto mais simples, sem IA generativa, mostrando a base de automação sobre a qual os agentes depois foram construídos.
**Stack:** n8n · Telegram Bot · Google Sheets · Power Query

> O projeto de extração automática de dados de clientes a partir de NF-e (usado para popular a base de clientes da clínica) está documentado dentro da pasta do Teka, como um sub-projeto de apoio.

---

## Sobre mim

Product Manager sênior (na prática) buscando transição para uma função híbrida de AI Product Engineer — construindo produtos com IA de ponta a ponta, não apenas especificando-os. Essa transição está documentada nos projetos acima, todos construídos e mantidos por mim, com uso real em um negócio real.

📍 São José, SC — Brasil
