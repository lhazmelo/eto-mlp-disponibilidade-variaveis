# Dados meteorológicos — estação INMET A601

A base atual é da estação **SEROPEDICA-ECOLOGIA AGRICOLA (A601)**. Os metadados foram fornecidos pelo responsável pelo estudo e conferidos no cabeçalho de `INMET_processado.csv`. Os arquivos-fonte são preservados; a preparação ocorre em memória e gera produtos separados.

## Estação e cobertura

| Campo | Valor |
| --- | --- |
| Nome | SEROPEDICA-ECOLOGIA AGRICOLA |
| Código da estação | A601 |
| Latitude | -22.75777777° |
| Longitude | -43.68472221° |
| Altitude | 35 m |
| Situação registrada | Operante |
| Data inicial do período fornecido | 2001-01-01 |
| Data final do período fornecido | 2026-08-31 |
| Periodicidade do produto meteorológico | Diária |

A situação descreve o metadado fornecido, não uma consulta operacional em tempo real. A radiação global possui um arquivo horário separado. O processamento mantém as datas originais sem deslocamento de fuso. O sistema geodésico e a definição exata da janela diária permanecem a documentar na fonte.

## Arquivos

| Arquivo | Conteúdo |
| --- | --- |
| `INMET_processado.csv` | Produto diário: temperaturas, ponto de orvalho, umidade, vento e variáveis auxiliares. |
| `INMET_radiacao.csv` | Radiação global horária, em kJ/m² por intervalo. |
| `Dataset3.csv` | Base anterior de desenvolvimento; não alimenta o fluxo atual. |
| `base_processamento_fao56.csv` e `dataset_mlp.csv` | Arquivos locais que não são lidos pelos dois notebooks do fluxo atual; a MLP usa o produto A601 em `FAO-56/dados_FAO-56/`. |
| `../FAO-56/dados_FAO-56/base_processamento_fao56_A601.csv` | 9.374 dias, incluindo ausências, derivados e flags. |
| `../FAO-56/dados_FAO-56/dataset_A601_mlp.csv` | 8.329 dias completos, data, seis entradas e alvo. |

Os arquivos INMET são lidos com `sep=';'`, `skiprows=10` e `encoding='utf-8-sig'`. Os produtos derivados usam vírgula, decimal ponto e não incluem índice. O [notebook FAO-56](../FAO-56/penman–monteith.ipynb) gera os derivados; o [notebook MLP](../mlp_FAO-56.ipynb) lê a base preparada.

## População do experimento

| Etapa | Registros |
| --- | ---: |
| Calendário completo, 2001-01-01 a 2026-08-31 | 9.374 |
| Grade horária da radiação, incluindo ausências | 224.976 |
| Dias com radiação global diária Rs válida | 8.538 |
| Dias com saldo de radiação Rn válido | 8.374 |
| Dias com ETo calculada | 8.331 |
| Dias completos no dataset MLP | 8.329 |

O dataset MLP vai de **2001-01-18 a 2026-08-30**, com lacunas. As 43 perdas entre Rn e ETo correspondem a vento ausente; as duas perdas adicionais na seleção MLP correspondem à temperatura média INMET ausente. Os 8.329 registros não possuem valores ausentes ou infinitos nem datas duplicadas na versão examinada.

Todos os cenários usam esses mesmos dias. A divisão aleatória da MLP, com seed 42, contém **5.830 registros de treino, 1.249 de validação e 1.250 de teste**. Datas são preservadas para rastreabilidade, mas não são entradas da rede.

## Dicionário do dataset MLP

| Coluna no CSV | Nome em memória na MLP | Significado | Unidade |
| --- | --- | --- | --- |
| `data` | `datas_observacoes`, separada do dataframe numérico | Dia da observação | AAAA-MM-DD |
| `rn_mj_m2_dia` | `Radiacao` | Saldo de radiação calculado: Rns − Rnl | MJ/m²/dia |
| `u2_ms` | `Vel_Vento` | Vento equivalente a 2 m, convertido do vento a 10 m | m/s |
| `ur_media_pct` | `umidade` | Umidade relativa média diária INMET | % |
| `tmax_c` | `Temp_Max` | Temperatura máxima diária | °C |
| `tmin_c` | `Temp_Min` | Temperatura mínima diária | °C |
| `tmedia_inmet_c` | `Temp_Med` | Temperatura média diária fornecida pelo INMET | °C |
| `eto_pmt` | `Eto_PMT` | ETo diária calculada por FAO-56 Penman–Monteith | mm/dia |

