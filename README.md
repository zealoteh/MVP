## 1. Visão Geral do Projeto
Este projeto é um MVP focado na construção de um pipeline analítico de ponta a ponta utilizando **Databricks** (PySpark e SQL). 

O objetivo principal é processar dados brutos do dataset `games-features.csv` (Steam), aplicar regras de qualidade de dados, modelar e extrair *insights* focados no mercado de jogos de PC, analisando precificação, popularidade e qualidade por gênero.

---

## 2. Arquitetura de Dados 

* **Extração de Dados:** Ingestão dos dados brutos a partir do Unity Catalog, com inferência automática de tipos e tratamento de caracteres de escape para textos complexos.
* **Qualidade e Transformação:** Aplicação de regras de governança, como:
    *   Remoção de registros nulos nas chaves de identificação.
    *   Exclusão de registros duplicados.
    *   Padronização da nomenclatura das colunas.
* **Modelagem Dimensional:** Os dados foram divididos em diferentes tabelas (Fato e Dimensões) para facilitar a análise e a criação dos gráficos.

---

## 3. Qualidade de Dados e Modelagem Dimensional
Para viabilizar as análises de negócio, o dado limpo foi modelado nas seguintes tabelas:

*   **`dim_jogo`:** Tabela de contexto do jogo. 
    *   *Tratamento aplicado:* A coluna original de data possuía formatos inconsistentes. Foi utilizada **Expressões Regulares (Regex)** para extrair exatamente o ano de lançamento e converter falhas para valores nulos de forma controlada.
*   **`dim_genero`:** Tabela de categorias.
    *   *Tratamento aplicado:* Como os gêneros originais estavam divididos em múltiplas colunas booleanas, foi utilizado **Unpivot** do PySpark, transformando as colunas em linhas.
*   **`fato_estatistica`:** Tabela central contendo as métricas numéricas de negócio (Preço, Notas do Metacritic, Recomendações e Estimativa de Jogadores).

---

## 4. Análise de Negócio e Dashboards

### 4.1. Analise da Monetização Grátis vs. Pago 

**Pergunta:** Jogos gratuitos atraem uma base de jogadores significativamente maior? Como a crítica avalia a qualidade desses jogos em comparação aos pagos?

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

> > **Conclusão Analítica:** A análise técnica demonstra que o modelo Gratuito possui um alcance de público significativamente maior, registrando uma base média de jogadores na casa de 3,1 milhões. Ao cruzar essa métrica de volume com a avaliação da crítica especializada, é possível observar um empate estatístico nas notas 72.2 para gratuitos vs. 72.1 para pagos. 

### 4.2. A Evolução dos Preços
**Pergunta:** Como o mercado de jogos de PC se comportou financeiramente ao longo dos anos? Existe uma tendência histórica de aumento no valor final dos jogos pagos?

<img width="1342" height="400" alt="Gráfico Histórico de Preço" src="https://github.com/user-attachments/assets/81278f78-786e-4073-8e5d-b01aa5b33c7c" />

> **Conclusão Analítica:** A análise histórica em série temporal filtrada entre 1997 a 2016 revela o comportamento inflacionário e de precificação da indústria. É possível observar quedas expressivas no final dos anos 90, seguidas por uma curva de ascensão que atinge o pico de preço médio 13,89 no ano de 2013, marcando a adoção de novas "tabelas de preços" pela indústria para jogos de maior orçamento.

### 4.3. Domínio de Engajamento
**Pergunta:** Quais gêneros geram a comunidade mais engajada em volume de recomendações?

<img width="1342" height="400" alt="Gráfico Engajamento por Gênero" src="https://github.com/user-attachments/assets/7dad975c-0c37-4f34-8490-3b12415c5bd5" />

> **Conclusão Analítica:** A análise técnica demonstra que os nichos tradicionais de **Ação** (com média de 7.050 recomendações/jogo) e **RPG** (com média de 5.462) dominam o engajamento da comunidade. Curiosamente, categorias com alto volume de publicações, como *Indie* e *Casual*, figuram na base do ranking de engajamento médio. A partir disso, é possível analisar o comportamento do mercado: as *Software Houses* que buscam comunidades massivas e altamente ativas tendem a focar seus investimentos em títulos de Ação e RPG.

---

## 5. Autoavaliação e Considerações Finais
A construção deste MVP foi um ótimo desafio para colocar em prática a estruturação de um pipeline completo. Para garantir a **Qualidade dos Dados e a Modelagem Dimensional**, precisei tomar algumas decisões importantes ao longo do caminho:

* **Padronização e Entendimento:** O primeiro passo foi analisar os tipos de dados originais e arrumar as nomenclaturas das colunas. Isso foi essencial para facilitar a leitura e o entendimento do negócio antes de começar a separar as tabelas.
* **Tratamento de Texto com Regex:** Ao analisar os dados para criar as tabelas dimensão, reparei que a coluna de data de lançamento estava como texto e com os formatos bagunçados. Para resolver isso, utilizei o *Regex* para manipular o texto e conseguir extrair apenas o ano exato de forma limpa.
* **Modelagem de Gêneros com Unpivot:** Também percebi que existiam várias colunas booleanas indicando o gênero do jogo. Como um mesmo jogo pode estar em mais de um gênero (ex: Ação e RPG), utilizei o *Unpivot* para separar essas informações em linhas. Eu precisava fazer isso para saber exatamente o gênero de cada jogo na hora da análise, sem que os números do painel ficassem distorcidos.

**Considerações Finais**
Este projeto também marcou a minha primeira experiência utilizando o **Databricks**. Tive que aprender a plataforma do zero, entendendo primeiro como funcionava a importação de um arquivo `.csv` para dentro do ambiente deles. Depois de conseguir ingerir o arquivo, aprendi como acessar e consultar esses dados através do notebook. Foi uma excelente oportunidade para vivenciar a criação de um pipeline de dados de ponta a ponta, desde a entrada do dado bruto até a visualização final, aprendendo na prática do mercado.

---
**Tecnologias Utilizadas:** Databricks, PySpark, SQL, Delta Lake, Unity Catalog, Markdown.
