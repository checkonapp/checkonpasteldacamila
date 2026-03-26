# CheckON — Especificação Técnica e de Negócio
**Pastel da Camilla · Versão 2026.14**
*Documento de referência para desenvolvimento, manutenção e tomada de decisão.*

---

## 1. O que é o CheckON

CheckON é um sistema de gestão operacional por turno para o Pastel da Camilla. Resolve três problemas reais:

1. **Accountability sem burocracia** — subgerentes preenchem checklists estruturados no celular, não formulários genéricos
2. **Rastreabilidade de passagem de turno** — o que a manhã deixou pendente, a noite recebe formalmente
3. **Visibilidade gerencial em tempo real** — a Camila (CEO) vê o status de cada área sem precisar perguntar

O sistema **não é** um app de tarefas genérico. É calibrado para a operação específica do Pastel da Camilla, com checklists, horários, e regras de negócio que refletem a realidade da loja.

---

## 2. Usuários e Perfis

| Login | Nome | Role | Tabela principal | Hub |
|---|---|---|---|---|
| `gabriela` | Gabriela | `subgerente` | `subger_checklists` | SubgerenteHub |
| `vitoria` | Vitória | `subgerente_noite` | `subger_noite_checklists` | SubgerenteNoiteHub |
| `viviane` | Viviane | `dp` | `dp_checklists` | DPHub |
| `derik` | Derik | `gerencia` | `ger_checklists` | GerenciaHub |
| `gerente` | Gerente | `gerente` | `checklists` | GerenteHub |
| `bruna` | Bruna | `suporte` | `checklists` | SuporteHub |
| `camila` | Camila | `ceo` | leitura de todas | CEOHub |

### Regras de roteamento (App.js)
```
role === 'ceo'               → CEOHub
role === 'suporte'           → SuporteHub
role === 'gerente'           → GerenteHub
role === 'gerencia'          → GerenciaHub
role === 'dp'                → DPHub
role === 'subgerente'        → SubgerenteHub
role === 'subgerente_noite'  → SubgerenteNoiteHub
qualquer outro role          → ChecklistScreen (legado)
```

### Invariante de segurança
`getUsers()` força os roles de gabriela e vitoria mesmo que estejam corrompidos no localStorage:
```js
if(u.gabriela.role !== 'subgerente') u.gabriela.role = 'subgerente'
if(u.vitoria.role !== 'subgerente_noite') u.vitoria.role = 'subgerente_noite'
```
**Nunca remover essa verificação.**

---

## 3. Fórmulas de Score — Canônico

> **PROBLEMA ATUAL:** existem três implementações diferentes de score no código. Esta seção define o que deveria ser e o que é hoje.

### 3.1 O que é hoje (por perfil)

| Perfil | Fórmula atual | Críticos com peso extra? |
|---|---|---|
| Subgerência Manhã (Gabriela) | `100 - (falhas/total × 50) - (criticos × 8)` | ✅ Sim |
| Subgerência Noite (Vitória) | `100 - (falhas/total × 50) - (criticos × 8)` | ✅ Sim |
| Gerência (Derik) | `100 - (falhas/total × 60)` | ❌ Não |
| Gerente/Suporte | `100 - (falhas/total × 40) - (criticos × 10)` | ✅ Sim |
| DP (Viviane) | `100 - (falhas/total × 60)` | ❌ Não |

Todos têm `Math.max(0, Math.round(...))` — score mínimo é 0.

### 3.2 Por que isso importa
O **ranking unificado** que construímos no CEO Dashboard compara esses scores diretamente. Um score 80 da Gabriela (com peso de crítico) não é o mesmo que um score 80 do Derik (sem peso de crítico). O ranking atual está comparando grandezas diferentes.

### 3.3 Fórmula canônica proposta (a implementar)
```
score = max(0, round(100 - (falhas/total × 50) - (criticos × 8)))
```
**Parâmetros:**
- `50` = penalidade proporcional por falha. Com 10 falhas em 20 itens = -25 pontos
- `8` = penalidade adicional por item marcado como `critico:true`. Com 2 críticos = -16 pontos adicionais
- `Math.max(0)` = floor em zero, score nunca negativo

**Por que esses valores:**
- Um turno com todas as falhas em itens não-críticos ainda pode ter score 50+
- Um único crítico não resolvido custa 8 pontos acima do proporcional — cria incentivo correto
- Score 80+ = operação controlada, 60–79 = atenção, <60 = crítico

