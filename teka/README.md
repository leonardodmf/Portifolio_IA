# Case Study: Teka — AI Scheduling Agent for Aesthetic Clinic

**Role:** AI Product Engineer (solo)  
**Stack:** n8n · Evolution API (Baileys) · DeepSeek V4 Flash · Google Calendar · Google Sheets · Redis · Groq Whisper  
**Status:** In active development / pre-production testing  
**Client:** Clínica Vitalle Estética — Santa Catarina, Brasil

---

## 1. CONTEXTO E PROBLEMA

### Problema de negócio
A Clínica Vitalle Estética, clínica de estética e harmonização facial conduzida pela Dra. Bianca, operava seu agendamento 100% via aplicativo proprietário (Masters APP) sem API pública. Isso impossibilitava qualquer automação. O atendimento ao cliente acontecia de forma manual no WhatsApp pessoal da médica, gerando:

- Perda de agendamentos fora do horário comercial
- Ausência de confirmações e lembretes automáticos
- Nenhum cadastro estruturado de clientes — dados dispersos em agenda de contatos e NFs
- Tempo da Dra. Bianca consumido com triagem de mensagens repetitivas (valores, disponibilidade, pós-procedimento)

### Usuário / beneficiário final
- **Primário:** clientes da clínica que entram em contato via WhatsApp
- **Secundário:** Dra. Bianca — que recebe notificações apenas quando necessário e mantém visibilidade da agenda via Google Calendar

### Restrições
- **Orçamento:** custo operacional próximo de zero (Google Workspace gratuito, DeepSeek ~$0.002/1K tokens, Groq Whisper free tier)
- **Infraestrutura:** n8n e Evolution API hospedados na Cloudfy (plataforma gerenciada)
- **Agenda da clínica:** sistema atual (Masters APP) sem API — necessidade de migração completa para Google Calendar
- **Perfil da usuária final:** Dra. Bianca não tem familiaridade técnica — toda interface precisa ser WhatsApp ou Google Calendar nativo
- **Projeto real:** este é simultaneamente o caso zero do produto comercial Agentize AI, usado como portfólio e laboratório de produto

---

## 2. DECISÕES TÉCNICAS

### Plataforma de agendamento — Google Calendar vs Cal.com vs Simplybook.me

| Opção | Custo | API | Decisão |
|---|---|---|---|
| Cal.com (self-hosted) | ~R$0 (infra) | REST + Webhooks | Descartado — complexidade de manutenção |
| Simplybook.me | ~R$60/mês | REST + Webhooks | Descartado — custo mensal por cliente |
| Google Calendar | Gratuito | Google API nativa | **Escolhido** |

**Porquê:** Google Calendar é universal, custo zero, tem node nativo no n8n e a Dra. Bianca já conhece a interface. O agente substitui o portal de auto-agendamento — a clínica não precisa de outra interface visual além do GCal.

---

### Armazenamento de dados — Google Sheets vs banco relacional

**Escolha:** Google Sheets como camada de dados principal (clientes, procedimentos, FAQ, lembretes), com Redis exclusivamente para sessão conversacional.

**Porquê:**
- Sheets é editável pela Dra. Bianca sem nenhum treinamento
- Elimina necessidade de painel administrativo separado
- Replicável para novos clientes — cada clínica tem sua própria planilha
- Redis para sessão: TTL nativo (36h), mais rápido que PostgreSQL para leitura/escrita frequente, sem necessidade de limpeza manual

PostgreSQL foi considerado inicialmente para sessão mas descartado — overhead desnecessário para dados efêmeros de conversa.

---

### Modelo de IA — DeepSeek V4 Flash vs alternativas

**Escolha:** DeepSeek V4 Flash via API compatível com OpenAI.

**Porquê:**
- Custo ~10x menor que GPT-4o para o mesmo volume
- Suporte sólido a tool calling — essencial para as 6 ferramentas do agente
- API OpenAI-compatible — sem lock-in, troca de modelo sem refatoração
- Temperatura 0.4 — respostas consistentes sem robotismo excessivo

---

### Transcrição de áudio — Groq Whisper vs OpenAI Whisper

