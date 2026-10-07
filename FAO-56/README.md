# Preparação da base meteorológica e cálculo da ETo

O [penman–monteith.ipynb](penman–monteith.ipynb) prepara os dados da estação automática INMET A601 — Seropédica/Ecologia Agrícola — e calcula a evapotranspiração de referência diária pelo método FAO-56 Penman–Monteith. O resultado fornece o alvo `eto_pmt` para um experimento posterior com MLP.

Os arquivos-fonte e suas pendências de proveniência estão descritos também no [README de dados](../dados/README.md). Esta pasta concentra a documentação da preparação, das variáveis e do alvo utilizado pela MLP.

## Arquivos e caminhos

| Arquivo | Função |
| --- | --- |
| [penman–monteith.ipynb](penman–monteith.ipynb) | Leitura, diagnóstico, preparação, cálculo e exportação; 25 blocos explicados em Markdown. |
| [INMET_processado.csv](../dados/INMET_processado.csv) | Dados meteorológicos diários, incluindo temperaturas, ponto de orvalho, umidade e vento. |
| [INMET_radiacao.csv](../dados/INMET_radiacao.csv) | Radiação global horária em kJ/m². |

Os dois arquivos de entrada possuem dez linhas anteriores ao cabeçalho e usam `;` como separador. A leitura usa UTF-8 com suporte a BOM. O código atual usa `ROOT = Path.cwd().parent` e `PASTA_SAIDA = Path("dados_FAO-56")`: portanto, o diretório de trabalho deste notebook deve ser **`FAO-56/`**. Assim, as entradas são localizadas em `../dados/` e as saídas em `FAO-56/dados_FAO-56/`.

## Como executar

1. Abra o projeto e selecione o kernel Python do ambiente `.venv`, com pandas e NumPy instalados. As dependências do projeto estão em [requirements.txt](../requirements.txt).
2. Abra `FAO-56/penman–monteith.ipynb` e confira o diretório de trabalho do kernel com `%pwd`. Ele deve ser `eto-mlp-fao56/FAO-56`.
3. Se o kernel estiver na raiz `eto-mlp-fao56`, execute `%cd FAO-56` antes das células do fluxo. Faça isso somente depois de conferir o diretório atual. O texto introdutório do notebook ainda menciona a raiz, mas as expressões de caminho do código atual exigem a subpasta `FAO-56`.
4. Confira a configuração inicial: período, latitude, altitude, altura do sensor, pasta de saída e checkpoints.
5. Reinicie o kernel, confirme novamente o diretório de trabalho e execute as células em sequência. Evite reaproveitar objetos de uma execução anterior ou executar cálculos fora de ordem.
6. Confira a mensagem de aprovação dos checkpoints e os dois arquivos exportados. A última célula **sobrescreve os CSVs correspondentes** em `PASTA_SAIDA`; configure outra pasta se precisar preservar uma versão anterior.


## Configuração da base atual

| Parâmetro | Valor |
| --- | --- |
| Nome da estação | SEROPEDICA-ECOLOGIA AGRICOLA |
| Código | A601 |
| Situação no metadado fornecido | Operante |
| Período | 01/01/2001 a 31/08/2026 |
| Latitude no metadado | -22.75777777° |
| Latitude usada no código | -22.757778° |
| Longitude no metadado | -43.68472221° |
| Altitude | 35 m |
| Periodicidade do produto meteorológico | Diária; arquivo de radiação separado em frequência horária |
| Altura do anemômetro | 10 m|
| Escala do cálculo | Diária |
| Fluxo de calor no solo | G = 0 |
| Política de ausências | Sem imputação meteorológica |

`VALIDAR_CHECKPOINTS_REFERENCIA = True` verifica a versão de referência da base. Ao trocar o período ou os arquivos, revise calendário, metadados e valores esperados; não atualize os checkpoints apenas para ocultar uma divergência inesperada.

## Etapas e decisões metodológicas

1. **Tabela diária:** padroniza nomes, remove somente colunas auxiliares comprovadamente vazias, valida números e calendário, ordena datas e diagnostica lacunas. Os dias incompletos permanecem na base de processamento.
2. **Termodinâmica:** calcula a média FAO `(Tmax + Tmin)/2`, as pressões de saturação, a pressão de vapor pelo ponto de orvalho e seu déficit. A média INMET permanece separada. A pressão estimada pela altitude determina gamma; a pressão observada é auxiliar.
3. **Radiação horária:** valida 24 timestamps distintos por dia, preserva a radiação bruta e cria uma coluna QC com negativos limitados a zero. NaN permanece NaN. Somente dias com 24 valores válidos recebem Rs.
4. **Integração e geometria solar:** soma radiação pela data original do INMET, sem deslocamento de fuso; integra por merge um-para-um e calcula Ra, Rso e Rns. A correspondência das datas não resolve, por si só, a definição dos intervalos do produto diário.
5. **Saldo de radiação:** calcula Rnl e Rn pela formulação principal, limitando Rs/Rso apenas acima a 1. A versão experimental com limite inferior 0,30 usa colunas `clip030` separadas e não alimenta ETo.
6. **Vento e alvo:** corrige a altura do vento para 2 m e calcula ETo diária, preservando a propagação das ausências.
7. **MLP:** seleciona data, seis features e alvo; somente nessa etapa remove linhas incompletas. Todos os cenários devem utilizar a mesma população resultante.

