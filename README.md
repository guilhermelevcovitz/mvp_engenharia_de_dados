## 1. Objetivo
O objetivo deste projeto é desenvolver um pipeline de dados completo utilizando a plataforma Databricks, contemplando ingestão, transformação, modelagem, carga e análise. O dataset escolhido é proveniente do IMDb e permite a construção de um Data Warehouse em modelo dimensional (esquema estrela), adequado para consultas analíticas sobre títulos, gêneros e avaliações.

As principais perguntas de negócio definidas para este MVP são:

1. Quais gêneros apresentam maiores avaliações médias ao longo do tempo?
2. Quais títulos são os mais votados por década?
3. Como evolui o volume de lançamentos por ano?
4. Existe relação entre a duração da obra e a avaliação média?
5. Como os tipos de títulos (ex.: movie, short, tvSeries) se distribuem em termos de popularidade e avaliação?

## 2. Fonte dos Dados
Os dados foram obtidos diretamente do repositório oficial do IMDb:

https://datasets.imdbws.com/

Arquivos utilizados:
- `title.basics.tsv.gz`
- `title.ratings.tsv.gz`

## 3. Arquitetura do Pipeline de Dados

O pipeline segue o padrão arquitetural **Bronze → Silver → Gold**, utilizando Delta Lake como formato de armazenamento e possibilitando versionamento e eficiência nas consultas.

### 3.1 Camada Bronze 
Nesta camada ficam os dados brutos, conforme extraídos da fonte original.  
Processos executados:

- Leitura direta dos arquivos compactados.
- Conversão de valores faltantes (`\N`) para nulos.
- Armazenamento sem transformação, preservando o formato original.

Local:/Volumes/workspace/default/imdb_bronze/

<img width="471" height="220" alt="Capturar" src="https://github.com/user-attachments/assets/3fe49eb1-c495-409b-b3b2-2a154378e99d" />

<img width="492" height="218" alt="Capturar2" src="https://github.com/user-attachments/assets/4fab3690-30da-4935-9945-c3d91d34b1f1" />

### 3.2 Camada Silver 
A camada Silver contém dados tratados e padronizados, prontos para modelagem.  
Principais transformações realizadas:

- Padronização de tipos (ex.: conversão de `startYear` para inteiro).
- Criação da coluna `genres_array` a partir do split de gêneros.
- Explosão dos gêneros, permitindo granularidade consistente.
- Padronização de colunas (`is_adult`, `runtimeMinutes`).
- Junção entre basics e ratings utilizando a chave `tconst`.
- Criação das colunas derivadas `year_key` e `decade`.

Local: /Volumes/workspace/default/imdb_silver/

### 3.3 Camada Gold 
A camada Gold contém o modelo dimensional utilizado para análise.  
Foi desenvolvido um modelo estrela contendo:

- **Tabela fato**: `fact_title_rating`
- **Dimensões**: `dim_title`, `dim_genre`, `dim_date`

As tabelas foram registradas no Unity Catalog para consulta direta via SQL.

Local:/Volumes/workspace/default/imdb_gold/

Catalogação:
imdb_mvp.fact_title_rating
imdb_mvp.dim_title
imdb_mvp.dim_genre
imdb_mvp.dim_date

## 4. Modelagem dos Dados

### 4.1 Esquema Estrela
O modelo dimensional foi construído com o objetivo de suportar consultas analíticas de forma eficiente e intuitiva. A estrutura consiste em uma tabela fato contendo métricas (ratings e votos) e dimensões com atributos descritivos.

Diagrama simplificado:

          dim_date                           dim_title
     -----------------                    -----------------
     year_key (PK)                        tconst (PK)
     year                                 primaryTitle
     decade                               titleType
               \                          runtimeMinutes
                \                        /
                 \ (FK)          (FK)  /
                  --------------------
                        fact_title_rating
                  ------------------------------
                  tconst (FK → dim_title)
                  year_key (FK → dim_date)
                  averageRating
                  numVotes
                          \
                           \
                        dim_genre
                     -----------------
                     tconst (FK → dim_title)
                     genre


### 4.2 Tabelas

**Fato — `fact_title_rating`**  
Contém avaliações e métricas agregáveis:
- tconst  
- averageRating  
- numVotes  
- year_key  