### 3.4 Status — valores esperados no banco

| Perfil | Valores possíveis de `status` |
|---|---|
| Subgerência (manhã e noite) | `'ok'` · `'atencao'` · `'critico'` |
| Gerência (Derik) | `'ok'` · `'atenção'` · `'crítico'` ← **com acento, inconsistente** |
| Gerente/Suporte | `'conforme'` · `'ressalvas'` · `'critico'` ← **valores diferentes** |
| DP | `'ok'` · `'atencao'` · `'critico'` |

**Problema:** o CEO Dashboard trata todos como se fossem iguais. A leitura `d.status === 'ok'` funciona para subger e DP, mas não para gerência (`'ok'` vs `'OK'`) nem para gerente (`'conforme'`).

**Regra canônica proposta:**
```
'ok'      = zero falhas
'atencao' = falhas mas nenhum crítico
'critico' = pelo menos um item crítico com falha
```
**Todos os perfis devem usar esses três valores exatos, minúsculo, sem acento.**

---

## 4. Banco de Dados — Contrato das Tabelas

### 4.1 `subger_checklists` (Gabriela — Manhã)
| Campo | Tipo | Quem grava | Invariante |
|---|---|---|---|
| `login` | string | SubgerChecklistScreen | sempre `'gabriela'` |
| `user_name` | string | idem | sempre `'Gabriela'` |
| `turno` | string | idem | sempre `'manha'` |
| `data` | string ISO | idem | formato `YYYY-MM-DD` |
| `data_fmt` | string | idem | formato `DD/MM/YYYY` |
| `hora` | string | idem | hora de envio, `HH:MM` |
| `hora_inicio` | string | idem | hora de início da etapa 1 |
| `score` | integer 0–100 | idem | fórmula: `100-(falhas/total×50)-(criticos×8)` |
| `total` | integer | idem | sempre 28 (soma dos itens das 3 etapas) |
| `falhas` | integer | idem | contagem de `resps[id]==='nao'` |
| `criticos` | integer | idem | contagem de `item.critico && resps[id]==='nao'` |
| `pendencias` | integer | idem | falhas com `negs[id].prox==='Sim, fica pendente'` |
| `status` | string | idem | `'ok'` · `'atencao'` · `'critico'` |
| `etapas_concluidas` | integer 0–3 | idem | etapas onde todos os itens têm resposta |
| `resps` | JSON object | idem | `{[item_id]: 'sim'|'nao'}` |
| `negs` | JSON object | idem | ver estrutura de neg abaixo |
| `relato` | string | idem | texto livre opcional |
| `sugestao` | JSON object | idem | `{atividade: string}` ou `{}` |

### 4.2 `subger_noite_checklists` (Vitória — Noite)
Mesma estrutura de `subger_checklists`, exceto:
- `turno` = sempre `'noite'`
- `total` = sempre 28 (soma dos itens das 3 etapas de noite)
- Após envio, chama `salvarPendencias()` → grava em `checkon_pendencias`

### 4.3 `ger_checklists` (Derik — Gerência)
| Campo | Observação |
|---|---|
| `turno` | `'manha'` ou `'tarde'` — determinado automaticamente por horário (<15h = manhã) |
| `score` | `100-(falhas/total×60)` — **sem peso de crítico, inconsistente** |
| `status` | `'ok'` · `'atenção'` · `'crítico'` — **com acento, inconsistente** |
| `criticos` | gravado como campo, mas sem peso no score |

### 4.4 `dp_checklists` (Viviane — DP)
| Campo | Observação |
|---|---|
| `freq` | `'diario'` · `'semanal'` · `'mensal'` |
| `checklist` | título da frequência, ex: `'DP — Checklist Diário'` |
| `blocos_concluidos` | blocos onde todos os itens têm resposta |
| `score` | `100-(falhas/total×60)` — **sem peso de crítico** |
| `ocorrencia` | observação de ocorrência do contexto, string |

### 4.5 `checklists` (Gerente/Suporte — legado)
| Campo | Observação |
|---|---|
| `turno` | `'manha'` · `'tarde'` |
| `status` | `'conforme'` · `'ressalvas'` · `'critico'` — **valores diferentes dos outros** |
| `score` | `100-(falhas/total×40)-(criticos×10)` — pesos diferentes |

