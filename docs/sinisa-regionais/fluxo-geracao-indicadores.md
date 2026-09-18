# SINISA Regionais — Mapeamento do Fluxo de Geração de Indicadores

> Conversão do documento `mapeamento-fluxo-geracao-indicadores.pdf`, que não está versionado neste repositório. Documento gerado em 22/07/2026.

Arquitetura e fluxo de geração de indicadores do SINISA Regionais: fluxo principal, consulta temporária, exportador antigo e responsabilidade de cada arquivo.

**Projeto:** `C:\xampp\htdocs\sinisa-regionais`

## 1. Visão geral

O projeto contém três implementações relacionadas a indicadores. O fluxo atual é a geração persistida em lotes. Há também uma consulta temporária por prestador, que reutiliza o mesmo motor, e um exportador antigo baseado em grandes consultas SQL e em outra tradução de fórmulas.

```text
Tela “Gerar Indicadores”
        ↓
Busca prestadores e municípios elegíveis
        ↓
JavaScript divide o trabalho em lotes de 50
        ↓
Controller entrega o lote ao IndicadoreService
        ↓
Recria a tabela dinâmica → Carrega regras e calcula fórmulas → Grava prestador/município
        ↓
Último lote: agrega Estado, Região e Brasil
        ↓
Relatório HTML ou exportação XLSX
```

## 2. Fluxo principal — geração persistida

### 2.1 Entrada pela interface

`views/gerar-indicadores/index.php`

Monta a tela administrativa. Exibe o botão de geração, tabela de erros, exportação XLSX e cancelamento. Também registra o JavaScript e informa ao frontend qual é o módulo em uso.

`web/js/app/gerar-indicadores.js`

Coordena todo o processamento no navegador:

- busca os prestadores pela rota `/<modulo>/gerar-indicadores/prestadores`;
- divide o retorno em lotes de 50;
- envia os lotes sequencialmente para a ação de cálculo;
- atualiza a barra de progresso e mostra os erros;
- repete uma requisição até três vezes quando há falha HTTP;
- habilita a exportação quando termina.

Não existe fila ou *worker* de background: o navegador mantém e controla a sequência das requisições.

### 2.2 Controller e rotas

`controllers/GerarIndicadoresController.php`

É a porta de entrada do backend:

- `actionIndex()`: exibe a página;
- `actionPrestadores()`: obtém os participantes;
- `actionCalcular()`: entrega cada lote ao serviço;
- `actionCancelarExecucao()`: remove a *flag* de execução do cache.

| Arquivo | Responsabilidade |
| --- | --- |
| `modules/agua/controllers/GerarIndicadoresController.php` | Adaptador vazio que disponibiliza o controller comum sob as rotas do módulo Água. |
| `modules/esgoto/controllers/GerarIndicadoresController.php` | Adaptador equivalente para as rotas do módulo Esgoto. |

### 2.3 Seleção dos prestadores

`models/TabPrestadoresSearch.php`

O método `getPrestadores()` relaciona prestador, vínculo com o módulo, participação no ano, municípios atendidos, situação de preenchimento, estado e região. Na geração principal são solicitadas as situações 58, 59, 60, 62 e 63. Registros sem atendimento/delegação são excluídos.

### 2.4 Orquestração e gravação

`services/IndicadoreService.php`

É o núcleo do fluxo atual. No primeiro lote, remove e recria `<modulo>.tab_indicadores_calculados`. As colunas de indicadores são criadas dinamicamente a partir das siglas ativas para o módulo e o ano. Para cada prestador/município, percorre os indicadores, executa o motor de fórmula, normaliza o resultado, grava uma linha e guarda os componentes usados na agregação. No último lote, produz os resultados geográficos.

| Tipo | Significado na tabela calculada |
| --- | --- |
| `M` | Resultado de prestador/município. |
| `E` | Agregado por estado. |
| `R` | Agregado por região. |
| `B` | Agregado nacional — Brasil. |

Para o módulo Esgoto, o serviço também pode incorporar os indicadores de balanço `IFB3001` a `IFB3009`.

## 3. Motor de cálculo

`sinisa/HasCalcIndicador.php`

*Trait* compartilhada por `IndicadoreService` e `RelatorioController`. Ela carrega glossários, formulários, unidades e indicadores em cache; identifica campos usados nas condições e fórmulas; consulta seus valores; e executa a regra cadastrada.

São aceitas referências locais, como `#GTA0001`, e referências qualificadas de outro módulo, como `#^GM:OGM5008`. Fórmulas comuns são normalizadas e avaliadas como PHP. Indicadores marcados como validação SQL executam a consulta cadastrada em `dsc_equacao`.

**Semântica da condição:**

- condição `TRUE` → não calcula o indicador;
- condição `FALSE` → executa a fórmula.

`services/IndicadorCampoResolver.php`

Resolve campos fora do módulo atual. Consulta o glossário para descobrir módulo, formulário e tabela de origem; localiza a participação correspondente ao município/ano; consulta o valor e informa situações como origem inválida, dado ausente ou participação ambígua. Formulários filhos externos ainda não são suportados.

## 4. Cadastro das regras

`models/TabGlossariosIndicadores.php`

Representa a tabela `dicionario.tab_glossarios_indicadores`. As fórmulas do fluxo atual ficam no banco, não em arquivos PHP. Nessa tabela estão sigla, ano, módulo, equação, condição de bloqueio, tipo de validação, unidade, família e situação ativo/inativo.

`models/TabGlossariosIndicadoresSearch.php`

Acrescenta filtros, cenários e validações utilizados pela administração e pelas buscas do motor.

