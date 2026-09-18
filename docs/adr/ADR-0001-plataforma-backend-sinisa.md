# ADR-0001: Plataforma de backend do SINISA — permanência em Yii2 e estratégia de evolução

**Status:** Proposto
**Data:** 2026-08-05
**Contexto do projeto:** TED nº 996611 — SNSA/Ministério das Cidades × UNIVASF (36 meses, mês 1 = maio/2026)
**Meta relacionada:** Meta 1 — Produtos 1.1, 1.3 e 1.4
**Decisores:** Coordenação Geral do TED (UNIVASF) · Equipe técnica da SNSA

---

## 1. Contexto

### 1.1 Estado atual do sistema

Os três sistemas sob escopo compartilham a mesma base tecnológica:

| Sistema | Framework | Volume |
|---|---|---|
| `sinisa-locais` | Yii2 2.0.52 / PHP 8.2 | 2.119 arquivos PHP, 42 controllers, 147 models, 9 módulos |
| `sinisa-regionais` | Yii2 2.0.52 / PHP 8.2 | mesma stack, `composer.json` praticamente idêntico |
| `sinisa-planos` | Yii2 2.0.52 / PHP 8.2 | escopo menor |

Infraestrutura já containerizada: `php:8.2.18-apache` + `postgres:17` via Docker Compose. Suíte Codeception presente, com cobertura a verificar.

Os módulos funcionais (`agua`, `esgoto`, `residuos_solidos`, `aguas_pluviais`, `gestao_municipal`, `relatorios` e respectivas consolidações) implementam o ciclo anual de coleta — o núcleo operacional que não pode parar.

### 1.2 Dívida técnica identificada

O framework em si está atualizado (2.0.52 é uma versão recente da linha 2.0). O problema não é o Yii — é a camada de apresentação e a gestão de dependências:

| Item | Versão em uso | Situação |
|---|---|---|
| `yiisoft/yii2-bootstrap` | 2.0.11 | Bootstrap 3 — sem suporte desde 2019 |
| `components/jquery` | 2.2.4 | linha descontinuada desde 2016 |
| `almasaeed2010/adminlte` | 2.3.6 | tema de 2016, acoplado ao Bootstrap 3 |
| 5 pacotes em `dev-master` | sem pin | `cyneek/yii2-widget-upload-crop`, `kartik-v/yii2-editors`, `maksyutin/yii2-dual-list-box`, `mg-code/yii2-session-timeout-warning`, `newerton/yii2-jcrop` |
| ~30 dependências com `"*"` | sem teto de versão | build não reprodutível a partir do `composer.json` |
| 4 patches manuais via `cweagans/composer-patches` | — | correções aplicadas sobre `vendor/`, inclusive no core do Yii |

Os pacotes em `dev-master` são o risco mais agudo: são forks pessoais sem releases, sem garantia de manutenção, e um `composer update` pode quebrar o sistema sem aviso. O `composer.lock` é hoje a única coisa segurando a reprodutibilidade.

### 1.3 O fator decisivo: janela de suporte

- **Yii 2.0** recebe correções de segurança e compatibilidade com PHP até **novembro de 2027**.
- O TED vai de maio/2026 a **abril/2029**.

Novembro de 2027 é o **mês 19 de 36**. O fim de suporte do framework cai no meio do projeto — durante a execução das Metas 3, 5, 6 e 7. Qualquer decisão que ignore isso deixa a SNSA com um sistema nacional sem suporte upstream antes mesmo da entrega final.

### 1.4 Ecossistema

Yii 3.0 tornou-se estável em 31/12/2025, mas não é um upgrade — é uma reescrita. A arquitetura mudou para um conjunto de pacotes independentes com versionamento próprio, e não há caminho de migração incremental a partir do 2.0.

Quanto à comunidade: Yii perdeu tração de forma consistente. Laravel concentra ~28% de mindshare no ecossistema PHP e Symfony ~18%, ambos em crescimento; Yii aparece em quinto lugar nos rankings de 2026, com atividade de comunidade e demanda de mercado em declínio. Isso tem efeito prático direto no TED: **contratar e repor desenvolvedores Yii em Petrolina, ao longo de 36 meses, é mais difícil e mais caro do que contratar para Laravel ou Symfony.**

### 1.5 Restrições

- O ciclo anual de coleta do SINISA não pode ser interrompido (Meta 3, meses 1–36).
- Metas 4, 5, 6 e 7 exigem entregas funcionais contínuas — não há janela para congelar o sistema e reescrever.
- Equipe: 3 desenvolvedores júnior PJ + 2 estagiários, sem sênior dedicado até o momento.
- O Plano de Trabalho já especifica React + Leaflet para a plataforma geoespacial (Meta 6), o que impõe uma fronteira front/back de qualquer forma.

