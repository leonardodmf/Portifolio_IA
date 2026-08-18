# Case Study: Extrator Automático de Dados de NF-e com n8n + IA

---

## 1. CONTEXTO E PROBLEMA

**Problema de negócio**
Uma clínica de estética avançada acumulou notas fiscais eletrônicas (NFS-e) emitidas ao longo de 2025 e 2026 sem nunca ter construído um cadastro de clientes estruturado. Todos os dados dos tomadores de serviço estavam dispersos nos PDFs das notas, sem centralização, sem busca, sem histórico consultável. Montar esse cadastro manualmente — abrindo nota por nota, copiando nome, CPF, endereço, telefone e e-mail — seria inviável dado o volume e o tempo disponível do gestor.

**Usuário / beneficiário final**
Gestor da clínica (perfil não-técnico), que precisava de uma planilha de clientes pronta para uso imediato — sem depender de sistema de gestão externo ou de entrada manual de dados.

**Restrições**
- Orçamento próximo de zero: sem licenças pagas de ferramentas de OCR ou automação enterprise
- Stack já existente: n8n.cloud (plano gratuito/trial), Google Gemini (API já credenciada em outro agente), Google Sheets
- Sem acesso a servidor próprio — tudo em SaaS
- Executor do projeto acumulando papel de PM e desenvolvedor simultaneamente
- PDFs organizados em estrutura de pastas por ano/mês no OneDrive (depois migrado para Google Drive)

---

## 2. DECISÕES TÉCNICAS

**1. n8n como orquestrador (mantido desde o início)**
Já era a ferramenta em uso para outros agentes na clínica. Evitou curva de aprendizado e reaproveitou credenciais existentes. Alternativa seria Make (Integromat), descartada por custo e por não ter familiaridade prévia.

**2. Extração via node nativo `Extract from File` (PDF → texto)**
Opção preferida por ser nativa no n8n, sem custo adicional e suficiente para NFS-e geradas digitalmente (texto selecionável). A alternativa — OCR via Google Document AI ou OpenAI Vision — foi identificada como fallback para notas escaneadas, mas não foi necessária dado o perfil dos arquivos.

**3. IA para estruturação dos dados (Gemini, já credenciado)**
Em vez de parsear o texto da NF-e com regex ou código fixo — frágil dado que o layout pode variar entre prefeituras — optou-se por passar o texto bruto para um LLM e solicitar retorno em JSON estruturado. Isso torna o extrator resiliente a variações de layout sem necessidade de manutenção de regras.

**4. OneDrive → migração para Google Drive**
Decisão forçada por bloqueio técnico: a criação do App Registration no Azure falhou devido a restrições de tenant na conta Microsoft pessoal do usuário. Em vez de continuar investindo tempo no desbloqueio do Azure, o projeto foi pivotado para Google Drive, onde a autenticação OAuth é trivial (um clique no n8n.cloud). Os PDFs foram realocados manualmente.

**5. Google Sheets como destino (mantido)**
Baixa fricção para o usuário final, sem necessidade de banco de dados, compatível com o nível de maturidade de dados da operação. Excel foi considerado inicialmente mas descartado junto com o OneDrive.

**6. Node `Get Rows` antes do `Append` para ID sequencial**
O Google Sheets não tem autonumérico nativo. Solução: antes de cada inserção, o fluxo lê a planilha para contar as linhas existentes e calcula o próximo ID com `padStart(3, '0')`, gerando o padrão `001, 002, 003...`.

**7. `Search Files` com filtro por pasta em vez de `List Files`**
O node `List Files` não estava disponível na versão do node Google Drive instalada. O `Search Files/Folders` com query `mimeType='application/pdf' and 'ID_DA_PASTA' in parents` cobriu o mesmo caso de uso com mais flexibilidade.

---

## 3. OBSTÁCULOS E ITERAÇÃO

**Bloqueio 1 — Node `Read Binary Files` inexistente**
Planejamento inicial assumia esse node para leitura de pasta local. Na prática, o n8n.cloud não tem acesso a disco local, e o node não existia na interface. Resolução: substituído pelo par `Search Files (Google Drive)` + `Download File`.

**Bloqueio 2 — Autenticação Microsoft / Azure**
Tentativa de usar OneDrive como fonte dos PDFs. O portal Azure retornou erro de tenant (`no_tokens_found`, conta pessoal sem acesso ao diretório corporativo Microsoft). Tentativas de contorno via `entra.microsoft.com` e aba anônima não resolveram. Resolução: decisão de abandonar OneDrive e migrar para Google Drive — decisão correta dado o custo de oportunidade.

**Bloqueio 3 — Loop Over Items mal conectado**
Após remoção do node placeholder `Replace Me` (que fechava o loop), o fluxo parou de iterar após o primeiro item. Diagnóstico: a saída do último node processador (Excel/Sheets) precisa reconectar na entrada do `SplitInBatches` para que ele avance para o próximo item. Resolução: conexão `Salvar no Google Sheets → Loop Over Items` adicionada.

**Bloqueio 4 — Confusão entre saídas do Loop**
O `SplitInBatches` tem duas saídas: `0` (done — quando acabam os itens) e `1` (loop — enquanto há itens). O fluxo estava com a conexão invertida, processando apenas quando o loop terminava. Resolução: conexão corrigida para saída `1` → `Baixar PDF`.