**Dimensão — `dim_title`**  
Atributos descritivos:
- tconst  
- primaryTitle  
- originalTitle  
- titleType  
- runtimeMinutes  
- is_adult  
- year  

**Dimensão — `dim_genre`**  
Estrutura normalizada com um gênero por linha:
- tconst  
- genre  

**Dimensão — `dim_date`**  
Informações temporais:
- year  
- decade  

A descrição completa das colunas está no arquivo data_catalog.md.

## 5. Carga (ETL)

A etapa de carga foi realizada em notebooks Databricks utilizando PySpark. 

### 5.1 Extração e Carga da Camada Bronze
Notebook: `01_ingest_imdb`

- Leitura dos arquivos originais do IMDb.
- Mapeamento de valores nulos.
- Definição explícita do schema.
- Escrita no formato Delta para a camada Bronze.

### 5.2 Transformações na Camada Silver
Notebook: `02_transform_imdb`

- Conversão de tipos (string → integer, string → array).
- Limpeza de registros com valores ausentes críticos.
- Separação dos gêneros em arrays e explosão para múltiplas linhas.
- Junção entre `title.basics` e `title.ratings`.
- Criação de colunas derivadas:
  - `year_key` (chave temporal)
  - `decade`
  - `is_adult` (boolean)

### 5.3 Construção e Carga da Camada Gold
Notebook: `03_modeling_imdb`

- Construção da dimensão de datas a partir dos anos identificados.
- Normalização dos gêneros em tabela própria.
- Criação da dimensão de títulos com todos os atributos necessários.
- Criação da tabela fato com:
  - métricas (averageRating, numVotes)
  - chaves estrangeiras (tconst, year_key)
- Escrita das tabelas no formato Delta.
- Registro das tabelas no catálogo Unity Catalog para consulta SQL.

## 6. Qualidade dos Dados

Antes da realização das análises exploratórias e de negócio, foi conduzida uma etapa de verificação da qualidade dos dados com o objetivo de garantir a confiabilidade dos resultados extraídos do Data Warehouse. Esta etapa teve como foco a validação da consistência das tabelas, a identificação de valores ausentes, a verificação de domínios esperados, a análise de integridade referencial entre as tabelas do modelo estrela e a detecção de registros com datas inconsistentes.

As verificações realizadas permitem assegurar que os dados utilizados nas análises apresentam estrutura adequada, relações corretas entre dimensões e fatos, além de estarem livres de anomalias que poderiam comprometer a interpretação dos resultados.

### 6.1 Volume de registros por tabela

| Tabela             | Quantidade de Registros |
|--------------------|--------------------------|
| fact_title_rating  | 2.341.369                |
| dim_title          | 2.341.369                |
| dim_genre          | 3.871.466                |
| dim_date           | 153                      |

### 6.2 Schema e tipos de dados

Nesta etapa foi realizada a verificação dos esquemas das tabelas do Data Warehouse, com o objetivo de validar os tipos de dados, os atributos existentes e a aderência ao modelo dimensional proposto.

#### Tabela: fact_title_rating

| Coluna        | Tipo    | Nulo |
|---------------|----------|------|
| tconst        | string   | Sim  |
| averageRating| double   | Sim  |
| numVotes      | long     | Sim  |
| year_key      | integer  | Sim  |
| titleType     | string   | Sim  |

#### Tabela: dim_title

| Coluna         | Tipo     | Nulo |
|----------------|-----------|------|
| tconst         | string    | Sim  |
| titleType      | string    | Sim  |
| primaryTitle   | string    | Sim  |
| originalTitle  | string    | Sim  |
| is_adult       | boolean   | Sim  |
| runtimeMinutes | integer   | Sim  |
| year           | integer   | Sim  |

#### Tabela: dim_genre

| Coluna | Tipo   | Nulo |
|--------|--------|------|
| tconst | string | Sim  |
| genre  | string | Sim  |

#### Tabela: dim_date

| Coluna  | Tipo    | Nulo |
|---------|----------|------|
| year    | integer  | Sim  |
| decade | integer  | Sim  |

A inspeção dos esquemas confirma que os tipos de dados estão compatíveis com os respectivos domínios esperados, permitindo a correta execução das transformações, junções e análises analíticas ao longo do projeto.

