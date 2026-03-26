# CheckON — Changelog e Registro de Decisões
**Pastel da Camilla**

*Cada entrada documenta o que mudou, por que mudou, e o que não pode ser revertido sem consequência.*

---

## [2026.14] — Março 2026 — Sessão atual

### Adicionado
- **Painel "Hoje" no CEO Dashboard** — card no overview mostrando status em tempo real de Gabriela, Vitória, Derik e Viviane no dia atual
- **Tab Turnos unificada** — histórico de todas as 5 tabelas numa view única, ordenada com hoje primeiro, badge "HOJE" destacado
- **Alertas CEO corrigidos** — críticos de `subger_checklists` e `subger_noite_checklists` agora aparecem na aba Alertas
- **Ranking Geral** — inclui Gabriela e Vitória além de gerente/suporte
- **`buscarPendenciasManhaParaNoite`** — função nova que lê o turno da manhã de hoje e conta pendências para exibir no briefing da Vitória
- **`verificarTurnoHoje`** — função nova com query limitada (3 registros) em vez de buscar histórico completo
- **`salvarPendencias` para a manhã** — Gabriela agora grava pendências em `checkon_pendencias` ao fechar turno
- **Botão de escape na tela de turno enviado** — Vitória (e qualquer usuário) pode iniciar novo turno mesmo após já ter enviado hoje

### Corrigido
- **Bug crítico Vitória** — tela travava sem saída quando turno já tinha sido enviado hoje
- **Item duplicado DP** — `dp_b4_i5` aparecia duas vezes no array de itens
- **Variáveis duplicadas** — `var _enviado` e `var _envScore` declarados duas vezes em SubgerChecklistScreen e DPHub
- **`buscarPendenciasManhaParaNoite`** — corrigida para filtrar por data no servidor em vez de buscar todos os registros de todos os usuários

### Decisões de design registradas
- O botão de ação principal do `BoasVindasInteligente` renderiza **imediatamente**, sem esperar carregamento do histórico. Histórico carrega progressivamente. Isso foi deliberado — UX não pode bloquear acesso.
- `buscarPendenciasManhaParaNoite` lê diretamente de `subger_checklists` (não de `checkon_pendencias`). Motivo: garante que o card da Vitória reflete o turno real de hoje, não pendências antigas de dias anteriores.
- O estado "turno já enviado" mostra resumo mas não bloqueia mais. Um subgerente pode querer registrar um segundo turno no mesmo dia (cobertura, emergência).

---

## [2026.13] — Março 2026 — Redesign da tela de início

### Adicionado
- **`BoasVindasInteligente` reescrito** — três camadas: contexto do turno (score, streak, alerta), estado do checklist (rascunho ou etapas), ação clara (botão único)
- **SubgerenteHub** — tela de briefing completa substituindo o hub simples com dois botões
- **SubgerenteNoiteHub** — idem, com card de pendências da manhã
- **GerenteHub, GerenciaHub, SuporteHub** — migrados para o padrão de briefing
- **Acesso ao histórico pelo header** — botão "📈 Evolução" no canto direito do header em vez de card na tela inicial

### Removido
- Tela de início dentro de `SubgerChecklistScreen` — hub agora faz o briefing, checklist começa direto na etapa 1
- Tela de início dentro de `NoiteChecklistScreen` — idem
- Spinner bloqueante do `BoasVindasInteligente` — não bloqueia mais o botão de início

### Decisões de design registradas
- O `ChecklistScreen` (gerente/suporte) mantém a tela de entrada com seleção de turno porque depende de contexto (manhã/tarde, delivery, novato, falta) que não faz sentido no briefing. Candidato a refatoração futura.

---

## [2026.12] — Fevereiro/Março 2026 — CEO Dashboard e pendências

### Adicionado
- **CEO Dashboard** com abas: Geral, Gerência, DP, Gabriela, Vitória, Pendências, Ranking, Alertas, Turnos
- **SubgerCEORow** — linha expansível no CEO com detalhamento por etapa e falhas
- **PendenciasBlock** — bloco de pendências abertas com resolução inline
- **HistoricoPanel** — painel de evolução pessoal com streak, diff de falhas, top falha
- **FeedbackCrescimento** — card motivacional na tela de summary

### Decisões de design registradas
- CEO lê todas as tabelas sem filtro de período por padrão. Com volume baixo (< 500 registros por tabela) isso é aceitável. Quando passar de 1.000 registros por tabela, adicionar filtro `gte` nos últimos 90 dias.
- Pendências são salvas em tabela separada (`checkon_pendencias`) em vez de derivadas on-the-fly das respostas. Motivo: permite marcar como resolvida sem reabrir o checklist original.