---

## 2. Decisão

**Manter o Yii2 como núcleo dos módulos de coleta e desenvolver todo artefato novo fora dele, atrás de uma API, adotando o padrão Strangler Fig.**

Em concreto:

1. Os módulos de coleta existentes (`agua`, `esgoto`, `residuos_solidos`, `aguas_pluviais`, `gestao_municipal`) permanecem em Yii2 e recebem apenas manutenção evolutiva e corretiva (Meta 3).
2. As Metas 4 (módulos Rural e Regulação), 6 (série histórica e geoespacial) e 7 (painéis e chatbot) são construídas como **serviços desacoplados**, consumindo o PostgreSQL compartilhado via API REST — não como novos módulos Yii2.
3. O Yii2 ganha uma camada de API para expor os dados de coleta aos novos serviços.
4. Executa-se um **saneamento de dependências** no primeiro trimestre, tratado como pré-requisito e não como melhoria opcional.
5. A decisão sobre o framework de destino final (Laravel, Symfony ou Yii3) é **deliberadamente adiada** para o mês 18, quando a equipe terá maturidade e o desenho de API estará provado. Este ADR será substituído por um ADR-0002 nessa data.

---

## 3. Opções consideradas

### Opção A — Manter Yii2 e apenas evoluir dentro dele

| Dimensão | Avaliação |
|---|---|
| Complexidade | Baixa |
| Custo | Baixo no curto prazo |
| Escalabilidade | Limitada — acoplamento crescente ao legado |
| Familiaridade da equipe | Precisa ser construída de qualquer forma |
| Risco de suporte | **Alto** — sem framework suportado a partir do mês 19 |

**Prós:** entrega mais rápida no curto prazo; nenhuma curva de aprendizado adicional; menor risco de regressão imediata.

**Contras:** entrega à SNSA, em 2029, um sistema sobre framework sem suporte; agrava o acoplamento a Bootstrap 3 e jQuery 2; incompatível com o React/Leaflet já previsto na Meta 6; mercado de contratação restrito por 36 meses.

### Opção B — Migração big-bang para Laravel ou Symfony

| Dimensão | Avaliação |
|---|---|
| Complexidade | **Muito alta** |
| Custo | Alto — consome orçamento de várias metas |
| Escalabilidade | Boa no destino |
| Familiaridade da equipe | Baixa (equipe júnior) |
| Risco de suporte | Baixo no destino |

**Prós:** resolve a dívida de uma vez; ecossistema com comunidade ativa e mão de obra abundante; alinhamento com o mercado.

**Contras:** reescrever 2.119 arquivos PHP e 147 models com equipe júnior, sem interromper o ciclo anual de coleta, não é executável. Consumiria as Metas 4 a 7 sem produzir os produtos que elas exigem. **Descartada.**

### Opção C — Strangler Fig: Yii2 congelado + serviços novos desacoplados

| Dimensão | Avaliação |
|---|---|
| Complexidade | Média |
| Custo | Médio — diluído ao longo do TED |
| Escalabilidade | Boa |
| Familiaridade da equipe | Construída gradualmente |
| Risco de suporte | Médio — mitigável |

**Prós:** cada meta entrega valor sem depender de uma migração concluída; os novos módulos nascem em stack moderna; a fronteira de API é exigida pela Meta 5 (integração Locais × Regionais) de qualquer forma; permite decidir o destino final com informação real; equipe júnior aprende em código novo, com menor risco de quebrar a coleta.

**Contras:** convivência de duas stacks aumenta a carga cognitiva e o custo de infraestrutura; exige disciplina para que a fronteira não seja contornada; parte do Yii2 pode sobreviver ao TED.

### Opção D — Migrar Yii2 → Yii3

| Dimensão | Avaliação |
|---|---|
| Complexidade | Alta |
| Custo | Alto |
| Escalabilidade | Boa |
| Familiaridade da equipe | Baixa |
| Risco de suporte | Médio — comunidade menor |

**Prós:** algum reaproveitamento conceitual; arquitetura moderna (PSR, DI).

**Contras:** o esforço é comparável ao da Opção B, mas o destino tem a menor comunidade entre as alternativas. Paga-se o preço de uma reescrita sem ganhar o mercado de contratação. **Pior relação custo-benefício das quatro.**

---

## 4. Análise de trade-offs

O trade-off central é **risco de entrega × risco de obsolescência**.

