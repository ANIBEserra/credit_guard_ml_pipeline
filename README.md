<p align="right">
  <b>Idiomas:</b>
  🇧🇷 Português | 
  <a href="README.en.md">🇺🇸 English</a> | 
  <a href="README.es.md">🇪🇸 Español</a>
</p>

---

Acesse o Looker Studio para visualizar o dashboard monitoring: [Credit Guard Pipeline](https://datastudio.google.com/s/oUAEZgpbSvY)

# Pipeline Serverless de Ingestão e Governança de Dados Cadastrais CNPJ (GCP & GitHub Actions)

## 🗺️ Sumário de Navegação

* [📌 1. Propósito do Projeto & Visão de Negócio](#-1-propósito-do-projeto--visão-de-negócio)
* [🏗️ 2. Arquitetura Medalhão Serverless](#️-2-arquitetura-medalhão-serverless)
* [📁 3. Organização do Repositório & Estrutura de Pastas](#-3-organização-do-repositório--estrutura-de-pastas)
* [🛡️ 4. Governança de Dados & Gestão de Metadados](#️-4-governança-de-dados--gestão-de-metadados)
* [📊 5. Monitoramento do Pipeline & Insights Operacionais](#-5-monitoramento-do-pipeline--insights-operacionais)

---

## 📌 1. Propósito do Projeto & Visão de Negócio

Em ambientes corporativos modernos, a consolidação de dados cadastrais vindos de fontes externas públicas (como a Receita Federal) costuma enfrentar desafios crônicos: payloads JSON complexos e aninhados, falta de padronização, ausência de dicionários de dados claros e riscos de exposição de dados sensíveis de pessoas físicas (sócios).

O objetivo deste projeto foi construir um **pipeline de dados 100% serverless e de baixo custo (FinOps)** para centralizar, estruturar e governar os dados de CNPJ e Quadro de Sócios e Administradores (QSA) consumidos via *BrasilAPI*. 

O projeto foca na excelência da **Engenharia e Governança de Dados**, demonstrando:
* **Automação Eficiente:** Ingestão agendada via CI/CD sem a necessidade de manter servidores próprios ligados 24/7.
* **Arquitetura Escalável:** Armazenamento otimizado utilizando o formato colunar `.parquet` para reduzir custos de varredura analítica.
* **Cultura de Governança:** Aplicação prática de catálogo de dados, dicionário de variáveis e técnicas de privacidade (*Privacy by Design*) diretamente na camada de entrega.

## 🏗️ 2. Arquitetura Medalhão Serverless

A arquitetura do projeto foi desenhada seguindo os princípios de desacoplamento de camadas (Medalhão) e computação em nuvem gerenciada. O fluxo garante o processamento eficiente de grandes volumes de dados de forma assíncrona, focando na otimização de custos de armazenamento e consulta.

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
        └─► [ 2. Conversão &   ] ──► Google Cloud Storage (Bucket Structured / Silver)
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

* **Camada Bronze (Raw/JSON):** Os dados retornados da API em formato JSON bruto são salvos diretamente no **Google Cloud Storage (GCS)**. Isso preserva a imutabilidade do dado original e permite reprocessamentos futuros (*replayabilidade*) sem onerar a API de origem.
* **Processamento e Otimização (.parquet):** Utilizando **Python**, o payload JSON é parseado, limpo e convertido para o formato colunar **.parquet**. Esta etapa reduz drasticamente o tamanho do arquivo final no Data Lake e otimiza a velocidade de leitura para as etapas analíticas.
* **Camada Silver (Structured/BigQuery):** Os arquivos `.parquet` estruturados no GCS são integrados ao **Google BigQuery**. Nesta camada analítica, os dados ganham tipagem rígida, schemas bem definidos e ficam organizados de forma relacional (Tabela de Empresas e Tabela de Quadro de Sócios).
* **Camada de Entrega (BI/Looker Studio):** O BigQuery expõe as tabelas diretamente para o **Looker Studio**, onde métricas operacionais e volumétricas são exibidas de forma fluida, sem a necessidade de processamentos complexos em tempo de execução no painel.


## 📁 3. Organização do Repositório & Estrutura de Pastas

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

## 🛡️ 4. Governança de Dados & Gestão de Metadados

A governança deste pipeline foi desenhada para garantir a rastreabilidade e o controle sobre o ciclo de vida do dado.

### ⏳ 1. Ciclo de Vida dos Dados (Data Lifecycle Management)
O pipeline adota uma política de persistência para garantir a conformidade com a LGPD e boas práticas de governança de dados:
* **Ingestão e Descarte Local:** Os arquivos gerados nas pastas `data/raw/` e `data/silver/` existem apenas durante a execução do script no runner e são expurgados imediatamente após a carga em nuvem.
* **Persistência em Camadas (GCP):** O dado bruto permanece imutável no Cloud Storage (Bronze) para fins de auditoria, enquanto a camada analítica (Silver) no BigQuery é atualizada para consumo, garantindo um histórico limpo e auditável.

### 📋 2. Matriz de Conformidade e Classificação de Dados
| Domínio dos Dados | Atributos Principais | Finalidade de Uso | Base Legal (LGPD) | Classificação de Segurança |
| :--- | :--- | :--- | :--- | :--- |
| **Cadastral PJ** | `NRCNPJ`, `NMRAZSOC`, `NMFANT`, `VLCPTSOC`, `IDMTZFIL`, `DSIDMTZFIL` | Centralização, higienização e enriquecimento cadastral de pessoas jurídicas para mitigação de riscos operacionais, prevenção a fraudes (KYC) e validação de dados em ecossistemas de negócios. | **Legítimo Interesse**<br>(Art. 7º, IX)<br><br>**Cumprimento de Obrigação Legal ou Regulatória**<br>(Art. 7º, II - Conformidade com PLD/Compliance) | **Público**<br>(Dados institucionais de registro público ostensivo na Receita Federal) |
| **QSA** *(Quadro de Sócios)* | Nome do Sócio, CPF (mascarado/parcial), Qualificação do Sócio | Avaliação de vínculos societários, identificação de beneficiários finais e geração de dashboards analíticos de monitoramento no Looker Studio. | **Legítimo Interesse**<br>(Art. 7º, IX - Proteção do próprio negócio)<br><br>**Cumprimento de Obrigação Legal ou Regulatória**<br>(Art. 7º, II - Normas de Compliance e Governança) | **Pessoal / PII**<br>(Dados de pessoas físicas. Exige mascaramento dinâmico na camada de visualização e controle de efemeridade com descarte local via `clean_local_temp_files`). |

### 📖 3. Catálogo de Dados & Linhagem
A estrutura de dados é mapeada de forma declarativa e centralizada:
* **Centralização de Schemas:** As definições de tipos, chaves e relacionamentos do BigQuery são controladas via código, garantindo que a volumetria ingerida respeite rigorosamente o contrato de dados estabelecido.
* **Separação de Contextos:** Divisão clara entre a entidade de dados cadastrais da Empresa e a entidade do Quadro de Sócios e Administradores (QSA).

### 📋 4. Tabela de Logs e Monitoramento
Para garantir a observabilidade do pipeline, o script alimenta uma tabela dedicada de logs operacionais no BigQuery a cada execução. Essa tabela registra:
* **Metadados de Ingestão:** Data e hora do processamento, volume de CNPJs consultados com sucesso, quantidade de falhas na API e o status final da carga (Sucesso/Erro). Isso permite rastrear a saúde do fluxo analítico sem a necessidade de abrir consoles de infraestrutura.

### 🔤 5. Padronização de Nomenclatura (Padrão DATASUS)
O dicionário de colunas no arquivo `configs/mapping.py` adota a **metodologia de compressão de vogais e prefixos taxonômicos inspirada no DATASUS**:

O de/para mapeia explicitamente os tipos e aliases curtos:
* `cnpj` ➔ `NRCNPJ` (Número CNPJ)
* `razao_social` ➔ `NMRAZSOC` (Nome Razão Social)
* `nome_fantasia` ➔ `NMFANT` (Nome Fantasia)
* `capital_social` ➔ `VLCPTSOC` (Valor Capital Social)
* `identificador_matriz_filial` ➔ `IDMTZFIL` (Identificador Matriz Filial)
* `descricao_identificador_matriz_filial` ➔ `DSIDMTZFIL` (Descrição Identificador Matriz Filial)

Essa abordagem reduz drasticamente o ruído dos metadados, economiza bytes de armazenamento de colunas, padroniza as chaves lógicas do Data Warehouse e assegura que a modelagem relacional seja limpa e previsível para qualquer analista da organização.

## 📊 5. Monitoramento do Pipeline & Insights Operacionais

O estágio final do ecossistema consiste em expor os dados processados e estruturados da camada Silver (BigQuery) para o **Looker Studio**, onde o fluxo de ingestão e a volumetria empresarial são monitorados visualmente.

### 📈 Métricas de Monitoramento e Observabilidade
O painel técnico consome diretamente a tabela de logs gerada pelo pipeline, permitindo o acompanhamento de indicadores vitais de DataOps:
* **Taxa de Sucesso da Ingestão:** Percentual de requisições enviadas para a BrasilAPI que retornaram com sucesso (*Status 200*) vs. falhas cadastrais (CNPJs inválidos ou inexistentes).
* **Volumetria Diária:** Monitoramento do volume de linhas injetadas por execução para garantir que não haja quedas drásticas ou anomalias no fluxo de processamento.
* **Métricas de FinOps:** Rastreabilidade do tempo de execução e volume de dados trafegados, ajudando a garantir o funcionamento do pipeline dentro do limite de gratuidade (*Free Tier*) da GCP.

### 💡 Extração de Insights Cadastrais Básicos
Além do monitoramento do pipeline, o painel consolida visões gerais sobre o portfólio de empresas processadas para fornecer inteligência de negócios:
* **Distribuição Geográfica:** Concentração das empresas ativas no território nacional por UF e município (utilizando o campo `cd_municipio` mapeado).
* **Análise de Porte e Capital:** Agrupamento do volume financeiro das organizações com base na distribuição do campo `VLCPTSOC` (Capital Social).
* **Visão de Quadro Societário:** Mapeamento quantitativo do número de sócios por organização, permitindo entender a densidade de vínculos societários no ecossistema consultado.