---

## [2026.11] — Fevereiro 2026 — Vitória (turno da noite)

### Adicionado
- **NoiteChecklistScreen** com 3 etapas por horário
- **NoiteSummaryScreen** com relato para a manhã
- **`NOITE_ETAPAS`** — 28 itens em 3 etapas (17h–18h, 18h–21h, 21h+)
- **`salvarPendencias`** — primeiro uso, só para o turno da noite

### Decisões de design registradas
- O relato final da noite é explicitamente direcionado "para a manhã" (placeholder diz "O que a Gabriela precisa saber ao chegar..."). Isso é intencional — cria consciência de passagem de turno.

---

## [2026.10] — Janeiro/Fevereiro 2026 — Subgerência manhã

### Adicionado
- **SubgerChecklistScreen** com 3 etapas por horário
- **`SUBGER_ETAPAS`** — 28 itens em 3 etapas (9h–11h, 11h–13h, 13h+)
- **Item especial `sg26`** (`SUBGER_RECHEIO_ID`) — item de verificação de recheios com aviso especial no formulário de neg
- **`SugestaoForm`** — campo de sugestão de melhoria no final do checklist

### Decisões de design registradas
- O id `sg26` tem comportamento especial em `SubgerNaoForm` (mostra aviso de contexto sobre dispensa). Se renumerar os itens, o `SUBGER_RECHEIO_ID` deve ser atualizado.
- Chips de "o que aconteceu" incluem causas de recheio (`'Não verifiquei a dispensa'`, `'Peguei recheio sem conferir validade'`, `'Organização incorreta na dispensa'`) — foram adicionados após identificar padrão de falha recorrente nesse item.

---

## [2026.09] — Janeiro 2026 — DP (Viviane)

### Adicionado
- **DPChecklistScreen** com três frequências: diário, semanal, mensal
- **Contexto adaptativo** — perguntas de contexto antes do checklist adaptam itens visíveis (admissão, assinatura, ocorrência, colaborador_trouxe)
- **`DPNaoForm`** — formulário de neg com detecção automática de categoria (`detectarCategoriaDP`) que muda os chips de ocorrência

### Decisões de design registradas
- `detectarCategoriaDP` usa keywords no texto do item para categorizar automaticamente. Isso é frágil a mudanças de texto — se um item for renomeado, pode perder a categoria. Candidato a usar campo `categoria` explícito no futuro.
- DP tem `freq` no banco porque a mesma usuária preenche três tipos de checklist. Outros perfis não têm — cada checklist é um único tipo de turno.

---

## [2026.08] — Dezembro 2025 — Gerência (Derik)

### Adicionado
- **GerChecklistScreen** com 17 itens em 4 blocos
- **`SharedNaoForm`** — formulário estruturado de neg com problema/causa/impacto/ação/resp/prox
- **`GerSummaryScreen`** com buildText para WhatsApp

### Decisões de design registradas
- Gerência não tem etapas com horário — o checklist é de visão geral do turno, não de execução por hora. Candidato a ganhar etapas no futuro.
- `validateNeg` para gerência exige 6 campos (problema, causa, impacto, ação, resp, prox) — mais rigoroso que os outros perfis (3 campos). Isso reflete o papel gerencial de análise mais completa.

---

## [2026.07] — Novembro/Dezembro 2025 — Base operacional

### Adicionado
- **ChecklistScreen** (legado) para gerente e suporte — sem etapas, por blocos
- **`ITEMS`** — 12 itens manhã + 12 itens tarde com contexto adaptativo (delivery, novato, falta, troco)
- **Sistema de score e webhook** — primeira versão
- **Login com hash SHA-256** — senhas não ficam em texto puro no localStorage
- **CEOHub** básico

### Decisões de design registradas
- SHA-256 para senha é aceitável para esse nível de sensibilidade — não é sistema bancário. A senha `'2026'` é usada como padrão de reset. Mudar para hash real da nova senha ao resetar.
- `ChecklistScreen` usa terminologia `'conforme'`/`'ressalvas'` herdada da versão original. Diferente dos novos perfis que usam `'ok'`/`'atencao'`. É uma inconsistência conhecida que não foi corrigida para não quebrar dados históricos.

---

*Este arquivo deve ser atualizado a cada mudança significativa no sistema.*
*Regra: se você precisou pensar mais de 5 minutos em uma decisão, ela entra aqui.*