**Escolha:** Groq Whisper Large v3 Turbo.

**Porquê:** free tier generoso, latência inferior ao OpenAI Whisper, API OpenAI-compatible. Para uma clínica onde a maioria das clientes usa áudio no WhatsApp, transcrição gratuita é diferencial de margem.

---

### Arquitetura de memória — memória nativa do n8n vs Redis externo

**Escolha:** Redis externo com injeção manual do histórico no prompt.

**Porquê:** a memória nativa do n8n (`Window Buffer Memory`) não persiste entre execuções de webhook — cada mensagem dispara uma nova execução. Redis resolve isso com chave por número de WhatsApp (`teka:{remoteJid}`), TTL de 36h e histórico das últimas 36 mensagens injetado no system prompt a cada request.

---

### Gestão de dados de cliente — cadastro progressivo

**Decisão de produto:** coleta mínima no primeiro contato (nome + CPF), complemento no momento do agendamento (endereço + email).

**Porquê:** pedir todos os dados de uma vez gera fricção e abandono. A justificativa para o cadastro inicial usa contexto real — "estamos migrando de sistema de agendamento".

---

### Sub-workflows como ferramentas do agente

**Decisão:** cada tool do agente (`salvar_cliente`, `criar_agendamento`, etc.) é um workflow n8n separado chamado via `toolWorkflow`.

**Porquê:** separação de responsabilidades, testabilidade independente, replicabilidade por cliente. Cada sub-workflow tem seu próprio `Execute Workflow Trigger` e retorna um campo `output` que o agente lê para decidir o próximo passo.

---

## 3. OBSTÁCULOS E ITERAÇÃO

### Webhook disparando para todos os eventos da Evolution API
**Problema:** a Evolution API enviava eventos de `connection.update`, `qrcode.updated` e outros além de `messages.upsert`. O fluxo processava todos como mensagens.  
**Solução:** configuração na própria Evolution API para filtrar apenas `MESSAGES_UPSERT`. Mais limpo do que filtrar no n8n.

### API key exposta no payload do webhook
**Problema:** a Evolution API enviava a API key dentro do body do webhook (`"apikey": "..."`), expondo a chave em logs.  
**Diagnóstico:** comportamento controlado pela variável de ambiente `EXPOSE_IN_FETCH_INSTANCES` no servidor.  
**Solução temporária:** regeneração da instância. Solução definitiva: solicitação à Cloudfy para desativar a variável.

### Paralelização de nós no n8n quebrava o fluxo
**Problema:** os quatro nós de carregamento de dados (Redis, Buscar Cliente, Procedimentos, FAQ) foram conectados em paralelo a partir do "Normalizar Mensagem". O n8n executava apenas os dois primeiros, deixando os demais sem dados.  
**Diagnóstico:** o n8n não paraleliza nativamente dentro de um workflow de webhook de forma confiável.  
**Solução:** sequencialização completa dos nós. Perda de performance irrelevante (~800ms) para o caso de uso.

### Redis retornando null bloqueava o fluxo
**Problema:** cliente nova sem sessão → Redis retorna null → n8n interpreta como "sem dados" → fluxo para.  
**Solução:** `continueRegularOutput` no nó Redis GET + tratamento defensivo com `try/catch` no nó "Montar Contexto", garantindo fallback para `{}` quando não há sessão.

### Chave Redis duplicada (`teka:teka:...`)
**Problema:** o GET do Redis tinha a chave configurada como `teka:{{ expressão }}` onde a expressão já incluía o prefixo `teka:`, resultando em `teka:teka:{remoteJid}`. O SET usava chave diferente — sessão nunca era recuperada.  
**Diagnóstico:** inspeção visual do campo Key no nó, comparação entre GET e SET.  
**Solução:** correção da Key do GET para `teka:{{ $('Normalizar Mensagem').first().json.remoteJid }}` sem prefixo duplicado.

### Google Sheets retornando vazio para clientes novas
**Problema:** quando o número não estava cadastrado, o nó "Buscar Cliente" não retornava dados e o n8n parava a execução.  
**Solução:** ativar "Always Output Data" especificamente no nó "Buscar Cliente" — único nó onde resultado vazio é comportamento esperado.