`controllers/GlossariosIndicadoresController.php`

Mantém o cadastro das definições dos indicadores e das imagens usadas para representar as equações.

## 5. Consulta temporária por prestador

`views/glossarios-indicadores/resultado.php`

Exibe o botão para calcular somente o prestador selecionado. Depois apresenta uma grade com valor, campos da condição, campos da fórmula, imagem da equação e condição interpretada.

`controllers/RelatorioController.php`

`actionGerarIndicadoresPrestadorSessao()` usa o mesmo motor, mas não grava na tabela calculada: mantém o resultado em cache por 30 minutos. Já `actionRelatorioIndicadoresGerados()` lê a tabela persistida e prepara o relatório geral.

`views/gerar-indicadores/relatorio-indicadores-gerados.php`

Renderiza os registros municipais e agregados retornados pelo relatório persistido.

## 6. Gerador/exportador antigo

`controllers/IndicadoresController.php`

Implementação independente do fluxo atual. Recebe tipo, formato e opção de incluir informações; escolhe um arquivo SQL; carrega todas as entradas; lê as fórmulas antigas `txt_formula_indicador` e `txt_formula_bloqueio`; traduz essas fórmulas para PHP; calcula; e devolve JSON, CSV ou planilha.

| Arquivo SQL | Finalidade |
| --- | --- |
| `INDICADORES-ANALITICOS.sql` | Produz dados analíticos por prestador/município. |
| `INDICADORES-MUNICIPAIS.sql` | Produz a consolidação municipal. |
| `INDICADORES-CONSOLIDADOS.sql` | Produz consolidações por UF, macrorregião e Brasil. |
| `INDICADORES-MUNICIPIO-ATENDIDO-POR-REGIONAL.sql` | Relaciona municípios atendidos por prestadores regionais. |
| `INDICADORES-BALANCO-PATRIMONIAL.sql` | Extrai informações financeiras dos prestadores. |

Os SQLs específicos de Água e Esgoto ficam sob `consultas/indicadores/agua` e `consultas/indicadores/esgoto`.

| Arquivo | Responsabilidade |
| --- | --- |
| `modules/agua/controllers/IndicadoresController.php` | Disponibiliza o exportador antigo nas rotas de Água. |
| `modules/esgoto/controllers/IndicadoresController.php` | Disponibiliza o exportador antigo nas rotas de Esgoto. |
| `models/CSV.php` | Materializa a saída em CSV. |
| `models/Planilha.php` | Materializa a saída em planilha. |
| `consultas/InformacoesIndicadoresSql.php` | Contém uma consulta PHP extensa importada pelo controller, mas não chamada pelo caminho atual de `actionIndex()`. |

## 7. Código legado separado

| Arquivo | Responsabilidade |
| --- | --- |
| `sinisa/service/Indicadores.php` | Abstração antiga que calcula e salva em `tab_indicadores`. |
| `sinisa/service/IndicadoresAP.php` | Fórmulas *hardcoded* de Águas Pluviais, como `in001`, `in002` e outras. |
| `modules/*/models/TabIndicadores.php` | Models das tabelas antigas. Não participam do fluxo principal, que usa `tab_indicadores_calculados`. |

## 8. Scripts de correção

Os arquivos `scripts/corrigir_condicoes_indicadores_*.sql` corrigem condições de Água e Esgoto para 2024 e 2025. Eles validam quantidades, criam *backup*, substituem condições incompatíveis e ajustam fórmulas específicas de forma transacional e idempotente.

Os correspondentes `scripts/rollback_condicoes_indicadores_*.sql` restauram condições e equações a partir das tabelas de *backup*.

## 9. Pontos de atenção

1. **Cancelamento parcial:** o botão remove apenas a *flag* do cache; não interrompe requisições já enviadas ou em execução.
2. **Tabela global recriada:** o primeiro lote derruba e recria a tabela calculada do módulo. Execuções concorrentes podem destruir resultados.
3. **Lotes ignorados:** após três falhas HTTP, o JavaScript pula o lote e continua; ainda assim a interface pode indicar conclusão.
4. **Execução dinâmica:** fórmulas PHP são avaliadas com `eval()` e fórmulas SQL são executadas diretamente. O cadastro deve ter acesso fortemente restrito.
5. **Credenciais em SQL:** os arquivos do exportador antigo contêm credenciais de banco em texto claro.
6. **Duas engines:** a geração persistida e o exportador antigo interpretam fórmulas de maneiras diferentes, podendo produzir divergências.
7. **Variável de ano:** a *view* recebe `$ano_ref`, mas registra `$anoRef` no frontend; o valor pode cair no ano corrente. O backend usa o parâmetro global correto.
8. **Scripts fora do Git:** no momento do levantamento, os oito scripts de correção e *rollback* estavam não rastreados.

## 10. Resumo arquitetural

| Caminho | Motor | Destino | Uso |
| --- | --- | --- | --- |
| Geração principal | `HasCalcIndicador` | `<modulo>.tab_indicadores_calculados` | Processamento geral e agregados. |
| Consulta do prestador | `HasCalcIndicador` | Cache por 30 minutos | Inspeção detalhada de um prestador. |
| Exportador antigo | `IndicadoresController` | JSON, CSV ou planilha | Extrações por tipo com SQL dedicado. |
| Águas Pluviais legado | `IndicadoresAP` | `tab_indicadores` | Fórmulas *hardcoded* antigas. |

---

Documento produzido a partir da leitura estática do código-fonte. Nenhuma regra ou arquivo da aplicação foi alterado durante o levantamento.