### 4.6 `checkon_pendencias` (compartilhada)
| Campo | Tipo | Invariante |
|---|---|---|
| `login` | string | login de quem gerou a pendência |
| `user_name` | string | nome de quem gerou |
| `turno` | string | `'manha'` ou `'noite'` |
| `data` | string ISO | data de geração |
| `data_fmt` | string | data formatada |
| `item_id` | string | id do item no checklist |
| `item_texto` | string | texto do item |
| `aconteceu` | string | chip selecionado em "o que aconteceu" |
| `feito` | string | chip selecionado em "o que foi feito" |
| `detalhe` | string | texto livre opcional |
| `status` | string | `'aberta'` · `'resolvida'` — **apenas esses dois valores** |
| `checklist_origem` | string | nome da tabela de origem |

**Quem grava:** `salvarPendencias()` — chamada por SubgerChecklistScreen e NoiteChecklistScreen ao finalizar
**Quem lê:** CEODashboard (aba Pendências), PendenciasBlock
**Quem atualiza:** `PendenciasBlock.resolver()` → muda status para `'resolvida'`

### 4.7 `dp_sugestoes`
Grava sugestões de melhoria do checklist submetidas pelo DP. Aparece no CEO → aba DP como badge de contador.

---

## 5. Estrutura de `negs` por Perfil

O campo `negs` é um objeto `{[item_id]: NegObject}` gravado como JSON no banco.

### 5.1 Subgerência (manhã e noite) — `SubgerNaoForm` / `NoiteNaoForm`
```js
{
  aconteceu: string,  // chip de "O que aconteceu?" — obrigatório
  feito:     string,  // chip de "O que foi feito?" — obrigatório
  prox:      string,  // 'Sim, fica pendente' | 'Não, encerrado' — obrigatório
  detalhe:   string,  // texto livre — opcional
}
```
**Validação:** `validateSubgerNeg` / `validateNoiteNeg` exige os três primeiros campos.

### 5.2 DP — `DPNaoForm`
```js
{
  ocorrencia:    string,  // chip de categoria — obrigatório
  acao:          string,  // 'resolvi' | 'pendente' | 'escalei' — obrigatório
  escaladoPara:  string,  // obrigatório se acao === 'escalei'
  detalhe:       string,  // obrigatório se isOutro || isPendente || isEscalei
  prox:          string,  // 'Sim' | 'Não' — obrigatório
  resp:          string,  // igual a escaladoPara quando escalei
}
```
**Atenção:** `prox` aqui aceita `'Sim'` ou `'Não'` — diferente da subgerência que usa `'Sim, fica pendente'`. A função `salvarPendencias` trata ambos: `n.prox === 'Sim, fica pendente' || n.prox === 'Sim'`.

### 5.3 Gerência (Derik) — `SharedNaoForm`
```js
{
  problema: string,  // texto — obrigatório
  causa:    string,  // texto — obrigatório
  impacto:  string,  // texto — obrigatório
  acao:     string,  // texto — obrigatório
  resp:     string,  // chip de responsável — obrigatório
  prox:     string,  // 'Sim' | 'Não' — obrigatório
}
```

### 5.4 Gerente/Suporte — legado
Não usa formulário estruturado de negativos — usa campo `obsGerais` (texto livre) e tela `OcorrenciasScreen`.

---

## 6. Chaves de localStorage

| Chave | Conteúdo | Quem usa | Quando limpa |
|---|---|---|---|
| `konclui_users` | JSON de todos os usuários | getUsers, saveUsers | nunca (persiste) |
| `konclui_session` | `{key: login}` | getSession, saveSession | ao fazer logout ou nova versão |
| `konclui_version` | string de versão (`'2026.14'`) | verificação de versão | nunca |
| `checkon_dp_[login]` | rascunho DP em andamento | DPChecklistScreen | ao enviar ou limpar |
| `konclui_ger_[login]` | rascunho gerência (legado) | GerChecklistScreen interno | ao enviar ou limpar |
| `ger_draft_[login]` | rascunho gerência (novo hub) | GerenciaHub | ao calcular rascunhoPct |
| `konclui_ck_[login]` | rascunho checklist legado | ChecklistScreen | ao enviar ou limpar |
| `checkon_subger_[login]` | rascunho subgerência manhã | SubgerChecklistScreen | ao enviar ou limpar |
| `checkon_noite_[login]` | rascunho subgerência noite | NoiteChecklistScreen | ao enviar ou limpar |
| `subger_draft_[login]` | referência do hub (leitura) | SubgerenteHub | não limpa |
| `checkon_ctx_[login]` | cache do analisarHistorico | analisarHistorico | válido por 4h |
| `checkon_hist_[login]_[tabela]` | fallback local de histórico | analisarHistorico fallback | nunca |

