# CheckON — Guia do Desenvolvedor
**Use antes de fazer qualquer mudança no sistema**

---

## Antes de qualquer alteração

- [ ] Li o SPEC.md na seção relevante?
- [ ] Atualizei o VERSION no topo do index.html? (`'2026.14'` → próximo número)
- [ ] A mudança afeta dados já gravados no Supabase? Se sim, dados antigos vão continuar funcionando?
- [ ] A mudança afeta o `analisarHistorico`? O cache de 4h pode estar servindo dados antigos.

---

## Adicionar um item ao checklist

### Subgerência Manhã (`SUBGER_ETAPAS`)
```
1. Definir o id: formato sg[número], ex: sg55
   - Verificar que o id não existe em nenhuma etapa
   - Ids são chaves no banco — reutilizar id quebra dados históricos

2. Decidir a etapa:
   - Etapa 1 (sg_e1): ações de abertura — 9h a 11h
   - Etapa 2 (sg_e2): controle durante operação — 11h a 13h
   - Etapa 3 (sg_e3): fechamento — 13h+

3. Definir critico:
   - true  = segurança, operação ou cliente impactado diretamente se não feito
   - false = importante mas não para a operação

4. Adicionar no array da etapa correta em SUBGER_ETAPAS

5. Verificar se o total de itens mudou:
   - SPEC.md seção 4.1 diz 'total sempre 28'
   - Atualizar o SPEC.md com o novo total
```

### Subgerência Noite (`NOITE_ETAPAS`)
Mesma lógica, ids no formato `vt[número]`.

### DP (`DP_BLOCOS_DATA`, `DP_BLOCOS_SEMANAL`, `DP_BLOCOS_MENSAL`)
```
1. Id no formato dp_[bloco]_i[número], ex: dp_b1_i5
2. Se o item só aparece em contexto específico, adicionar campo cond:
   cond: 'admissao' | 'assinatura' | 'comunicado' | 'uniforme' | 'exame' | 'colaborador_trouxe'
3. Verificar se a lógica em DPChecklistScreen allItems filtra corretamente o cond
```

---

## Adicionar um novo usuário

```
1. Adicionar em DEFAULT_USERS:
   novouser: {
     name: 'Nome',
     login: 'novouser',
     role: 'role_existente',   // usar role já mapeado no App()
     pwd: PWD_2026,
     first: false,
     color: '#HEX',
     active: true
   }

2. Ou criar pelo UserManager (Bruna tem acesso) — mais seguro

3. Se for um novo role:
   a. Adicionar rota em App(): if(user.role === 'novo_role') return <NovoHub/>
   b. Criar NovoHub seguindo o padrão dos outros hubs
   c. Criar tabela no Supabase
   d. Adicionar ao CEO Dashboard
   e. Documentar no SPEC.md seção 2 e 4
```

---

## Mudar a fórmula de score

**ATENÇÃO: scores históricos no banco não são recalculados.**
Mudar a fórmula quebra comparabilidade histórica. Só fazer se:
- O novo score for claramente superior
- Houver consciência de que o ranking histórico vai ser inconsistente
- O período de transição for documentado no CHANGELOG.md

```
1. Localizar TODAS as ocorrências:
   grep -n "var score=Math.max" index.html

2. Atualizar TODAS (não deixar uma de fora — isso cria inconsistência)

3. Atualizar SPEC.md seção 3

4. Atualizar CHANGELOG.md

5. Bump no VERSION
```

---

## Mudar um chip de resposta (aconteceu/feito/prox)

Os valores dos chips são gravados como strings no banco em `negs`.
Mudar um chip significa que dados históricos vão ter o valor antigo e dados novos vão ter o valor novo.

```
Antes de mudar:
- O novo valor vai quebrar alguma lógica que compara a string?
- Verificar: salvarPendencias(), verificarTurnoHoje(), buscarPendenciasManhaParaNoite()
- Verificar: CEO Dashboard — qualquer lugar que filtra por valor de neg

Se o valor de 'prox' mudar:
- salvarPendencias() usa: n.prox === 'Sim, fica pendente' || n.prox === 'Sim'
- Adicionar o novo valor ao OR, não remover os antigos
```

---

## Adicionar uma nova aba ao CEO Dashboard

```
1. Adicionar tab ao array de navegação (linha ~1295):
   ['nova_tab', '🎯 Nome']

2. Adicionar fetch no useEffect load():
   if(tab==='overview'||tab==='nova_tab'){...}

3. Adicionar state no componente:
   var _novaData=useState([]);var novaData=_novaData[0];var setNovaData=_novaData[1];

4. Adicionar render:
   {!loading&&tab==='nova_tab'&&(...)}

5. Se a aba mostra dados que devem aparecer no ranking ou alertas:
   Adicionar 'nova_tab' nas condições de load de subgerData e noiteData
```

---

## Fazer deploy (atualizar o arquivo)