### 6.3 Análise de valores nulos por tabela

A análise de valores nulos foi realizada para identificar possíveis lacunas de informação que possam impactar as análises posteriores.

#### Tabela: fact_title_rating

| Coluna         | Registros Nulos | Percentual |
|----------------|------------------|------------|
| tconst         | 0                | 0,00%      |
| averageRating | 1.643.814        | 70,21%     |
| numVotes       | 1.643.814        | 70,21%     |
| year_key       | 184.242          | 7,87%      |
| titleType      | 0                | 0,00%      |

#### Tabela: dim_title

| Coluna          | Registros Nulos | Percentual |
|------------------|------------------|------------|
| tconst           | 0                | 0,00%      |
| titleType        | 0                | 0,00%      |
| primaryTitle     | 0                | 0,00%      |
| originalTitle    | 0                | 0,00%      |
| is_adult         | 0                | 0,00%      |
| runtimeMinutes   | 937.871          | 40,06%     |
| year             | 184.242          | 7,87%      |

#### Tabela: dim_genre

| Coluna | Registros Nulos | Percentual |
|--------|------------------|------------|
| tconst | 0                | 0,00%      |
| genre  | 0                | 0,00%      |

#### Tabela: dim_date

| Coluna | Registros Nulos | Percentual |
|--------|------------------|------------|
| year   | 0                | 0,00%      |
| decade | 0                | 0,00%      |

Observa-se elevado percentual de valores nulos em `averageRating` e `numVotes`, indicando que grande parte dos títulos não possui avaliações registradas no IMDb, o que é esperado para produções com baixa visibilidade ou muito recentes.  

A coluna `runtimeMinutes` também apresenta volume significativo de valores ausentes, especialmente em conteúdos como curtas, séries ou registros incompletos.  

Os valores nulos em `year` e `year_key` correspondem a títulos sem ano de lançamento informado na base original. Para evitar impactos nas análises temporais, esses registros foram excluídos das consultas por ano e década por meio de filtros específicos.  

As tabelas `dim_genre` e `dim_date` não apresentam valores nulos, indicando completa consistência estrutural nessas dimensões.

### 6.4 Análise de domínios dos dados numéricos

Nesta etapa foram analisados os valores mínimos e máximos dos principais atributos numéricos para validação de coerência com o domínio do problema.

#### Métricas de avaliação (fact_title_rating)

| Métrica      | Valor |
|---------------|--------|
| Nota mínima   | 1,0    |
| Nota máxima   | 10,0   |
| Votos mínimos| 5      |
| Votos máximos| 3.127.637 |

Esses valores estão totalmente coerentes com o padrão do IMDb, no qual as avaliações variam de 1 a 10 e o número de votos pode atingir milhões em títulos de altíssima popularidade.

#### Atributos de duração e ano (dim_title)

| Métrica                | Valor |
|-------------------------|--------|
| Duração mínima (min)   | 0      |
| Duração máxima (min)   | 3.692.080 |
| Ano mínimo              | 1874   |
| Ano máximo              | 2115   |

A presença de duração igual a 0 indica registros sem informação válida de tempo. A duração máxima extremamente elevada caracteriza um **outlier evidente**, típico de erros de registro ou agregações indevidas no dado original.  

O intervalo de anos inicia-se em 1874, compatível com os primórdios do cinema. Já valores superiores ao ano atual (2115) representam **anos futuros inconsistentes**, tratados posteriormente com filtros nas análises temporais.

#### Atributos temporais consolidados (dim_date)

| Métrica        | Valor |
|----------------|--------|
| Ano mínimo     | 1874   |
| Ano máximo     | 2115   |
| Década mínima | 1870   |
| Década máxima | 2110   |

A dimensão temporal reflete diretamente os valores observados nos títulos. Os registros com anos futuros foram mantidos fisicamente na base para rastreabilidade, porém desconsiderados nas análises por meio de filtros (`year <= year(current_date())`).

**Interpretação geral:**

Os domínios dos dados numéricos são, em sua maioria, coerentes com o contexto do problema. As exceções identificadas referem-se a valores extremos de duração e a anos futuros, que não comprometem os resultados devido ao tratamento aplicado nas consultas analíticas.

### 6.5 Análise de distribuição dos dados (quartis)

