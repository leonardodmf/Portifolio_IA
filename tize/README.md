# Case Study — Tize / Taya: AI Virtual Assistant for SMB Operations
## Agentize AI · 2025–2026

---

## 1. CONTEXTO E PROBLEMA

**Problema de negócio**

Uma clínica de estética de pequeno porte operava sem nenhum sistema de gestão integrado. Contas a pagar eram controladas mentalmente ou em papel. O estoque de insumos clínicos (toxina botulínica, ácido hialurônico, fios de PDO) era gerenciado de memória. A agenda estava em um aplicativo proprietário sem API externa (Masters APP). Não havia visibilidade financeira proativa — a proprietária só descobria que uma conta estava vencida quando o fornecedor cobrava.

O problema central: **tempo e atenção do gestor drenados por tarefas operacionais repetitivas**, sem nenhuma automação em lugar algum.

**Contexto do projeto**

O projeto nasceu como laboratório pessoal de um desenvolvedor independente construindo uma empresa de automações com IA (Agentize AI). A clínica da parceira foi usada como caso real de validação — não como cliente pagante, mas como ambiente de produção para aprender, errar e iterar com dados reais.

**Usuários e beneficiários**

- Proprietária da clínica (Bianca): recebe alertas, faz perguntas, registra movimentos de estoque e agenda compromissos via WhatsApp
- Sócio operacional (Léo): monitora financeiro, agenda e opera o agente pelo mesmo canal

**Restrições**

| Restrição | Detalhe |
|-----------|---------|
| Orçamento | Mínimo. Toda a infraestrutura deveria custar menos de R$200/mês |
| Ferramentas | n8n já disponível via Cloudfy. Sem budget para ferramentas adicionais pagas |
| Canal obrigatório | WhatsApp — única interface que o usuário final aceitaria usar |
| Agenda | Masters APP sem API pública — integração direta impossível |
| Tempo | Projeto paralelo ao trabalho principal. Desenvolvimento incremental, noite e fim de semana |
| Expertise | Desenvolvedor aprendendo n8n e LangChain durante o próprio projeto |

---

## 2. DECISÕES TÉCNICAS

Em ordem cronológica:

**1. Google Sheets como banco de dados**
*Alternativa considerada:* banco relacional (PostgreSQL), Airtable.
*Decisão:* Google Sheets.
*Porquê:* Usuário final já conhece planilhas. Edição manual possível sem dependência do agente. API gratuita e nativa no n8n. Custo zero. Para o volume de dados de uma PME (dezenas de linhas), performance é irrelevante.

**2. WhatsApp via Evolution API (Baileys)**
*Alternativa considerada:* Z-API (pago, ~R$80/mês), Twilio WhatsApp Business API (caro e burocrático).
*Decisão:* Evolution API self-hosted na Cloudfy.
*Porquê:* Custo zero de API por mensagem. Já incluído no plano Cloudfy. Controle total da instância. Suporte a grupos nativamente.

**3. Arquitetura em dois workflows separados**
*Alternativa considerada:* tudo em um único workflow com roteamento condicional.
*Decisão:* dois workflows independentes — Briefing Diário (Schedule Trigger) e Assistente Conversacional (Webhook Trigger).
*Porquê:* Falha em um não afeta o outro. Triggers fundamentalmente diferentes. Facilita debug e manutenção. Permite vender os produtos separadamente.

**4. Google Calendar como sistema de agenda**
*Alternativa considerada:* Cal.com (self-hosted), Simplybook.me (~R$60/mês), manter Masters APP.
*Decisão:* Google Calendar.
*Porquê:* Masters APP não tem API — integração impossível. Google Calendar é gratuito, API madura, nó nativo no n8n, usuários já têm conta Google. Cal.com foi considerado para o futuro como alternativa mais robusta para agendamento de clientes externos.

**5. Modelo de IA — DeepSeek V4 Flash**
*Alternativas consideradas e descartadas:*
- Gemini 2.5 Flash (Cloudfy): atingiu cota gratuita rapidamente em produção
- Gemini 1.5 Pro: não aparecia na lista de modelos disponíveis
- Gemini 2.0 Flash: erro 429 (quota exceeded) no free tier
- Modelos Cloudfy (Mistral, Llama 3.1): fracos para tool calling
- GPT-4o Mini: custo maior sem justificativa técnica
*Decisão:* DeepSeek V4 Flash via API OpenAI-compatible.
*Porquê:* Custo irrisório (~$0.14/1M tokens input). Suporte nativo a Tool Calling. Compatível com nó OpenAI Chat Model do n8n. Estimativa de custo mensal para o caso de uso: menos de $0.10.

