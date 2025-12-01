# Catálogo de Dados

## 1. Fonte dos Dados
Os dados foram obtidos do repositório oficial IMDb (https://datasets.imdbws.com). Arquivos utilizados:
- title.basics.tsv.gz
- title.ratings.tsv.gz

Data de download: 01/12/2025

## 2. Linhagem dos Dados

Site IMDb → Camada Bronze → Camada Silver → Camada Gold (DW)

- Bronze: dados brutos conforme arquivos originais.
- Silver: dados tratados, padronizados e com tipos corrigidos.
- Gold: modelo dimensional pronto para análise.

## 3. Tabelas da Camada Gold

### 3.1 fact_title_rating
Contém as métricas relacionadas às avaliações dos títulos.

| Coluna        | Tipo   | Descrição                             | Intervalo Esperado |
|---------------|--------|---------------------------------------|--------------------|
| tconst        | string | Identificador único do título         | —                  |
| averageRating | double | Nota média dada pelos usuários        | 1.0 – 10.0         |
| numVotes      | long   | Número total de votos                 | 1 – 2.000.000      |
| year_key      | int    | Ano de lançamento (chave da dim_date) | 1874 – 2025        |

### 3.2 dim_title
Tabela contendo informações descritivas dos títulos.

| Coluna         | Tipo    | Descrição                     | Categorias / Intervalos                       |
|----------------|---------|-------------------------------|-----------------------------------------------|
| tconst         | string  | Identificador único do título | —                                             |
| titleType      | string  | Tipo da obra                  | movie, short, tvSeries, tvMovie, tvMiniSeries |
| primaryTitle   | string  | Título principal              | texto livre                                   |
| originalTitle  | string  | Título original               | texto livre                                   |
| runtimeMinutes | int     | Duração em minutos            | 1 – 500+                                      |
| is_adult       | boolean | Indicador de conteúdo adulto  | true/false                                    |
| year           | int     | Ano de lançamento             | 1874 – 2025                                   |

### 3.3 dim_genre
Gênero associado ao título. Um título pode conter múltiplos gêneros.

| Coluna | Tipo   | Descrição               | Categorias                                                |
|--------|--------|-------------------------|-----------------------------------------------------------|
| tconst | string | Identificador do título | —                                                         |
| genre  | string | Gênero                  | Action, Comedy, Drama, Horror, Romance, Documentary, etc. |

---

### 3.4 dim_date
Tabela temporal para análises por período.

| Coluna | Tipo | Descrição             | Intervalo           |
|--------|------|-----------------------|---------------------|
| year   | int  | Ano                   | 1874 – 2025         |
| decade | int  | Década correspondente | 1870, 1880, …, 2020 |
