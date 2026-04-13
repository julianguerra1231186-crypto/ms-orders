![](https://github.com/julianguerra1231186-crypto/ms-orders/blob/main/miroservicio3.png)
# ms-orders — Microservicio de Gestión de Pedidos
### Se deja evidencia de todas las Historias de Usuario en la mesa de trabajo:
   - [Mesa De Trabajo](https://julianguerra1231186-1773894024267.atlassian.net/?continue=https%3A%2F%2Fjulianguerra1231186-1773894024267.atlassian.net%2Fwelcome%2Fsoftware%3FprojectId%3D10000&atlOrigin=eyJpIjoiOTdhMWY4ZGU5N2YwNDQ0MDk3NTZjODkxYTU5ZWVlZWQiLCJwIjoiamlyYS1zb2Z0d2FyZSJ9)

## Descripción general

`ms-orders` es el microservicio responsable de registrar y consultar los pedidos de compra de PulpApp. Cuando un cliente finaliza su compra en el carrito, este servicio recibe los productos seleccionados, consulta el precio real y vigente de cada uno directamente en `ms-products`, calcula el total y persiste el pedido completo con su detalle.

Una característica clave es el **precio histórico**: el precio de cada producto se guarda en el momento exacto de la compra. Si el precio cambia después, el pedido siempre refleja lo que el cliente pagó originalmente.

---

## Puerto

| Entorno | Puerto |
|---------|--------|
| Local | 8083 |
| Docker (interno) | 8083 |
| Acceso desde host | http://localhost:8083 |
| Acceso vía Gateway | http://localhost:8090/orders |

---

## Stack tecnológico

| Tecnología | Versión | Propósito |
|-----------|---------|-----------|
| Java | 17 | Lenguaje base |
| Spring Boot | 4.0.3 | Framework principal |
| Spring Data JPA | Incluido | Persistencia con Hibernate |
| Spring Validation | Incluido | Validaciones con @Valid |
| RestClient | Incluido en Boot | Comunicación HTTP con ms-products |
| PostgreSQL | 15 | Base de datos relacional |
| Liquibase | Incluido | Versionado del esquema de BD |
| Lombok | Incluido | Reducción de código boilerplate |

---

## Estructura de paquetes

```
com.pulpapp.msorders/
│
├── config/
│   └── RestClientConfig.java
│         Configura el bean RestClient apuntando a la URL base de ms-products.
│         La URL se inyecta desde la variable de entorno MS_PRODUCTS_BASE_URL.
│         Dentro de Docker usa http://ms-products:8082 (DNS interno de Docker).
│
├── controller/
│   └── OrderController.java
│         Expone los endpoints REST de pedidos:
│         POST /orders      → crea un nuevo pedido (201 Created)
│         GET  /orders      → lista todos los pedidos
│         GET  /orders/{id} → consulta un pedido específico por ID
│
├── service/
│   └── OrderService.java
│         Contiene la lógica de negocio principal del microservicio.
│
│         createOrder() — flujo completo de creación:
│           1. Crea la cabecera del pedido con el userId
│           2. Por cada item del request:
│              a. Consulta el producto en ms-products via ProductClient
│              b. Verifica que el producto esté disponible (available = true)
│              c. Verifica que haya stock suficiente
│              d. Captura el precio actual como precio histórico
│              e. Calcula el subtotal (precio × cantidad)
│              f. Crea el OrderItem y lo agrega al pedido
│           3. Suma todos los subtotales para calcular el total
│           4. Persiste el pedido completo con todos sus items en una
│              sola transacción (@Transactional)
│
│         findAll()   → retorna todos los pedidos con sus items
│         findById()  → busca por ID, lanza ResourceNotFoundException si no existe
│
├── client/
│   └── ProductClient.java
│         Cliente HTTP que se comunica con ms-products usando RestClient.
│         getProductById(productId) → GET /products/{id} en ms-products.
│         Maneja los errores de comunicación:
│           - 404 en ms-products → lanza ResourceNotFoundException
│           - Otro error HTTP → lanza ExternalServiceException
│           - ms-products no responde → lanza ExternalServiceException
│         Esto garantiza que los errores externos se traduzcan en
│         respuestas claras para el cliente del API.
│
├── entity/
│   ├── Order.java
│   │     Entidad JPA mapeada a la tabla orders.
│   │     Campos: id, userId (referencia lógica a ms-users, sin FK física),
│   │     total, fecha (se asigna automáticamente con @PrePersist).
│   │     Relación OneToMany con OrderItem (1 pedido → N items).
│   │     El método addItem() agrega un item y establece la relación bidireccional.
│   │
│   └── OrderItem.java
│         Entidad JPA mapeada a la tabla order_items.
│         Campos: id, productId (referencia lógica a ms-products, sin FK física),
│         cantidad, precioUnitario (precio histórico capturado al crear el pedido),
│         subtotal (precioUnitario × cantidad).
│         Relación ManyToOne con Order.
│
├── dto/
│   ├── CreateOrderRequestDTO.java
│   │     Entrada para crear un pedido:
│   │     {userId, items: [{productId, cantidad}]}
│   │
│   ├── OrderItemRequestDTO.java
│   │     Detalle de cada item en el request: {productId, cantidad}
│   │
│   ├── OrderResponseDTO.java
│   │     Salida completa del pedido:
│   │     {id, userId, total, fecha, items: [OrderItemResponseDTO]}
│   │
│   ├── OrderItemResponseDTO.java
│   │     Detalle de cada item en la respuesta:
│   │     {id, productId, cantidad, precioUnitario, subtotal}
│   │
│   └── ProductResponseDTO.java
│         DTO local que replica la estructura de respuesta de ms-products.
│         Se usa para deserializar la respuesta del ProductClient.
│         Campos: {id, name, price, stock, available}
│
├── repository/
│   └── OrderRepository.java
│         Extiende JpaRepository<Order, Long>. CRUD básico.
│         Spring Data JPA genera todas las consultas necesarias.
│
└── exception/
    ├── GlobalExceptionHandler.java
    │     Captura excepciones y las convierte en respuestas JSON estructuradas.
    │     Maneja: ResourceNotFoundException (404), ExternalServiceException (502),
    │     IllegalStateException (409 para stock insuficiente o producto no disponible),
    │     MethodArgumentNotValidException (400), Exception (500).
    │
    ├── ResourceNotFoundException.java
    │     Lanzada cuando no se encuentra un pedido o producto por ID.
    │
    └── ExternalServiceException.java
          Lanzada cuando ms-products no responde o retorna un error inesperado.
          Permite distinguir errores propios de errores de servicios externos.
```

---

## Comunicación con ms-products

Este es el único microservicio del sistema que se comunica con otro microservicio directamente (comunicación sincrónica).

### ¿Cómo funciona?

```
ms-orders recibe POST /orders
        ↓
Por cada producto en el pedido:
        ↓
ProductClient → GET http://ms-products:8082/products/{id}
        ↓
ms-products retorna: {id, name, price, stock, available}
        ↓
ms-orders captura el precio y verifica disponibilidad y stock
        ↓
Crea el OrderItem con el precio histórico
        ↓
Persiste el pedido completo
```

### ¿Por qué no hay FK física entre orders y products?

Porque son microservicios independientes con bases de datos separadas (aunque compartan la misma instancia de PostgreSQL). Si hubiera una FK física, eliminar un producto rompería los pedidos históricos. Con referencias lógicas, cada servicio es autónomo.

---

## Liquibase — Versionado de base de datos

### ¿Qué es Liquibase?

Liquibase gestiona los cambios en la estructura de la base de datos de forma controlada. Define **changesets** en YAML que se ejecutan automáticamente al arrancar. Lleva un registro en la tabla `databasechangelog` para no ejecutar el mismo changeset dos veces.

`onFail: MARK_RAN` hace que si una tabla o índice ya existe, el changeset se registre como ejecutado sin error — los changesets son **idempotentes**.

### Changesets de ms-orders

#### Changeset 1 — Tabla `orders`

Crea la cabecera del pedido. `user_id` es una referencia lógica (sin FK física) al usuario de ms-users.

```
orders
├── id      BIGINT PK autoincrement
├── user_id BIGINT NOT NULL (referencia lógica a users.id en ms-users)
├── total   DOUBLE PRECISION NOT NULL
└── fecha   TIMESTAMP NOT NULL
```

#### Changeset 2 — Tabla `order_items`

Crea el detalle de cada pedido. Guarda el precio en el momento de la compra.

```
order_items
├── id              BIGINT PK autoincrement
├── order_id        BIGINT NOT NULL (FK → orders)
├── product_id      BIGINT NOT NULL (referencia lógica a products.id en ms-products)
├── cantidad        INTEGER NOT NULL
├── precio_unitario DOUBLE PRECISION NOT NULL (precio histórico)
└── subtotal        DOUBLE PRECISION NOT NULL
```

#### Changeset 3 — FK `order_items → orders`

Crea la clave foránea entre `order_items.order_id` y `orders.id`.
`onDelete: CASCADE` significa que al eliminar un pedido, todos sus items se eliminan automáticamente.

#### Changeset 4 — Índice en `order_items.order_id`

Crea un índice en la columna `order_id` de `order_items` para optimizar las consultas de detalle de pedido. Sin este índice, consultar los items de un pedido requeriría un full scan de la tabla.

---

## Modelo de datos

### Tabla `orders`

| Columna | Tipo | Restricción | Descripción |
|---------|------|-------------|-------------|
| id | BIGINT | PK, autoincrement | Identificador único |
| user_id | BIGINT | NOT NULL | ID del usuario (referencia lógica) |
| total | DOUBLE PRECISION | NOT NULL | Suma de todos los subtotales |
| fecha | TIMESTAMP | NOT NULL | Fecha y hora de creación |

### Tabla `order_items`

| Columna | Tipo | Restricción | Descripción |
|---------|------|-------------|-------------|
| id | BIGINT | PK, autoincrement | Identificador único |
| order_id | BIGINT | FK → orders (CASCADE) | Pedido al que pertenece |
| product_id | BIGINT | NOT NULL | ID del producto (referencia lógica) |
| cantidad | INTEGER | NOT NULL | Unidades compradas |
| precio_unitario | DOUBLE PRECISION | NOT NULL | Precio al momento de la compra |
| subtotal | DOUBLE PRECISION | NOT NULL | precio_unitario × cantidad |

### Relación entre tablas

```
orders (1) ──────────────── (N) order_items
   id                               id
   user_id                          order_id (FK → orders, CASCADE DELETE)
   total                            product_id
   fecha                            cantidad
                                    precio_unitario
                                    subtotal
```

---

## Endpoints con ejemplos

### POST /orders — Crear pedido

```json
// Request
{
  "userId": 1,
  "items": [
    { "productId": 1, "cantidad": 2 },
    { "productId": 3, "cantidad": 1 }
  ]
}

// Response 201
{
  "id": 1,
  "userId": 1,
  "total": 24900.0,
  "fecha": "2026-04-10T17:30:00",
  "items": [
    {
      "id": 1,
      "productId": 1,
      "cantidad": 2,
      "precioUnitario": 8500.0,
      "subtotal": 17000.0
    },
    {
      "id": 2,
      "productId": 3,
      "cantidad": 1,
      "precioUnitario": 7900.0,
      "subtotal": 7900.0
    }
  ]
}
```

### Errores posibles al crear pedido

```json
// Producto no disponible — 409
{ "error": "Product with id 1 is not available" }

// Stock insuficiente — 409
{ "error": "Insufficient stock for product with id 1" }

// ms-products no responde — 502
{ "error": "ms-products is unavailable or did not respond correctly" }
```

---

## Manejo de errores

| Excepción | HTTP | Cuándo ocurre |
|-----------|------|---------------|
| ResourceNotFoundException | 404 | Pedido no encontrado por ID |
| IllegalStateException | 409 | Producto no disponible o sin stock |
| ExternalServiceException | 502 | ms-products no responde |
| MethodArgumentNotValidException | 400 | Campos inválidos en el request |
| Exception (fallback) | 500 | Error interno no controlado |

---

## Configuración

```properties
spring.application.name=ms-orders
server.port=${SERVER_PORT:8083}

spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5434/pulpapp_db}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:postgres}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:1234}

# URL de ms-products para consultar precios
ms-products.base-url=${MS_PRODUCTS_BASE_URL:http://localhost:8082}

spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true

spring.liquibase.change-log=classpath:db/changelog/changelog-master.yml
spring.liquibase.enabled=true
```

---

## Levantar el servicio

```bash
docker-compose up --build ms-orders
docker-compose logs -f ms-orders
```