**6. Redis para memória conversacional**
*Alternativa considerada:* PostgreSQL (disponível e gratuito na Cloudfy), Simple Memory (nativo n8n).
*Decisão:* Redis.
*Porquê:* TTL nativo — sessões de 36h expiram automaticamente sem manutenção. Simple Memory perde contexto quando o workflow reinicia. PostgreSQL acumula histórico indefinidamente sem mecanismo de expiração nativo. Redis é a solução mais limpa para memória conversacional com janela de tempo definida.

**7. Briefing em série, não paralelo**
*Problema identificado:* tentativa inicial de rodar leitura de Sheets (Contas + Estoque) e Calendar em paralelo, com os três convergindo para um nó de Code.
*Decisão:* execução em série com nós Aggregate entre cada leitura.
*Porquê:* O nó Code referencia dados de nós anteriores via `$('Nome do Nó').first().json`. Em execução paralela, o n8n não garante que todos os nós upstream já executaram quando o Code roda. Em série com Aggregate, os dados chegam consolidados e a referência é determinística.

---

## 3. OBSTÁCULOS E ITERAÇÃO

**OAuth Google Calendar expirando a cada 7 dias**
*Sintoma:* agente parava de criar eventos sem aviso.
*Causa:* Google Cloud project em modo "Testing" — tokens OAuth expiram em 7 dias.
*Solução:* publicar o OAuth consent screen como "In Production" no Google Cloud Console.

**Datas chegando como DD/MM/YY do Google Sheets**
*Sintoma:* briefing exibia "Invalid Date" para todos os vencimentos.
*Causa:* Google Sheets envia datas via API no formato `DD/MM/YY` (dois dígitos no ano) quando células são formatadas como Data. `new Date("20/05/26")` retorna Invalid Date em JavaScript.
*Solução:* regex específico para capturar o formato e reconstruir o objeto Date manualmente:
```javascript
const matchBR2 = str.match(/^(\d{2})\/(\d{2})\/(\d{2})$/);
if (matchBR2) {
  const ano = parseInt(matchBR2[3]) + 2000;
  return new Date(ano, parseInt(matchBR2[2]) - 1, parseInt(matchBR2[1]));
}
```

**Timezone errado nos horários de eventos**
*Sintoma:* evento às 20h aparecia como "23:00" na mensagem do briefing.
*Causa:* `new Date(evento.start.dateTime).toLocaleTimeString()` converte para UTC antes de formatar.
*Solução:* extrair o horário diretamente da string ISO sem passar pelo objeto Date:
```javascript
evento.start.dateTime.substring(11, 16) // extrai "20:00" de "2026-05-27T20:00:00-03:00"
```

**Schedule Trigger disparando no horário errado**
*Sintoma:* briefing configurado para 9h disparava às 6h ou 1h.
*Causa:* n8n roda em UTC por padrão. Configuração do horário sem ajuste de timezone.
*Solução:* configurar timezone diretamente no workflow (Settings do workflow → Timezone → America/Sao_Paulo).

**Agente inventando datas ao agendar eventos**
*Sintoma:* pedido para agendar "amanhã" resultava em evento em datas aleatórias do passado.
*Causa:* data atual não estava disponível para o modelo. System Message não processa expressões `{{ }}` do n8n — é campo de texto estático.
*Solução:* injetar data via User Message usando expressão n8n:
```
={{ 'Hoje é ' + $now.format('DD/MM/YYYY') + '. ' + $json.pushName + ': ' + $json.mensagem }}
```

**Google Sheets Tool nodes exigindo parâmetro com $fromAI**
*Sintoma:* erro "No parameters are set up to be filled by AI" ao tentar usar tool de leitura.
*Causa:* nós do tipo Tool no n8n exigem pelo menos um campo marcado com o botão ✨ (fromAI), mesmo para operações de leitura simples.
*Solução:* ativar o botão ✨ em pelo menos um campo do nó tool.

**JSON inválido no nó de envio WhatsApp**
*Sintoma:* erro "JSON parameter needs to be valid JSON" ao tentar enviar resposta.
*Causa:* expressão `{{ $json.output }}` dentro de JSON raw quebra a sintaxe quando o valor contém aspas ou quebras de linha.
*Solução:* trocar de "JSON raw" para "Using Fields Below" com Content Type JSON, e usar modo Expression no campo value sem as chaves `{{ }}`.

**Agente identificando todos os usuários como Léo**
*Sintoma:* quando a Bianca enviava mensagem, a resposta começava com "Bom dia, Léo!".
*Causa:* System Message não referenciava múltiplos usuários. Modelo assumia identidade padrão.
*Solução:* passar o `pushName` capturado do webhook como prefixo no User Message (`Bianca: [mensagem]`) e adicionar instrução no System Message para identificar o remetente pelo nome.

