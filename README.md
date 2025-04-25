markdown

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



