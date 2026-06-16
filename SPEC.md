# Especificação — Gerador de Projeto Comercial DWV

> **Para o Claude Code:** este documento descreve a lógica completa de um wizard que monta uma proposta comercial com precificação oculta. O arquivo `projeto-comercial-dwv.html` (anexo) é a **implementação funcional de referência** — quando houver qualquer dúvida de comportamento ou visual, ele é a fonte de verdade. Este `.md` existe para permitir reimplementação em outro stack (Next.js, React, Vue, etc.) preservando exatamente as regras de negócio.

---

## 1. Resumo

Wizard multi-etapa onde o cliente seleciona serviços organizados **por tema**. O sistema calcula o preço **por trás**, sem nunca revelar ao cliente qual "plano" ele está comprando. Ao final, gera uma **proposta arquitetada** (capa, métricas, módulos, cronograma, investimento).

### Regra inviolável
O cliente **NUNCA** vê o nome do plano (Essencial / Avançado / Nexus). Os planos são uma estrutura interna de precificação. Na UI, os serviços aparecem agrupados por tema funcional.

---

## 2. Modelo de dados

### 2.1 Planos (internos / ocultos)

| Plano | Piso (R$) | Teto (R$) | Nº atividades | Incremento por atividade (R$) |
|-------|-----------|-----------|---------------|-------------------------------|
| E (Essencial) | 828 | 1.200 | 4 | 124,00 |
| A (Avançado) | 1.590 | 2.500 | 11 | 91,00 |
| N (Nexus) | 5.200 | 8.000 | 11 | 280,00 |

- **Incremento** = `(teto − piso) / (nº_atividades − 1)`.
- Hierarquia: **N > A > E**. Cada plano engloba os de baixo.

```js
const PLAN = {
  E: { piso: 828,  teto: 1200, inc: 124 },
  A: { piso: 1590, teto: 2500, inc: 91  },
  N: { piso: 5200, teto: 8000, inc: 280 }
};
const RANK = ['N', 'A', 'E']; // ordem de prioridade do plano cobrado
const IMPLEMENTACAO = 5800;   // taxa única
const SETUP_DAYS = 30;
const RAMPUP_DAYS = 60;       // "aceleração das vendas"
```

### 2.2 Serviços (26 no total)

Cada serviço tem: `id`, `nome`, `desc`, `plan` (E/A/N, **oculto**), `etapa`, e opcionalmente `extra` (custo variável exibido como selo).

| id | Serviço | Plano | Etapa |
|----|---------|-------|-------|
| i1 | Exibição no Marketplace da DWV | E | 1 |
| i2 | Gerador de Landing Pages ilimitadas | E | 1 |
| i9 | 12 Stories na plataforma | A | 1 |
| i27 | 2 mil corretores no funil | N | 1 |
| i3 | Central de ações de parceiros | E | 2 |
| i7 | Receba e gerencie intenções de propostas | A | 2 |
| i8 | Construa sua rede social de corretores | A | 2 |
| i4 | Integração com outros CRMs imobiliários | E | 3 |
| i5 | CRM completo com Kanban | A | 3 |
| i10 | Painel de gestão do time comercial | A | 3 |
| i13 | Sistema de gestão de visitas presenciais | A | 3 |
| i6 | IA personalizada para autoatendimento 24h | A | 4 |
| i14 | SDR por IA para qualificação de contatos | A | 4 |
| i11 | Separação da base em grupos pra disparo | A | 5 |
| i12 | E-mail marketing | A | 5 |
| i15 | Campanhas pelo WhatsApp (+R$0,60/msg) | A | 5 |
| i18 | Ativação de base de corretores ociosos | N | 5 |
| i19 | Arquitetura para cliente final | N | 6 |
| i20 | Arquitetura para atrair corretores | N | 6 |
| i21 | 1 Webinar estruturado pela DWV | N | 6 |
| i16 | Inteligência de mercado | N | 7 |
| i23 | Pesquisas de satisfação de corretores | N | 7 |
| i24 | Comparação com concorrentes | N | 7 |
| i25 | Diagnóstico comercial do setor de parcerias | N | 7 |
| i26 | Playbook comercial (tese + objeções) | N | 8 |
| i17 | Concierge dedicada | N | 8 |

Distribuição: **E = 4, A = 11, N = 11**. (As contagens batem com `nº atividades` de cada plano — importante para o teto.)

> O texto exato de `nome` e `desc` de cada serviço está no array `STEPS` dentro do HTML de referência. Copie de lá.

### 2.3 Etapas (8) — fachada neutra para o cliente