A renomeação ocorre apenas em memória. A média INMET é preservada como entrada e não é substituída por `(Tmax + Tmin)/2`. Essa segunda média, `tmean_fao_c`, é usada no cálculo físico do alvo.

## Preparação e construção do alvo

O alvo atual é calculado em **Python**, no notebook FAO-56 do projeto. As informações anteriores sobre cálculo em Excel e localização não confirmada pertenciam à base de desenvolvimento e não descrevem a A601.

O fluxo valida calendários e números, preserva ausências e não imputa dados meteorológicos. Na radiação, mantém a coluna bruta e limita negativos a zero em uma coluna QC. A soma diária exige 24 valores presentes e é dividida por 1.000 para converter kJ/m² em MJ/m²/dia.

A pressão de vapor real é obtida da temperatura do ponto de orvalho. A pressão atmosférica usada na constante psicrométrica é estimada pela altitude de 35 m. A geometria solar usa o dia do ano e latitude -22,757778° no código, arredondada em relação ao metadado. O vento medido a 10 m é convertido para 2 m. O cálculo diário utiliza fluxo de calor no solo G = 0.

Rn é derivado de radiação global, temperaturas, ponto de orvalho e geometria solar. A razão Rs/Rso principal é limitada apenas acima a 1; a alternativa com limite inferior 0,30 é calculada separadamente e não alimenta ETo. Detalhes, constantes e checkpoints estão no [README FAO-56](../FAO-56/README.md).

Retirar uma temperatura da entrada da MLP mantendo Rn não remove toda a informação dessa temperatura do processamento anterior. Os cenários representam conjuntos de entradas da rede, não necessariamente conjuntos independentes de sensores disponíveis.

## Normalização e controles na MLP

O notebook separa a data, seleciona as sete colunas numéricas, remove linhas com ausências e verifica tipos numéricos e valores finitos. Na versão atual, nenhuma linha adicional é removida nessa etapa.

Conforme a metodologia definida com o orientador, média e desvio-padrão amostral (`ddof=1`) são calculados por coluna sobre **todos os 8.329 registros**, incluindo o alvo. Essas estatísticas normalizam treino, validação e teste e revertem as previsões para mm/dia. Desvios nulos ou não finitos interrompem a execução. O teste participa desse pré-processamento global, embora não participe das atualizações dos pesos ou da seleção automática.

O SHA-256 do CSV é registrado em `configuracao_execucao.json`, exportado com o modelo e as tabelas após o treinamento. O protocolo dos 63 cenários com seed 42, as instruções de execução e as seis tabelas CSV estão no [README da MLP](../README.md).

## Qualidade e pendências

- **31 dias da MLP têm média INMET inferior à temperatura mínima**, ainda mantidos e sujeitos a investigação.
- A formulação principal produz **820 dias com Rnl negativo** na base completa. Esses valores são preservados; os checkpoints não constituem validação física independente.
- A disponibilidade varia entre anos. Não há imputação nem exclusão adicional por cenário.
- A altura de 10 m do anemômetro foi confirmada pelo responsável pelo estudo; o vento da MLP já está convertido para 2 m.
- Documentar URL ou identificador do produto INMET, data de obtenção, responsável pela preparação e referência de citação.
- Confirmar janela temporal e fuso do produto diário, sistema de referência das coordenadas e procedimentos originais de agregação/controle de qualidade do provedor.
- Registrar licença, atribuição e condições de redistribuição dos arquivos originais e derivados. A autorização relatada para a antiga base de exemplo não define os termos desta base.

O objetivo é aproximar ETo calculada e comparar disponibilidade de entradas na A601. Não há medição independente de evapotranspiração nem demonstração de transferência para outras estações.
