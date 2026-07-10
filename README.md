## 🏛️ 1. Governança de Dados & Conformidade (LGPD)

Este projeto foi desenhado sob os princípios de *Privacy by Design*, simulando a estrutura de governança de um **Bureau de Crédito**. Como o pipeline processa dados cadastrais de pessoas jurídicas e de seus respectivos quadros societários para subsidiar análises de risco, a estratégia de ingestão segrega os ativos conforme sua finalidade de negócio e enquadramento na Lei Geral de Proteção de Dados (LGPD -Lei nº 13.709/18).

### 📋 Matriz de Conformidade e Classificação de Dados

| Domínio dos Dados | Atributos Principais | Finalidade de Uso | Base Legal (LGPD) | Classificação de Segurança |
| :--- | :--- | :--- | :--- | :--- |
| **Cadastral PJ** | CNPJ, Razão Social, CNAE, Endereço, Situação Cadastral | Higienização de bases, enriquecimento cadastral, mitigação de risco de fraude de terceiros e geração de indicadores de mercado. | **Legítimo Interesse**<br>(Art. 7º, IX) | **Público**<br>(Dados institucionais de registro público) |
| **QSA** *(Quadro de Sócios)* | Nome do Sócio, CPF (mascarado/parcial), Qualificação | Avaliação de interdependência financeira, identificação de beneficiários finais para concessão de crédito e **geração de dashboards analíticos de BI (Business Intelligence)** para monitoramento de risco. | **Proteção do Crédito**<br>(Art. 7º, X) | **Pessoal / PII**<br>(Exige controle de acesso restrito via RBAC no ambiente interno, embora os CPFs já possuam mascaramento parcial nativo da origem). |

### 🔒 Diretrizes de Segurança e Privacidade Aplicadas
* **Pseudonimização na Origem:** Os documentos de pessoas físicas no domínio QSA (CPFs dos sócios) já são ingeridos em formato mascarado (`***.XXX.XXX-**`) fornecido pela BrasilAPI. O pipeline preserva essa característica de privacidade ao longo das camadas (Raw ➔ Silver), mitigando riscos de reidentificação direta do titular.
* **Controle de Acesso e Governança de Visualização Pública:** No 'ambiente corporativo simulado' (GCP), o acesso aos nomes completos dos sócios para fins de BI é restrito e auditado via RBAC. Contudo, dado que este portfólio é público, foi aplicada uma camada de mascaramento dinâmico nos campos nominais de pessoas físicas diretamente na camada de visualização (Looker Studio). Essa abordagem atende ao princípio da finalidade da LGPD, impedindo a exposição indexada e em lote de nomes na internet e mitigando riscos de *data scraping* por terceiros.


## 🔄 2. Ciclo de Vida dos Dados & Metodologia Smart Data

O pipeline foi estruturado seguindo o framework de gerenciamento do ciclo de vida dos dados (Coleta, Armazenamento, Recuperação, Uso e Expurgo), garantindo escalabilidade, arquitetura 100% serverless e otimização de custos (FinOps) em cada etapa da jornada analítica.

```text
 [Origem: BrasilAPI] ➔ [1. Coleta] ➔ [2. Armazenamento (Raw)] ➔ [3. Processamento (Silver)] ➔ [4. Uso (BI/Looker)] ➔ [5. Expurgo]
 ```

* **1. Coleta (Ingestão):** Orquestrada via **GitHub Actions**, a coleta consome dados cadastrais e societários da *BrasilAPI* por meio de scripts **Python** executados de forma automatizada (via gatilhos de tempo - *cron*). Essa abordagem elimina a necessidade de gerenciar servidores ou clusters de orquestração dedicados, garantindo resiliência e baixo custo de execução.
* **2. Armazenamento (Camada Raw/Bronze):** Os dados brutos em formato JSON são persistidos diretamente no **Google Cloud Storage (GCS)** em buckets. Esta camada funciona como um *Data Lake* primário, preservando o histórico e a fidelidade do dado original sem modificações conforme a fonte.
* **3. Recuperação & Processamento (Camada Silver):** Os arquivos JSON do GCS são lidos, limpos e estruturados em .parquet de forma relacional dentro do **Google BigQuery**. Nesta etapa, os dados sofrem transformações essenciais para o negócio: tipagem de dados, padronização de strings, tratamento de nulos e estruturação da tabela de Quadro de Sócios e Administradores (QSA).
* **4. Uso & Inteligência (Smart Data / BI):** É a fase onde o dado bruto se transforma em inteligência. Os dados estruturados da camada Silver alimentam um painel no **Looker Studio**, gerando métricas agregadas de interdependência de sócios, concentração de risco por CNAE e distribuição geográfica das empresas. O acesso na nuvem é restrito via políticas de **RBAC (Role-Based Access Control)** no BigQuery.
* **5. Arquivamento & Expurgo (Retenção):** Para conformidade com a LGPD e otimização de custos de armazenamento no BigQuery e GCS, aplicam-se regras de ciclo de vida (*Object Lifecycle Management*):
  * **Dados Brutos (GCS):** Movidos automaticamente para classes de armazenamento de cold storage (Archive) após 90 dias.