| # | Título | Subtítulo |
|---|--------|-----------|
| 1 | Exibição e apresentação | Como o empreendimento aparece para corretores e clientes. |
| 2 | Relacionamento com parceiros | O vínculo com os corretores e a entrada de propostas. |
| 3 | Operação e carteira | A gestão interna do time e das carteiras. |
| 4 | Atendimento automático | Inteligência artificial para nenhum corretor ou lead ficar sem resposta. |
| 5 | Disparos e ativação | Comunicação ativa com a base de corretores. |
| 6 | Aquisição com tráfego | Atração de novos corretores e clientes com mídia paga. |
| 7 | Pesquisas e inteligência | Diagnóstico do mercado e da operação antes de agir. |
| 8 | Material e suporte estratégico | Argumento de venda estruturado e acompanhamento dedicado. |

Regra de telas: **máximo 4 serviços por etapa**. Ao adicionar serviços novos, dividir em mais etapas mantendo esse limite.

---

## 3. Lógica de precificação (núcleo)

```
ENTRADA: lista de serviços selecionados (cada um com seu plano oculto)

1. Conte os selecionados por plano → counts = { E, A, N }
2. Determine o PLANO COBRADO = primeiro plano em [N, A, E] cujo count > 0.
   (= o plano mais alto que o cliente tocou)
3. Se nenhum selecionado → mensal = 0.
4. Caso contrário, seja `c` o plano cobrado:
   mensal = min( PLAN[c].piso + (counts[c] − 1) × PLAN[c].inc , PLAN[c].teto )
5. Itens de planos INFERIORES ao cobrado NÃO somam valor (vêm englobados).
   Só os itens DO plano cobrado movem a posição na faixa.

DERIVADOS:
   anual        = mensal × 12
   total_ano_1  = anual + IMPLEMENTACAO (5800)
   pct_vgv      = total_ano_1 / VGV × 100   (ver §6)
```

Implementação de referência:

```js
function calc(selecionados) {
  const counts = { E: 0, A: 0, N: 0 };
  selecionados.forEach(s => counts[s.plan]++);
  let cobrado = null;
  for (const p of ['N', 'A', 'E']) { if (counts[p] > 0) { cobrado = p; break; } }
  if (!cobrado) return { mensal: 0, cobrado: null };
  const cfg = PLAN[cobrado];
  const mensal = Math.min(cfg.piso + (counts[cobrado] - 1) * cfg.inc, cfg.teto);
  return { mensal, cobrado };
}
```

### Comportamento esperado (importante para QA)
- Tocar **1** item de um plano já cobra o **piso** daquele plano.
- Marcar **todos** os itens de um plano cobra o **teto**.
- Remover o último item do plano mais alto faz o valor **cair de degrau** para o plano imediatamente abaixo (salto esperado, não é bug).

---

## 4. Casos de teste (devem passar exatamente)

| Cenário (counts E/A/N) | Mensal esperado |
|------------------------|-----------------|
| 1 / 0 / 0 | R$ 828 |
| 4 / 0 / 0 | R$ 1.200 |
| 0 / 1 / 0 | R$ 1.590 |
| 0 / 11 / 0 | R$ 2.500 |
| 0 / 0 / 1 | R$ 5.200 |
| 0 / 0 / 11 | R$ 8.000 |
| 0 / 5 / 3 | R$ 5.760 |
| 4 / 11 / 1 | R$ 5.200 |
| 3 / 2 / 0 | R$ 1.681 |
| 0 / 0 / 0 | R$ 0 |

---

## 5. Cronograma (90 dias)

A partir da `data_inicio` informada no formulário:

```
fim_setup = data_inicio + 30 dias        (fase "Preparação")
entrega   = data_inicio + 90 dias        (fim da fase "Aceleração das vendas", 60 dias)
```

Atenção a fuso: fazer o parse de `YYYY-MM-DD` como data **local** (não UTC), para não perder 1 dia:

```js
function parseISO(v){ const [y,m,d]=v.split('-').map(Number); return new Date(y, m-1, d); }
function addDays(dt,n){ const d=new Date(dt.getTime()); d.setDate(d.getDate()+n); return d; }
```

Se `data_inicio` estiver vazia, exibir a estrutura (30/60/90 dias) com datas como "—".

---

## 6. Percentual do VGV

```
pct = total_ano_1 / VGV × 100
casas decimais = 3 se pct < 0,01 ; senão 2
```

Formatar em pt-BR. Só exibir o bloco de % se `VGV > 0`.

---

## 7. Fluxo de telas

```
Step 0  → Dados do projeto (formulário)
Step 1..8 → Seleção de serviços por tema (1 etapa por tela, máx. 4 itens)
Step 9  → Proposta (gerada ao entrar)
```

