# Case Study: Automação de Fluxo de Caixa via Telegram + n8n + Google Sheets

---

## 1. CONTEXTO E PROBLEMA

### Qual problema de negócio motivou esse projeto?
Uma clínica de estética avançada operava sem controle automatizado de movimentação de caixa. Os lançamentos financeiros eram feitos manualmente — processo suscetível a erros, esquecimentos e falta de rastreabilidade. O objetivo foi criar um sistema de registro financeiro que pudesse ser acionado de qualquer lugar, pelo celular, com o mínimo de fricção possível.

### Quem era o usuário/beneficiário final?
A proprietária da clínica (biomedical aesthetician, perfil clínico forte, baixa afinidade com ferramentas de gestão). O sistema precisava ser simples o suficiente para ser operado via mensagem de texto, sem abrir planilhas, sem logar em sistemas.

### Que restrições existiam?
- **Orçamento**: zero ou próximo de zero — sem licenças pagas, sem banco de dados gerenciado
- **Infraestrutura**: n8n hospedado em plataforma cloud (Cloudfy), sem acesso root
- **Perfil do usuário final**: baixíssima tolerância a interfaces complexas — a entrada de dados precisava ser via chat
- **Tempo**: projeto construído em sprints curtos, sem dedicação exclusiva

---

## 2. DECISÕES TÉCNICAS

### Alternativas consideradas e decisões tomadas (ordem cronológica)

**Decisão 1 — Canal de entrada: WhatsApp vs Telegram**
- _Considerado_: WhatsApp Business API (Meta oficial), Evolution API, Z-API
- _Escolhido_: Telegram
- _Porquê_: WhatsApp exige camada intermediária (Evolution API ou serviço pago), risco de ban por uso automatizado, configuração complexa. Telegram tem bot nativo via BotFather, trigger direto no n8n em minutos, zero custo, zero burocracia.

**Decisão 2 — Armazenamento: Excel vs Google Sheets vs banco de dados**
- _Considerado_: Microsoft Excel 365 (Graph API), Google Sheets, banco relacional
- _Escolhido_: Google Sheets como backend, Excel como interface de visualização
- _Porquê_: Excel 365 exige autenticação OAuth via Azure AD — processo instável e já problemático em projetos anteriores. Google Sheets tem node nativo no n8n com OAuth simples. Banco de dados foi descartado por complexidade desnecessária para o volume. A separação backend/interface resolve a preferência da operadora por Excel sem abrir mão da simplicidade da integração.

**Decisão 3 — Autenticação Google Sheets: OAuth vs Service Account**
- _Considerado_: OAuth2 com Client ID/Secret, Service Account
- _Escolhido_: OAuth2
- _Porquê_: Resolvido após adicionar o domínio `cloudfy.live` nos Authorized Domains da Consent Screen. Service Account ficou como fallback documentado caso o OAuth instabilizasse.

**Decisão 4 — Geração de ID único: timestamp vs ID do Telegram vs sequencial com lookup**
- _Considerado_: Timestamp em milissegundos (`ID-1714392000000`), sequencial com busca da última linha no Sheets, ID nativo da mensagem Telegram
- _Escolhido_: ID da mensagem Telegram (`ID-XXX`)
- _Porquê_: Timestamp gerava IDs longos e sem legibilidade. Sequencial com lookup exigia node extra de busca + lógica de incremento. O `message_id` do Telegram já é único, crescente, e chega gratuitamente no payload — zero overhead.

**Decisão 5 — Sincronização Excel: fórmula vs Power Query**
- _Considerado_: Fórmula `WEBSERVICE`, importação manual, Power Query
- _Escolhido_: Power Query com atualização automática ao abrir
- _Porquê_: `WEBSERVICE` não suporta CSV adequadamente no Excel desktop. Power Query é a solução nativa para conexões externas, permite atualização automática ao abrir o arquivo e transforma os dados antes de carregar.

---

## 3. OBSTÁCULOS E ITERAÇÃO

**Obstáculo 1 — OAuth Google rejeitava a redirect URI**
- _Problema_: A URL `https://squeakingseacucumber-n8n.cloudfy.live/rest/oauth2-credential/callback` era recusada pelo Google como inválida
- _Diagnóstico_: O Google exige que o domínio da redirect URI esteja listado como Authorized Domain na Consent Screen
- _Solução_: Adicionado `cloudfy.live` nos Authorized Domains via OAuth Consent Screen → Edit App → aba Branding/Scopes

**Obstáculo 2 — Bug no formatador de valor monetário**
- _Problema_: Valores já formatados como `2.500,00` retornavam como `2,50` após o processamento
- _Diagnóstico_: O `replace(',', '.')` trocava a vírgula decimal por ponto, mas o ponto de milhar permanecia — gerando `2.500.00`, que o `parseFloat` interpretava como `2.5`
- _Solução_: Adicionado `.replace(/\./g, '')` com regex global antes do replace da vírgula, removendo todos os pontos antes do parse

**Obstáculo 3 — Fuso horário incorreto na captura de hora**
- _Problema_: `new Date().getHours()` retornava hora do servidor (UTC), não o horário de Brasília
- _Solução_: Substituído por `Intl.DateTimeFormat` com `timeZone: 'America/Sao_Paulo'` e `formatToParts()`, independente do fuso do servidor

**Obstáculo 4 — ID sequencial travado em ID-001**
- _Problema_: Todos os registros geravam `ID-001` porque `lastId` sempre chegava como `0` (node de busca não configurado)
- _Solução_: Descartada a abordagem de lookup. Adotado `message_id` do Telegram como identificador — mais simples e sem dependência de estado externo