A Opção A minimiza o risco de entrega e maximiza o de obsolescência: entrega tudo no prazo, sobre uma base sem suporte. A Opção B faz o inverso e provavelmente não entrega. A Opção C aceita um custo médio permanente — duas stacks convivendo — em troca de não ter que apostar tudo em uma única decisão tomada no mês 4, quando a equipe ainda está se internalizando no sistema.

Vale nomear o que a Opção C **não** resolve: o Yii2 continua rodando a coleta e continua sem suporte a partir do mês 19. A mitigação é reduzir sua superfície (nada novo é construído nele) e manter o PHP atualizado, já que a maior parte do risco de segurança em fim de suporte vem da runtime, não do framework. Isso é mitigação, não solução — e precisa ser declarado explicitamente à SNSA.

Há também um argumento organizacional a favor da C: com 3 desenvolvedores júnior, código novo e isolado é um ambiente de aprendizado muito melhor do que refatoração de legado. Trabalhar dentro de 147 models desconhecidos, sem um sênior revisando, é receita para regressão na coleta.

---

## 5. Consequências

**Fica mais fácil:**

- Entregar as Metas 6 e 7 — React/Leaflet e chatbot já nascem fora do Yii2, como o Plano de Trabalho pressupõe.
- Cumprir a Meta 5, já que a interoperabilidade Locais × Regionais passa a ser exercitada na própria fronteira de API.
- Contratar e repor desenvolvedores para a parte nova do sistema.
- Testar: serviços novos nascem com CI/CD e testes automatizados (Meta 3), sem herdar a suíte Codeception legada.

**Fica mais difícil:**

- Operar: duas stacks, dois pipelines, dois conjuntos de logs e observabilidade.
- Manter consistência de dados entre o que o Yii2 escreve e o que os serviços leem — exige contrato de dados formal.
- Onboarding: um desenvolvedor novo precisa entender dois mundos.
- Custo de infraestrutura sobe.

**Precisará ser revisitado:**

- **Mês 18** — decisão sobre o framework de destino final (ADR-0002), com a experiência real da equipe como insumo.
- **Mês 19 (novembro/2027)** — fim de suporte do Yii 2.0; reavaliar o risco residual com a SNSA e registrar formalmente.
- **Mês 30** — definir se o Yii2 é aposentado dentro do TED ou entregue como legado documentado, com plano de sucessão.

---

## 6. Itens de ação

**Trimestre 1 (meses 4–6) — saneamento, pré-requisito para tudo**

1. ☐ Fixar (pin) as ~30 dependências com `"*"` em versões explícitas no `composer.json`.
2. ☐ Substituir ou internalizar os 5 pacotes em `dev-master` — avaliar fork sob controle da UNIVASF para cada um.
3. ☐ Documentar os 4 patches manuais: motivo, escopo e condição para remoção.
4. ☐ Rodar `composer audit` + SCA completo nos três repositórios (Produto 1.1).
5. ☐ Registrar os ambientes dev/homolog/prod em IaC a partir do Docker Compose existente (Produto 1.2).
6. ☐ Levantar a cobertura real da suíte Codeception e definir a linha de base.

**Trimestre 2 (meses 7–9) — fronteira**

7. ☐ Especificar o contrato de API do Yii2 (OpenAPI) para os módulos de coleta.
8. ☐ Implementar a camada de API no Yii2, sem alterar a lógica de coleta.
9. ☐ Definir a stack dos serviços novos e registrar em ADR-0002-prévio.
10. ☐ Implantar o pipeline de CI/CD com branching por ciclo de coleta (Meta 3).

**Contínuo**

11. ☐ Manter o PHP na versão suportada mais recente compatível com Yii 2.0.
12. ☐ Nenhum módulo funcional novo dentro do Yii2 — regra de governança, verificada em code review.
13. ☐ Comunicar formalmente à SNSA o marco de novembro/2027 e o risco residual assumido.

---

## 7. Referências

- Plano de Trabalho do TED nº 996611, SEI 80000.002526/2026-19
- [Yii Release History and End-of-Life Status — VersionLog](https://versionlog.com/yii-framework/)
- [Release Cycle | Yii PHP Framework](https://www.yiiframework.com/release-cycle)
- [Upgrading from Version 2.0 | Yii3 Documentation](https://yiisoft.github.io/docs/guide/intro/upgrade-from-v2.html)
- [PHP Framework Popularity: 2025–2026 Breakdown](https://devabit.com/blog/php-framework-popularity/)
- [Best PHP Frameworks for 2026 — PeerSpot](https://www.peerspot.com/categories/php-frameworks)
