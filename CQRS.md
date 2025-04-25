# CQRS: Command Query Responsibility Segregation

## What is CQRS?

CQRS is a pattern that separates the responsibility for handling **commands** (write operations that change state) from **queries** (read operations that retrieve data). This separation allows for optimized performance, scalability, and maintainability by using different models for reading and writing.

### Key Concepts
- **Command**: An action that changes the system's state (e.g., `CreateOrder`, `UpdateUser`).
- **Query**: A request to retrieve data without modifying state (e.g., `GetOrderById`, `ListUsers`).
- **Separation**: Commands and queries use distinct models, often with separate databases (one for writes, one for reads).
- **Event Sourcing (Optional)**: Often paired with CQRS, where state changes are stored as a sequence of events.

### Benefits
- **Scalability**: Optimize read and write operations independently (e.g., scale read database for heavy query loads).
- **Flexibility**: Use different data models for reads (e.g., denormalized views) and writes (e.g., normalized data).
- **Maintainability**: Clear separation of concerns simplifies complex systems.

### Trade-offs
- **Complexity**: Separate models and synchronization (e.g., eventual consistency) add overhead.
- **Learning Curve**: Requires understanding of asynchronous patterns and event-driven architecture.
- **Eventual Consistency**: Read models may lag behind write models.

## Simple Example (Pseudo-Code)

### Context
A basic e-commerce system where users can place orders (write) and view order details (read).

### Command Side
Handles state changes (writes).

```csharp
// Command
public class CreateOrderCommand {
    public Guid OrderId { get; set; }
    public string CustomerName { get; set; }
    public List<string> Items { get; set; }
}

// Command Handler
public class OrderCommandHandler {
    private readonly IWriteRepository _writeRepository;

    public OrderCommandHandler(IWriteRepository writeRepository) {
        _writeRepository = writeRepository;
    }

    public async Task Handle(CreateOrderCommand command) {
        var order = new Order {
            Id = command.OrderId,
            CustomerName = command.CustomerName,
            Items = command.Items
        };
        await _writeRepository.SaveAsync(order);
        // Publish event for read model update
        await PublishEvent(new OrderCreatedEvent(order));
    }
}
```
### Query Side
Handles data retrieval (reads).
```csharp
// Query
public class GetOrderQuery {
    public Guid OrderId { get; set; }
}

// Query Handler
public class OrderQueryHandler {
    private readonly IReadRepository _readRepository;

    public OrderQueryHandler(IReadRepository readRepository) {
        _readRepository = readRepository;
    }

    public async Task<OrderDto> Handle(GetOrderQuery query) {
        return await _readRepository.GetByIdAsync(query.OrderId);
    }
}

// DTO for read model
public class OrderDto {
    public Guid Id { get; set; }
    public string CustomerName { get; set; }
    public List<string> Items { get; set; }
}
```
### Database Setup
**Write Database**: Normalized relational DB (e.g., SQL Server) for commands.

**Read Database**: Denormalized NoSQL DB (e.g., MongoDB) for queries, updated via events.

### Event Synchronization
When a command creates an order, an OrderCreatedEvent is published to update the read database asynchronously.
```csharp

public class OrderCreatedEvent {
    public Order Order { get; set; }
}

public class ReadModelUpdater {
    public async Task Handle(OrderCreatedEvent evt) {
        var orderDto = new OrderDto {
            Id = evt.Order.Id,
            CustomerName = evt.Order.CustomerName,
            Items = evt.Order.Items
        };
        await _readRepository.SaveAsync(orderDto);
    }
}
```
## When to Use CQRS
Systems with high read/write disparity (e.g., reporting vs. transactions).

Complex domains requiring distinct models for reads and writes.

Applications needing scalability for specific operations.