## 🛠️ 3. Arquitetura Técnica & Infraestrutura Serverless

O ecossistema foi projetado utilizando o conceito de **Infrastructure as Code (IaC)** e desacoplamento de camadas, sendo sustentado por serviços gerenciados (Serverless) da **Google Cloud Platform (GCP)** e automatizado via CI/CD no **GitHub Actions**.

### 🏗️ Desenho de Arquitetura

```text
+-------------------+      +-------------------+      +-----------------------+
|  GitHub Actions   | ---> |   Cloud Storage   | ---> |    Google BigQuery    |
| (Orquestração/    |      | (Camada Raw/JSON) |      | (Camada Silver/OLAP)  |
|  Scripts Python)  |      +-------------------+      +-----------------------+
+-------------------+                                             |
                                                                  v
                                                      +-----------------------+
                                                      |     Looker Studio     |
                                                      | (Visualização/BI)     |
                                                      +-----------------------+
```

### 🧰 Componentes do Ecossistema

* **Orquestrador de Workflows (GitHub Actions):** 
  * Atua como o motor do pipeline, executando os scripts Python de ingestão por meio de gatilhos agendados (*cron*).
  * Gerencia *secrets* (chaves de acesso à GCP e chaves de API), garantindo que credenciais sensíveis nunca fiquem expostas no código-fonte.
* **Storage de Arquivos Brutos (Google Cloud Storage):**
  * Armazenamento de objetos de baixo custo focado na retenção do dado bruto (JSON) em sua forma original.
* **Data Warehouse Analítico (Google BigQuery):**
  * Mecanismo de processamento de dados em larga escala (OLAP). É responsável por receber as cargas transformadas e expor as tabelas relacionais prontas para consumo de BI e Analytics.
* **Autenticação e Permissões (GCP IAM & Service Accounts):**
  * O acesso entre as ferramentas (GitHub ➔ GCP ➔ Looker Studio) é regido pelo **Princípio do Menor Privilégio**, utilizando chaves de contas de serviço limitadas estritamente às permissões de leitura/escrita necessárias em cada recurso.


## 🧠 4. Feature Engineering & Modelagem Analítica

A transformação dos dados da camada Raw para a camada Silver focou em gerar variáveis estratégicas (features) essenciais para modelos de concessão de crédito, mitigação de riscos e análise de fraude.

### 📊 Principais Features Desenvolvidas

* **Mapeamento de Vínculos Societários (QSA):**
  * Criação de chaves de relacionamento para identificar interdependência entre empresas por meio de sócios em comum. Essa métrica é crucial para identificar grupos econômicos ocultos e contágio de inadimplência.
* **Métricas de Concentração Setorial e Geográfica:**
  * Consolidação do volume de empresas ativas por região e segmento (CNAE), permitindo identificar a exposição de risco de crédito em setores específicos da economia.
* **Estruturação de Qualificação do Quadro Societário:**
  * Padronização dos cargos e níveis de responsabilidade dos administradores (ex: Diretor, Sócio-Administrador) para modelagem de score baseado no perfil do corpo diretivo.


## 📈 5. Resultados Práticos, Insights de Negócio & Próximos Passos

O objetivo final éconsolidar dados dispersos em uma solução centralizada de inteligência cadastral, fornecendo insumos claros para a tomada de decisão em mesas de crédito e compliance.

### 💡 Insights Gerados pelo Dashboard (Looker Studio)

* **Visão 360° do Ecossistema Societário:** Centralização de consultas que antes exigiam buscas manuais ou individuais, permitindo cruzar dados de CNPJ e QSA de forma instantânea.
* **Análise de Concentração de Risco:** Identificação visual rápida de setores econômicos (CNAE) ou regiões geográficas com maior volume de empresas ativas no portfólio analisado.

### 🚀 Próximos Passos & Evolução do Projeto

Para as próximas iterações da arquitetura, estão planejadas as seguintes implementações:
* **Camada de Machine Learning (Gold):** Integração com o BigQuery ML ou Vertex AI para construir um modelo preditivo de *Score de Sobrevivência* empresarial ou propensão à inadimplência baseado no tempo de abertura e capital social.
* **Monitoramento e Alertas de Data Quality:** Implementação de testes automatizados de qualidade de dados (via Great Expectations ou queries agendadas) para monitorar anomalias na ingestão (ex: volumetria abaixo do esperado ou campos nulos inesperados).
* **CI/CD para Mudanças de Schema:** Automação de testes de integração nos scripts Python para validar o schema dos dados caso a API de origem sofra atualizações na estrutura dos payloads JSON.