# Chess Game - Game Rules and Edge Cases

## Standard Chess Rules Implementation

### Basic Movement Rules

1. **Pawn Movement**
   - Moves forward one square (toward opponent)
   - On first move, can move two squares forward
   - Captures diagonally one square forward
   - Cannot move backward
   - Cannot capture forward

2. **Rook Movement**
   - Moves horizontally or vertically any number of squares
   - Cannot jump over pieces
   - Captures by moving to an occupied square

3. **Knight Movement**
   - Moves in L-shape: 2 squares in one direction, 1 square perpendicular
   - Can jump over pieces
   - Only piece that can jump over other pieces

4. **Bishop Movement**
   - Moves diagonally any number of squares
   - Cannot jump over pieces
   - Stays on the same color squares

5. **Queen Movement**
   - Combines Rook and Bishop movement
   - Moves horizontally, vertically, or diagonally any number of squares
   - Cannot jump over pieces

6. **King Movement**
   - Moves one square in any direction
   - Cannot move into check
   - Special move: Castling (see below)

### Special Moves

#### 1. Castling

**Rules**:
- King and rook must not have moved previously
- No pieces between king and rook
- King must not be in check
- King must not pass through check
- King must not end up in check

**Types**:
- **Kingside Castling**: King moves 2 squares toward the rook on the right, rook moves to the square next to the king
- **Queenside Castling**: King moves 2 squares toward the rook on the left, rook moves to the square next to the king

**Implementation Considerations**:
- Track `hasMoved` flag for both king and rook
- Validate all castling conditions before allowing the move
- Move both pieces in a single move operation

#### 2. En Passant

**Rules**:
- Occurs when a pawn moves two squares forward from its starting position
- An opponent pawn on an adjacent file can capture it "en passant" on the next move only
- The capturing pawn moves diagonally to the square the opponent pawn passed over

**Implementation Considerations**:
- Track the last move to detect two-square pawn moves
- Store the "en passant target square" in game state
- Validate en passant capture on the move immediately following the two-square pawn move
- Remove the captured pawn from the square it passed over, not the destination square

#### 3. Pawn Promotion

**Rules**:
- When a pawn reaches the opposite end of the board (8th rank for white, 1st rank for black)
- Pawn must be promoted to Queen, Rook, Bishop, or Knight
- Cannot remain a pawn or be promoted to a King
- Promotion is mandatory

**Implementation Considerations**:
- Detect when a pawn reaches the promotion rank
- Prompt player to choose promotion piece (or default to Queen)
- Replace pawn with chosen piece immediately
- Handle promotion in the move execution logic

### Game State Rules

#### Check

**Definition**: The king is under attack by an opponent's piece.

**Rules**:
- Player in check must make a move that removes the check
- Cannot make a move that leaves or puts own king in check
- Must be resolved on the next move

**Detection**:
- After each move, check if opponent's king is under attack
- Use `isSquareUnderAttack()` method to check if king's position is attacked
- Update game state to reflect check condition

#### Checkmate

**Definition**: The king is in check and there are no legal moves to escape.

**Rules**:
- Game ends immediately
- Player in checkmate loses
- Opponent wins

**Detection**:
1. Verify king is in check
2. Generate all possible moves for the player
3. For each move, simulate it and check if king is still in check
4. If no move removes the check, it's checkmate

#### Stalemate

**Definition**: The player to move is not in check but has no legal moves.

**Rules**:
- Game ends in a draw
- Neither player wins

**Detection**:
1. Verify player is NOT in check
2. Generate all possible moves for the player
3. For each move, simulate it and check if it's legal (doesn't leave king in check)
4. If no legal moves exist, it's stalemate

## Edge Cases and Validation

### 1. Invalid Move Scenarios

**Case**: Moving opponent's piece
- **Validation**: Check that piece at source position belongs to current player
- **Error Handling**: Return false or throw exception with clear message

**Case**: Moving to same position
- **Validation**: Source and destination must be different
- **Error Handling**: Reject move as invalid

**Case**: Moving piece off the board
- **Validation**: Both source and destination must be within board bounds (0-7)
- **Error Handling**: `Position` constructor throws exception for invalid coordinates

**Case**: Moving through own pieces
- **Validation**: Path must be clear (except for Knight which can jump)
- **Error Handling**: Piece's `isValidMove()` returns false

**Case**: Capturing own piece
- **Validation**: Destination square cannot contain a piece of the same color
- **Error Handling**: `isEmptyOrOpponent()` check in piece validation

### 2. Check-Related Edge Cases

**Case**: Moving into check
- **Validation**: After simulating move, check if own king is in check
- **Error Handling**: Move is rejected, must choose different move

**Case**: Moving piece that exposes king to check
- **Validation**: Simulate move and check if king is attacked
- **Error Handling**: Move is rejected (e.g., moving a piece that was blocking check)

**Case**: Multiple pieces giving check
- **Validation**: Must capture checking piece or block all checking pieces
- **Error Handling**: Only moves that resolve all checks are valid

**Case**: King cannot escape check
- **Validation**: Check all possible king moves, all leave king in check
- **Error Handling**: Results in checkmate

### 3. Special Move Edge Cases

**Castling Edge Cases**:
- **King has moved**: Cannot castle
- **Rook has moved**: Cannot castle with that rook
- **Pieces between**: Cannot castle if any pieces between king and rook
- **King in check**: Cannot castle to escape check
- **King passes through check**: Cannot castle if king would pass through an attacked square
- **King ends in check**: Cannot castle if final position is under attack

