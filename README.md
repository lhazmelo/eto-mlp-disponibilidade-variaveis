# MLP FAO-56 — aproximação da evapotranspiração de referência

O notebook [mlp_FAO-56.ipynb](mlp_FAO-56.ipynb) compara redes neurais MLP para aproximar a ETo calculada pelo método FAO-56 Penman–Monteith na estação INMET A601. O experimento avalia todas as 63 combinações não vazias de seis entradas, com uma única seed (42).

**As informações sobre os dados estão na pasta [FAO-56](FAO-56/README.md)**: estação, período, variáveis, unidades, preparação, cálculo do alvo e limitações da base. Os arquivos-fonte ficam em `dados/`, cujo [README](dados/README.md) complementa a documentação.

## Arquivos utilizados

| Caminho | Função |
| --- | --- |
| `mlp_FAO-56.ipynb` | Preparação das entradas, treinamento, comparação dos cenários e exportações. |
| [requirements.txt](requirements.txt) | Dependências do ambiente de referência. |
| [FAO-56/README.md](FAO-56/README.md) | Documentação dos dados e do cálculo físico. |
| `FAO-56/dados_FAO-56/dataset_A601_mlp.csv` | Dataset preparado, com data, seis entradas e alvo. |
| `matriz_correlacao/resultados/matriz_correlacao_pearson.csv` | Matriz lida pela MLP e conferida contra o dataset após a limpeza. |

A matriz de Pearson é uma entrada obrigatória. Execute antes o [notebook de correlação](matriz_correlacao/matriz.ipynb), seguindo o [README da pasta](matriz_correlacao/README.md), para gerá-la em `matriz_correlacao/resultados/`. O notebook verifica rótulos, valores finitos, simetria, diagonal e correspondência com as correlações da população efetivamente utilizada. Se a base mudar, a matriz precisa acompanhar essa mudança.

## Como executar

O `requirements.txt` atual foi preparado para Python 3.12 no Windows x64 e contém uma referência direta ao pacote PyTorch com CUDA 12.8. A instalação em outro sistema exige adaptar essa dependência. O notebook usa a GPU quando `torch.cuda.is_available()` retorna verdadeiro; caso contrário, usa CPU.

Na raiz do projeto, em PowerShell, crie o ambiente se ele ainda não existir e instale as dependências:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

1. Abra `mlp_FAO-56.ipynb` em um editor compatível com notebooks e selecione o kernel de `.venv`.
2. Confira com `%pwd` se o diretório do kernel é a raiz `eto-mlp-fao56/`. Os caminhos da MLP são relativos a essa raiz.
3. Confirme a presença do dataset e da matriz de Pearson. Para regenerar o dataset, siga primeiro as instruções da [pasta FAO-56](FAO-56/README.md); aquele notebook exige o diretório de trabalho `FAO-56/`.
4. Ajuste as opções de saída: `EXECUTAR_GRAFICOS = True` gera figuras; `EXPORTAR_EXCEL = False` evita a cópia adicional em Excel.
5. Reinicie o kernel, confirme o diretório de trabalho e execute as células em ordem.
6. Ao concluir, confira os arquivos e o campo `status` de `configuracao_execucao.json`. O valor `concluida` só é gravado após a verificação das tabelas exportadas.

O treinamento não possui checkpoints nem retomada automática. Métricas, históricos e pesos ficam em memória até a etapa de exportação; uma interrupção pode exigir novo treinamento. O notebook cita uma duração anterior de 202,9 minutos como referência, sem identificação do hardware dessa medição; não é uma estimativa garantida para esta execução.

## Protocolo da MLP

As entradas candidatas são saldo de radiação (Rn), vento a 2 m (u2), umidade relativa média (UR), temperatura máxima (Tmax), mínima (Tmin) e média INMET (Tmed). A data é preservada para rastreabilidade e não entra na rede. O alvo é `eto_pmt`, calculado no processamento FAO-56.

| Configuração | Valor atual |
| --- | --- |
| Cenários | 63: 6 individuais, 15 pares, 20 trios, 15 quartetos, 6 quintetos e 1 completo. |
| Repetições | Uma por cenário, seed 42; total de 63 treinamentos. |
| Divisão | Aleatória fixa, aproximadamente 70% treino, 15% validação e 15% teste. |
| População atual | 8.329 observações: 5.830 de treino, 1.249 de validação e 1.250 de teste. |
| Arquitetura | Entradas do cenário → 32 neurônios com ReLU → uma saída linear. |
| Otimizador e perda | Adam, taxa de aprendizado 0,001 e MSE. |
| Tamanho do lote | 32. |
| Limite e parada | Até 10.000 épocas; paciência de 400 épocas sem melhoria da perda de validação. |
| Pesos utilizados | Pesos da época com menor perda de validação de cada cenário. |
| Seleção final | Menor RMSE de validação; empates seguem C01–C63. Sem retreinamento. |

Todos os cenários compartilham as mesmas observações e partições. Média e desvio-padrão amostral (`ddof=1`) são calculados sobre a base completa após a limpeza, incluindo o alvo. Portanto, validação e teste participam das estatísticas de normalização. Essa é a implementação atual e deve ser considerada na interpretação das métricas.

O teste não determina cenário, época nem hiperparâmetros. RMSE é apresentado em mm/dia, R² é adimensional e o MSE exportado é calculado na escala normalizada. Com uma única seed, não há estimativa de variabilidade entre inicializações.

## Saídas

Cada execução recebe um identificador exclusivo. As tabelas e o modelo ficam em `resultados/dataframe/<execução>/`:

| Arquivo | Conteúdo |
| --- | --- |
| `tabela_mestre.csv` | Uma linha por cenário: entradas, seed, métricas de validação e teste, melhor época e época de parada; ordenada pelo RMSE de validação. |
| `loss_epocas.csv` | Histórico de perdas de treino e validação por cenário e época. |
| `pearson_pares.csv` | Correlações dos 15 pares; a coluna `cenario` permite relacioná-las à tabela mestre. |
| `previsoes_modelo_final_teste.csv` | Datas, ETo de referência calculada, previsões e resíduos do modelo escolhido. |
| `normalizacao.csv` | Média e desvio amostral utilizados por variável. |
| `split.csv` | Índice original, data, partição e ordem dentro da partição. |
| `tempos_treinamento.csv` | tempo de treinamento de cada combinação. |
| `modelo_final_evapotranspiracao.pt` | Pesos, arquitetura, entradas, normalização e métricas do modelo final. |
| `configuracao_execucao.json` | Configuração, versões, dispositivo, hashes do dataset e notebook, duração, seleção final e estado das exportações. |


Por padrão são **seis CSVs, um modelo `.pt` e um JSON**. Com `EXPORTAR_EXCEL = True`, também é criado `resultados.xlsx`, uma cópia de consulta com as abas `Cenarios`, `Pearson_Pares` e `Previsoes_Teste`. O histórico extenso fica apenas no CSV. As tabelas exportadas são relidas e comparadas com as versões em memória.

Com os gráficos habilitados, `resultados/imagens/<execução>/` recebe PNGs e PDFs das comparações de RMSE/R² e dos diagramas de dispersão dos 63 cenários. As exportações de execuções anteriores são preservadas.

## Interpretação

A MLP aproxima um alvo calculado por uma fórmula física; não há avaliação contra evapotranspiração medida independentemente em campo. A divisão aleatória não representa uma avaliação prospectiva de anos futuros, e o estudo se restringe à estação A601.

 As limitações de qualidade e de proveniência da base estão documentadas em [FAO-56](FAO-56/README.md) e em [dados](dados/README.md).