**Problema identificado:** `GER_SAVE_KEY` tem dois valores diferentes no código:
- `'konclui_ger_' + user.login` — em `GerChecklistScreen`
- `'ger_draft_' + user.login` — em `GerenciaHub`
Isso significa que o hub lê uma chave diferente da que o checklist grava. O progresso do rascunho exibido no hub pode estar sempre zerado para a gerência.

---

## 7. Fluxo por Etapas — Como Funciona

### 7.1 Subgerência Manhã (Gabriela)
```
SubgerenteHub (briefing) 
  → onIniciar() 
  → SubgerChecklistScreen (screen='inicio') 
  → auto-avança para screen='checklist'
  → Etapa 1: Abertura 9h–11h (12 itens)
  → Etapa 2: Controle da Operação 11h–13h (10 itens)
  → Etapa 3: Fechamento 13h+ (6 itens)
  → screen='ocorrencias' (resumo + relato)
  → handleFinalizar() → sbInsert + salvarPendencias + sendWebhook
  → screen='summary' (SubgerSummaryScreen)
  → TurnoEncerrado (tela final motivacional)
```

### 7.2 Subgerência Noite (Vitória)
```
SubgerenteNoiteHub (briefing + pendências da manhã)
  → onIniciar()
  → NoiteChecklistScreen (screen='inicio')
  → auto-avança para screen='checklist'
  → Etapa 1: Chegada e Recebimento 17h–18h (9 itens)
  → Etapa 2: Operação e Pico 18h–21h (9 itens)
  → Etapa 3: Fechamento Completo 21h+ (10 itens)
  → screen='resumo' (resumo + relato para a manhã)
  → handleFinalizar() → sbInsert + salvarPendencias + sendWebhook
  → screen='summary' (NoiteSummaryScreen)
  → TurnoEncerrado
```

### 7.3 Regra de avançar etapa
Antes de avançar da etapa N para N+1, `validateEtapa(idx)` verifica:
- Todo item tem resposta (`resps[item.id]` existe)
- Todo item com `resps[item.id]==='nao'` tem neg válido (`validateSubgerNeg` / `validateNoiteNeg`)

Se falhar → `setShowErr(true)` + toast de aviso + scroll para o topo. Não avança.

### 7.4 Rascunho automático
Ao sair pelo botão "⏸ Depois" ou "← Início", o estado é salvo no localStorage. Ao retornar ao hub, o `BoasVindasInteligente` detecta o rascunho e exibe barra de progresso + botão "Continuar turno →".

---

## 8. BoasVindasInteligente — Estados

O componente tem **quatro estados mutuamente exclusivos**:

| Estado | Condição | O que mostra |
|---|---|---|
| **Turno já enviado** | `verificarTurnoHoje` retorna registro com `data === hoje` | Card de resumo do turno + botão "Iniciar novo mesmo assim" |
| **Com rascunho** | `rascunhoPct !== null` | Barra de progresso + botão "Continuar turno →" |
| **Zerado** | sem rascunho, sem turno hoje | Preview das etapas com horários + botão "Iniciar [turno] →" |
| **Carregando** | `turnoHoje === null` (ainda resolvendo) | Header renderiza imediatamente, cards carregam progressivamente |

**Regra importante:** o botão de ação principal aparece **imediatamente**, sem esperar o carregamento do histórico. O histórico (`analisarHistorico`) carrega em background e atualiza os cards progressivamente.

---

## 9. Regras de Pendências

### 9.1 O que vira pendência
Um item vira pendência quando:
```
resps[item.id] === 'nao'
&& (negs[item.id].prox === 'Sim, fica pendente'   // subger, noite
    || negs[item.id].prox === 'Sim')               // DP, gerência
```

