# Electrum Domain AI Coding Instructions

## Project Overview
**Electrum** is a narrative AI system (C# .NET 4.7) combining **CQRS/Mediator pattern** with a **character-driven narrative engine**. It manages character decision-making through actions, influences, and world models, integrating with ElectrumX blockchain client for cryptocurrency operations.

## Architecture

### Core Patterns
- **CQRS (Command Query Responsibility Segregation)**: Commands (`Commanding/`) mutate state with access control; Queries (`Querying/`) read data immutably
- **Mediator Pattern** (`Mediation/`): Central `Mediator` class routes all commands/queries to registered handlers via type-name lookup
- **Result Types**: Use `AccessResult` for authorization, `ElectrumValidationResult` for chained validations, `Perhaps<T>` monad for optional values

### Key Components

| Component | Purpose | Files |
|-----------|---------|-------|
| **Mediation Layer** | Routes commands/queries to handlers | `Mediation/Mediator.cs`, `IMediator.cs` |
| **Command/Query Handlers** | Business logic with built-in validation | `Commanding/CommandHandler.cs`, `Querying/QueryHandler.cs` |
| **Character System** | AI agents with traits, relationships, goals | `Character.cs`, `Models.cs`, `Action.cs`, `Condition.cs` |
| **Action Selection** | Evaluates action affinity, utility, bindings | `Character.ChooseAction()` |
| **Security** | Password hashing, access validation | `Security/KnoxHasher.cs`, `AccessResult.cs` |
| **ElectrumX Client** | Blockchain RPC requests | `Extensions/Request/` and `Extensions/Response/` |

## Development Patterns

### Implementing Commands
```csharp
// Define command in Commands/ folder
public record SaveToJsonFileCommand(object Object, string Directory, string FileName) : Command();

// Implement handler - override InternalValidateAccessAsync and InternalExecuteAsync
public class SaveToJsonFileCommandHandler : CommandHandler<SaveToJsonFileCommand>
{
    protected override Task<AccessResult> InternalValidateAccessAsync(SaveToJsonFileCommand command) 
        => Task.FromResult(AccessResult.Success()); // Or Fail("message")
    
    protected override Task InternalExecuteAsync(SaveToJsonFileCommand command)
    {
        // Mutate state here
        return Task.CompletedTask;
    }
}

// Register in Mediator
mediator.Register<SaveToJsonFileCommand>(new SaveToJsonFileCommandHandler());

// Execute via mediator
await mediator.ExecuteCommandAsync(new SaveToJsonFileCommand(...));
```

### Implementing Queries
```csharp
public record GetObjectFromJsonFile(Type SerializeType, string ObjectName) : Query;

public class GetObjectFromJsonFileHandler : QueryHandler<GetObjectFromJsonFile, object>
{
    protected async override Task<object> InternalRequestAsync(GetObjectFromJsonFile query)
    {
        // Return immutable data
        return JsonSerializer.Deserialize(text, query.SerializeType);
    }
}

// Execute via mediator
var result = await mediator.RequestResponseAsync(new GetObjectFromJsonFile(...));
```

### Character Decision-Making
Characters use multi-phase action selection:
1. **Filter by affinity**: Discard actions below `settings.affinityTreshold` based on trait curves
2. **Find bindings**: Recursively match available characters to action roles with conditions
3. **Evaluate quality**: Score each binding via **DesirabilityRules** and **LikelyhoodRules**
4. **Calculate utility**: Simulate action effects on world model, compare to character goals
5. **Select**: Return highest expected utility instance

Key files: `Character.ChooseAction()`, `ActionInstance.cs`, `InfluenceRules.cs`, `Condition.cs`

### Validation Pattern - Chained Results
Use `ElectrumValidationResult.Verify()` to compose validations:
```csharp
var result1 = ElectrumValidationResult.Verify(() => x > 0, "X must be positive");
var result2 = ElectrumValidationResult.Verify(() => y > 0, "Y must be positive", result1);
if (!result2) Debug.Log(string.Join(", ", result2.Messages)); // Aggregated errors
```

### Perhaps<T> Monad for Optional Values
```csharp
Perhaps<T> value = Perhaps<T>.ToPerhaps(someValue); // Null/empty string → empty
if (!value.IsEmpty) { var item = value.Get(); }
var fallback = value.Else(defaultValue);
var safe = value.ElseThrow(new Exception("..."));
```

## Extension Patterns
- **Extensions/** folder: Utility methods on standard types (LINQ for `Perhaps<T>`, serialization, task helpers)
- **Task chain ending**: Use `.FromResult(item)` helper instead of `Task.FromResult()`

## Blockchain Integration
ElectrumX RPC client in `Extensions/Request/Response/`:
- **Request classes** inherit `RequestBase`, set `Method` and `Parameters` in constructor
- **Response classes** inherit `ResponseBase`, implement `FromJson(string)` deserialization
- Requests/Responses use Newtonsoft.Json with custom converters for complex types

## Critical Notes
- **Namespaces**: Main domain logic uses `Vulpes.Electrum.Domain.*`; Unity/Form code is root namespace
- **Async-first**: All handlers are async; use `await` throughout command/query execution
- **No direct state mutation outside handlers**: Characters' `ChooseAction()` is Unity-only; domain logic stays pure
- **SerializableDictionary dependency**: Uses RotaryHeart.Lib for Unity serialization of generic dictionaries
- **Type-name lookup in Mediator**: Handler registry keys are `typeof(TCommand).Name` strings—ensure unique class names

## File Structure Quick Reference
```
Domain/
├── Commanding/          # Command definitions & handlers (access-controlled mutations)
├── Querying/            # Query definitions & handlers (immutable reads)
├── Mediation/           # Mediator pattern implementation
├── Security/            # Hashing, access control, validation results
├── Exceptions/          # Custom exceptions, Perhaps monad, custom handlers
├── Extensions/          # Utility extensions for standard types + blockchain client
├── ConsoleCommanding/   # CLI interface for commands (debug only)
└── [Character, Action, Condition, etc].cs  # Game logic (narrative AI)
```

---

**Update this file when introducing new patterns, architectural changes, or critical workflows that require reading multiple files.**