Nesta etapa foi analisada a distribuição dos principais atributos numéricos por meio dos quartis, com o objetivo de compreender a concentração dos dados e identificar possíveis assimetrias.

#### Distribuição do número de votos (fact_title_rating)

| Percentil | Número de Votos |
|-----------|------------------|
| 25% (P25) | 14               |
| 50% (Mediana) | 34           |
| 75% (P75) | 156              |

Observa-se que a maioria dos títulos possui baixo volume de votos, com 75% dos registros apresentando até 156 votos. Isso indica uma forte concentração de votos em um pequeno subconjunto de títulos muito populares, caracterizando um efeito de cauda longa.

#### Distribuição da duração dos títulos (dim_title)

| Percentil | Duração (min) |
|-----------|----------------|
| 25% (P25) | 10             |
| 50% (Mediana) | 29         |
| 75% (P75) | 81             |

Nota-se que 75% dos títulos possuem duração inferior a 81 minutos, o que reforça a predominância de curtas, episódios de séries e conteúdos de menor duração na base, em contraste com a minoria de títulos muito longos.

### 6.6 Análise de duplicidade de registros

Nesta etapa foi verificada a existência de registros duplicados nas chaves primárias e compostas das tabelas do Data Warehouse, como forma de garantir a unicidade dos dados.

| Tabela              | Chave Avaliada        | Registros Duplicados |
|---------------------|------------------------|-----------------------|
| fact_title_rating   | tconst                | 0                     |
| dim_title           | tconst                | 0                     |
| dim_genre           | (tconst, genre)       | 0                     |
| dim_date            | year                  | 0                     |

Os resultados indicam que não há registros duplicados nas chaves avaliadas, garantindo a integridade das chaves primárias e compostas do modelo dimensional e assegurando a consistência das junções entre as tabelas.

### 6.7 Análise de integridade referencial

Nesta etapa foi validada a consistência das chaves estrangeiras entre as tabelas do modelo dimensional, assegurando que todos os relacionamentos estejam devidamente representados.

#### Relação: fact_title_rating.tconst → dim_title.tconst

| Métrica                                      | Valor     |
|----------------------------------------------|-----------|
| Total de linhas na tabela com FK             | 2.341.369 |
| Linhas com FK nula                           | 0         |
| Linhas sem correspondência (inclui nulos)   | 0         |
| Linhas sem correspondência (FK não nula)    | 0         |

Todos os registros da tabela fato possuem correspondência na dimensão de títulos, indicando integridade referencial completa nesse relacionamento.

#### Relação: fact_title_rating.year_key → dim_date.year

| Métrica                                      | Valor     |
|----------------------------------------------|-----------|
| Total de linhas na tabela com FK             | 2.341.369 |
| Linhas com FK nula                           | 184.242   |
| Linhas sem correspondência (inclui nulos)   | 184.242   |
| Linhas sem correspondência (FK não nula)    | 0         |

Os registros sem correspondência estão integralmente associados a valores nulos de `year_key`. Não existem chaves não nulas sem correspondência na dimensão temporal. Esses registros foram desconsiderados nas análises temporais por meio de filtros específicos.

#### Relação: dim_genre.tconst → dim_title.tconst

| Métrica                                      | Valor     |
|----------------------------------------------|-----------|
| Total de linhas na tabela com FK             | 3.871.466 |
| Linhas com FK nula                           | 0         |
| Linhas sem correspondência (inclui nulos)   | 0         |
| Linhas sem correspondência (FK não nula)    | 0         |

Todos os registros da dimensão de gêneros possuem correspondência válida na dimensão de títulos, garantindo a consistência do relacionamento entre essas tabelas.

**Resumo da integridade referencial:**

Não foram identificados problemas de integridade referencial entre as tabelas do modelo estrela. As únicas ocorrências de não correspondência estão associadas a valores nulos de `year_key` na tabela fato, que não caracterizam violação de integridade e foram devidamente tratadas nas análises por meio de filtros.


## 7. Análise dos Dados
As análises foram realizadas no notebook `04_analysis_imdb` utilizando SQL e PySpark.  

### 7.1 Evolução do volume de lançamentos por ano

<img width="1341" height="480" alt="visualization" src="https://github.com/user-attachments/assets/fe628283-2b46-42da-89e0-666007833980" />

