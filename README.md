# score_rh

### *Importando bibliotecas necessárias e conectando ao banco de dados SQLite*

Aqui optei por um SQL com as colunas que iria usar para o sistema de pontuação.

```python
# código Python
import sqlite3
import pandas as pd

conn = sqlite3.connect('teste___desafio_tcnico_-_analista_de_dados.db')

# Consulta para extrair os dados necessários
query = """
SELECT *
FROM (
    SELECT
        candidatos.id AS id,
        candidatos.nome AS nome,
        vagas.nome AS vaga,
        vagas.nivel_vaga,
        COUNT(DISTINCT vagas.departamento) AS candidaturas,
        CASE
            WHEN candidatos.endereco LIKE '%SP%' THEN 'Estado alvo'
            WHEN candidatos.endereco LIKE '%SC%' THEN 'Estado alvo'
            WHEN candidatos.endereco LIKE '%PB%' THEN 'Estado alvo'
            ELSE 'Outros'
        END AS estado,
        CASE
            WHEN tempo_de_experiencia LIKE '%1 ano%' THEN 1
            WHEN tempo_de_experiencia LIKE '%18 meses%' THEN 1.6
            WHEN tempo_de_experiencia LIKE '%2 anos%' THEN 2
            WHEN tempo_de_experiencia LIKE '%3,5 anos%' THEN 3.5
            WHEN tempo_de_experiencia LIKE '%5 anos%' THEN 5
            WHEN tempo_de_experiencia LIKE '%72 meses%' THEN 6
            WHEN tempo_de_experiencia LIKE '%36 meses%' THEN 3
            ELSE 0
        END AS tempo_exp_em_anos,
        minimo_experiencia AS min_experiencia,
        CASE WHEN competencias.tipo = 'Atitude' THEN 1 ELSE 0 END AS fit_cultural,
        competencias.area AS area,
        competencias.descricao AS descricao,
        localizacao AS localizacao_da_vaga,
        salario_minimo,
        pretensao_salarial,
        salario_maximo
    FROM candidatos
    LEFT JOIN candidato_vaga ON candidatos.id = candidato_vaga.id_candidato
    LEFT JOIN vagas ON candidato_vaga.id_vaga = vagas.id
    LEFT JOIN vagacompetencia ON vagas.id = vagacompetencia.id_vaga
    LEFT JOIN competencias ON vagacompetencia.id_competencia = competencias.id
    LEFT JOIN competencia_experiencia ON competencias.id = competencia_experiencia.id_competencia
    GROUP BY candidatos.id
) AS x
"""
candidatos_df = pd.read_sql_query(query, conn)

```
## *Função para calcular o score*
### Requisitos:
 1. **Localização** - Se o candidato está em um dos `Estado alvo` (São Paulo, Santa Catarina ou Paraíba), 20 pontos são adicionados ao score. Isso incentiva a seleção de candidatos que estão em áreas geográficas desejadas pela empresa.

 2. **Pretensão Salarial** - Se a pretensão salarial do candidato está dentro de um intervalo considerado aceitável (entre o mínimo mais 100 e o máximo menos 100), o candidato recebe 30 pontos. Isso significa que o candidato está alinhado com a faixa salarial ideal. Se a pretensão salarial está dentro do intervalo definido por `salario_minimo` e `salario_maximo`, mas não exatamente nos valores extremos (ou seja, muito perto do mínimo ou do máximo), o candidato recebe 15 pontos. Pensei em um critério mais flexível, mas que ainda reconhece que o candidato é uma boa opção.

 3. **Fit Cultural** - O fit cultural é uma medida de como os valores e comportamentos do candidato se alinham com os da empresa. Se o candidato tem um fit cultural avaliado como 1 (ou seja, atende aos critérios de alinhamento), ele recebe 30 pontos.

 4. **Experiência Técnica** - Se o candidato tem 5 anos ou mais de experiência em sua área de atuação, ele recebe 20 pontos. Isso valoriza a experiência técnica.

 4. **Cargos não relacionados** - Se o candidato tem mais 1 candidatura que seja em um departamento diferente, ele não pontua, recebe 5 pontos se tiver candidaturas para o mesmo departamento com vagas diferentes.

``` python
def calcular_score(row, salario_minimo, salario_maximo):
    score = 0

    if row['estado'] == 'Estado alvo':
        score += 20

    if salario_minimo + 100 <= row['pretensao_salarial'] <= salario_maximo - 100:
        score += 30
    elif salario_minimo <= row['pretensao_salarial'] <= salario_maximo:
        score += 15

    if row['fit_cultural'] == 1:
        score += 30

    if row['tempo_exp_em_anos'] >= 5:
        score += 20

    if row['candidaturas'] == 1:
        score += 5

    return min(score, 100)
```
### *Calcular o Score para Cada Candidato*  

Aplicação do Score: Aplicação da função `calcular_scores` onde ela aplica `calcular_score` a cada linha da base de dados, adicionando uma nova coluna chamada score.


``` python
def calcular_scores(df, salario_minimo, salario_maximo):
    df['score'] = df.apply(lambda x: calcular_score(x, salario_minimo, salario_maximo), axis=1)
    return df
```
### *Salvar os Resultados em um Arquivo CSV*

Passando a função `salvar_resultados` para criação da exportação dos resultados para um arquivo CSV.
* Configurações Iniciais:
Passando o valor Min/Max de sálario retirei esses valores da base de dados usando a função min/max das colunas `salario_maximo` e `salario_minimo`.

1. Minimo = R$ 1214

2. Máximo = R$ 13389

Seguindo as configurações salvei os resultados em um CSV.
```python
def salvar_resultados(df, filename):
    df.to_csv(filename, index=False)

salario_minimo = 1214
salario_maximo = 13389

resultados_df = calcular_scores(candidato, salario_minimo, salario_maximo)

salvar_resultados(resultados_df, 'resultados_candidatos.csv')
```
# **Resultado**
```python
resultados_df = pd.read_csv('resultados_candidatos.csv')

resultados_df
