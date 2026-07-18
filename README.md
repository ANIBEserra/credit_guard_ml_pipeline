<p align="center">
  <b>Idiomas:</b>
  🇧🇷 Português | 
  <a href="README.en.md">🇺🇸 English</a> | 
  <a href="README.es.md">🇪🇸 Español</a>
</p>

> 🎓 **Nota de Desenvolvimento:** Este projeto foi desenhado e implementado como aplicação prática dos conhecimentos adquiridos na **Formação de Governança de Dados da Alura**. O objetivo é consolidar conceitos estruturados de qualidade de dados, privacidade e gestão de metadados em um cenário simulado de uma empresa data driven.

---

Acesse o Looker Studio para visualizar o Portal de Monitoramento: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)


# Case Study: CreditGuard — Pipeline de Engenharia de Dados Governado

Este projeto apresenta o ciclo de vida completo de um pipeline de dados governado e automatizado para uma empresa de análise de risco financeiro, seguindo práticas reais de mercado, segurança da informação e conformidade com a LGPD.

---

## 🗺️ Sumário de Navegação

* [1. Contexto de Negócio, Desafio e Propósito](#1-contexto-de-negócio-desafio-e-propósito)
* [2. Requisitos do Projeto](#2-requisitos-do-projeto)
* [3. Arquitetura Medalhão Serverless](#3-arquitetura-medalhão-serverless)
* [4. Organização do Repositório & Estrutura de Pastas](#4-organização-do-repositório--estrutura-de-pastas)
* [5. Política de Governança de Dados & Gestão de Metadados](#5-política-de-governança-de-dados--gestão-de-metadados)
* [6. Ciclo de Testes e Entrega](#6-ciclo-de-testes-e-entrega)
* [7. Monitoramento do Pipeline & Insights Operacionais](#7-monitoramento-do-pipeline--insights-operacionais)
* [8. Entregáveis do Projeto](#8-entregáveis-do-projeto)

---

## 1. Contexto de Negócio, Desafio e Propósito

A **CreditGuard** é uma fintech de inteligência de crédito especializada na avaliação de riscos e compliance (*Know Your Customer* - KYC) para transações comerciais B2B de pequeno e médio porte. 

No ecossistema de crédito, a velocidade e o custo de aquisição de informações são fatores inerentes. Para conceder limites financeiros, a fintech consome dados de **bureaus de crédito pagos** (como Serasa). No entanto, realizar chamadas em APIs pagas para absolutamente todas as propostas recebidas gera um custo operacional (exemplo: Custo de Aquisição de Cliente - CAC).

### Dor do negócio
Antes da implementação deste projeto, a CreditGuard sofria com três grandes dores em sua esteira de aprovação de crédito:
* **Custos e Ineficiência:** A fintech gastava recursos excessivos consultando bureaus pagos para empresas que já poderiam ser sumariamente reprovadas por estarem inativas na Receita Federal ou possuírem CNAEs fora da política de crédito.
* **Gargalo Sistêmico e Operacional:** Consultas pontuais a fontes públicas eram feitas de forma fragmentada e manual pela equipe de risco, impedindo que a aprovação de crédito fosse feita de forma 100% automatizada por um Motor de Regras.
* **Falta de Auditoria e Rastreabilidade de QSA:** O quadro societário não era armazenado de forma relacional. Isso impedia análises de risco complexas (ex: identificar automaticamente se o sócio da empresa atual já faliu uma empresa anterior na base da fintech) e impossibilitava auditar a situação exata da empresa no dia em que o crédito foi concedido.

### A Solução & Objetivo
Para resolver essa dor, foi desenvolvido um **pipeline de dados 100% serverless e governado**, que atua como uma **camada primária de enriquecimento de dados (Tier 1)** utilizando fontes públicas gratuitas (BrasilAPI). 

Este pipeline intercepta e processa os dados de CNPJ e Quadro de Sócios (QSA) antes que as chamadas a bureaus pagos sejam necessárias, focando em:
* **Estratégia FinOps:** A ingestão automatizada e estruturada permite que o motor de decisão da fintech negue propostas inviáveis a custo zero, utilizando dados públicos, acionando APIs pagas apenas para os clientes que passam no primeiro filtro.
* **Engenharia para Decisão em Escala:** Armazenamento otimizado no formato colunar `.parquet` e consolidação no BigQuery (Data Warehouse), permitindo que modelos de Machine Learning e painéis de risco consumam milhares de registros instantaneamente.
* **Cultura de Governança e LGPD:** Aplicação prática de *Privacy by Design*. Dados pessoais de sócios (PII) são mascarados na camada de entrega, garantindo compliance total com as leis de proteção de dados, mantendo um dicionário de variáveis padronizado.

---

## 2. Requisitos do Projeto

### Requisitos Funcionais (O que o pipeline entrega)
* **RF-01:** Extração automatizada e em lote dos dados cadastrais de CNPJs fornecidos como insumo de entrada.
* **RF-02:** Armazenamento seguro do payload bruto em nuvem para viabilizar reprocessamentos futuros.
* **RF-03:** Padronização taxonômica e limpeza dos dados de empresas e sócios, removendo caracteres especiais e aplicando tipagem estrita.
* **RF-04:** Criação de uma matriz de monitoramento contínuo para avaliar a integridade de preenchimento e completude das colunas carregadas.
* **RF-05:** Disponibilização de uma tabela central de logs para auditoria de execuções do pipeline (sucesso, tempo de processamento, erros e volumes).

### Requisitos Não Funcionais (Regras arquiteturais e técnicas)
* **RNF-01 (FinOps/Custo):** O projeto deve operar 100% sob os limites gratuitos (*Free Tier*) de serviços em nuvem (GCP) e runners de orquestração.
* **RNF-02 (Armazenamento Eficiente):** Utilização do formato colunar `.parquet` para reduzir custos de transporte no Data Lake e aumentar a performance de leitura.
* **RNF-03 (Segurança):** Nenhum dado bruto de pessoas físicas (PII) deve permanecer salvo em servidores temporários ou runners locais após o término da execução.
* **RNF-04 (Automação):** Agendamento e execução isolada via pipeline de CI/CD, eliminando a dependência de infraestruturas locais ativas.

---

## 3. Arquitetura Medalhão Serverless

A arquitetura do projeto foi desenhada seguindo os princípios de desacoplamento de camadas (Medalhão) e computação em nuvem gerenciada. O fluxo garante o processamento eficiente de dados de forma assíncrona, focando na otimização de custos de armazenamento e consulta.

```text
  [ BrasilAPI ]
        │ (Request Assíncrona / Python)
        ▼
 ┌────────────────────────────────────────┐
 │            GitHub Actions              │ ◄── [ Gatilho Cron / CI/CD ]
 └────────────────────────────────────────┘
        │
        ├─► [ 1. Extração JSON ] ──► Google Cloud Storage (Bucket Raw / Bronze)
        │
        └─► [ 2. Transformação &    ] ──► Google Cloud Storage (Bucket Structured / Silver)
             Processamento .parquet
        │
        ▼
 ┌────────────────────────────────────────┐
 │        Google BigQuery (OLAP)          │ ◄── [ Armazenamento de Tabelas Silver ]
 └────────────────────────────────────────┘
        │
        ▼
 ┌────────────────────────────────────────┐
 │         Looker Studio (BI)             │ ◄── [ Entrega e Mascaramento de Dados ]
 └────────────────────────────────────────┘
```

### Detalhamento Técnico das Camadas
* **Camada Bronze (Raw/JSON):** Os dados retornados da API em formato JSON bruto são salvos diretamente no Google Cloud Storage (GCS). Isso preserva a imutabilidade do dado original e permite reprocessamentos futuros (replayabilidade) sem onerar a API de origem.
* **Processamento e Otimização (.parquet):** Utilizando Python, o payload JSON é parseado, limpo e convertido para o formato colunar `.parquet`. Esta etapa reduz drasticamente o tamanho do arquivo final no Data Lake e otimiza a velocidade de leitura para as etapas analíticas.
* **Camada Silver (Structured/BigQuery):** Os arquivos `.parquet` estruturados no GCS são integrados ao Google BigQuery. Nesta camada analítica, os dados ganham tipagem rígida, schemas bem definidos e ficam organizados de forma relacional (Tabela de Empresas e Tabela de Quadro de Sócios).
* **Camada de Entrega (BI/Looker Studio):** O BigQuery expõe as tabelas diretamente para o Looker Studio, onde métricas operacionais e volumétricas são exibidas de forma fluida, sem a necessidade de processamentos complexos em tempo de execução no painel.

---

## 4. Organização do Repositório & Estrutura de Pastas

O projeto foi estruturado de forma prática e organizada, separando os scripts de execução, as configurações de metadados e as documentações de suporte.

```text
├── configs/
│   ├── __init__.py
│   └── mapping.py           # Dicionários de mapeamento (De/Para) e renomeação de colunas
├── data/
│   └── input/
│       └── cnpjs.csv        # Lista inicial de CNPJs que serve de insumo para a busca na API
│   ├── raw/                 # [Dinâmico] Criado na execução para armazenar temporariamente os arquivos JSON brutos
│   └── silver/              # [Dinâmico] Criado na execução para armazenar temporariamente os arquivos .parquet
├── docs/                    # Documentações auxiliares e dicionários de dados do projeto
├── notebooks/
│   ├── analyse_silver.ipynb # Análise e validação dos dados estruturados na camada Silver
│   └── brasil-api.ipynb     # Jupyter Notebook utilizado para exploração inicial da BrasilAPI
├── scripts/
│   └── main.py              # Script principal contendo as funções de extração, limpeza e carga na GCP
├── .gitignore               # Definição de arquivos e credenciais locais que não devem ser versionados
├── README.md                # Documentação completa do ecossistema
└── requirements.txt         # Dependências do projeto (pandas, pyarrow, google-cloud-storage, etc.)
```

### Organização do Fluxo & Gerenciamento de Arquivos Temporários
* **Insumo de Entrada (`data/input/cnpjs.csv`):** Arquivo estático contendo a relação de CNPJs que o pipeline deve processar a cada execução.
* **Ciclo de Vida Local e Efemeridade:** Para garantir a resiliência e o isolamento de etapas, o script cria as pastas `data/raw/` e `data/silver/` em tempo de execução para apoiar o processamento local. No entanto, ao final do pipeline, a função `clean_local_temp_files` expurga esses diretórios. Isso garante que o runner do GitHub Actions não acumule lixo eletrônico e atenda a boas práticas de segurança de dados.
* **Isolamento de Regras (`configs/`):** Centraliza os mapeamentos e renomeações de colunas. Caso a API de origem altere o nome de algum campo, a manutenção é feita apenas neste arquivo, mantendo o script principal intacto.
* **Exploratório (`notebooks/`):** Espaço focado no desenvolvimento incremental, testes de conexões e prototipação das transformações analíticas antes de integrá-las ao código de produção.

---

## 5. Política de Governança de Dados & Gestão de Metadados

Para garantir que a CreditGuard opere de acordo com padrões éticos, legais e regulatórios, o pipeline segue as diretrizes estruturadas do nosso framework de governança:

```text
┌───────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   Propósito   │ ───► │    Escopo     │ ───► │    Papéis     │ ───► │  Diretrizes   │ ───► │ Conformidade  │
│ Assegurar LGPD│      │ Todo o fluxo  │      │ Eng/Gov/Risk  │      │  PII/Efêmero  │      │ CI-CD/Logs/DQ │
└───────────────┘      └───────────────┘      └───────────────┘      └───────────────┘      └───────────────┘
```
<details>
<summary>📌 <b>Clique aqui para expandir os papéis de governança</b></summary>

<br>

<details>
<summary><font size="3"><b> 5.1. Propósito</b></font></summary>

Garantir a integridade e rastreabilidade durante todo o ciclo de vida da informação no pipeline da CreditGuard. Esta política existe para assegurar que as decisões de análise de crédito sejam tomadas com base em dados de qualidade e auditáveis.
</details>

<details>
<summary><font size="3"><b> 5.2. Escopo</b></font></summary>

Esta política aplica-se de forma obrigatória a todas as etapas e ativos de dados envolvidos no projeto:
* Arquivos de entrada estáticos de CNPJs (`cnpjs.csv`).
* Ambiente de execução temporário (Runner local ou GitHub Actions).
* Camadas física e lógica na nuvem (Google Cloud Storage e Google BigQuery).
* Painéis visuais de consumo analítico (Looker Studio).
</details>

<details>
<summary><font size="3"><b> 5.3. Papéis e Responsabilidades</b></font></summary>

A estrutura de responsabilidades do projeto segue o modelo de governança descentralizada, separando a custódia técnica, a facilitação metodológica e a propriedade de negócio dos dados:

* **Analista de Governança de Dados:**
  * Define o framework de governança, padrões de nomenclatura e regras gerais de classificação de segurança.
  * Facilita a criação e gerencia a sustentação do Catálogo de Dados, Inventário e Glossário no Looker Studio.
  * Formaliza Data Owner, Data Steward e Data Custodian, garantindo que os papéis de negócio estejam claramente definidos e documentados.
  * Apoia o Data Steward na definição de métricas de qualidade de dados (DQ) e monitora a conformidade do processo.

* **Engenharia de Dados (Data Custodian):**
  * Implementa e mantém a infraestrutura técnica do pipeline (CI/CD, scripts em Python e cargas no GCP).
  * Executa os procedimentos automáticos de expurgo e mascaramento técnico de dados locais e na nuvem.
  * Garante a estabilidade, a performance operacional do fluxo e a aplicação das chaves de segurança de forma protegida.

* **Head/Gestor de Risco de Crédito (Data Owner):**
  * Assume a responsabilidade final pelo ciclo de vida, conformidade legal (LGPD) e riscos financeiros associados ao uso dos dados de CNPJ e Sócios.
  * Define os níveis de criticidade dos dados e aprova o compartilhamento ou novos usos da base para modelos de crédito.
  * Estipula o apetite a risco em relação à qualidade da informação (ex: se o pipeline pode ou não continuar operando caso faltem determinados dados opcionais da API).

* **Analista de Risco Sênior (Data Steward):**
  * Atua como a referência de negócio para definir e documentar os conceitos das variáveis no Glossário de Dados.
  * Homologa e valida se as regras de transformação e limpeza criadas pela engenharia refletem fielmente as regras de política de crédito da empresa.
  * Define os critérios de aceitação para a qualidade dos dados (ex: quais CNAEs são impeditivos) e consome os alertas de desvios gerados pelo monitoramento.

* **Analistas de Risco e Modelagem (Data Consumers):**
  * Consomem as tabelas tratadas na camada Silver e os dashboards operacionais para análises diárias e tomada de decisão.
  * Utilizam os dados de pessoas físicas (sócios) de forma estritamente enmascarada na camada visual, reportando qualquer inconsistência ou suspeita de vazamento de informações à equipe de Governança.
</details>

<details>
<summary><font size="3"><b> 5.4. Diretrizes</b></font></summary>

* **Diretriz de Descarte:** Os arquivos locais criados nas pastas `data/raw/` e `data/silver/` devem ser eliminados pelo script (`clean_local_temp_files`) imediatamente após o upload bem-sucedido para o Cloud Storage.
* **Diretriz de Nomenclatura (Metadados):** O mapeamento de colunas no arquivo `configs/mapping.py` deve adotar o padrão de redução taxonômica inspirado no DATASUS (ex: `NRCNPJ`, `NMRAZSOC`), padronizando o dicionário de dados corporativo.
* **Diretriz de Privacidade (LGPD):** Dados pessoais identificáveis (PII) dos sócios, como o CPF parcial, devem receber tratamento de mascaramento na camada de exibição analítica.
* **Diretriz de Qualidade de Dados:** O preenchimento das variáveis de identificação essenciais (`NRCNPJ` e `CDCNAEFISC`) deve ser mantido em 100%. A qualidade de preenchimento das demais variáveis deve ser auditada e monitorada visualmente através de uma View de Data Quality dinâmica no BigQuery.
</details>

<details>
<summary><font size="3"><b> 5.5. Conformidade, Barreiras e Exceções</b></font></summary>

Para garantir a resiliência operacional e a segurança jurídica da esteira de crédito, o pipeline implementa controles automáticos divididos em três camadas:

* **1. Controles Preventivos (Esteira de CI/CD)***🚧 Status: Em desenvolvimento*

* **2. Controles Detectivos (Qualidade e SLAs em Produção)***🚧 Status: Em desenvolvimento*

* **3. Regras de Contingência e Continuidade de Negócio (Fallback)***🚧 Status: Em desenvolvimento*
</details>

<details>
<summary><font size="3"><b> 5.6. Gestão de Metadados: Inventário, Catálogo e Glossário</b></font></summary>

Para garantir a escalabilidade da governança e facilitar a usabilidade (*Data Discovery*) tanto para perfis técnicos quanto de negócio, a gestão detalhada dos metadados está centralizada em um **Portal Interativo no Looker Studio**.

O ambiente foi desenhado para oferecer uma navegação fluida em diferentes níveis de granularidade:

* **🌐 Visão de Inventário de Dados (Macro):** Uma visão gerencial focada no ciclo de vida e na conformidade do ativo de dados. Apresenta informações a nível de *dataset* e tabela, documentando a frequência de atualização, responsáveis (*Data Owner* e *Data Steward*), descrições gerais, além do endquadramento LGPD (base legal, finalidade de uso e classificação de confidencialidade).
* **🔍 Visão de Catálogo de Dados (Micro):** Acessada através da coluna interativa de "Detalhes" no Inventário, esta página faz o *drill-down* para o nível físico. Ela detalha cada campo (coluna), especificando a tipagem do dado, tamanho e eventuais regras de transformação ou mascaramento aplicadas durante o pipeline, sem perder o contexto de negócio da tabela pai.
* **📚 Glossário de Dados:** Acessível via link fixo no canto superior direito do painel, garantindo que o vocabulário de negócio e as métricas organizacionais estejam sempre a um clique de distância durante a exploração.

🔗 **[Acesse o Portal de Governança e Catálogo de Dados aqui](inserir-link-do-looker-aqui)** *🚧 Status: Em desenvolvimento*
</details>

</details>

---

## 6. Ciclo de Testes e Entrega

### Como Testamos
* **Validação Local:** Testes funcionais unitários utilizando uma amostra reduzida de CNPJs para certificar que o script `main.py` consegue realizar a paginação e o consumo da API sem quebras.
* **Mapeamento de Esquemas:** O arquivo de configuração `configs/mapping.py` atua como o nosso "contrato de dados", forçando a conversão correta de tipos de dados (tipagem explícita) antes do envio ao BigQuery.

### Como Enviamos para Produção (CI/CD)
* **Ambiente Serverless:** O pipeline é totalmente implantado via **GitHub Actions**.
* **Proteção de Chaves (Secrets):** Nenhuma credencial de infraestrutura é salva no repositório. As credenciais de produção do GCP estão criptografadas e são injetadas em tempo de execução através das chaves de segurança secretas do repositório (GitHub Secrets).
* **Orquestração por Cron:** O agendador automatizado roda o script de forma programada em janelas de menor atividade, atualizando as bases cadastrais sem qualquer necessidade de gatilho manual.

---

## 7. Monitoramento do Pipeline & Insights Operacionais

O estágio final do ecossistema consiste em expor os dados processados e estruturados da camada Silver (BigQuery) para o Looker Studio, onde o fluxo de ingestão e a volumetria empresarial são monitorados visualmente.

### 📈 Métricas de Monitoramento e Observabilidade
O painel técnico consome diretamente a tabela de logs gerada pelo pipeline, permitindo o acompanhamento de indicadores vitais de DataOps:
* **Taxa de Sucesso da Ingestão:** Percentual de requisições enviadas para a BrasilAPI que retornaram com sucesso (Status 200) vs. falhas cadastrais (CNPJs inválidos ou inexistentes).
* **Volumetria:** Monitoramento do volume de linhas injetadas por execução para garantir que não haja quedas drásticas ou anomalias no fluxo de processamento.
* **Métricas de FinOps:** Rastreabilidade do tempo de execução e volume de dados trafegados, ajudando a garantir o funcionamento do pipeline dentro do limite de gratuidade (Free Tier) da GCP.

### 💡 Extração de Insights Cadastrais Básicos
Além do monitoramento do pipeline, o painel consolida visões gerais sobre o portfólio de empresas processadas para fornecer inteligência de negócios:
* **Distribuição Geográfica:** Concentração das empresas ativas no território nacional por UF e município (utilizando o campo `cd_municipio` mapeado).
* **Análise de Porte e Capital:** Agrupamento do volume financeiro das organizações com base na distribuição do campo `VLCPTSOC` (Capital Social).
* **Visão de Quadro Societário:** Mapeamento quantitativo do número de sócios por organização, permitindo entender a densidade de vínculos societários no ecossistema consultado.

---

## 8. Entregáveis do Projeto

Ao final da implantação, a CreditGuard possui um ambiente de dados moderno constituído por:
1. **Camada Bronze Física (GCS):** Armazenamento em arquivos JSON de segurança, garantindo a imutabilidade dos dados de entrada históricos.
2. **Camada Silver Analítica (BigQuery):** Dados normalizados, higienizados e indexados de forma relacional entre Empresas e Sócios. *🚧 Status: Em desenvolvimento*
3. **Tabela de Logs Consolidados (BigQuery):** Visibilidade ponta a ponta sobre o consumo de cota da API, taxas de sucesso de execução e tempo de resposta do pipeline.
4. **View de Qualidade de Dados (Metadata DQ View):** Uma tabela dinâmica que calcula em tempo real o percentual de completude de cada uma das colunas para auditoria de integridade. *🚧 Status: Em desenvolvimento*
5. **Dashboard Looker Studio:** Um cockpit unificado apresentando a volumetria diária, monitoramento de falhas, métricas de FinOps e resumo de dados da base cadastral.