### 9.2 Ciclo completo
```
Gabriela marca item como NÃO + "Sim, fica pendente"
  → handleFinalizar() chama salvarPendencias()
  → grava em checkon_pendencias com status='aberta'
  → SubgerenteNoiteHub chama buscarPendenciasManhaParaNoite()
  → Vitória vê card de alerta no briefing
  → CEO vê na aba Pendências
  → PendenciasBlock.resolver() atualiza status='resolvida'
```

### 9.3 `buscarPendenciasManhaParaNoite` — como funciona
Busca os últimos 5 registros de `subger_checklists` com `data = hoje`. Pega o mais recente. Conta itens onde `resps[id]==='nao'` e `negs[id].prox==='Sim, fica pendente'`. Retorna `{responsavel, score, pendencias, horario}`.

**Não usa a tabela `checkon_pendencias`** — lê diretamente do turno da manhã. Isso é intencional: é mais rápido e garante que o card da Vitória reflete o turno real de hoje, não pendências antigas.

---

## 10. Inteligência de Histórico — `analisarHistorico`

### Parâmetros de negócio
- **Cache válido:** 4 horas (evita query no Supabase a cada abertura do app)
- **Janela de score médio:** últimos 7 turnos
- **Janela de falhas frequentes:** últimos 5 turnos
- **Threshold de falha frequente:** 2+ ocorrências em 5 turnos → gera alerta
- **Threshold de streak de conquista:** 5+ turnos sem crítico → gera badge de conquista
- **Threshold de score baixo:** média < 70 com 3+ registros → gera alerta de atenção

### Por que esses valores
- 7 turnos para média = ~2 semanas de operação, suaviza outliers sem perder relevância
- 5 turnos para falhas frequentes = ~1 semana, detecta padrão recente
- 2 ocorrências em 5 = 40% de frequência — limiar razoável para alertar sem ruído
- 5 turnos para streak = ~1 semana sem crítico merece reconhecimento

---

## 11. Webhook e Envio de E-mail

### Payload do webhook (Make.com)
```js
{
  texto:          string,  // texto do relatório (multilinha)
  timestamp:      string,  // ISO datetime
  sistema:        'CheckON Pastel da Camilla',
  email_destino:  'pasteldacamilasuporte@gmail.com',
  email_assunto:  string,  // ex: '[CheckON Subgerência] OK – Gabriela'
  email_html:     string,  // texto com \n → <br>
  email_texto:    string,  // mesmo que texto
}
```

### Quando dispara
- Ao finalizar qualquer checklist (todos os perfis)
- **Não dispara por etapa** — apenas ao enviar o turno completo

### Problema atual
`sendWebhook` falha silenciosamente. Se o Make.com estiver fora, o usuário não sabe. O `sbInsert` também falha silenciosamente. **Não há confirmação real de que os dados chegaram.**

---

## 12. CEO Dashboard — Fontes de Dados por Aba

| Aba | Tabelas lidas | O que mostra |
|---|---|---|
| `overview` | todas | KPIs gerais + painel de hoje + pendências abertas |
| `gerencia` | `ger_checklists` | tabela de histórico por turno |
| `dp` | `dp_checklists` + `dp_sugestoes` | lista clicável com drilldown por bloco |
| `subgerencia` | `subger_checklists` | lista com SubgerCEORow (expansível por etapa) |
| `vitoria` | `subger_noite_checklists` | lista de turnos da noite |
| `pendencias` | `checkon_pendencias` | pendências com status='aberta' de toda a equipe |
| `ranking` | `checklists` + `subger_checklists` + `subger_noite_checklists` | ranking unificado por score médio |
| `alertas` | `checklists` + `subger_checklists` + `subger_noite_checklists` | turnos com criticos>0, hoje primeiro |
| `turnos` | todas (5 tabelas) | histórico unificado, hoje em destaque |

### Painel "Hoje" no overview
Lê `hojeKpi` (useMemo) que filtra por `data === todayISO()` em cada conjunto de dados. Mostra status e horário de envio de cada perfil. Se não enviou, mostra "Não enviado".

---

## 13. Problemas Conhecidos e Dívidas Técnicas

