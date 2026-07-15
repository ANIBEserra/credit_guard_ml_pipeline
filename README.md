<p align="center">
  <b>Idiomas:</b>
  🇧🇷 Português | 
  <a href="README.en.md">🇺🇸 English</a> | 
  <a href="README.es.md">🇪🇸 Español</a>
</p>

---

Acesse o Looker Studio para visualizar o monitoramento no dashboard: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)


# Case Study: CreditGuard — Inteligência de Crédito, Engenharia e Governança de Dados

Este projeto apresenta o ciclo de vida completo de um pipeline de dados governado e automatizado para uma empresa de análise de risco financeiro, seguindo práticas reais de mercado, segurança da informação e conformidade com a LGPD.

---

## 🗺️ Sumário de Navegação

* [🏢 1. Contexto de Negócio, Desafio e Propósito](#-1-contexto-de-negócio-desafio-e-propósito)
* [📋 2. Requisitos do Projeto](#-2-requisitos-do-projeto)
* [🏗️ 3. Arquitetura Medalhão Serverless](#-3-arquitetura-medalhão-serverless)
* [📁 4. Organização do Repositório & Estrutura de Pastas](#-4-organização-do-repositório--estrutura-de-pastas)
* [🛡️ 5. Política de Governança de Dados & Gestão de Metadados](#-5-política-de-governança-de-dados--gestão-de-metadados)
* [🚀 6. Ciclo de Testes e Entrega (Do Desenvolvimento à Produção)](#-6-ciclo-de-testes-e-entrega-do-desenvolvimento-à-produção)
* [📊 7. Monitoramento do Pipeline & Insights Operacionais](#-7-monitoramento-do-pipeline--insights-operacionais)
* [📦 8. Entregáveis do Projeto](#-8-entregáveis-do-projeto)

---

## 🏢 1. Contexto de Negócio, Desafio e Propósito

A **CreditGuard** é uma fintech de inteligência de crédito especializada no mapeamento de riscos e compliance (*Know Your Customer* - KYC) para transações comerciais de pequeno e médio porte. Antes de aprovar limites financeiros ou parcerias comerciais, a fintech precisa avaliar a saúde cadastral das empresas e de seus respectivos sócios.

Em ambientes corporativos modernos, a consolidação desses dados cadastrais vindos de fontes públicas enfrenta desafios como: payloads JSON complexos e aninhados, falta de padronização, ausência de dicionários de dados claros e riscos de exposição de dados sensíveis de pessoas físicas (sócios).

### O Problema (A dor do negócio)
Antes da implementação deste projeto, a equipe de análise operacional realizava consultas manuais no portal da Receita Federal e em outras fontes públicas. Isso gerava:
* **Gargalo operacional:** Lentidão na esteira de aprovação de crédito para novos parceiros comerciais.
* **Falta de histórico consolidado:** Impossibilidade de auditar a situação cadastral de uma empresa no momento exato em que o crédito foi concedido (falta de rastreabilidade temporal).
* **Riscos de Segurança e LGPD:** Dados de pessoas físicas (CPFs e nomes de sócios) eram salvos em planilhas locais e compartilhados sem qualquer controle ou anonimização.

### A Solução & Objetivo
Desenvolver um **pipeline de dados 100% serverless e de baixo custo (FinOps)** para centralizar, estruturar e governar os dados de CNPJ e Quadro de Sócios e Administradores (QSA) consumidos via *BrasilAPI*. O pipeline foca na excelência da engenharia e governança através de:
* **Automação Eficiente:** Ingestão agendada via CI/CD sem a necessidade de manter servidores ligados 24/7.
* **Arquitetura Escalável:** Armazenamento otimizado utilizando o formato colunar `.parquet` para reduzir custos de varredura analítica.
* **Cultura de Governança:** Aplicação prática de catálogo de dados, dicionário de variáveis e técnicas de privacidade (*Privacy by Design*) diretamente na camada de entrega.

---

## 📋 2. Requisitos do Projeto

### Requisitos Funcionais (O que o pipeline entrega)
* **RF-01:** Extração automatizada e em lote dos dados cadastrais de CNPJs fornecidos como insumo de entrada.
* **RF-02:** Armazenamento seguro do payload bruto em nuvem para viabilizar reprocessamentos futuros (*replayability*).
* **RF-03:** Padronização taxonômica e limpeza dos dados de empresas e sócios, removendo caracteres especiais e aplicando tipagem estrita.
* **RF-04:** Criação de uma matriz de monitoramento contínuo para avaliar a integridade de preenchimento e completude das colunas carregadas.
* **RF-05:** Disponibilização de uma tabela central de logs para auditoria de execuções do pipeline (sucesso, tempo de processamento, erros e volumes).

### Requisitos Não Funcionais (Regras arquiteturais e técnicas)
* **RNF-01 (FinOps/Custo):** O projeto deve operar 100% sob os limites gratuitos (*Free Tier*) de serviços em nuvem (GCP) e runners de orquestração.
* **RNF-02 (Armazenamento Eficiente):** Utilização do formato colunar `.parquet` para reduzir custos de transporte no Data Lake e aumentar a performance de leitura.
* **RNF-03 (Segurança e Efimeridade):** Nenhum dado bruto de pessoas físicas (PII) deve permanecer salvo em servidores temporários ou runners locais após o término da execução.
* **RNF-04 (Automação):** Agendamento e execução isolada via pipeline de CI/CD, eliminando a dependência de infraestruturas locais ativas.

---

## 🏗️ 3. Arquitetura Medalhão Serverless

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
        └─► [ 2. Conversão &    ] ──► Google Cloud Storage (Bucket Structured / Silver)
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

### 🧱 Detalhamento Técnico das Camadas
* **Camada Bronze (Raw/JSON):** Os dados retornados da API em formato JSON bruto são salvos diretamente no Google Cloud Storage (GCS). Isso preserva a imutabilidade do dado original e permite reprocessamentos futuros (replayabilidade) sem onerar a API de origem.
* **Processamento e Otimização (.parquet):** Utilizando Python, o payload JSON é parseado, limpo e convertido para o formato colunar `.parquet`. Esta etapa reduz drasticamente o tamanho do arquivo final no Data Lake e otimiza a velocidade de leitura para as etapas analíticas.
* **Camada Silver (Structured/BigQuery):** Os arquivos `.parquet` estruturados no GCS são integrados ao Google BigQuery. Nesta camada analítica, os dados ganham tipagem rígida, schemas bem definidos e ficam organizados de forma relacional (Tabela de Empresas e Tabela de Quadro de Sócios).
* **Camada de Entrega (BI/Looker Studio):** O BigQuery expõe as tabelas diretamente para o Looker Studio, onde métricas operacionais e volumétricas são exibidas de forma fluida, sem a necessidade de processamentos complexos em tempo de execução no painel.

---

## 📁 4. Organização do Repositório & Estrutura de Pastas

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

### ⚙️ Organização do Fluxo & Gerenciamento de Arquivos Temporários
* **Insumo de Entrada (`data/input/cnpjs.csv`):** Arquivo estático contendo a relação de CNPJs que o pipeline deve processar a cada execução.
* **Ciclo de Vida Local e Efemeridade:** Para garantir a resiliência e o isolamento de etapas, o script cria as pastas `data/raw/` e `data/silver/` em tempo de execução para apoiar o processamento local. No entanto, ao final do pipeline, a função `clean_local_temp_files` expurga esses diretórios. Isso garante que o runner do GitHub Actions não acumule lixo eletrônico e atenda a boas práticas de segurança de dados.
* **Isolamento de Regras (`configs/`):** Centraliza os mapeamentos e renomeações de colunas. Caso a API de origem altere o nome de algum campo, a manutenção é feita apenas neste arquivo, mantendo o script principal intacto.
* **Exploratório (`notebooks/`):** Espaço focado no desenvolvimento incremental, testes de conexões e prototipação das transformações analíticas antes de integrá-las ao código de produção.

---

## 🛡️ 5. Política de Governança de Dados & Gestão de Metadados

Para garantir que a CreditGuard opere de acordo com padrões éticos, legais e regulatórios, o pipeline segue as diretrizes estruturadas do nosso framework de governança:

```text
┌───────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   Propósito   │ ───► │    Escopo     │ ───► │    Papéis     │ ───► │  Diretrizes   │ ───► │ Conformidade  │
│ Assegurar LGPD│      │ Todo o fluxo  │      │ Eng/Gov/Risk  │      │  PII/Efêmero  │      │ CI-CD/Logs/DQ │
└───────────────┘      └───────────────┘      └───────────────┘      └───────────────┘      └───────────────┘
```

### 🟢 1. Propósito
Garantir a integridade, rastreabilidade e privacidade dos dados de pessoas jurídicas durante todo o ciclo de vida da informação no pipeline da CreditGuard. Esta política existe para mitigar riscos de vazamento de dados confidenciais (em conformidade com a LGPD) e assegurar que as decisões de análise de crédito sejam tomadas com base em dados de qualidade e auditáveis.

### 🔵 2. Escopo
Esta política aplica-se de forma obrigatória a todas as etapas e ativos de dados envolvidos no projeto:
* Arquivos de entrada estáticos de CNPJs (`cnpjs.csv`).
* Ambiente de execução temporário (Runner local ou GitHub Actions).
* Camadas física e lógica na nuvem (Google Cloud Storage e Google BigQuery).
* Painéis visuais de consumo analítico (Looker Studio).

### 🔵 3. Papéis e Responsabilidades
* **Engenharia de Dados (DE):** Responsável por implementar e manter a segurança técnica do pipeline, garantir a limpeza de arquivos locais, o versionamento do código e o provisionamento correto de acessos via Service Accounts no GCP.
* **Analista de Governança de Dados (DG):** Responsável por definir as regras de classificação de dados, validar as políticas de privacidade (máscaras de CPF), gerenciar o dicionário de dados de metadados e monitorar os indicadores de qualidade (completude).
* **Analistas de Negócio / Risco de Crédito:** Consumidores finais das tabelas tratadas na camada Silver e dos dashboards. Não possuem privilégios de escrita nos bancos de dados e devem consumir os dados de pessoas físicas de forma estritamente enmascarada.

### 🔵 4. Diretrizes
* **Diretriz de Efimeridade (Descarte):** Os arquivos locais criados nas pastas `data/raw/` e `data/silver/` devem ser eliminados pelo script (`clean_local_temp_files`) imediatamente após o upload bem-sucedido para o Cloud Storage.
* **Diretriz de Nomenclatura (Metadados):** O mapeamento de colunas no arquivo `configs/mapping.py` deve adotar o padrão de redução taxonômica inspirado no DATASUS (ex: `NRCNPJ`, `NMRAZSOC`), padronizando o dicionário de dados corporativo.
* **Diretriz de Privacidade (LGPD):** Dados pessoais identificáveis (PII) dos sócios, como o CPF parcial, devem receber tratamento de mascaramento na camada de exibição analítica.
* **Diretriz de Qualidade de Dados:** O preenchimento das variáveis de identificação essenciais (`NRCNPJ` e `CDCNAEFISC`) deve ser mantido em 100%. A qualidade de preenchimento das demais variáveis deve ser auditada e monitorada visualmente através de uma View de Data Quality dinâmica no BigQuery.

### 🔵 5. Conformidade e Exceções
* **Bloqueio em CI/CD:** Scripts que exponham chaves de segurança (GCP Credentials JSON) ou que não contenham o método de exclusão automática de arquivos temporários serão sumariamente bloqueados durante a etapa de pull-request no GitHub Actions.
* **Tratamento de Exceções Operacionais:** Se a taxa de sucesso das requisições na API cair abaixo do limite crítico de 90%, o pipeline registrará o incidente como um log de alerta na tabela de logs. A equipe de Governança deve ser notificada para auditar eventuais alterações na estrutura de retorno da API de origem.

---

### ⏳ 6. Matriz de Conformidade e Classificação de Dados

| Domínio dos Dados | Atributos Principais | Finalidade de Uso | Base Legal (LGPD) | Classificação de Segurança |
| :--- | :--- | :--- | :--- | :--- |
| **Cadastral PJ** | `NRCNPJ`, `NMRAZSOC`, `NMFANT`, `VLCPTSOC`, `IDMTZFIL`, `DSIDMTZFIL` | Centralização, higienização e enriquecimento cadastral de pessoas jurídicas para mitigação de riscos operacionais, prevenção a fraudes (KYC) e validação de dados em ecossistemas de negócios. | **Legítimo Interesse**<br>(Art. 7º, IX)<br><br>**Cumprimento de Obrigação Legal ou Regulatória**<br>(Art. 7º, II - Conformidade com PLD/Compliance) | **Público**<br>(Dados institucionais de registro público ostensivo na Receita Federal) |
| **QSA** *(Quadro de Sócios)* | Nome do Sócio, CPF (mascarado/parcial), Qualificação do Sócio | Avaliação de vínculos societários, identificação de beneficiários finais e geração de dashboards analíticos de monitoramento no Looker Studio. | **Legítimo Interesse**<br>(Art. 7º, IX - Proteção do próprio negócio)<br><br>**Cumprimento de Obrigação Legal ou Regulatória**<br>(Art. 7º, II - Normas de Compliance e Governança) | **Pessoal / PII**<br>(Dados de pessoas físicas. Exige mascaramento dinâmico na camada de visualização e controle de efemeridade com descarte local via `clean_local_temp_files`). |

---

### 🔤 7. Padronização de Nomenclatura (Padrão DATASUS)
O dicionário de colunas no arquivo `configs/mapping.py` adota a metodologia de compressão de vogais e prefixos taxonômicos inspirada no DATASUS. O de/para mapeia explicitamente os tipos e aliases curtos:
* `cnpj` ➔ `NRCNPJ` (Número CNPJ)
* `razao_social` ➔ `NMRAZSOC` (Nome Razão Social)
* `nome_fantasia` ➔ `NMFANT` (Nome Fantasia)
* `capital_social` ➔ `VLCPTSOC` (Valor Capital Social)
* `identificador_matriz_filial` ➔ `IDMTZFIL` (Identificador Matriz Filial)
* `descricao_identificador_matriz_filial` ➔ `DSIDMTZFIL` (Descrição Identificador Matriz Filial)

Essa abordagem reduz o ruído dos metadados, economiza bytes de armazenamento de colunas, padroniza as chaves lógicas do Data Warehouse e assegura que a modelagem relacional seja limpa e previsível para qualquer analista da organização.

---

## 🚀 6. Ciclo de Testes e Entrega

### Como Testamos
* **Validação Local:** Testes funcionais unitários utilizando uma amostra reduzida de CNPJs para certificar que o script `main.py` consegue realizar a paginação e o consumo da API sem quebras.
* **Mapeamento de Esquemas:** O arquivo de configuração `configs/mapping.py` atua como o nosso "contrato de dados", forçando a conversão correta de tipos de dados (tipagem explícita) antes do envio ao BigQuery.
* **Auditoria Operacional de Logs:** O comportamento do pipeline é validado inspecionando a tabela de logs localmente antes de promover o código à produção, checando se os campos de tempo (`EXECUTION_TIME_SECONDS`) e status estão sendo computados corretamente.

### Como Enviamos para Produção (CI/CD)
* **Ambiente Serverless:** O pipeline é totalmente implantado via **GitHub Actions**.
* **Proteção de Chaves (Secrets):** Nenhuma credencial de infraestrutura é salva no repositório. As credenciais de produção do GCP estão criptografadas e são injetadas em tempo de execução através das chaves de segurança secretas do repositório (GitHub Secrets).
* **Orquestração por Cron:** O agendador automatizado roda o script de forma programada em janelas de menor atividade, atualizando as bases cadastrais sem qualquer necessidade de gatilho manual.

---

## 📊 7. Monitoramento do Pipeline & Insights Operacionais

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

## 📦 8. Entregáveis do Projeto

Ao final da implantação, a CreditGuard possui um ambiente de dados moderno constituído por:
1. **Camada Bronze Física (GCS):** Armazenamento em arquivos JSON de segurança, garantindo a imutabilidade dos dados de entrada históricos.
2. **Camada Silver Analítica (BigQuery):** Dados normalizados, higienizados e indexados de forma relacional entre Empresas e Sócios.
3. **Tabela de Logs Consolidados (BigQuery):** Visibilidade ponta a ponta sobre o consumo de cota da API, taxas de sucesso de execução e tempo de resposta do pipeline.
4. **View de Qualidade de Dados (Metadata DQ View):** Uma tabela dinâmica que calcula em tempo real o percentual de completude de cada uma das colunas para auditoria de integridade.
5. **Dashboard Looker Studio:** Um cockpit unificado apresentando a volumetria diária, monitoramento de falhas, métricas de FinOps e visualizações geográficas da base cadastral.