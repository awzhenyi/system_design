# Chess Game - Design Patterns and Principles

## Design Patterns Applied

### 1. Strategy Pattern

**Usage**: Each chess piece type implements its own movement strategy.

**Implementation**:
- The abstract `Piece` class defines the interface (`isValidMove()`, `getPossibleMoves()`)
- Each concrete piece class (`Pawn`, `Rook`, `Knight`, `Bishop`, `Queen`, `King`) implements its own movement logic
- The `Board` and `Game` classes work with the `Piece` interface, not concrete implementations

**Benefits**:
- Easy to add new piece types without modifying existing code
- Each piece encapsulates its own movement rules
- Polymorphic behavior allows treating all pieces uniformly

**Example**:
```java
// Client code doesn't need to know the specific piece type
Piece piece = board.getPiece(position);
List<Position> moves = piece.getPossibleMoves(board, position);
// Works for any piece type: Pawn, Rook, Queen, etc.
```

### 2. Template Method Pattern

**Usage**: The `Piece` abstract class defines the template for piece behavior.

**Implementation**:
- `Piece` provides common functionality (color, position, hasMoved tracking)
- Abstract methods (`isValidMove()`, `getPossibleMoves()`) define the template
- Concrete classes implement the specific behavior

**Benefits**:
- Code reuse for common piece functionality
- Consistent interface across all pieces
- Easy to extend with new piece types

### 3. Factory Pattern (Implicit)

**Usage**: Creating pieces during board initialization.

**Implementation**:
- The `Board.initialize()` method creates all pieces
- Could be extended to use a `PieceFactory` for more complex piece creation

**Potential Enhancement**:
```java
public class PieceFactory {
    public static Piece createPiece(PieceType type, Color color, Position position) {
        switch (type) {
            case PAWN: return new Pawn(color, position);
            case ROOK: return new Rook(color, position);
            // ... other pieces
            default: throw new IllegalArgumentException("Unknown piece type");
        }
    }
}
```

## SOLID Principles Application

### Single Responsibility Principle (SRP)

Each class has a single, well-defined responsibility:

- **`Board`**: Manages the 8x8 grid and piece placement
- **`Piece`**: Represents a chess piece and its movement rules
- **`Game`**: Orchestrates game flow and move execution
- **`GameState`**: Tracks game state (check, checkmate, stalemate)
- **`Move`**: Represents a single move
- **`Position`**: Represents a coordinate on the board
- **`Player`**: Represents a player

### Open/Closed Principle (OCP)

The design is open for extension but closed for modification:

- **Open for Extension**: New piece types can be added by extending the `Piece` class
- **Closed for Modification**: Existing code doesn't need to change when adding new pieces
- **Example**: To add a "Super Pawn" with different movement, just create a new class extending `Piece`

### Liskov Substitution Principle (LSP)

Any concrete piece can be substituted for the `Piece` abstract class:

- All concrete piece classes (`Pawn`, `Rook`, etc.) can be used wherever a `Piece` is expected
- The `Board` and `Game` classes work with `Piece` references, not specific types
- This enables polymorphic behavior

### Interface Segregation Principle (ISP)

Classes only depend on methods they actually use:

- `Board` only uses methods from `Piece` that it needs (`isValidMove()`, `getPossibleMoves()`)
- `Game` works with high-level interfaces, not implementation details
- No class is forced to depend on methods it doesn't use

### Dependency Inversion Principle (DIP)

High-level modules depend on abstractions, not concretions:

- **High-level**: `Game` and `Board` depend on the `Piece` abstraction
- **Low-level**: Concrete piece classes implement the `Piece` interface
- This allows for easy testing and flexibility

## Design Decisions and Trade-offs

### 1. Piece Movement Strategy

**Decision**: Each piece type implements its own movement logic.

**Trade-off**:
- **Pros**: Clear separation of concerns, easy to understand and maintain, follows Strategy pattern
- **Cons**: Some code duplication (e.g., path checking logic in Rook, Bishop, Queen)

