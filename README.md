# Microsserviços com Saga Orquestrada

Projeto de estudo de uma arquitetura de microsserviços que usa o **padrão Saga orquestrado** para manter a consistência de uma transação distribuída (a criação de um pedido) entre vários serviços, cada um com seu próprio banco de dados. Toda a comunicação entre serviços é assíncrona, via **Apache Kafka**.

## Stack

- Java 17 + Spring Boot 3.1.2
- Spring Kafka
- Spring Data MongoDB (order-service) e Spring Data JPA + PostgreSQL (demais serviços)
- Lombok
- Gradle
- Docker / Docker Compose
- Redpanda Console (interface web para inspecionar os tópicos do Kafka)

## Arquitetura

| Serviço | Porta | Banco | Responsabilidade |
|---|---|---|---|
| `order-service` | 3000 | MongoDB (`order-db`) | Recebe o pedido via REST, dispara a saga e guarda o resultado final dos eventos |
| `orchestrator-service` | 8080 | — | Orquestra a saga: decide para qual tópico o evento vai a seguir |
| `product-validation-service` | 8090 | PostgreSQL (`product-db`) | Valida se os produtos do pedido existem |
| `payment-service` | 8091 | PostgreSQL (`payment-db`) | Calcula o valor total e registra o pagamento |
| `inventory-service` | 8092 | PostgreSQL (`inventory-db`) | Baixa o estoque dos produtos |

Infraestrutura:

| Container | Porta(s) |
|---|---|
| `kafka` | 9092 (externo), 29092 (interno, entre containers) |
| `redpanda-console` | 8081 |
| `order-db` (MongoDB) | 27017 |
| `product-db` / `payment-db` / `inventory-db` (PostgreSQL) | 5432 / 5433 / 5434 |

### Fluxo da saga

```
                         start-saga
  order-service ───────────────────────────▶ orchestrator-service
       ▲                                         │   ▲
       │ notify-ending                           │   │ orchestrator
       │                                         ▼   │
       │                    product-validation-success/fail ──▶ product-validation-service
       │                    payment-success/fail            ──▶ payment-service
       └──────────────────  inventory-success/fail          ──▶ inventory-service
```

1. O `order-service` salva o pedido no MongoDB, gera um `transactionId` e publica o evento em `start-saga`.
2. O orquestrador recebe o evento e o envia para o primeiro passo (`product-validation-success`).
3. Cada serviço executa sua etapa e devolve o evento ao tópico `orchestrator`, com `source` (quem processou) e `status` (`SUCCESS`, `ROLLBACK_PENDING` ou `FAIL`).
4. O orquestrador consulta a tabela de transições em `SagaHandler` e encaminha para o próximo tópico.
5. Ao final, o orquestrador publica em `notify-ending` e o `order-service` grava o evento com todo o histórico.

### Tabela de transições (`SagaHandler`)

| Origem | Status | Próximo tópico |
|---|---|---|
| ORCHESTRATOR | SUCCESS | `product-validation-success` |
| ORCHESTRATOR | FAIL | `finish-fail` |
| PRODUCT_VALIDATION_SERVICE | SUCCESS | `payment-success` |
| PRODUCT_VALIDATION_SERVICE | ROLLBACK_PENDING | `product-validation-fail` |
| PRODUCT_VALIDATION_SERVICE | FAIL | `finish-fail` |
| PAYMENT_SERVICE | SUCCESS | `inventory-success` |
| PAYMENT_SERVICE | ROLLBACK_PENDING | `payment-fail` |
| PAYMENT_SERVICE | FAIL | `product-validation-fail` |
| INVENTORY_SERVICE | SUCCESS | `finish-success` |
| INVENTORY_SERVICE | ROLLBACK_PENDING | `inventory-fail` |
| INVENTORY_SERVICE | FAIL | `payment-fail` |

### Compensação (rollback)

Quando uma etapa falha, o serviço marca o evento como `ROLLBACK_PENDING`. O orquestrador manda o evento para o tópico `*-fail` desse mesmo serviço, que desfaz o que tiver feito e devolve o evento com status `FAIL`. A partir daí o rollback volta pela cadeia, serviço por serviço, até chegar em `finish-fail`:

- **inventory-service**: restaura as quantidades anteriores do estoque (`OrderInventory.oldQuantity`).
- **payment-service**: muda o pagamento para `REFUND`.
- **product-validation-service**: marca a validação como `success = false`.

Cada serviço também confere se já existe um registro com o mesmo `orderId` + `transactionId`, para não processar o mesmo evento duas vezes.

## Regras de negócio

- **Validação de produtos**: a lista de produtos não pode estar vazia, e todo `code` precisa existir na tabela `product`.
- **Pagamento**: total = soma de `quantity * unitValue`. O valor mínimo é `0.1`.
- **Estoque**: falha com `Product is out of stock!` se a quantidade pedida for maior que a disponível.

Dados iniciais (`import.sql`):

| Produto | Estoque inicial |
|---|---|
| `COMIC_BOOKS` | 4 |
| `BOOKS` | 2 |
| `MOVIES` | 5 |
| `MUSIC` | 9 |

## Como executar

### Pré-requisitos

- Docker e Docker Compose
- Java 17
- Python 3 (só para o script de build)

### Opção 1: script de build

```bash
python3 build.py
```

O script compila os 5 serviços em paralelo (`./gradlew build -x test`), remove containers existentes e sobe tudo com `docker compose up --build -d`.

> **Atenção:** a etapa de limpeza do `build.py` para e remove **todos** os containers Docker da máquina, não só os deste projeto.

### Opção 2: manualmente

```bash
# compilar cada serviço
for s in order-service orchestrator-service product-validation-service payment-service inventory-service; do
  (cd $s && ./gradlew build -x test)
done

# subir a stack
docker compose up --build -d
```

### Rodar localmente (fora do Docker)

Suba só a infraestrutura e rode os serviços pela IDE ou com `./gradlew bootRun`. Os valores padrão do `application.yml` já apontam para `localhost`:

```bash
docker compose up -d order-db product-db payment-db inventory-db kafka redpanda-console
```

Variáveis de ambiente aceitas: `KAFKA_BROKER`, `MONGO_DB_URI`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.

## API (order-service)

> Os controllers usam `@Controller("/api/order")` e `@Controller("/api/event")`. Nessa anotação o valor é o **nome do bean**, não um caminho, então os endpoints ficam na raiz. Para usar os prefixos `/api/...`, troque por `@RequestMapping("/api/order")` / `@RequestMapping("/api/event")`.

### Criar pedido

```bash
curl -X POST http://localhost:3000/ \
  -H "Content-Type: application/json" \
  -d '{
    "products": [
      { "product": { "code": "COMIC_BOOKS", "unitValue": 15.50 }, "quantity": 3 },
      { "product": { "code": "BOOKS", "unitValue": 9.90 }, "quantity": 1 }
    ]
  }'
```

### Consultar o resultado da saga

```bash
# pelo orderId ou pelo transactionId
curl "http://localhost:3000/?orderId=<id>"
curl "http://localhost:3000/?transactionId=<transactionId>"

# todos os eventos, mais recentes primeiro
curl http://localhost:3000/all
```

O evento retornado tem o campo `eventHistory`, com cada etapa da saga (origem, status, mensagem e data).

### Testar o rollback

- Use um `code` que não existe, como `"XYZ"`: falha logo na validação de produtos.
- Use `unitValue: 0`: falha no pagamento e desfaz a validação.
- Peça mais itens do que há em estoque, como `"BOOKS"` com `quantity: 10`: falha no estoque e desfaz pagamento e validação.

Os eventos podem ser acompanhados em tempo real no Redpanda Console: http://localhost:8081

O `springdoc-openapi` está nas dependências, então a documentação Swagger fica em `http://localhost:<porta>/swagger-ui.html`.

## Estrutura dos serviços

Todos os serviços seguem a mesma organização:

```
src/main/java/br/com/microservices/orchestrated/<servico>/
├── config/
│   ├── exception/   # ValidationException + handler global
│   └── kafka/       # Configuração de producer/consumer e criação dos tópicos
└── core/
    ├── consumer/    # @KafkaListener
    ├── producer/    # KafkaTemplate
    ├── dto/         # Event, Order, Product, History...
    ├── model/       # Entidades JPA (document/ no order-service)
    ├── repository/
    ├── service/     # Regra de negócio
    └── utils/       # JsonUtil (serialização com Jackson)
```