### Envio de resposta quebrando JSON
**Problema:** respostas do agente com aspas, quebras de linha ou caracteres especiais quebravam o JSON body do HTTP Request para a Evolution API.  
**Solução:** trocar de Raw JSON body para `bodyParameters` individuais com `contentType: json` — o n8n serializa e escapa automaticamente.

### Campo `delay` rejeitado pela Evolution API
**Problema:** a Evolution API exigia `delay` como número inteiro, não string. O n8n enviava `"1200"` (string).  
**Solução:** usar `{{ 1200 }}` ou configurar o campo como tipo Number nos body parameters.

### Extração de dados de clientes de 119 NFs em PDF
**Problema:** base histórica de clientes inexistente em formato estruturado — dados dispersos em NFs da prefeitura.  
**Solução:** script Python com `pdfplumber` que extrai o bloco "TOMADOR DE SERVIÇOS" de cada NF usando regex, deduplica por CPF mantendo dados mais recentes e soma valor total por cliente. Executado via Google Colab com Drive montado.

---

## 4. ARQUITETURA FINAL

### Fluxo ponta a ponta — atendimento

```
Cliente (WhatsApp)
    ↓
Evolution API (Baileys) — instância teka_dona_harmonia
    ↓ webhook POST
n8n — Webhook Trigger
    ↓
Responder 200 OK (paralelo — evita timeout)
    ↓
Filtrar Mensagens Válidas (IF)
    → descarta: fromMe=true, grupos (@g.us)
    ↓
É Áudio? (IF — messageType === 'audioMessage')
    → SIM: Download Base64 → Groq Whisper → texto transcrito
    → NÃO: extrai message.conversation ou extendedTextMessage.text
    ↓
Normalizar Mensagem (Code)
    → remoteJid, pushName, messageText, isAudio
    ↓
Carregar Sessão Redis (GET teka:{remoteJid})
    ↓
Buscar Cliente — Google Sheets (aba: clientes)
    ↓
Carregar Procedimentos — Google Sheets (aba: procedimentos)
    ↓
Carregar FAQ Intercorrências — Google Sheets (aba: faq_intercorrencias)
    ↓
Montar Contexto (Code)
    → merge: sessão + cliente + procedimentos + FAQ + histórico
    ↓
Teka — Agente IA (LangChain Agent)
    → modelo: DeepSeek V4 Flash
    → tools disponíveis:
        - verificar_disponibilidade → Google Calendar (sub-workflow)
        - criar_agendamento → Google Calendar + Sheets lembretes
        - alterar_agendamento → Google Calendar + Sheets
        - cancelar_agendamento → Google Calendar + Sheets
        - salvar_cliente → Google Sheets (aba: clientes)
        - notificar_dra_bianca → Evolution API (WhatsApp)
    ↓
Atualizar Sessão (Code)
    → append histórico, trim para 36 mensagens
    ↓
Salvar Sessão Redis (SET teka:{remoteJid}, TTL 129600s)
    ↓
Enviar Resposta WhatsApp
    → Evolution API POST /message/sendText/teka_dona_harmonia
```

### Fluxo de lembretes (Schedule — a cada hora)

```
Schedule Trigger (1h)
    ↓
Carregar lembretes_agendamentos (Sheets — status=pendente)
    ↓
Avaliar Lembretes (Code)
    → janela 24h antes → lembrete_24h
    → janela 6h antes (se não confirmou) → lembrete_6h
    → janela 2h antes (se ainda não confirmou) → aviso_bianca
    ↓
É Aviso para Bianca? (IF)
    → SIM: WhatsApp para Dra. Bianca
    → NÃO: WhatsApp para cliente
```

### Estrutura de dados (Google Sheets)

```
Planilha: Clínica Vitalle Estética
├── clientes           (cadastro — nome, CPF, telefone, email, endereço)
├── procedimentos      (nome, valor, duração, buffer, instruções pré/pós)
├── faq_intercorrencias (gatilho, resposta, escalar: SIM/NÃO/CONDICIONAL)
├── lembretes_agendamentos (event_id_gcal, status_lembrete, timestamps)
└── config_clinica     (telefone_bianca, horários, IDs de sistema)
```