**Alternative Considered**: Centralized movement logic in a `MoveValidator` class
- **Rejected because**: Would violate Open/Closed Principle, harder to extend

### 2. Board Representation

**Decision**: Use a 2D array (`Piece[8][8]`) to represent the board.

**Trade-off**:
- **Pros**: Simple, direct access O(1), easy to understand
- **Cons**: Fixed size, less flexible for variations

**Alternative Considered**: Map-based representation (`Map<Position, Piece>`)
- **Rejected because**: More complex, O(log n) access time, unnecessary for fixed-size board

### 3. Move Validation Approach

**Decision**: Validate moves by simulating them on a board copy.

**Trade-off**:
- **Pros**: Accurate validation, handles complex scenarios (check detection)
- **Cons**: Creates temporary board copies (memory overhead)

**Alternative Considered**: Rule-based validation without simulation
- **Rejected because**: More complex logic, harder to maintain, less accurate

### 4. Game State Management

**Decision**: Separate `GameState` class to track game state.

**Trade-off**:
- **Pros**: Clear separation of concerns, easy to query state, follows SRP
- **Cons**: Additional class, need to keep state synchronized with board

**Alternative Considered**: Embed state in `Game` class
- **Rejected because**: Would violate SRP, harder to test state logic independently

### 5. Position Immutability

**Decision**: `Position` class is immutable (final fields).

**Trade-off**:
- **Pros**: Thread-safe, prevents accidental modification, can be used as map keys
- **Cons**: Need to create new instances for different positions

**Alternative Considered**: Mutable Position class
- **Rejected because**: Risk of bugs from accidental modification, not thread-safe

## Why These Patterns Were Chosen

### Strategy Pattern for Piece Movement

**Reason**: Chess pieces have fundamentally different movement rules. The Strategy pattern allows each piece to encapsulate its own movement logic while maintaining a consistent interface. This makes the code:
- Easy to understand (each piece's logic is self-contained)
- Easy to extend (add new pieces without modifying existing code)
- Easy to test (test each piece type independently)

### Template Method Pattern in Piece Class

**Reason**: All pieces share common attributes (color, position, hasMoved) and behavior (getting color, position). The Template Method pattern allows:
- Code reuse for common functionality
- Consistent interface across all pieces
- Easy addition of new pieces by extending the base class

### Separation of Concerns

**Reason**: Breaking the system into focused classes (`Board`, `Game`, `GameState`, `Piece`) makes the code:
- Easier to understand (each class has a clear purpose)
- Easier to maintain (changes are localized)
- Easier to test (can test each component independently)
- More flexible (can modify one component without affecting others)

## Extensibility Considerations

The design allows for easy extension in several ways:

1. **New Piece Types**: Simply extend the `Piece` class and implement movement logic
2. **Custom Rules**: Can extend `Game` class to add custom rules
3. **Different Board Sizes**: Could make `Board` size configurable (though would require more changes)
4. **Move History**: `GameState` already tracks moves, easy to add undo/redo
5. **AI Players**: `Player` class can be extended to create AI players
6. **Special Game Modes**: Can extend `Game` class for variants (Chess960, etc.)

## Performance Considerations

1. **Move Validation**: Uses board copying for validation - acceptable for single-threaded game, but could be optimized with move undo mechanism
2. **Check Detection**: Iterates through all pieces - O(n) where n is number of pieces, acceptable for 32 pieces
3. **Possible Moves**: Calculated on-demand - could be cached for performance if needed
4. **Board Representation**: 2D array provides O(1) access, optimal for this use case

## Testing Considerations

The design supports testability:

1. **Dependency Injection**: Classes accept dependencies through constructors
2. **Immutable Value Objects**: `Position` is immutable and easy to test
3. **Separation of Concerns**: Each class can be tested independently
4. **Interface-Based Design**: Can easily mock `Piece` objects for testing
5. **Pure Functions**: Many methods are pure functions (no side effects), easy to test

