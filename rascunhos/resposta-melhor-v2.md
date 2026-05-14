Para resolver esse desafio arquitetônico, precisamos balancear a alta performance (tempo de resposta de 25ms) com a garantia de entrega, ordenação estrita e flexibilidade de schema para uma auditoria robusta.

A solução ideal para este cenário é a adoção de uma **Arquitetura Orientada a Eventos (Event-Driven Architecture)** combinada com os padrões de **Event Sourcing** e o uso de um **Banco de Dados de Série Temporal (Time-Series Database - TSDB)**.

Abaixo, detalho como estruturar essa solução atendendo a cada uma das suas restrições.

### 1. Desacoplamento e Baixa Latência (Mensageria Asíncrona)

Para garantir que o acréscimo da persistência não afete os 25ms de tempo de resposta do serviço `authorization`, a gravação **não pode ser síncrona**.

* O serviço `authorization` atuará apenas como um *Producer*. Após agregar o JSON de pendências dos serviços downstream, ele fará um disparo do tipo *fire-and-forget* para um Message Broker (como **Apache Kafka**).
* Isso garante a **consistência eventual** exigida e absorve o grande volume de dados (alta taxa de ingestão) sem penalizar as requisições HTTP da API REST.

### 2. Ordenação Estrita de Transições

A ordem com que as pendências mudam é crítica para a auditoria antifraude. Em sistemas distribuídos, a ordem global é um desafio, mas a ordem *por cliente* é perfeitamente factível.

* Utilizando o Kafka, você configurará o tópico de eventos usando uma chave de particionamento (Partition Key) baseada no identificador único do cliente (ex: `client_id` ou CPF).
* Isso garante que todas as requisições e transições de estado de um mesmo usuário caiam sempre na mesma partição, garantindo a ordenação FIFO (First-In, First-Out) no processamento.

### 3. Filtro de Alteração de Estado (Diffing)

O requisito exige que **apenas as alterações** sejam persistidas para evitar redundância. Fazer uma consulta no banco de dados a cada requisição para comparar o estado anterior destruiria o tempo de resposta. A solução para isso é um **Cache Distribuído Rápido**:

* Utilize um cluster **Redis** para armazenar o estado mais recente (ou um hash criptográfico do JSON, como SHA-256) das pendências de cada `client_id`.
* Quando o `authorization` compilar a resposta, ele gera o hash do JSON e compara com o cache em O(1).
* Se o hash for igual, nenhuma pendência mudou. O sistema apenas retorna a resposta REST para o cliente.
* Se o hash for diferente (ou não existir no cache), ocorreu uma transição de estado. O serviço envia o payload inteiro para o Kafka e atualiza o hash no Redis.

### 4. Persistência Append-Only e Sem Schema Rígido

Para o armazenamento voltado à auditoria antifraude, a persistência de eventos no mercado financeiro (como o acompanhamento de abertura de contas) se beneficia enormemente de bancos otimizados para tempo e imutabilidade.

* **Solução Recomendada:** Um **Time-Series Database (TSDB)** que possua suporte nativo a dados semiestruturados (JSON), como o **TimescaleDB** (extensão do PostgreSQL) usando colunas `JSONB`, ou o **InfluxDB**.
* **Por que uma TSDB?** Elas são projetadas nativamente para cargas de trabalho *append-only* (insert-heavy), lidam excepcionalmente bem com altíssimo volume de dados e particionam os dados em "chunks" baseados em janelas de tempo, o que torna as consultas analíticas e de auditoria incrivelmente rápidas.
* O schema flexível do `JSONB` no TimescaleDB, por exemplo, permite que novos serviços integrem diferentes formatos de pendências futuramente sem necessidade de rodar migrações estruturais pesadas.

### 5. Disponibilidade para Sistemas Analíticos

Uma vez que os dados repousam em uma TSDB (ou através de um conector do Kafka para um Data Lake, como Amazon S3 com formato Parquet), as equipes de antifraude e dados poderão executar queries analíticas complexas. Elas poderão reconstruir a linha do tempo exata de tentativas de abertura de conta de um fraudador, verificando exatamente quando a pendência de "dispositivo não habilitado" mudou ou quantas vezes a senha falhou, aproveitando as funções nativas de agregação temporal desses bancos.

### Resumo do Fluxo da Requisição

1. Requisição chega no `authorization` via API REST.
2. `authorization` consulta os microserviços e monta o JSON final de pendências.
3. `authorization` calcula o hash do JSON e verifica no Redis se houve alteração desde a última chamada daquele `client_id`.
4. Havendo alteração, envia o JSON de forma assíncrona para o tópico Kafka (usando `client_id` como key) e atualiza o Redis.
5. Retorna a resposta HTTP (mantendo os ~25ms).
6. Um *Consumer Worker* lê as mensagens do Kafka na ordem correta e insere (*append-only*) na Time-Series Database.