Observa-se crescimento contínuo no número de títulos lançados entre 1940 e o final da década de 1990, seguido por uma forte aceleração a partir dos anos 2000. O período entre 2005 e 2018 concentra a maior expansão, impulsionada pela digitalização da produção audiovisual e pela consolidação das plataformas de streaming. A partir de 2019 ocorre uma queda gradual no volume de lançamentos, possivelmente associada principalmente aos impactos da pandemia de COVID-19.

### 7.2 Títulos mais votados por década (a partir de 2000)

| Década | Título                                       | Número de Votos |
|--------|----------------------------------------------|------------------|
| 2000   | The Dark Knight                              | 3.103.210        |
| 2000   | Breaking Bad                                | 2.434.902        |
| 2000   | The Lord of the Rings: The Fellowship of the Ring | 2.160.893        |
| 2010   | Inception                                   | 2.757.756        |
| 2010   | Game of Thrones                             | 2.506.361        |
| 2010   | Interstellar                                | 2.438.660        |
| 2020   | Dune: Part One                              | 990.589          |
| 2020   | Spider-Man: No Way Home                     | 988.797          |
| 2020   | Oppenheimer                                 | 960.504          |


### 7.3 Popularidade por gênero (volume total de votos)

A popularidade foi medida pelo volume total de votos por gênero, refletindo o engajamento do público com cada categoria.

**Top 5 gêneros mais populares (mais votos):**

| Gênero     | Total de Votos   |
|------------|------------------|
| Drama      | 772.952.317      |
| Action     | 448.199.458      |
| Comedy     | 444.177.660      |
| Adventure  | 369.855.310      |
| Crime      | 296.602.540      |

**Bottom 5 gêneros menos populares (menos votos):**

| Gênero       | Total de Votos |
|--------------|----------------|
| Adult        | 389.453        |
| News         | 828.664        |
| Game-Show    | 1.391.747      |
| Talk-Show    | 1.416.899      |
| Reality-TV   | 3.168.894      |

Observa-se que gêneros tradicionais do cinema comercial, como *Drama*, *Action* e *Comedy*, concentram a maior parte do engajamento do público. Em contrapartida, gêneros televisivos e de menor apelo comercial apresentam baixa participação no volume total de votos.

### 7.4 Avaliação média por gênero

Esta análise considera a nota média dos títulos dentro de cada gênero, independentemente do volume de votos.

**Top 5 gêneros com maior avaliação média:**

| Gênero        | Avaliação Média | Quantidade de Títulos |
|----------------|------------------|------------------------|
| Documentary    | 7.08             | 107.794               |
| Biography      | 7.01             | 17.397                |
| History        | 6.99             | 18.376                |
| Music          | 6.84             | 17.453                |
| Short          | 6.80             | 158.090               |

**Bottom 5 gêneros com menor avaliação média:**

| Gênero     | Avaliação Média | Quantidade de Títulos |
|------------|------------------|------------------------|
| Western    | 6.04             | 6.473                 |
| Thriller   | 5.97             | 42.317                |
| Adult      | 5.71             | 6.232                 |
| Horror     | 5.68             | 43.979                |

Observa-se que gêneros informativos e biográficos apresentam as maiores avaliações médias, enquanto gêneros de forte apelo comercial, como *Horror* e *Adult*, concentram as menores médias. Nota-se também que popularidade (votos) e qualidade percebida (nota média) não apresentam, necessariamente, correlação direta.

### 7.5 Relação entre duração e avaliação média

| Faixa de Duração (min) | Quantidade de Títulos | Avaliação Média |
|------------------------|------------------------|------------------|
| < 60                   | 231.627                | 6,84             |
| 60–89                  | 140.196                | 6,15             |
| 90–119                 | 154.656                | 6,12             |
| 120–149                | 30.664                 | 6,54             |
| 150–179                | 8.241                  | 6,71             |
| ≥ 180                  | 6.335                  | 6,90             |

A análise estatística mostra uma correlação praticamente nula entre a duração dos títulos e a nota média (correlação ≈ 0,00006), indicando que, de forma geral, o tempo de duração não exerce influência relevante sobre a avaliação do público.

### 7.6 Avaliação média por tipo de título

