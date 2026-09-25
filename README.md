# 🎮 MVP: Pipeline Analítico da Steam

## 1. Contexto de Negócios e Origem dos Dados
Este projeto é um MVP focado no mercado de jogos de PC. O objetivo é criar um pipeline analítico de ponta a ponta no Databricks para extrair *insights* sobre precificação, popularidade e qualidade por gênero.

**Sobre os Dados Brutos:**
O projeto utiliza o dataset `games-features.csv`, extraído de informações da plataforma Steam. 
* **Estrutura Bruta:** Os dados originais consistem em um arquivo plano (CSV) contendo múltiplas colunas mistas, como identificadores, texto descritivo, datas em formatos variados e diversas colunas booleanas (`True/False`) para representar os gêneros dos jogos.
* **Licença e Créditos:** O dataset base ("Steam Game Data") foi disponibilizado na plataforma [Kaggle](https://www.kaggle.com/datasets/ibriiee/stream-game-date) sob a licença **CC0: Public Domain**, permitindo seu uso livre, modificação e distribuição para fins educacionais e de pesquisa analítica.

**Perguntas de Negócio Formuladas:**
1. Jogos gratuitos atraem uma base de jogadores maior, e como a crítica os avalia em comparação aos pagos?
2. Como o mercado de jogos de PC se comportou financeiramente ao longo dos anos?
3. Quais gêneros geram a comunidade mais engajada em volume de recomendações?

---

## 2. Pipeline de Dados e Carga
**Carga dos Dados:** A ingestão dos dados brutos foi realizada via *upload* do arquivo CSV diretamente para o DBFS (Databricks File System). O Spark fez a leitura inferindo os tipos iniciais e tratando caracteres de escape.

**Organização do Pipeline (ETL):**    
Todo o processo de Extração, Transformação e Carga foi desenvolvido, orquestrado e centralizado no databricks.
> **O script completo pode ser acessado neste repositório no arquivo:** [MVP - Jogos da STEAM](MVP%20-%20Jogos%20da%20STEAM.ipynb)

Tabelas no Data Lake:

<img width="317" height="519" alt="BandodedadosDataBricks" src="https://github.com/user-attachments/assets/2e9db2fb-149e-460c-80ec-5f33ebf67729" />


---

## 3. Qualidade de Dados
Durante a analise dos dados, detectei e resolvi os seguintes problemas para garantir a integridade:

* **Padronização:** As nomenclaturas das colunas originais eram confusas. Renomei seguindo o padrão em português.
* **Inconsistência de Datas:** A coluna de data de lançamento estava no formato texto (String) com preenchimentos irregulares. Apliquei **Expressões Regulares (Regex)** via PySpark para identificar e extrair apenas os quatro dígitos numéricos correspondentes ao ano de lançamento..
* **Múltiplas Colunas de Gênero:** Os gêneros dos jogos estavam fragmentados em várias colunas booleanas (ex: `GenreIsIndie`, `GenreIsAction`). Utilizei a função `stack` (**Unpivot**) para transformar essas colunas em linhas. Isso resolveu o problema da multiplicidade, permitindo agregações matemáticas precisas.

---

## 4. Modelagem e Catálogo de Dados
Os dados foram modelados no formato *Star Schema* e salvos como **Delta Tables**.

### Catálogo de Dados (Dicionário)
* **`dim_jogo` (Tabela de Contexto):**
  * `id_jogo` (Int): Chave primária.
  * `nome_jogo` (String): Nome oficial do título.
  * `ano_lancamento` (Int): Ano extraído via Regex.
  * `eh_gratuito` (Boolean): Flag de monetização.
* **`dim_genero` (Tabela de Categoria):**
  * `id_jogo` (Int): Chave estrangeira.
  * `genero` (String): Categoria do jogo (Ação, RPG, Indie, etc).
* **`fato_estatistica` (Tabela de Métricas):**
  * `id_jogo` (Int): Chave estrangeira.
  * `preco` (Float): Preço de venda.
  * `nota_metacritic` (Int): Avaliação da crítica.
  * `total_jogadores_estimado` (Int): Estimativa de base de usuários.
  * `total_recomendacoes` (Int): Volume de engajamento da comunidade.

<img width="1373" height="490" alt="EstruturaBanco" src="https://github.com/user-attachments/assets/abf7a92b-1103-4753-aee9-b95b58d1a990" />

---

## 5. Análise de Dados

### 5.1. Análise de Monetização
**Pergunta:** Jogos gratuitos atraem uma base maior de jogadores? E a qualidade?

**Consulta SQL:**
```sql
SELECT 
    CASE 
        WHEN d.eh_gratuito = True THEN 'Gratuito' 
        ELSE 'Pago' 
    END AS `Modelo de Monetização`,
    COUNT(d.id_jogo) AS total_jogos,
    ROUND(AVG(f.total_jogadores_estimado), 0) AS media_jogadores,
    ROUND(AVG(f.nota_metacritic), 1) AS media_metacritic
FROM workspace.mvp.fato_estatistica f
INNER JOIN workspace.mvp.dim_jogo d USING (id_jogo)
WHERE f.nota_metacritic > 0 
GROUP BY 1 
ORDER BY media_jogadores DESC;
```
<img width="1342" height="400" alt="Gráfico Grátis vs Pago" src="https://github.com/user-attachments/assets/3a7d623a-076b-4a38-8f3e-fdb710d21e53" />

>**Conclusão**: Os dados mostram que o modelo gratuito atinge um público bem maior — em média, 3,1 milhões de jogadores, contra 480 mil dos jogos pagos. Já quando comparamos as notas da crítica, há um empate técnico (72,2 contra 72,1). Ou seja: os jogos gratuitos têm muito mais adesão, mas, mesmo assim, as avaliações da crítica ficam praticamente iguais.

### 5.2. A Evolução do Preço no Tempo
**Pergunta:** Como o mercado de jogos se comportou financeiramente ao longo dos anos?

**Consulta SQL:**
```sql
SELECT 
    d.ano_lancamento,
    COUNT(d.id_jogo) AS total_lancamentos,
    ROUND(AVG(f.preco_final), 2) AS preco_medio
FROM workspace.mvp.fato_estatistica f
INNER JOIN workspace.mvp.dim_jogo d USING (id_jogo)
WHERE d.ano_lancamento BETWEEN 1997 AND 2016 
  AND d.eh_gratuito = False 
GROUP BY d.ano_lancamento
ORDER BY d.ano_lancamento;
```

<img width="1342" height="400" alt="Gráfico Histórico de Preço" src="https://github.com/user-attachments/assets/81278f78-786e-4073-8e5d-b01aa5b33c7c" />

> **Conclusão Analítica:** A análise da série temporal entre 1997 e 2016 mostra como a indústria se comportou em termos de inflação e precificação. Há quedas no final dos anos 90, seguidas de uma trajetória de alta que chega ao pico de preço médio de 13,89 em 2013 — momento que marca a adoção de novas "tabelas de preços" para jogos de maior orçamento.

### 5.3. Engajamento por Gênero
Pergunta: Quais gêneros geram a comunidade mais engajada em volume de recomendações?
```sql
SELECT 
    g.genero,
    COUNT(d.id_jogo) AS total_jogos,
    ROUND(AVG(f.nota_metacritic), 1) AS media_metacritic,
    ROUND(AVG(f.total_recomendacoes), 0) AS media_recomendacoes
FROM workspace.mvp.fato_estatistica f
INNER JOIN workspace.mvp.dim_jogo d USING (id_jogo)
INNER JOIN workspace.mvp.dim_genero g USING (id_jogo)
WHERE f.nota_metacritic > 0
GROUP BY g.genero
ORDER BY media_recomendacoes DESC;
```
<img width="1342" height="400" alt="Gráfico Engajamento por Gênero" src="https://github.com/user-attachments/assets/7dad975c-0c37-4f34-8490-3b12415c5bd5" />

> **Conclusão Analítica:** Os dados mostram que os nichos tradicionais de Ação (com média de 7.050 recomendações por jogo) e RPG (5.462) lideram o engajamento da comunidade. Curiosamente, categorias que publicam muito, como Indie e Casual, ficam na base do ranking de engajamento médio. Isso ajuda a entender o comportamento do mercado: Software Houses em busca de comunidades grandes e ativas tendem a investir principalmente em títulos de Ação e RPG.

---

## 6. Autoavaliação e Considerações Finais
A construção deste MVP foi um desafio para colocar em prática a estruturação de um pipeline completo e atendeu aos objetivos propostos. Para garantir a **Qualidade dos Dados e a Modelagem Dimensional**, precisei tomar algumas decisões importantes ao longo do caminho:

* **Padronização e Entendimento:** O primeiro passo foi analisar os tipos de dados originais e arrumar as nomenclaturas das colunas. Isso foi essencial para facilitar a leitura e o entendimento do negócio antes de começar a separar as tabelas.
* **Tratamento de Texto com Regex:** Ao analisar os dados para criar as tabelas dimensão, reparei que a coluna de data de lançamento estava como texto e com os formatos bagunçados. Para resolver isso, utilizei o *Regex* para manipular o texto e conseguir extrair apenas o ano exato de forma limpa.

**Dificuldades Encontradas:**
A principal dificuldade foi a **Modelagem de Gêneros com Unpivot**. Percebi que existiam várias colunas booleanas indicando o gênero do jogo e, como um mesmo jogo pode ter mais de um gênero, precisei entender como utilizar o *Unpivot* para separar essas informações em linhas sem distorcer os números da análise.
Além disso, foi o meu projeto utilizando o **Databricks**. Tive que aprender a plataforma do zero, entender como funciona a importação de um arquivo `.csv`, como acessar estes dados e entender como consultar os dados pelo notebook.

**Trabalhos Futuros:**
Foi uma excelente oportunidade para vivenciar a criação de um pipeline de ponta a ponta na prática. Para evoluir este projeto no meu portfólio no futuro, pretendo:
1. Substituir a carga manual do `.csv` por uma ingestão automatizada consumindo a API oficial da Steam (Steamworks).
2. Conectar a base final a uma ferramenta de BI externa para a criação de um Dashboard interativo.

---
**Tecnologias Utilizadas:** Databricks, PySpark, SQL, Delta Lake, Unity Catalog, Markdown.
