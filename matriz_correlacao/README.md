# Correlação de Pearson — INMET A601

O [matriz.ipynb](matriz.ipynb) calcula a matriz de correlação de Pearson entre as seis entradas da MLP e a ETo calculada por FAO-56 Penman–Monteith. A análise é descritiva e usa a mesma seleção de casos completos da MLP.

As informações sobre a estação, o processamento, as unidades e as limitações dos dados estão na pasta [FAO-56](../FAO-56/README.md). O protocolo da rede está no [README da MLP](../README.md).

## Entrada e variáveis

O notebook lê `FAO-56/dados_FAO-56/dataset_A601_mlp.csv`, usando a data para validação e as seguintes colunas para o cálculo:

| Coluna | Rótulo na figura | Unidade |
| --- | --- | --- |
| `rn_mj_m2_dia` | Rn | MJ/m²/dia |
| `u2_ms` | u2 | m/s |
| `ur_media_pct` | UR | % |
| `tmax_c` | Tmax | °C |
| `tmin_c` | Tmin | °C |
| `tmedia_inmet_c` | Tmed | °C |
| `eto_pmt` | ETo | mm/dia |

Na base atual, são 8.329 observações completas, de 2001-01-18 a 2026-08-30, com lacunas. Esses números são calculados novamente a cada execução, não impostos como condição para usar outra versão dos dados.

## Execução

1. Use o ambiente Python do projeto, com as dependências de [requirements.txt](../requirements.txt). Este notebook utiliza pandas, NumPy e Matplotlib, além da biblioteca padrão.
2. Abra `matriz.ipynb` e selecione o kernel do ambiente. O diretório de trabalho pode ser a raiz do projeto ou `matriz_correlacao/`; os caminhos são resolvidos nos dois casos.
3. Confirme que o dataset preparado existe. Se necessário, gere-o seguindo o [README FAO-56](../FAO-56/README.md).
4. Reinicie o kernel e execute todas as células em ordem.
5. Confira o resumo da população e a mensagem final de exportação. A matriz gerada é a entrada de correlação do notebook MLP.

`EXPORTAR_TABELA_ETO = False` é o padrão. Altere para `True` somente se quiser um CSV adicional contendo a coluna de correlações com ETo, ordenada por valor absoluto.

## Validações e método

O fluxo rejeita cabeçalhos duplicados, colunas obrigatórias ausentes, datas ausentes ou repetidas, valores não numéricos, infinitos, variáveis constantes e menos de três observações completas. Valores inválidos não são convertidos silenciosamente em ausências.

Linhas com valores ausentes em qualquer uma das sete variáveis são removidas em conjunto, sem imputação. Todos os pares usam exatamente as mesmas linhas. Quantidades de entrada, remoções, ausências por coluna e período utilizado são registradas. O dataset de origem não é modificado.

O cálculo usa `DataFrame.corr(method='pearson')` nas unidades originais. São verificadas finitude, faixa de −1 a 1, simetria e diagonal unitária. O CSV é relido e comparado com a matriz em memória; o arredondamento ocorre apenas na apresentação. A figura usa escala divergente fixa de −1 a 1 e informa o tamanho da população.

## Resultados

As saídas são salvas exclusivamente em **`matriz_correlacao/resultados/`**, criada automaticamente:

| Arquivo | Conteúdo |
| --- | --- |
| `matriz_correlacao_pearson.csv` | Matriz 7 × 7, com nomes originais das variáveis e precisão numérica preservada; lida pela MLP. |
| `matriz_correlacao_pearson.png` | Mapa de calor a 300 dpi, com coeficientes anotados. |
| `configuracao_correlacao.json` | Hash SHA-256 do dataset, população, colunas, política de ausências, versões, opções e horário UTC de conclusão. |
| `correlacao_com_ETo.csv` | Opcional: seis correlações com ETo e seus valores absolutos. |

Os nomes são fixos e os arquivos habilitados são sobrescritos ao executar novamente. Para preservar uma análise anterior, copie a pasta antes de executar. Desativar a tabela opcional não apaga um CSV antigo; o JSON lista quais produtos foram gerados na execução atual.

Os CSVs e a figura que já existiam diretamente em `matriz_correlacao/` são artefatos anteriores, preservados. O fluxo atualizado e a MLP usam `resultados/`.

## Interpretação científica

Pearson mede associação linear, não causalidade. O ranking das correlações com ETo não determina o desempenho de uma rede neural nem seleciona automaticamente suas entradas.

A análise usa a população completa antes da divisão da MLP, incluindo observações que posteriormente pertencem à validação e ao teste. Portanto, não deve ser apresentada como seleção de variáveis ajustada apenas no treino.

ETo é um alvo calculado e Rn compartilha componentes com sua construção. Correlações altas não representam validação independente contra evapotranspiração medida em campo. A série também pode apresentar sazonalidade e dependência temporal; este notebook não produz p-valores nem intervalos de confiança que pressupõem observações independentes.