| Tipo de Título | Avaliação Média | Quantidade de Títulos |
|----------------|------------------|------------------------|
| tvMiniSeries   | 6,99             | 23.782                |
| tvSeries       | 6,83             | 107.485               |
| short          | 6,81             | 174.591               |
| tvMovie        | 6,59             | 55.768                |
| movie          | 6,18             | 335.929               |

Observa-se que minisséries e séries de TV apresentam as maiores avaliações médias, indicando maior percepção de qualidade nesse formato. Filmes para TV e filmes de cinema apresentam médias inferiores, possivelmente associadas à maior heterogeneidade de produções e ao elevado volume de títulos lançados nesses formatos.

### 7.7 Evolução da avaliação média ao longo das décadas (a partir de 1940)

<img width="1341" height="480" alt="visualization (2)" src="https://github.com/user-attachments/assets/6ffcfb3a-892f-45bd-91b7-b1f603d5610d" />

A partir da década de 1940, as avaliações médias apresentam comportamento predominantemente estável, variando entre 6,2 e 6,4 até os anos 1990. Na década de 1970 observa-se um leve recuo, seguido por retomada gradual nas décadas seguintes.

A partir dos anos 2000, verifica-se crescimento contínuo da avaliação média, com destaque para as décadas de 2010 e 2020, que atingem os maiores valores da série (acima de 6,6). Esse movimento sugere uma tendência recente de aumento da percepção de qualidade das produções, possivelmente associada à maior profissionalização do setor e à consolidação das plataformas de streaming.

### 7.8 Evolução do volume de lançamentos por gênero ao longo das décadas

<img width="1258" height="480" alt="visualization (3)" src="https://github.com/user-attachments/assets/627c8d58-f55c-47b9-9342-fced16eed439" />

A distribuição de títulos por gênero mostra crescimento contínuo desde 1940, com aceleração expressiva a partir dos anos 2000. Gêneros como *Short*, *Drama*, *Documentary* e *Comedy* apresentam os maiores volumes absolutos, refletindo sua ampla produção global. Na década de 2010, observa-se um pico significativo em praticamente todos os gêneros, impulsionado pela expansão de plataformas digitais, redução de barreiras de produção e diversificação de formatos.

A década de 2020 mostra retração em vários gêneros, atribuída ao período ainda incompleto e aos efeitos da pandemia na produção audiovisual. Apesar disso, o padrão histórico reforça que o aumento da capacidade de produção e distribuição transformou o perfil da indústria, ampliando a oferta em praticamente todas as categorias.

## 8. Autoavaliação

O objetivo principal deste trabalho foi construir um pipeline completo de dados na nuvem, desde a coleta até a análise, utilizando dados públicos do IMDb e a plataforma Databricks. De modo geral, os objetivos propostos foram atingidos de forma satisfatória. Foi possível coletar, modelar, carregar e analisar os dados, bem como responder às principais perguntas definidas no início do projeto por meio de consultas analíticas e visualizações.

A etapa de modelagem permitiu a construção de um modelo dimensional em esquema estrela, garantindo organização e eficiência nas consultas. A carga dos dados foi realizada por meio de pipelines no Databricks, possibilitando a consolidação das camadas Bronze, Silver e Gold. A análise de qualidade dos dados foi fundamental para identificar valores nulos, domínios inconsistentes e garantir a integridade referencial entre as tabelas, assegurando maior confiabilidade nos resultados analíticos.

Entre as principais dificuldades enfrentadas, destacam-se a configuração inicial do ambiente no Databricks, a organização dos arquivos no DBFS e a necessidade de ajustes nos dados temporais devido à presença de anos futuros e valores ausentes. Além disso, a elevada quantidade de dados exigiu cuidados adicionais com performance nas consultas e na execução das transformações.

Como trabalhos futuros, sugere-se a incorporação de novas fontes de dados, como informações sobre elenco e diretores, permitindo análises mais avançadas sobre influência de equipes nas avaliações. Também seria possível evoluir o projeto com a criação de dashboards interativos, aplicação de técnicas de machine learning para previsão de avaliações e automatização completa das pipelines de atualização dos dados.

De forma geral, o projeto contribuiu significativamente para a consolidação prática dos conceitos de engenharia de dados, modelagem dimensional e análise de BI em ambiente de nuvem.
