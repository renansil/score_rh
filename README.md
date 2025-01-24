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