**Bloqueio 5 — Code node com código placeholder**
O node foi importado com o código padrão gerado pelo n8n (`myNewField = 1`). Substituído por lógica real de `JSON.parse()` da resposta do Gemini, com tratamento de markdown residual (```` ```json ``` ````) e mapeamento explícito de cada campo para a coluna correspondente na planilha.

**Redesign mid-flight**
- Destino: Excel (OneDrive) → Google Sheets
- Fonte dos arquivos: pasta local → OneDrive → Google Drive
- Estrutura de busca: `List Files` → `Search Files` com query por mimeType e ID de pasta

---

## 4. ARQUITETURA FINAL

**Fluxo ponta a ponta**

```
Manual Trigger
    ↓
Google Drive — Search Files
(query: mimeType=PDF + pasta Notas Fiscais)
    ↓
Loop Over Items (SplitInBatches)
    ↓ [saída 1 — enquanto há itens]
Google Drive — Download File
(fileId: {{ $json.id }})
    ↓
Extract from File
(tipo: PDF → texto bruto)
    ↓
AI Agent (Gemini)
(prompt: extrai tomador de serviços → JSON)
    ↓
Google Sheets — Get Rows
(conta linhas existentes para calcular próximo ID)
    ↓
Code Node (JS)
(JSON.parse + gera Cod sequencial 001/002/...)
    ↓
Google Sheets — Append Row
(insere linha na planilha de cadastro)
    ↓ [retorno ao loop]
Loop Over Items
    ↓ [saída 0 — quando todos os itens foram processados]
FIM
```

**Componentes**

| Componente | Função |
|---|---|
| n8n.cloud | Orquestrador do fluxo |
| Google Drive node (Search) | Lista todos os PDFs da pasta |
| Google Drive node (Download) | Baixa o binário de cada PDF |
| Extract from File | Converte PDF em texto plano |
| AI Agent + Gemini | Interpreta o texto e retorna JSON estruturado |
| Google Sheets (Get Rows) | Conta registros para gerar ID sequencial |
| Code node (JS) | Parse do JSON + montagem do objeto final |
| Google Sheets (Append) | Grava uma linha por NF processada |

**Campos extraídos por NF-e**
`Cod · Nome · CPF · Telefone · Endereço · Bairro · Complemento · Município · Estado · CEP · Email`

**Diagrama simplificado**
```
[Drive: Search PDFs] → [Loop] → [Drive: Download] → [Extract PDF]
                          ↑                                 ↓
                    [Sheets: Append]              [AI Agent: Gemini]
                          ↑                                 ↓
                    [Code: parse+ID]          [Sheets: Get Rows]
```

---

## 5. RESULTADO E MÉTRICAS

**O que entrega**
Cadastro de clientes populado automaticamente a partir do acervo histórico de NFS-e, sem entrada manual de dados. Cada NF processada vira uma linha na planilha com 11 campos estruturados.

**Status**
Solução desenvolvida e em fase final de configuração (credenciais Google a conectar, planilha a selecionar nos nodes). Ainda não executada em produção sobre o acervo completo.

**Ganho estimado**
Considerando ~100 notas fiscais emitidas (estimativa conservadora para 2 anos de operação), o processo manual levaria entre 3 e 5 horas de trabalho repetitivo. O agente processa o mesmo volume em minutos, com padronização de campos e sem erro de digitação.

**Limitação conhecida**
A solução atual não deduplica clientes — se o mesmo CPF aparece em múltiplas notas, ele será inserido múltiplas vezes. Isso é aceitável para um cadastro inicial (limpeza pontual posterior), mas seria um débito técnico a endereçar em uma v2.

---

## 6. APRENDIZADOS

**O que faria diferente**

- **Validar o stack de autenticação antes de projetar o fluxo.** O bloqueio do Azure consumiu tempo significativo que poderia ter sido evitado com um teste de credencial de 5 minutos no início. "Testar o acesso antes de desenhar a arquitetura" vira regra.
- **Começar com Google Drive/Sheets desde o início.** Para usuários com conta pessoal Microsoft, OneDrive via API é uma armadilha de configuração. Google é o caminho de menor resistência no ecossistema n8n.cloud.
- **Adicionar deduplicação por CPF desde a v1.** Um `IF node` verificando se o CPF já existe na planilha antes do `Append` eliminaria retrabalho de limpeza posterior.

**Lições técnicas**

- O `SplitInBatches` tem comportamento contraintuitivo nas saídas: saída `0` = fim do loop, saída `1` = próximo item. Inverter isso quebra silenciosamente o fluxo sem erro visível.
- LLMs são superiores a regex para parsing de documentos com layout variável. O custo de uma chamada de API por documento é negligenciável frente à manutenção de regras de extração baseadas em posição de texto.
- Prompts para extração estruturada precisam ser explícitos em dois pontos: (1) retornar **somente** JSON, sem markdown, sem texto extra; (2) definir o comportamento para campos ausentes (`""`). Sem isso, o `JSON.parse()` quebra na primeira nota que o modelo responder diferente.

**Lição de produto**

Um fluxo de automação para usuário não-técnico precisa ser à prova de esquecimento: sem configuração manual por execução, sem dependência de lembrar de trocar um parâmetro. O design ideal é "abre, clica em executar, fecha". Este projeto ainda exige que o usuário mova os PDFs para a pasta correta antes de rodar — isso é um passo manual que seria o próximo a eliminar em uma versão futura (trigger por upload no Drive, por exemplo).