**Obstáculo 5 — Power Query criando aba nova em vez de carregar na tabela existente**
- _Problema_: Ao configurar a conexão, o Excel criava uma nova aba com os dados em vez de popular o Modelo 2
- _Solução_: Ajuste em Consultas e Conexões → Carregar para → selecionar "Tabela" + "Planilha existente" apontando para célula A2 do Modelo 2

**Obstáculo 6 — Fórmulas de saldo quebrando com tabela estruturada**
- _Problema_: Referências como `=J2+...` em tabelas do Excel se reajustavam automaticamente de forma incorreta por conta do cabeçalho e linhas puladas
- _Solução_: Substituição por referências estruturadas (`[@Tipo]`, `[@Valor]`) e fórmula de saldo acumulado com `SOMASE` comparando IDs, eliminando dependência de referência à linha anterior

---

## 4. ARQUITETURA FINAL

### Fluxo ponta a ponta

```
[Usuário]
    |
    | Envia mensagem no formato:
    | Tipo/ Valor/ Forma Pgto/ Cliente_forn/ Histórico
    v
[Telegram Bot]
    |
    v
[n8n — Trigger: Telegram]
    |
    v
[n8n — Node: Code (JavaScript)]
    | - Extrai message_id como tx_id (ex: ID-031)
    | - Captura data e hora (fuso America/Sao_Paulo)
    | - Faz split da mensagem por "/"
    | - Formata valor para padrão pt-BR (1.234,56)
    | - Retorna objeto com 8 campos padronizados
    v
[n8n — Node: Google Sheets]
    | - Operação: Append Row
    | - Mapeia os 8 campos para as colunas da planilha
    | - Colunas: tx_id, dt_data, dt_hora, tx_tipo,
    |            vl_valor, tx_forma_pgto, tx_cliente_forn, tx_historico
    v
[Google Sheets — Base de dados]
    |
    v
[n8n — Node: Telegram (resposta)]
    | - Envia confirmação formatada em Markdown
    | - Exibe todos os campos do registro inserido
    v
[Usuário recebe confirmação no Telegram]

            ↕  (Power Query — atualização ao abrir)

[Excel — Planilha de Controle Mensal]
    - Importa dados do Google Sheets via Power Query
    - Colunas A-G: dados vindos do Sheets
    - Colunas H-J: Entrada, Saída, Saldo (fórmulas locais)
    - Saldo acumulado via SOMASE por ID
```

### Componentes da solução

| Componente | Função |
|---|---|
| Telegram Bot | Interface de entrada de dados |
| n8n (Cloudfy) | Orquestrador da automação |
| Node Code (JS) | Parsing, formatação e geração de ID |
| Node Google Sheets | Persistência dos dados |
| Node Telegram (send) | Feedback de confirmação ao usuário |
| Google Sheets | Backend / base de dados central |
| Power Query (Excel) | Sincronização e visualização |
| Excel | Interface de análise e gestão mensal |

---

## 5. RESULTADO E MÉTRICAS

### O que a solução entrega hoje
- Registro de movimentação financeira via mensagem de texto no Telegram
- Dados persistidos automaticamente em planilha Google Sheets com ID único, data, hora e formatação monetária brasileira
- Confirmação imediata no próprio chat com resumo do lançamento
- Planilha Excel sincronizada com a base, com cálculo automático de entradas, saídas e saldo acumulado

### Status
Sistema em uso ativo na clínica.

### Ganhos estimados
- **Tempo de registro**: de ~3-5 minutos (abrir planilha, encontrar linha, digitar, salvar) para ~10 segundos (enviar mensagem)
- **Rastreabilidade**: cada lançamento tem ID único, timestamp e origem identificável
- **Erros de formatação**: eliminados pelo processamento no node Code
- **Barreira de adoção**: zero — usuária já usava Telegram diariamente

---

## 6. APRENDIZADOS

### O que faria diferente se recomeçasse hoje

1. **Validação de formato antes de inserir**: adicionar um node de verificação que checa se a mensagem tem os 5 campos esperados antes de tentar o split — hoje uma mensagem mal formatada gera um registro sujo na planilha sem aviso claro
2. **Service Account desde o início**: o caminho OAuth com domínio de subdomínio de plataforma compartilhada gerou fricção desnecessária; Service Account é mais simples para uso pessoal/interno
3. **Coluna de hora separada desde o design**: foi adicionada como iteração, mas poderia já estar no modelo inicial
4. **Convenção de nomenclatura de colunas desde o início**: a padronização com prefixos `tx_`, `dt_`, `vl_` veio no meio do projeto — idealmente seria definida antes de criar a planilha

### Lições técnicas e de produto

- **Simplicidade de entrada é o produto**: o usuário não adota ferramentas, adota comportamentos. Fazer o registro caber numa mensagem de Telegram foi a decisão de produto mais importante do projeto — não a tecnologia por trás.
- **ID externo elimina estado**: usar o `message_id` do Telegram como chave única simplificou a arquitetura, removeu um node e eliminou uma classe inteira de bugs de concorrência.
- **Separar backend de interface**: Google Sheets como banco + Excel como visualização é uma arquitetura válida para pequenas operações — resolve a preferência do usuário sem comprometer a confiabilidade da integração.
- **Regex global importa**: `.replace(',', '.')` vs `.replace(/,/g, '.')` — em formatação de dados financeiros, um replace não-global pode mascarar bugs silenciosos que só aparecem com dados reais.
- **Fuso horário em automações**: sempre explicitar o timezone. `new Date()` em servidor cloud é UTC — isso vai quebrar em produção e é difícil de debugar depois.