**Briefing paralelo gerando duas mensagens separadas**
*Sintoma:* ao rodar Contas e Estoque em paralelo, chegavam duas mensagens distintas no grupo em vez de uma consolidada.
*Causa:* cada branch paralelo completava independentemente e disparava seu próprio envio.
*Solução:* reestruturar para execução em série com Aggregate nodes antes do Code.

**Perda total de infraestrutura**
*Ocorrência:* infraestrutura Cloudfy original (prefixo `squeakingseacucumber`) foi perdida junto com todos os workflows.
*Impacto:* necessidade de recriar toda a configuração do zero na nova infraestrutura (prefixo `iconicllama`).
*Lição:* exportar e versionar os JSONs dos workflows regularmente no Git. Documentação arquitetural completa (como este documento) permite reconstrução rápida.

---

## 4. ARQUITETURA FINAL

### Fluxo 1 — Tize Briefing Diário

```
[Schedule Trigger - 9h]
        │
        ▼
[Google Sheets - Ler Contas a Pagar]
        │
        ▼
[Aggregate - contas{}]
        │
        ▼
[Google Sheets - Ler Estoque]
        │
        ▼
[Aggregate - estoque{}]
        │
        ▼
[Google Calendar - Ler Eventos do Dia]
        │
        ▼
[Aggregate - agenda{}]
        │
        ▼
[Code - Montar Mensagem]
  ├─ Parse datas DD/MM/YY
  ├─ Classifica: atrasadas / hoje / semana
  ├─ Filtra estoque REPOR/ZERADO
  ├─ Extrai horários via substring(11,16)
  └─ Monta texto condicional
        │
        ▼
[HTTP Request POST - Evolution API]
  └─ Envia para grupo WhatsApp
```

### Fluxo 2 — Tize Assistente Conversacional

```
[WhatsApp - Usuário envia mensagem]
        │
        ▼
[Evolution API - Webhook MESSAGES_UPSERT]
        │
        ▼
[n8n Webhook - POST /tize-webhook]
        │
        ▼
[Code - Processar Mensagem]
  ├─ Extrai: mensagem, remoteJid, pushName, fromMe
  ├─ Filtra: fromMe=true → para (evita loop)
  ├─ Filtra: remoteJid ≠ grupo autorizado → para
  └─ Filtra: mensagem vazia → para
        │
        ▼
[LangChain AI Agent - DeepSeek V4 Flash]
  ├─ User Message: "Hoje é DD/MM/YYYY. {pushName}: {mensagem}"
  ├─ System Message: persona + instruções operacionais
  ├─ Memory: Redis (TTL 36h, key=tize:{remoteJid})
  └─ Tools:
       ├─ Ler Contas a Pagar (Sheets GET)
       ├─ Adicionar Conta (Sheets APPEND)
       ├─ Atualizar Conta (Sheets UPDATE)
       ├─ Ler Estoque (Sheets GET)
       ├─ Atualizar Estoque (Sheets UPDATE)
       ├─ Ler Agenda (Calendar GET - semana completa)
       ├─ Criar Evento (Calendar CREATE)
       ├─ Atualizar Evento (Calendar UPDATE)
       └─ Deletar Evento (Calendar DELETE)
        │
        ▼
[HTTP Request POST - Evolution API]
  └─ Envia resposta para grupo WhatsApp
        │
        ▼
[Respond to Webhook - 200 OK]
```

### Componentes e integrações

| Componente | Tecnologia | Função |
|-----------|-----------|--------|
| Orquestrador | n8n (Cloudfy) | Runtime dos workflows |
| Gateway WhatsApp | Evolution API (Baileys) | Envio/recebimento de mensagens |
| Modelo de IA | DeepSeek V4 Flash | Interpretação e geração de linguagem natural |
| Framework de agente | LangChain (via n8n) | Tool calling, memória, raciocínio |
| Memória | Redis (Cloudfy) | Contexto conversacional com TTL |
| Dados financeiros | Google Sheets | Contas a pagar, estoque |
| Agenda | Google Calendar | Eventos e compromissos |

### Sugestão de diagrama

```
┌─────────────────────────────────────────────────────┐
│                    USUÁRIO                          │
│              (WhatsApp - grupo)                     │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │  Evolution API  │
              │  (Baileys)      │
              └────────┬────────┘
                       │ webhook
              ┌────────▼────────┐
              │      n8n        │
              │  ┌───────────┐  │
              │  │  AI Agent │  │
              │  │ DeepSeek  │  │
              │  └─────┬─────┘  │
              └────────│────────┘
                       │ tools
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼──────┐ ┌─────▼──────┐ ┌───▼────────┐
│ Google       │ │  Google    │ │   Redis    │
│ Sheets       │ │  Calendar  │ │  (memória) │
│ (financeiro/ │ │  (agenda)  │ │            │
│  estoque)    │ │            │ │            │
└──────────────┘ └────────────┘ └────────────┘
```