| # | Problema | Severidade | Onde | Status |
|---|---|---|---|---|
| 1 | Três fórmulas de score diferentes | Alta | Todas as telas | Não resolvido |
| 2 | Status com valores inconsistentes entre perfis | Alta | CEO Dashboard | Não resolvido |
| 3 | GER_SAVE_KEY diferente entre GerenciaHub e GerChecklistScreen | Média | GerenciaHub/GerChecklistScreen | Não resolvido |
| 4 | sbInsert e sendWebhook falham silenciosamente | Alta | Todos os envios | Não resolvido |
| 5 | analisarHistorico busca histórico completo sem limite | Média | BoasVindasInteligente | Parcialmente (cache 4h) |
| 6 | ChecklistScreen (gerente/suporte) é legado sem etapas | Baixa | GerenteHub/SuporteHub | Não resolvido |
| 7 | Sem alertas em tempo real durante turno em andamento | Média | Toda a operação | Não resolvido |
| 8 | Arquivo único com 3.400+ linhas | Baixa | index.html | Estrutural |
| 9 | prox aceita 'Sim' ou 'Sim, fica pendente' dependendo do perfil | Média | salvarPendencias | Tratado com OR |

---

## 14. Convenções de Desenvolvimento

### Ao adicionar um novo item de checklist
1. Criar id único no formato `[prefixo][número]` (ex: `sg55`, `vt30`, `dp_b5_i1`)
2. Nunca reutilizar ids — ids são chaves em `resps` e `negs` no banco
3. Marcar `critico: true` apenas para itens que, se não feitos, comprometem diretamente segurança ou operação
4. Adicionar o item na etapa correta com base no horário

### Ao adicionar um novo perfil (role)
1. Adicionar em `DEFAULT_USERS` com role único
2. Adicionar rota em `App()` — `if(user.role === 'novo_role') return <NovoHub/>`
3. Criar tabela no Supabase com os campos canônicos
4. Usar a fórmula de score canônica: `100 - (falhas/total × 50) - (criticos × 8)`
5. Usar status canônicos: `'ok'` · `'atencao'` · `'critico'`
6. Chamar `salvarPendencias()` ao finalizar se o perfil tiver itens com prox
7. Adicionar ao CEO Dashboard em todas as abas relevantes

### Ao mudar uma regra de negócio
1. Atualizar este SPEC.md primeiro
2. Atualizar o código
3. Bump no `VERSION` (ex: `'2026.14'` → `'2026.15'`) para invalidar sessões antigas

### Convenção de nomenclatura de chaves localStorage
```
konclui_*   = dados de sessão e usuário (sistema)
checkon_*   = dados de checklist e rascunho (operacional)
ger_*       = rascunhos de gerência (legado — migrar para checkon_ger_*)
```

---

## 15. Variáveis de Ambiente e Configuração

Todas as configurações estão no topo do `<script>`:

```js
const SUPABASE_URL = 'https://crmeztihgrwnpoovzpqk.supabase.co'
const SUPABASE_KEY = 'sb_publishable_...'   // chave pública (anon key) — ok expor
const SISTEMA      = 'CheckON Pastel da Camilla'
const WEBHOOK      = 'https://hook.us2.make.com/...'
const WA_GROUP     = 'https://chat.whatsapp.com/...'
const VERSION      = '2026.14'
```

**`SUPABASE_KEY` é a chave anon/pública** — só tem acesso ao que as Row Level Security policies permitirem. Não é uma secret key. Ainda assim, não compartilhar publicamente.

**`WEBHOOK`** é o endpoint do Make.com que processa o relatório e envia e-mail. Se o Make.com mudar, atualizar aqui.

**`VERSION`** deve ser incrementado a cada deploy que mude dados do localStorage — invalida sessões e força releitura dos usuários.

---

## 16. O que Não Existe Ainda (Roadmap)

| Feature | Impacto | Esforço |
|---|---|---|
| Fórmula de score unificada | Alto — ranking comparável | Baixo |
| Confirmação real de envio (retry) | Alto — confiabilidade | Médio |
| Alerta por etapa concluída (webhook parcial) | Alto — CEO em tempo real | Médio |
| Filtro de período no CEO Dashboard | Alto — decisão gerencial | Baixo |
| Migração ChecklistScreen para etapas | Médio — consistência | Alto |
| Push notification | Alto — CEO proativo | Alto (requer service worker) |
| Score por etapa (peso diferente) | Médio — precisão | Médio |
| Painel de configuração (sem hard-code) | Muito alto — escalabilidade | Muito alto |

---

*Última atualização: Março 2026*
*Mantenedor: desenvolvimento CheckON*