### Conversões usadas

| Operação | Implementação |
| --- | --- |
| kJ/m² por intervalo horário → MJ/m²/dia | Soma diária das 24 observações QC e divisão por 1.000. |
| Latitude em graus → radianos | `np.deg2rad(LATITUDE)`. |
| °C → Kelvin para Rnl | Soma de 273,16, conforme a convenção numérica da FAO-56, antes da quarta potência. |
| Vento a 10 m → equivalente a 2 m | Multiplicação por `4.87 / ln(67.8 * 10 - 5.42)`, aproximadamente 0,747951; permanece em m/s. |

A soma diária define o período da radiação; não se divide por 24 nem se multiplica por 3.600. A conversão exata moderna de Celsius para Kelvin usa 273,15; o notebook preserva a constante apresentada na referência metodológica.

## Produtos exportados

| Produto em `FAO-56/dados_FAO-56/` | Conteúdo |
| --- | --- |
| `base_processamento_fao56_A601.csv` | Base diária completa: observações, derivados, flags, sensibilidade e ETo; inclui dias incompletos. |
| `dataset_A601_mlp.csv` | Apenas linhas completas de data, seis features e alvo. |

Ambos são exportados com separador **vírgula**, decimal ponto e sem índice. A série bruta horária não é incorporada linha a linha à base diária; conserve os arquivos-fonte para reprodução.

| Coluna no dataset MLP | Papel | Unidade |
| --- | --- | --- |
| `data` | Identificador temporal; não é uma das seis features | Data |
| `rn_mj_m2_dia` | Saldo de radiação | MJ/m²/dia |
| `u2_ms` | Vento a 2 m | m/s |
| `ur_media_pct` | Umidade relativa média | % |
| `tmax_c` | Temperatura máxima | °C |
| `tmin_c` | Temperatura mínima | °C |
| `tmedia_inmet_c` | Temperatura média fornecida pelo INMET | °C |
| `eto_pmt` | Alvo calculado | mm/dia |

O [mlp_FAO-56.ipynb](../mlp_FAO-56.ipynb) já lê `FAO-56/dados_FAO-56/dataset_A601_mlp.csv` a partir da raiz do projeto. Renomeia as colunas em memória e preserva a data fora das entradas, incluindo-a no CSV de previsões. O dataset tem 8.329 dias completos, de **2001-01-18 a 2026-08-30**, com lacunas; o período do calendário completo continua sendo 2001-01-01 a 2026-08-31.

A MLP utiliza normalização global das sete colunas numéricas, divisão aleatória fixa 70%/15%/15%, **63 cenários e uma única seed (42)**, limite de 10.000 épocas e paciência 400. O modelo final reutiliza os pesos do cenário escolhido pelo menor RMSE de validação, sem retreinamento. O [README da MLP](../README.md) descreve o protocolo, a matriz de Pearson exigida e as exportações: seis CSVs, modelo `.pt`, configuração JSON e um único Excel opcional. Os checkpoints abaixo se referem à preparação dos dados.

## Checkpoints e limites da validação

| Verificação | Referência atual |
| --- | ---: |
| Dias na base completa | 9.374 |
| Registros horários | 224.976 |
| Dias com Rs válido | 8.538 |
| Dias com Rn válido | 8.374 |
| Dias com ETo válida | 8.331 |
| Linhas completas da MLP | 8.329 |
| Rnl negativo na formulação principal | 820 |
| ETo mínima / máxima | 0,641494 / 8,342550 mm/dia |

As 43 perdas entre Rn e ETo decorrem de vento ausente. Duas perdas adicionais na MLP decorrem da média INMET ausente. Esses números verificam a reprodução da versão atual; não validam independentemente as medições.

A base contém **31 registros da MLP com média INMET inferior à mínima**, ainda mantidos e sujeitos a investigação. A cobertura varia entre anos e há sensibilidade de ETo à formulação da radiação em dias nublados. Não houve exclusão automática desses casos nem imputação. A base permanece destinada a desenvolvimento e análise, com essas limitações documentadas.

Rn é derivado de radiação, temperaturas e ponto de orvalho: remover uma temperatura da MLP mantendo Rn não elimina toda a informação associada a ela. A interpretação da ablação deve considerar essa dependência. ETo é um alvo físico calculado, não uma medição independente de evapotranspiração em campo.

## Referência

Allen, R. G.; Pereira, L. S.; Raes, D.; Smith, M. (1998). *Crop evapotranspiration: Guidelines for computing crop water requirements*. FAO Irrigation and Drainage Paper 56. [Capítulo 2: equação FAO Penman–Monteith](https://www.fao.org/4/x0490e/x0490e06.htm); [capítulo 3: dados meteorológicos](https://www.fao.org/4/x0490e/x0490e07.htm).