### Diagrama simplificado

```
[WhatsApp Cliente]
       ↓
[Evolution API] ←→ [n8n Webhook]
                         ↓
                  [Redis — Sessão]
                  [Google Sheets — Dados]
                         ↓
                  [DeepSeek V4 Flash]
                         ↓
              ┌──────────┴──────────┐
         [Google Calendar]    [WhatsApp Bianca]
              (agenda)         (notificações)
```

---

## 5. RESULTADO E MÉTRICAS

### Estado atual
- Fluxo principal funcionando end-to-end: recebe mensagem → responde via WhatsApp
- Cadastro de clientes novas operacional (nome + CPF → Google Sheets)
- Sub-workflows de Calendar criados e integrados, em fase de teste
- Lembretes automáticos estruturados, aguardando dados reais para validação

### Ganhos esperados (estimativa pré-produção)

| Métrica | Antes | Depois (estimado) |
|---|---|---|
| Tempo de resposta fora do horário | Sem resposta | Imediato (24/7) |
| Confirmações de agendamento | Manual (100%) | Automatizado (100%) |
| Lembretes enviados | 0 | 2 por agendamento |
| Tempo da Dra. em triagem/dia | ~40min | <5min (só escalamentos) |
| Cadastro de clientes novas | Manual | Automático via conversa |

### Escalabilidade
Arquitetura projetada para replicação: novo cliente = nova instância Evolution + nova planilha Google Sheets + duplicação do workflow n8n com IDs atualizados. Redis e n8n compartilhados entre clientes com isolamento por prefixo de chave.

---

## 6. APRENDIZADOS

### O que faria diferente

**Testar o payload do webhook antes de construir o fluxo inteiro.**  
A estrutura real da mensagem da Evolution API diferia do esperado (campo `message` ausente em eventos de `connection.update`, `messageType` variando por dispositivo). Isso causou retrabalho no IF de áudio e no nó de normalização. Um teste de webhook isolado teria revelado isso em 10 minutos.

**Sequenciar desde o início, nunca paralelizar em n8n sem validação.**  
A tentativa de paralelizar os quatro nós de dados custou tempo de debug. Em n8n, paralelismo dentro de um único workflow de webhook não é confiável. Sequência é sempre mais segura.

**Validar IDs e placeholders antes de importar JSON.**  
Manter um checklist de placeholders (`ID_CALENDARIO_GCAL`, `SUA_API_KEY_EVOLUTION`, etc.) e verificar cada um antes do primeiro teste teria poupado vários ciclos de erro/diagnóstico.

### Lições técnicas

- **Redis é a escolha certa para sessão conversacional** — TTL nativo, velocidade, sem manutenção. PostgreSQL adiciona complexidade sem benefício para dados efêmeros.
- **Expressões n8n em nós Redis são case-sensitive e compostas** — `teka:{{ expressão }}` vs `{{ 'teka:' + expressão }}` produzem resultados diferentes dependendo de onde o prefixo é declarado. Sempre inspecionar o valor real da chave antes de assumir que está correto.
- **`Always Output Data` é essencial em nós de lookup** — qualquer nó que busca dados externos e pode não encontrar nada precisa dessa configuração ou o fluxo para silenciosamente.
- **A Evolution API envia eventos além de mensagens** — configurar o filtro de eventos na origem (painel da Evolution) é mais limpo do que filtrar no n8n.
- **LLMs tendem ao verbose** — sem instrução explícita de concisão, o agente confirma cada ação, repete o nome do usuário e usa saudações excessivas. Instruções como "não confirme que recebeu dados, vá direto ao próximo passo" são necessárias.

### Lição de produto

**O usuário final da ferramenta (Dra. Bianca) não pode depender de nenhuma interface nova.**  
Toda decisão de arquitetura foi orientada por esse princípio: agenda no Google Calendar (ela já usa), dados no Sheets (editável sem treinamento), notificações no WhatsApp (canal que já está aberto). O agente é invisível para ela — ela só vê o resultado.

Isso também define o modelo de venda: você não está vendendo tecnologia, está vendendo tempo da médica de volta.
