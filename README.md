# Projeto Comercial · DWV

Wizard de proposta comercial. O cliente seleciona ações agrupadas por **tema/frente** (8 etapas) e o sistema calcula o investimento por trás, via planos internos ocultos (Essencial / Avançado / Nexus) — o nome do plano nunca aparece na interface. Ao final, gera uma proposta arquitetada (capa, métricas, blueprint de módulos, cronograma de 90 dias e investimento).

## Rodar localmente

É um app estático de arquivo único. Basta abrir `index.html` no navegador, ou servir:

```bash
npx http-server -p 8755
```

## Estrutura

- `index.html` — implementação completa (UI + lógica de precificação + geração da proposta).
- `SPEC.md` — especificação das regras de negócio (fonte para reimplementação em outro stack).

## Regras de preço (resumo)

| Plano (oculto) | Piso | Teto | Atividades | Incremento |
|----------------|------|------|------------|------------|
| Essencial | R$ 828 | R$ 1.200 | 4 | R$ 124 |
| Avançado | R$ 1.590 | R$ 2.500 | 11 | R$ 91 |
| Nexus | R$ 5.200 | R$ 8.000 | 11 | R$ 280 |

Cobra-se o plano mais alto que o cliente tocou; itens de planos inferiores vêm englobados. Implementação única de R$ 5.800.
