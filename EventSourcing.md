# Event Sourcing Architecture 
**Description**
This demonstrates a basic Event Sourcing architecture in C# (.NET). It models a bank account where state is derived from events (e.g., AccountCreated, MoneyDeposited, MoneyWithdrawn). Key components include:
- **Events**: Immutable state changes.

- **Aggregate**: The entity (BankAccount) that applies events to compute its state.

- **Event Store**: Persists events and retrieves them for replay.

- **Commands**: Trigger new events based on business logic.

**Files**
EventSourcing.cs

```csharp
csharp

using System;
using System.Collections.Generic;
using System.Linq;

namespace EventSourcingDemo
{
    // Event base class
    public abstract class Event
    {
        public Guid EventId { get; init; } = Guid.NewGuid();
        public DateTime Timestamp { get; init; } = DateTime.UtcNow;
    }

    // Specific event types
    public class AccountCreated : Event
    {
        public string AccountId { get; init; }
        public string Owner { get; init; }

        public AccountCreated(string accountId, string owner)
        {
            AccountId = accountId;
            Owner = owner;
        }
    }

    public class MoneyDeposited : Event
    {
        public string AccountId { get; init; }
        public decimal Amount { get; init; }

        public MoneyDeposited(string accountId, decimal amount)
        {
            AccountId = accountId;
            Amount = amount;
        }
    }

    public class MoneyWithdrawn : Event
    {
        public string AccountId { get; init; }
        public decimal Amount { get; init; }

        public MoneyWithdrawn(string accountId, decimal amount)
        {
            AccountId = accountId;
            Amount = amount;
        }
    }

    // Aggregate (BankAccount)
    public class BankAccount
    {
        public string AccountId { get; private set; }
        public string Owner { get; private set; }
        public decimal Balance { get; private set; }
        public int Version { get; private set; }
        public List<Event> Changes { get; } = new List<Event>();

        public static BankAccount FromEvents(IEnumerable<Event> events)
        {
            var account = new BankAccount();
            foreach (var @event in events)
            {
                account.Apply(@event);
            }
            return account;
        }

        private void Apply(Event @event)
        {
            Version++;
            switch (@event)
            {
                case AccountCreated created:
                    AccountId = created.AccountId;
                    Owner = created.Owner;
                    break;
                case MoneyDeposited deposited:
                    Balance += deposited.Amount;
                    break;
                case MoneyWithdrawn withdrawn:
                    Balance -= withdrawn.Amount;
                    break;
            }
            Changes.Add(@event);
        }

        // Command handlers
        public void Create(string accountId, string owner)
        {
            if (!string.IsNullOrEmpty(AccountId))
                throw new InvalidOperationException("Account already created");
            Apply(new AccountCreated(accountId, owner));
        }

        public void Deposit(decimal amount)
        {
            if (amount <= 0)
                throw new ArgumentException("Deposit amount must be positive");
            if (string.IsNullOrEmpty(AccountId))
                throw new InvalidOperationException("Account not created");
            Apply(new MoneyDeposited(AccountId, amount));
        }

        public void Withdraw(decimal amount)
        {
            if (amount <= 0)
                throw new ArgumentException("Withdrawal amount must be positive");
            if (string.IsNullOrEmpty(AccountId))
                throw new InvalidOperationException("Account not created");
            if (Balance < amount)
                throw new InvalidOperationException("Insufficient funds");
            Apply(new MoneyWithdrawn(AccountId, amount));
        }
    }

    // Event Store
    public class EventStore
    {
        private readonly Dictionary<string, List<Event>> _events = new();

        public void SaveEvents(string accountId, IEnumerable<Event> events, int expectedVersion)
        {
            if (!_events.ContainsKey(accountId))
                _events[accountId] = new List<Event>();

            var currentVersion = _events[accountId].Count;
            if (currentVersion != expectedVersion)
                throw new InvalidOperationException($"Concurrency conflict: expected version {expectedVersion}, got {currentVersion}");

            _events[accountId].AddRange(events);
        }

        public IEnumerable<Event> GetEvents(string accountId)
        {
            return _events.ContainsKey(accountId) ? _events[accountId].ToList() : Enumerable.Empty<Event>();
        }
    }

    // Example usage
    public static class Program
    {
        public static void Main()
        {
            var eventStore = new EventStore();
            var account = new BankAccount();

            // Create account
            account.Create("acc123", "John Doe");
            eventStore.SaveEvents("acc123", account.Changes, 0);
            account.Changes.Clear();

            // Perform transactions
            account.Deposit(100.0m);
            account.Withdraw(30.0m);
            eventStore.SaveEvents("acc123", account.Changes, 1);
            account.Changes.Clear();

            // Rebuild state from events
            var events = eventStore.GetEvents("acc123");
            var rebuiltAccount = BankAccount.FromEvents(events);
            Console.WriteLine($"Account ID: {rebuiltAccount.AccountId}");
            Console.WriteLine($"Owner: {rebuiltAccount.Owner}");
            Console.WriteLine($"Balance: {rebuiltAccount.Balance}");
            Console.WriteLine($"Version: {rebuiltAccount.Version}");
        }
    }
}
```
**How It Works**
- **Events**: Immutable records of state changes (AccountCreated, MoneyDeposited, MoneyWithdrawn) inherit from an abstract Event class.

- **Aggregate (BankAccount)**:
  - Maintains state by applying events in a switch statement.

  - Handles commands (Create, Deposit, Withdraw) to generate new events.

  - Tracks uncommitted changes and version for optimistic concurrency.

- **Event Store**:
  - Persists events in an in-memory dictionary (keyed by accountId).

  - Supports saving events with version checks to prevent conflicts.

  - Retrieves events for state reconstruction.

- **Rebuilding State**:
  - The FromEvents method replays events to reconstruct the aggregate's state.

- **Concurrency**:
 - Uses optimistic concurrency by checking the expectedVersion when saving events.

**Running the Example**
 - Create a new .NET Console Application (e.g., dotnet new console -n EventSourcingDemo).

 - Replace the contents of Program.cs with the code above (or organize into appropriate files).

 - Run the application:
   ```bash
   bash

   dotnet run
   ```
 - Output:
   ```bash
 
   Account ID: acc123
   Owner: John Doe
   Balance: 70.0
   Version: 3
   ```
**Notes**
This is a simplified example. In production, you’d need:
- A proper event store like EventStoreDB, Cosmos DB, or SQL Server with a schema for events.

- Snapshotting to optimize state reconstruction for large event streams.

- Robust error handling and retries for concurrency conflicts.

- Event versioning for schema evolution (e.g., using upcasting).

Consider using .NET libraries like:
- Marten for PostgreSQL-based event sourcing.

- NEventStore for a lightweight event store.

- MassTransit or MediatR for command/event handling and dispatching.

For serialization, you might use **System.Text.Json** or **Newtonsoft.Json** to store events in a database.

**Project Setup**
To use this in a .NET project:
- Create a new .NET 8 Console Application:
  ```bash
  bash
  
  dotnet new console -n EventSourcingDemo

- Add the code to the project.

- If using a real event store, add dependencies (e.g., Marten):
  ```bash
  bash

  dotnet add package Marten

