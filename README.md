# 🚀 Data Engineering Challenge - SIAPE

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache_Spark-FFFFFF?style=for-the-badge&logo=apachespark&logoColor=#E35A16)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-00AEEF?style=for-the-badge&logo=deltalake&logoColor=white)

## Visão Geral do Projeto
Este repositório contém a solução arquitetural e o código fonte para o desafio de Engenharia de Dados. O objetivo do projeto foi ingerir, higienizar e modelar a complexa base de dados de Servidores Públicos Federais (SIAPE), transformando dados legados do governo em um **Produto de Dados** de alto valor para os times de **Modelagem de Risco** e **Políticas de Crédito**.

---

## Arquitetura e Decisões de Engenharia

O projeto foi construído sobre a arquitetura **Medallion Lakehouse** utilizando **Databricks Volumes** para governança de arquivos brutos e **Delta Lake** para as tabelas gerenciadas. O foco do desenvolvimento foi escalabilidade, resiliência e autonomia.



### 1. Extract & Load: Ingestão Autônoma e Gestão de Memória (`00_ingestion`)
* **Dinamismo Temporal (M-2):** Desenvolvi um motor matemático que identifica o dia de execução e recua estrategicamente a safra para coletar sempre a posição atual (M-2) e 3 meses de histórico, garantindo que o pipeline possa rodar agendado sem intervenção humana.
* **Prevenção de Gargalos (OOM):** Para evitar *Out of Memory* no processamento de ZIPs gigabytes do governo, implementei requisições HTTP via *Streaming* integradas ao módulo `BytesIO`, realizando a extração dos CSVs diretamente na **Memória RAM** sem onerar o disco físico do cluster.
* **Hive Partitioning Físico:** Os arquivos são roteados para subdiretórios no formato `posicao_base=YYYYMM`, pavimentando o terreno para o *Native Partition Discovery* do Spark nas próximas camadas.

### 2. Camada Bronze: Landing e Lineage (`01_bronze`)
* **Sanitização Universal de Esquemas:** Funções com RegEx varrem e higienizam caracteres inválidos nos cabeçalhos originais, impedindo que o Delta Lake corrompa por mudanças não avisadas pelo governo.
* **Data Lineage:** Uso nativo do `_metadata.file_path` do Apache Spark para carimbar a linhagem de origem do dado em cada registro (Auditoria Row-Level).

### 3. Camada Silver: Motor de Data Quality Dinâmico (`02_silver`)
Para proteger os modelos preditivos de *Garbage In, Garbage Out*, criei um Motor de DQ Universal (agnóstico ao domínio):
* **Extermínio de Magic Strings:** Identifica e anula dados fictícios governamentais (`-1`, `Sem informação`, `Não informada`), convertendo-os em verdadeiros `NULLs` relacionais, impedindo agrupamentos em "estados fantasmas" pela Inteligência Artificial.
* **Schema Evolution & Casting Rigoroso:** Varredura dinâmica de metadados. Converteu massivamente colunas financeiras (com formatação PT-BR) para `DecimalType(18,2)` e aplicou *Time Travel Parsing* para `DateType`. A trava de *Schema Enforcement* do Delta foi manipulada com segurança via `.option("overwriteSchema")`.

### 4. Camada Gold: Business Model & OBT (`03_gold`)
* **The Big Join (OBT):** Criação de uma *One Big Table* tendo a tabela de `Cadastro` como espinha dorsal, absorvendo a complexidade relacional que antes recaía sobre os Analistas e Cientistas de Dados.
* **Prevenção de Fan-out (Explosão Cartesiana):** Tabelas satélites de comportamento (como Afastamentos e Observações) foram pré-agregadas antes dos `JOINs`, criando variáveis binárias (*Flags* booleanas) que impedem a duplicação indevida da renda do servidor e são altamente otimizadas para os algoritmos de Credit Scoring.
* **Business Naming:** Renomeação semântica de todas as features para a linguagem de domínio dos *Stakeholders*.

---

## O Valor Gerado (Analytics)
Além do pipeline de engenharia, foi desenvolvido um Jupyter Notebook de Analytics (`04_metricas_gold`) consumindo diretamente a Camada Gold via SQL, provando a eficácia do modelo de dados criado. Foram respondidas questões como:
1. **Termômetro de Risco por Estabilidade:** Cruzamento da estabilidade do servidor (ex: Temporário vs. Aposentado) com a taxa mensal de afastamentos.
2. **Mapa de Calor Comercial:** Top 10 Estados com maior Massa Salarial Líquida.
3. **Radar High-Ticket:** Órgãos de origem (ex: Banco Central) que possuem o maior Ticket Médio salarial para embasar limites automáticos de Cartões Black/Premium.

---

## Governança e Contratos de Dados
Na pasta `/caatalog`, você encontrará o arquivo `.yml` da nossa tabela final. Todas as colunas possuem tipagem forte, descrições semânticas e mapeamento de dependências, prontos para integração com dbt Docs ou DataHub.

---

## Como Executar
1. Clone este repositório no seu Databricks Workspace.
2. Garanta a existência de um Volume no Unity Catalog (`/Volumes/workspace/default/staging-siape-tables`).
3. Execute os notebooks na pasta `/pipelines` em ordem sequencial (00 ao 03).
4. Para consumir os insights, abra a pasta `/analytics` e execute o notebook 04.