```
1. Testar localmente abrindo index.html no browser
2. Verificar login de cada perfil:
   - gabriela / 2026
   - vitoria / 2026
   - camila / 2026 (CEO)
3. Verificar que o painel "Hoje" do CEO está funcionando
4. Verificar que o briefing da Vitória carrega sem travar
5. Bump no VERSION se houver mudança de dados
6. Subir o arquivo para o servidor/hosting
```

---

## Quando o Supabase der erro

```
Sintomas: dados não aparecem, envio parece funcionar mas não grava

Verificar:
1. Console do browser (F12 > Console) — sbInsert e sbSelect logam erros
2. SUPABASE_KEY ainda é válida? (chave anon não expira, mas pode ser revogada)
3. Row Level Security no Supabase permite a operação?
4. A tabela existe com os campos esperados?

Fallback:
- analisarHistorico tem fallback para localStorage ('checkon_hist_[login]_[tabela]')
- Mas sbInsert não tem — dados podem se perder silenciosamente
```

---

## Quando um usuário reclamar que perdeu um rascunho

```
Rascunhos ficam no localStorage do dispositivo do usuário.
Se ela:
- Trocou de dispositivo: rascunho perdido (não há sync)
- Limpou dados do browser: rascunho perdido
- O VERSION mudou: SESSION foi limpa, mas rascunho pode ter sobrado

Verificar no console do dispositivo dela:
localStorage.getItem('checkon_subger_gabriela')  // subgerência manhã
localStorage.getItem('checkon_noite_vitoria')     // noite
localStorage.getItem('checkon_dp_viviane')        // DP
```

---

## Quando o CEO não ver dados de hoje

```
1. O turno foi enviado? (dados só aparecem ao finalizar, não durante)
2. A aba está carregando? (loading spinner)
3. O filtro de data está correto? hojeKpi usa todayISO() = new Date().toISOString().slice(0,10)
   Verificar se o dispositivo da Camila tem data/hora correta
4. Dados estão em qual tabela? Verificar no Supabase diretamente
```

---

## Regras que nunca devem ser quebradas

```
❌ Nunca reutilizar um id de item (sg1, vt1, dp_b1_i1, etc.)
❌ Nunca remover a verificação de role de gabriela e vitoria em getUsers()
❌ Nunca gravar status com valor diferente de 'ok'/'atencao'/'critico' em tabelas novas
❌ Nunca mudar o valor de prox sem adicionar o valor antigo ao OR em salvarPendencias()
❌ Nunca subir o arquivo sem bump no VERSION quando mudar dados de localStorage
❌ Nunca deletar registros do Supabase — apenas marcar status como inativo/resolvida
```

---

## Estrutura do arquivo index.html (seções principais)

```
Linha 1–240:     HTML head, CSS
Linha 241–300:   Constantes globais (SUPABASE, WEBHOOK, USERS_KEY, VERSION)
Linha 301–315:   Funções utilitárias (sbInsert, sbSelect, sendWebhook, Toast, useToast)
Linha 316–453:   Dados de checklists (GER_ITEMS, ITEMS, DP_BLOCOS_*)
Linha 454–550:   Validações e formulários de neg (DP)
Linha 551–785:   Componentes DP (DPBlocoItem, DPBlocoAcc, DPSummaryScreen)
Linha 786–960:   DPChecklistScreen e DPHub
Linha 961–1000:  GerNaoForm, GerChkItem, GerBlocoAcc
Linha 1001–1065: GerSummaryScreen, GerChecklistScreen
Linha 1066–1150: ChecklistScreen (legado — gerente/suporte)
Linha 1151–1225: SubgerCEORow
Linha 1226–1430: CEODashboard
Linha 1431–1685: CEOHub, GerenteHub, GerenciaHub, SuporteHub
Linha 1686–1715: UserManager
Linha 1716–1875: SUBGER_ETAPAS, chips, validação, SubgerBlocoItem, SubgerEtapaAcc
Linha 1876–2000: SubgerSummaryScreen, TurnoEncerrado
Linha 2001–2115: Funções de inteligência (verificarTurnoHoje, buscarPendencias, analisarHistorico)
Linha 2116–2450: BoasVindasInteligente, FeedbackCrescimento, SugestaoForm
Linha 2451–2615: SubgerChecklistScreen
Linha 2616–2680: HistoricoPanel
Linha 2681–2740: salvarPendencias, PendenciasBlock
Linha 2741–2895: NOITE_ETAPAS, chips, NoiteNaoForm, NoiteBlocoItem, NoiteEtapaAcc
Linha 2896–3000: NoiteSummaryScreen
Linha 3001–3130: NoiteChecklistScreen
Linha 3131–3250: SubgerenteNoiteHub
Linha 3251–3400: SubgerenteHub
Linha 3401–3435: Login, App (roteamento principal)
```

---

*Atualizado: Março 2026*