---

## 5. RESULTADO E MÉTRICAS

**Status atual:** em produção. Ambos os workflows rodando na nova infraestrutura (iconicllama). Usuários ativos: 2 (proprietária da clínica e sócio operacional).

**O que a solução entrega hoje:**

- Briefing diário às 9h com contas atrasadas, vencimentos da semana (com totais), alertas de estoque e agenda do dia — em uma única mensagem no WhatsApp
- Agente conversacional que responde perguntas sobre financeiro, estoque e agenda em linguagem natural
- Criação, edição e deleção de eventos no Google Calendar via mensagem de texto
- Atualização de status de contas a pagar (marcar como pago) via mensagem
- Identificação de quem está falando no grupo (múltiplos usuários)
- Memória de contexto de 36h entre mensagens

**Ganhos mensuráveis (estimados):**

| Tarefa | Antes | Depois |
|--------|-------|--------|
| Verificar contas da semana | ~10 min/dia (abrir planilha, filtrar, calcular) | 0 min (briefing automático às 9h) |
| Agendar compromisso | Abrir app, navegar, preencher formulário | 1 mensagem de texto |
| Checar estoque | Ir até o estoque fisicamente ou lembrar de cabeça | 1 pergunta no WhatsApp |
| Visibilidade financeira | Reativa (descobria quando o fornecedor cobrava) | Proativa (alerta 7 dias antes) |

**Custo operacional mensal estimado:**
- n8n + Redis + Evolution API: incluso no plano Cloudfy (~R$156/mês para toda a infraestrutura)
- DeepSeek V4 Flash: ~$0.05–0.10/mês para o volume atual
- Google APIs: gratuito

---

## 6. APRENDIZADOS

**O que faria diferente se recomeçasse:**

1. **Versionar workflows no Git desde o primeiro dia.** A perda total da infraestrutura Cloudfy e dos workflows mostrou que confiar em uma única plataforma sem backup externo é risco inaceitável. JSONs exportados do n8n + documentação no GitHub teriam evitado retrabalho completo.

2. **Documentar decisões técnicas em tempo real, não depois.** Reconstruir o raciocínio por trás de cada decisão semanas depois é difícil. Um ADR (Architecture Decision Record) simples por decisão teria economizado tempo na documentação posterior.

3. **Começar pelo agente conversacional, não pelo briefing.** O briefing é mais simples tecnicamente, mas o agente conversacional é o produto mais valioso. Teria aprendido mais rápido sobre tool calling, memória e LangChain se tivesse começado por ele.

4. **Definir o modelo de IA antes de construir.** Metade dos problemas de debug foram causados por trocar de modelo no meio do desenvolvimento (Gemini → DeepSeek). Cada modelo tem quirks próprios de tool calling. Escolher e travar o modelo primeiro teria evitado retrabalho.

5. **Separar ambiente de desenvolvimento de produção.** Testar diretamente no grupo real com a usuária presente gerou situações constrangedoras (agente respondendo com datas erradas, eventos sendo criados no dia errado). Um grupo de sandbox separado seria o correto.

**Lições técnicas:**

- **Google Sheets via API não é um banco de dados**, mas funciona surpreendentemente bem para PMEs com volume baixo. A restrição real não é performance — é a falta de transações e a possibilidade de edição manual bagunçar a estrutura.

- **Tool calling em modelos de linguagem não é confiável para operações destrutivas sem confirmação.** O agente deletou e editou eventos sem pedir confirmação ao usuário. Em produção, operações irreversíveis deveriam ter uma etapa de confirmação antes de executar.

- **O campo System Message em nós de agente do n8n não processa expressões `{{ }}`** — é texto estático. Qualquer dado dinâmico (data atual, nome do usuário, contexto do sistema) precisa ser injetado no User Message.

- **Redis com TTL é a solução correta para memória conversacional** com janela de tempo. Simple Memory perde estado em restarts. PostgreSQL acumula sem expirar. Redis resolve os dois problemas nativamente.

- **Parsing de datas requer tratamento defensivo.** Diferentes sistemas enviam datas em formatos diferentes (ISO 8601, DD/MM/YYYY, DD/MM/YY, serial numérico do Excel). Uma função de parsing com múltiplos fallbacks é essencial para qualquer integração com planilhas.

**Lição de produto:**

O usuário não queria um sistema de gestão. Queria não precisar pensar em gestão. A diferença é sutil mas fundamental para o design: em vez de uma dashboard para consultar, construímos um agente que informa proativamente e executa quando solicitado. O WhatsApp como interface não foi uma concessão técnica — foi a decisão de produto mais importante do projeto.

---

*Projeto desenvolvido por Léo Fagundes — Agentize AI*
*Automações e Agentes Inteligentes para Pequenas e Médias Empresas*
