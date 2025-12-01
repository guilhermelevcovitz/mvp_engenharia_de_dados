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

**Fato — fact_title_rating**  
Contém avaliações e métricas agregáveis:
- tconst  
- averageRating  
- numVotes  
- year_key  

**Dimensão — dim_title**  
Atributos descritivos:
- tconst  
- primaryTitle  
- originalTitle  
- titleType  
- runtimeMinutes  
- is_adult  
- year  

**Dimensão — dim_genre**  
Estrutura normalizada com um gênero por linha:
- tconst  
- genre  

**Dimensão — dim_date**  
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
- Junção entre `title.basics` e `title.ratings`, documentando:
  - chave usada (`tconst`)
  - granularidade resultante
  - colunas mantidas e colunas descartadas
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

## 6. Análise dos Dados
As análises foram realizadas no notebook `04_analysis_imdb` utilizando SQL e PySpark.  

xxxxxxxxxxxxxxxxxx

## 7. Qualidade dos Dados
Foi conduzida uma avaliação da qualidade dos dados em todas as colunas do modelo, observando:

- Distribuição de valores nulos.
- Verificação de intervalos válidos para ratings, votos, duração e anos.
- Padronização de categorias de gênero e tipo de título.
- Consistência de chaves entre fato e dimensões.
- Duplicatas e integridade referencial.

## 8. Autoavaliação
xxxxxxxxxxxxx