Barra de progresso = `step_atual / (total_steps − 1) × 100`.

### 7.1 Campos do formulário (Step 0)
`empresa` (texto), `responsavel`, `contato`, `cidade`, `data_apresentacao` (date), `data_inicio` (date), `representante`, `vgv` (numérico com máscara de milhar pt-BR).

Máscara do VGV: remover não-dígitos, formatar com `toLocaleString('pt-BR')` a cada input.

---

## 8. Tela de proposta (Step 9)

Montada ao entrar na tela. Seções, em ordem:

1. **Capa** — eyebrow "Proposta · Projeto Comercial", nome da empresa (h1), e linha de metadados: nº da proposta, representante, cidade, data de apresentação. Omitir cada metadado vazio.
2. **Métricas** (4 cards): `Frentes ativadas` (nº de etapas com ≥1 selecionado), `Ações no projeto` (total selecionado), `Implementação` (90 dias), `Investimento` (mensal).
3. **Arquitetura do projeto** — para cada etapa com ≥1 serviço selecionado, renderizar um **módulo numerado** (01, 02…) conectado por uma linha vertical (estética de blueprint), contendo o título da frente e seus componentes (os serviços selecionados daquela etapa). Serviços com `extra` exibem o selo.
4. **Plano de implementação** — cronograma de 90 dias (Preparação 30d → Aceleração 60d) com as datas calculadas.
5. **Investimento** — painel com: mensal (destaque, count-up), operação 12 meses (= mensal×12), implementação única (R$ 5.800), total no 1º ano, e o % do VGV.
6. **Nota de próximo passo** + botão "Exportar proposta (PDF)" (`window.print()`).

> Agrupamento na proposta é **sempre por tema/frente**, nunca por plano.

### 8.1 Número de proposta (determinístico)
```js
function propNumber(empresa){
  const y = new Date().getFullYear();
  let h = 0; for (const c of (empresa||'DWV')) h = (h*31 + c.charCodeAt(0)) % 9000;
  return 'DWV-' + y + '-' + String(1000 + h).padStart(4,'0');
}
```

---

## 9. Design system (tons de cinza, sóbrio)

```
--bg:        #1B1B1E   (fundo)
--surface:   #232327   (cards)
--surface-2: #2A2A2F   (painel de investimento, badges de módulo)
--border:    #36363B
--border-soft:#2E2E33
--text:      #F4F4F5
--muted:     #9A9AA0
--faint:     #5A5A60
--accent:    #C8C8CC   (eyebrows, barra de progresso, marcadores, bordas de seleção)
botão primário: fundo #E6E6E8 / texto #0E0E10
```

Tipografia: **Space Grotesk** (display/títulos), **Inter** (corpo), **JetBrains Mono** (rótulos, números, eyebrows). Sem cor de marca — apenas a escala de cinza. Sentence case.

Logo DWV: `https://site.dwvapp.com.br/wp-content/uploads/2025/07/cropped-cropped-Prancheta-1@2x-8.png`

Componentes-chave: card de serviço clicável (checkbox custom com check), faixa de progresso, cards de métrica, módulo de arquitetura (badge numerado + linha vertical + componentes), painel de investimento (mensal grande + 3 células de breakdown), bloco de cronograma (2 fases proporcionais 30/60).

---

## 10. Notas de implementação

- **WhatsApp (i15)** tem custo extra variável de **+R$ 0,60/mensagem** — é informativo (selo), **não entra** no cálculo do valor fixo mensal.
- Todos os valores de plano, textos de serviço e etapas devem ser **configuráveis** (idealmente em banco / arquivo de config), não hard-coded espalhados pela UI.
- A taxa de implementação é um valor único (`IMPLEMENTACAO = 5800`), exibida separada do mensal.
- Arredondar todo número exibido (`Math.round` / `toLocaleString('pt-BR')`).
- Acessibilidade mínima: foco visível, responsivo até mobile (métricas viram 2 colunas, cronograma empilha).
- Sugestão de modelagem (Supabase): tabelas `plans(code, piso, teto, inc)`, `services(id, nome, desc, plan_code, etapa_id, extra)`, `etapas(id, ordem, titulo, subtitulo)`, e `propostas(...)` para persistir cada proposta gerada com os serviços escolhidos e o snapshot de preço.

---

## 11. Arquivo de referência

`projeto-comercial-dwv.html` — implementação funcional completa e exata (UI + lógica). Use como verdade absoluta para textos, comportamento de cálculo, animações (count-up) e layout. Este `.md` é o mapa; o HTML é o território.