**En Passant Edge Cases**:
- **Not immediately after two-square move**: En passant only available on the very next move
- **No adjacent pawn**: No pawn on adjacent file to capture en passant
- **Wrong rank**: Capturing pawn must be on the correct rank (5th for white, 4th for black)

**Pawn Promotion Edge Cases**:
- **Reaching promotion rank**: Must promote, cannot remain a pawn
- **Invalid promotion piece**: Cannot promote to King or another Pawn
- **Promotion during check**: Must still promote even if in check

### 4. Turn Management Edge Cases

**Case**: Player tries to move out of turn
- **Validation**: Check `gameState.getCurrentPlayer()` matches piece color
- **Error Handling**: Reject move with appropriate error message

**Case**: Game already over
- **Validation**: Check `gameState.isGameOver()` before allowing moves
- **Error Handling**: Reject move, inform player game has ended

### 5. Board State Edge Cases

**Case**: Piece not found at source position
- **Validation**: Check `board.getPiece(from)` is not null
- **Error Handling**: Throw exception or return false

**Case**: King missing from board
- **Validation**: Should never happen in valid game, but handle gracefully
- **Error Handling**: Return null from `getKingPosition()`, handle in calling code

**Case**: Multiple pieces of same type and color
- **Validation**: This is normal (e.g., two rooks, eight pawns)
- **Implementation**: Handle by checking all pieces of that type

### 6. Move History Edge Cases

**Case**: Undo functionality (if implemented)
- **Consideration**: Need to store enough information to restore previous state
- **Implementation**: Store full board state or just moves with ability to reverse

**Case**: Move history size
- **Consideration**: Games can have many moves (50+ moves common)
- **Implementation**: Use efficient data structure (ArrayList), consider limiting history size

## Implementation Details for Special Rules

### Check Detection Algorithm

```java
public boolean isInCheck(Board board, Color color) {
    Position kingPos = board.getKingPosition(color);
    if (kingPos == null) return false;
    
    // Check if any opponent piece can attack the king
    return board.isSquareUnderAttack(kingPos, color.opposite());
}
```

### Checkmate Detection Algorithm

```java
public boolean isInCheckmate(Board board, Color color) {
    // Must be in check
    if (!isInCheck(board, color)) return false;
    
    // Must have no valid moves
    List<Move> validMoves = getAllValidMoves(board, color);
    return validMoves.isEmpty();
}
```

### Stalemate Detection Algorithm

```java
public boolean isStalemate(Board board, Color color) {
    // Must NOT be in check
    if (isInCheck(board, color)) return false;
    
    // Must have no valid moves
    List<Move> validMoves = getAllValidMoves(board, color);
    return validMoves.isEmpty();
}
```

### Valid Move Generation

```java
public List<Move> getAllValidMoves(Board board, Color color) {
    List<Move> validMoves = new ArrayList<>();
    
    // For each piece of the current color
    for (each piece of color) {
        // Get possible moves for the piece
        List<Position> possibleMoves = piece.getPossibleMoves(board, from);
        
        // For each possible move
        for (Position to : possibleMoves) {
            // Simulate the move
            Board testBoard = board.copy();
            testBoard.movePiece(from, to);
            
            // Check if move leaves king in check
            if (!isInCheck(testBoard, color)) {
                validMoves.add(new Move(from, to, piece));
            }
        }
    }
    
    return validMoves;
}
```

## Move Validation Flow

1. **Basic Validation**:
   - Check if source position has a piece
   - Check if piece belongs to current player
   - Check if it's the player's turn
   - Check if game is not over

2. **Piece-Specific Validation**:
   - Check if move follows piece's movement rules
   - Check if path is clear (for pieces that can't jump)
   - Check if destination is valid (empty or opponent piece)

3. **Special Move Validation**:
   - Castling: Validate all castling conditions
   - En Passant: Validate en passant conditions
   - Pawn Promotion: Detect and handle promotion

4. **Check Validation**:
   - Simulate the move on a board copy
   - Check if move leaves own king in check
   - Reject move if king would be in check

5. **Execute Move**:
   - Move piece to destination
   - Handle capture if applicable
   - Handle special moves (castling, en passant, promotion)
   - Update piece's `hasMoved` flag
   - Update game state

## Error Handling Strategy

1. **Invalid Input**: Throw `IllegalArgumentException` with descriptive message
2. **Invalid Move**: Return `false` from `makeMove()`, don't throw exception
3. **Game State Errors**: Return appropriate boolean or enum value
4. **Null Checks**: Always validate null before operations
5. **Bounds Checking**: Validate positions are within board bounds

## Testing Considerations

Key test cases to cover:

1. **Normal Moves**: All piece types moving normally
2. **Captures**: All piece types capturing opponent pieces
3. **Check Scenarios**: Various check situations
4. **Checkmate Scenarios**: Different checkmate patterns
5. **Stalemate Scenarios**: Various stalemate situations
6. **Castling**: Both kingside and queenside, all edge cases
7. **En Passant**: Valid and invalid en passant attempts
8. **Pawn Promotion**: All promotion piece types
9. **Invalid Moves**: All types of invalid moves
10. **Edge Cases**: Boundary conditions, null checks, etc.

