# Case Operations - Documentação

Este projeto documenta o processo de auditoria, limpeza e análise de uma base de oportunidades (pipeline de vendas).

## 1. Documentação da Base Inicial

O diagnóstico inicial revelou as seguintes características da base antes de qualquer tratamento:

- **Arquivo Original**: `opps_corrupted - opportunities.csv`
- **Total de linhas**: *413*
- **Total de colunas**: *18*
- **Oportunidades Únicas**: *261*
- **Contas Únicas**: *130*

### Descobertas e Inconsistências (Auditoria de Qualidade)
*Registre aqui as descobertas feitas durante o diagnóstico, como tipos de dados, valores nulos, discrepâncias financeiras, etc.*
- **Registros fora do escopo**: Identifiquei 108 registros com o campo `Type` inválido para a análise (diferentes de Project - Competitive, Project - Not Competitive, ou Change Order/Upsell).
- **Valores Nulos (NaN) Esperados**: Encontrei 21 oportunidades com os campos `Amount` e `Product_Name` vazios (NaN). Verifiquei que todas estão em estágios iniciais (`Opportunity Identified` ou `Qualified`), o que é perfeitamente validado pela regra de negócio. Como curiosidade analítica, 3 dessas oportunidades já tinham o prazo (`Delivery_Length_Months`) preenchido pelo vendedor, mesmo sem o preço definido.
- **Oportunidades Vencidas (Alerta Operacional)**: Os testes cronológicos confirmaram que a base não possui datas invertidas (nenhuma data de fechamento é anterior à de criação). Contudo, descobri 4 oportunidades ativas (em aberto) cuja data de previsão de fechamento (`Close_Date`) já expirou no passado. Isso indica desatualização do CRM pelos vendedores.

## 2. Registro de Alterações (Data Cleaning)

As seguintes ações foram tomadas para limpar e padronizar a base de dados:

- **Escopo e Filtros**:
  - Remoção de 108 linhas contendo tipos de oportunidade fora do escopo definido pelas regras de negócio. A base foi reduzida de 413 para 305 linhas.
- **Padronização Textual**:
  - Correção de erros de digitação e variações na coluna `Stage` (ex: "Clossed Won", "Pitchinng", "Qualifyed" padronizados para seus valores canônicos).
  - Correção de erros de digitação na coluna `Lead_Office` (ex: "São Paulo", "SP", "sao paulo" padronizados para "Sao Paulo, BR").
  - Mapeamento e normalização da coluna `Lead_Source`, consolidando as 24 variações e erros de digitação encontrados na base original nas 6 categorias principais (Inbound, Outbound, Referral, Customer Success, Event, Unknown) definidas pela taxonomia.
- **Conversões**:
  - Conversão dos campos de data (`Close_Date` e `Created_Date`) para formato `datetime`.
  - Limpeza e formatação da coluna `Amount` (remoção de "R$" e espaços, substituição de pontos de milhar e vírgulas decimais) e conversão para o tipo numérico (`float`).
- **Correções Financeiras**:
  - Identificação de divergência financeira em 22 oportunidades, nas quais o `Amount` (valor geral) diferia da soma individual dos produtos (`Total_Product_Amount`). A coluna `Amount` de toda a base foi corrigida para refletir o valor real da soma dos produtos, zerando as divergências.
- **Criação de Novas Colunas**:
  - Renomeação/Criação da coluna `Lead_Source_Category` para abrigar a taxonomia limpa (as 6 categorias mestre), conforme exigido pela regra do case.


## 3. Base Analítica Consolidada

Características da base final exportada para o negócio:

- **Arquivo Final**: `outputs/opps_corrigido.csv`
- **Total de Linhas Final**: *305*
- **Garantia de Qualidade**: 
  - 100% dos registros foram validados contra as regras de escopo, formatação textual (Listas Canônicas) e auditoria financeira (soma dos produtos). Base livre de dados corrompidos.

## 4. Construção e Auditoria do Dashboard Analítico

Após o tratamento da base, desenvolvi um painel em HTML estático (`outputs/analise.html`) utilizando a biblioteca Plotly. Abaixo estão as principais decisões de design, engenharia de dados e refinamento de negócios que apliquei:

### Decisões Técnicas e de Design
- **Agregação e Deduplicação Inteligente:** Para evitar a inflação irreal dos valores financeiros (`Amount`) ao listar produtos, os dados foram agrupados por `Opportunity_ID`. A coluna de produtos foi reestruturada para exibir todos os itens atrelados a uma oportunidade em uma única célula usando quebras de linha em *bullet points* (`<br>• `), otimizando a leitura humana na tabela de *Top 10 Focus List*.
- **Ajuste de Visualização:** O gráfico de "Pipeline em aberto por Stage" migrou do formato de Funil para um Gráfico de Barras Horizontais, entregando uma aderência muito superior ao briefing e facilitando a comparação de proporções financeiras entre as etapas.
- **Transparência de Dados:** Foi inserida uma **Nota Metodológica** no cabeçalho do dashboard explicando a engenharia de dados por trás dos números, garantindo máxima transparência para a diretoria.

### Auditoria Criteriosa das Observações de Negócio
Realizei uma análise minuciosa nas inferências automáticas, resultando em correções essenciais para a leitura do cenário:
- **Gráfico 1 (Receita MoM):** Corrigida a premissa de "estabilidade". Os dados comprovaram que houve uma queda brusca de mais de 60% na receita nos meses de março e abril em relação ao primeiro bimestre, configurando um alerta vermelho para previsibilidade.
- **Gráfico 2 (Mix por Lead Source):** Ajustada a narrativa para refletir a dependência massiva em **Customer Success** (responsável por R$ 17,4 milhões) e Referrals, corrigindo o peso equivocado antes atribuído ao Inbound.
- **Gráfico 3 (Top 10 Open Pipeline - Focus List):** Implementada a métrica avançada de `Score_Prioridade` (`Amount` dividido por `Dias_Ate_Fechamento`). Isso permitiu elencar verdadeiramente as oportunidades que unem **alto valor financeiro e alta urgência de tempo**, cumprindo com precisão o requisito do briefing em vez de apenas ordenar por tamanho.
- **Gráfico 5 (Ticket Médio por Tipo):** A leitura foi invertida após análise numérica. Os dados mostraram categoricamente que os projetos **Competitivos** possuem um ticket médio bem maior (~R$ 937 mil) do que os Não-Competitivos (~R$ 347 mil).
- **Gráfico 6 (Pipeline por Stage):** Corrigida a localização do gargalo. Mais de 65% do valor financeiro do pipeline já se concentra nas fases finais (*Pitched* e *Negotiation*), exigindo foco imediato em táticas de "fechamento" (closing).
- **Gráfico 8 (Ciclo Médio por Escritório):** A observação genérica deu espaço para apontar ativamente um gargalo operacional: a filial de **São Paulo** lidera a ineficiência com 90 dias de ciclo de fechamento médio, cerca de 30% mais lenta que filiais do interior (Votorantim e São Carlos).
- **Gráfico 10 (Idade do Pipeline):** Confirmei via estatística a existência de uma robusta *long right tail* (cauda estendida à direita). Embora a mediana seja de 34 dias, 14% do funil está envelhecendo há mais de 100 dias (com picos estagnados de até 330 dias), validando a recomendação de rotina de limpeza de pipeline.
