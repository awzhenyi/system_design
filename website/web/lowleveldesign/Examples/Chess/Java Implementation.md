# Chess Game - Java Implementation

## Package Structure

```
com.chess
├── model
│   ├── Color.java
│   ├── PieceType.java
│   ├── Position.java
│   ├── Piece.java
│   ├── Pawn.java
│   ├── Rook.java
│   ├── Knight.java
│   ├── Bishop.java
│   ├── Queen.java
│   ├── King.java
│   ├── Board.java
│   ├── Move.java
│   └── GameState.java
├── player
│   └── Player.java
└── game
    └── Game.java
```

## Core Classes Implementation

### Color.java

```java
package com.chess.model;

public enum Color {
    WHITE,
    BLACK;
    
    public Color opposite() {
        return this == WHITE ? BLACK : WHITE;
    }
}
```

### PieceType.java

```java
package com.chess.model;

public enum PieceType {
    PAWN,
    ROOK,
    KNIGHT,
    BISHOP,
    QUEEN,
    KING
}
```

### Position.java

```java
package com.chess.model;

public class Position {
    private final int row;
    private final int col;
    private static final int BOARD_SIZE = 8;
    
    public Position(int row, int col) {
        if (!isValid(row, col)) {
            throw new IllegalArgumentException("Invalid position: row=" + row + ", col=" + col);
        }
        this.row = row;
        this.col = col;
    }
    
    public int getRow() {
        return row;
    }
    
    public int getCol() {
        return col;
    }
    
    public boolean isValid() {
        return isValid(row, col);
    }
    
    private static boolean isValid(int row, int col) {
        return row >= 0 && row < BOARD_SIZE && col >= 0 && col < BOARD_SIZE;
    }
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Position position = (Position) obj;
        return row == position.row && col == position.col;
    }
    
    @Override
    public int hashCode() {
        return row * 31 + col;
    }
    
    @Override
    public String toString() {
        return "(" + row + ", " + col + ")";
    }
}
```

### Piece.java (Abstract Class)

```java
package com.chess.model;

import java.util.List;

public abstract class Piece {
    protected Color color;
    protected Position position;
    protected boolean hasMoved;
    
    public Piece(Color color, Position position) {
        this.color = color;
        this.position = position;
        this.hasMoved = false;
    }
    
    public Color getColor() {
        return color;
    }
    
    public Position getPosition() {
        return position;
    }
    
    public void setPosition(Position position) {
        this.position = position;
        this.hasMoved = true;
    }
    
    public boolean hasMoved() {
        return hasMoved;
    }
    
    public abstract boolean isValidMove(Board board, Position from, Position to);
    
    public abstract List<Position> getPossibleMoves(Board board, Position from);
    
    public abstract PieceType getType();
    
    protected boolean isOpponentPiece(Board board, Position pos) {
        Piece piece = board.getPiece(pos);
        return piece != null && piece.getColor() != this.color;
    }
    
    protected boolean isOwnPiece(Board board, Position pos) {
        Piece piece = board.getPiece(pos);
        return piece != null && piece.getColor() == this.color;
    }
    
    protected boolean isEmptyOrOpponent(Board board, Position pos) {
        return board.isSquareEmpty(pos) || isOpponentPiece(board, pos);
    }
}
```

### Pawn.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Pawn extends Piece {
    
    public Pawn(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        int direction = color == Color.WHITE ? -1 : 1;
        int startRow = color == Color.WHITE ? 6 : 1;
        
        // Forward move (one square)
        if (to.getCol() == from.getCol()) {
            if (to.getRow() == from.getRow() + direction) {
                return board.isSquareEmpty(to);
            }
            // Forward move (two squares from starting position)
            if (from.getRow() == startRow && to.getRow() == from.getRow() + 2 * direction) {
                Position intermediate = new Position(from.getRow() + direction, from.getCol());
                return board.isSquareEmpty(intermediate) && board.isSquareEmpty(to);
            }
        }
        
        // Diagonal capture
        if (Math.abs(to.getCol() - from.getCol()) == 1 && 
            to.getRow() == from.getRow() + direction) {
            return isOpponentPiece(board, to);
        }
        
        return false;
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        int direction = color == Color.WHITE ? -1 : 1;
        int startRow = color == Color.WHITE ? 6 : 1;
        
        // Forward one square
        Position oneForward = new Position(from.getRow() + direction, from.getCol());
        if (board.isValidPosition(oneForward) && board.isSquareEmpty(oneForward)) {
            moves.add(oneForward);
            
            // Forward two squares from starting position
            if (from.getRow() == startRow) {
                Position twoForward = new Position(from.getRow() + 2 * direction, from.getCol());
                if (board.isValidPosition(twoForward) && board.isSquareEmpty(twoForward)) {
                    moves.add(twoForward);
                }
            }
        }
        
        // Diagonal captures
        for (int colOffset = -1; colOffset <= 1; colOffset += 2) {
            Position capturePos = new Position(from.getRow() + direction, from.getCol() + colOffset);
            if (board.isValidPosition(capturePos) && isOpponentPiece(board, capturePos)) {
                moves.add(capturePos);
            }
        }
        
        return moves;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.PAWN;
    }
}
```

### Rook.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Rook extends Piece {
    
    public Rook(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        // Rook moves horizontally or vertically
        if (from.getRow() != to.getRow() && from.getCol() != to.getCol()) {
            return false;
        }
        
        // Check if path is clear
        return isPathClear(board, from, to) && isEmptyOrOpponent(board, to);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}}; // up, down, left, right
        
        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(from.getRow() + dir[0] * i, 
                                               from.getCol() + dir[1] * i);
                if (!board.isValidPosition(newPos)) break;
                if (isOwnPiece(board, newPos)) break;
                moves.add(newPos);
                if (isOpponentPiece(board, newPos)) break;
            }
        }
        
        return moves;
    }
    
    private boolean isPathClear(Board board, Position from, Position to) {
        int rowStep = Integer.compare(to.getRow(), from.getRow());
        int colStep = Integer.compare(to.getCol(), from.getCol());
        
        int currentRow = from.getRow() + rowStep;
        int currentCol = from.getCol() + colStep;
        
        while (currentRow != to.getRow() || currentCol != to.getCol()) {
            Position pos = new Position(currentRow, currentCol);
            if (!board.isSquareEmpty(pos)) {
                return false;
            }
            currentRow += rowStep;
            currentCol += colStep;
        }
        
        return true;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.ROOK;
    }
}
```

### Knight.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Knight extends Piece {
    
    public Knight(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());
        
        // Knight moves in L-shape: 2 squares in one direction, 1 square perpendicular
        boolean isValidLShape = (rowDiff == 2 && colDiff == 1) || (rowDiff == 1 && colDiff == 2);
        
        return isValidLShape && isEmptyOrOpponent(board, to);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        int[][] knightMoves = {
            {-2, -1}, {-2, 1}, {-1, -2}, {-1, 2},
            {1, -2}, {1, 2}, {2, -1}, {2, 1}
        };
        
        for (int[] move : knightMoves) {
            Position newPos = new Position(from.getRow() + move[0], from.getCol() + move[1]);
            if (board.isValidPosition(newPos) && isEmptyOrOpponent(board, newPos)) {
                moves.add(newPos);
            }
        }
        
        return moves;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.KNIGHT;
    }
}
```

### Bishop.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Bishop extends Piece {
    
    public Bishop(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        // Bishop moves diagonally
        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());
        
        if (rowDiff != colDiff) {
            return false;
        }
        
        // Check if path is clear
        return isPathClear(board, from, to) && isEmptyOrOpponent(board, to);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {{-1, -1}, {-1, 1}, {1, -1}, {1, 1}}; // diagonals
        
        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(from.getRow() + dir[0] * i, 
                                               from.getCol() + dir[1] * i);
                if (!board.isValidPosition(newPos)) break;
                if (isOwnPiece(board, newPos)) break;
                moves.add(newPos);
                if (isOpponentPiece(board, newPos)) break;
            }
        }
        
        return moves;
    }
    
    private boolean isPathClear(Board board, Position from, Position to) {
        int rowStep = Integer.compare(to.getRow(), from.getRow());
        int colStep = Integer.compare(to.getCol(), from.getCol());
        
        int currentRow = from.getRow() + rowStep;
        int currentCol = from.getCol() + colStep;
        
        while (currentRow != to.getRow() || currentCol != to.getCol()) {
            Position pos = new Position(currentRow, currentCol);
            if (!board.isSquareEmpty(pos)) {
                return false;
            }
            currentRow += rowStep;
            currentCol += colStep;
        }
        
        return true;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.BISHOP;
    }
}
```

### Queen.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Queen extends Piece {
    
    public Queen(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        // Queen combines Rook and Bishop movement
        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());
        
        // Horizontal, vertical, or diagonal
        boolean isStraight = (from.getRow() == to.getRow() || from.getCol() == to.getCol());
        boolean isDiagonal = (rowDiff == colDiff);
        
        if (!isStraight && !isDiagonal) {
            return false;
        }
        
        // Check if path is clear
        return isPathClear(board, from, to) && isEmptyOrOpponent(board, to);
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        // All 8 directions: horizontal, vertical, and diagonal
        int[][] directions = {
            {-1, -1}, {-1, 0}, {-1, 1},
            {0, -1},           {0, 1},
            {1, -1},  {1, 0},  {1, 1}
        };
        
        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(from.getRow() + dir[0] * i, 
                                               from.getCol() + dir[1] * i);
                if (!board.isValidPosition(newPos)) break;
                if (isOwnPiece(board, newPos)) break;
                moves.add(newPos);
                if (isOpponentPiece(board, newPos)) break;
            }
        }
        
        return moves;
    }
    
    private boolean isPathClear(Board board, Position from, Position to) {
        int rowStep = Integer.compare(to.getRow(), from.getRow());
        int colStep = Integer.compare(to.getCol(), from.getCol());
        
        int currentRow = from.getRow() + rowStep;
        int currentCol = from.getCol() + colStep;
        
        while (currentRow != to.getRow() || currentCol != to.getCol()) {
            Position pos = new Position(currentRow, currentCol);
            if (!board.isSquareEmpty(pos)) {
                return false;
            }
            currentRow += rowStep;
            currentCol += colStep;
        }
        
        return true;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.QUEEN;
    }
}
```

### King.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class King extends Piece {
    
    public King(Color color, Position position) {
        super(color, position);
    }
    
    @Override
    public boolean isValidMove(Board board, Position from, Position to) {
        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());
        
        // King moves one square in any direction
        if (rowDiff <= 1 && colDiff <= 1 && (rowDiff + colDiff > 0)) {
            return isEmptyOrOpponent(board, to);
        }
        
        // Castling (handled separately in Game class)
        return false;
    }
    
    @Override
    public List<Position> getPossibleMoves(Board board, Position from) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {
            {-1, -1}, {-1, 0}, {-1, 1},
            {0, -1},           {0, 1},
            {1, -1},  {1, 0},  {1, 1}
        };
        
        for (int[] dir : directions) {
            Position newPos = new Position(from.getRow() + dir[0], from.getCol() + dir[1]);
            if (board.isValidPosition(newPos) && isEmptyOrOpponent(board, newPos)) {
                moves.add(newPos);
            }
        }
        
        return moves;
    }
    
    public boolean canCastle(Board board, boolean kingside) {
        if (hasMoved) return false;
        
        int row = color == Color.WHITE ? 7 : 0;
        int rookCol = kingside ? 7 : 0;
        Position rookPos = new Position(row, rookCol);
        Piece rook = board.getPiece(rookPos);
        
        if (rook == null || rook.getType() != PieceType.ROOK || rook.hasMoved()) {
            return false;
        }
        
        // Check if squares between king and rook are empty
        int startCol = Math.min(position.getCol(), rookCol) + 1;
        int endCol = Math.max(position.getCol(), rookCol);
        for (int col = startCol; col < endCol; col++) {
            if (!board.isSquareEmpty(new Position(row, col))) {
                return false;
            }
        }
        
        // Check if king is not in check and doesn't pass through check
        if (board.isSquareUnderAttack(position, color.opposite())) {
            return false;
        }
        
        int kingStepCol = kingside ? 1 : -1;
        Position intermediatePos = new Position(row, position.getCol() + kingStepCol);
        if (board.isSquareUnderAttack(intermediatePos, color.opposite())) {
            return false;
        }
        
        return true;
    }
    
    @Override
    public PieceType getType() {
        return PieceType.KING;
    }
}
```

### Board.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class Board {
    private static final int SIZE = 8;
    private Piece[][] squares;
    
    public Board() {
        squares = new Piece[SIZE][SIZE];
    }
    
    public Piece getPiece(Position position) {
        if (!isValidPosition(position)) {
            return null;
        }
        return squares[position.getRow()][position.getCol()];
    }
    
    public void setPiece(Position position, Piece piece) {
        if (!isValidPosition(position)) {
            throw new IllegalArgumentException("Invalid position");
        }
        squares[position.getRow()][position.getCol()] = piece;
        if (piece != null) {
            piece.setPosition(position);
        }
    }
    
    public void removePiece(Position position) {
        setPiece(position, null);
    }
    
    public void movePiece(Position from, Position to) {
        Piece piece = getPiece(from);
        if (piece == null) {
            throw new IllegalArgumentException("No piece at source position");
        }
        removePiece(from);
        setPiece(to, piece);
    }
    
    public boolean isValidPosition(Position position) {
        return position != null && position.isValid();
    }
    
    public boolean isSquareEmpty(Position position) {
        return getPiece(position) == null;
    }
    
    public boolean isSquareOccupied(Position position) {
        return !isSquareEmpty(position);
    }
    
    public Position getKingPosition(Color color) {
        for (int row = 0; row < SIZE; row++) {
            for (int col = 0; col < SIZE; col++) {
                Position pos = new Position(row, col);
                Piece piece = getPiece(pos);
                if (piece != null && piece.getType() == PieceType.KING && piece.getColor() == color) {
                    return pos;
                }
            }
        }
        return null; // Should never happen in a valid game
    }
    
    public boolean isSquareUnderAttack(Position position, Color byColor) {
        for (int row = 0; row < SIZE; row++) {
            for (int col = 0; col < SIZE; col++) {
                Position pos = new Position(row, col);
                Piece piece = getPiece(pos);
                if (piece != null && piece.getColor() == byColor) {
                    if (piece.isValidMove(this, pos, position)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }
    
    public Board copy() {
        Board copy = new Board();
        for (int row = 0; row < SIZE; row++) {
            for (int col = 0; col < SIZE; col++) {
                Position pos = new Position(row, col);
                Piece piece = getPiece(pos);
                if (piece != null) {
                    // Create a new piece instance (simplified - in real implementation, 
                    // you'd need a factory or clone method)
                    copy.setPiece(pos, piece);
                }
            }
        }
        return copy;
    }
    
    public void initialize() {
        // Initialize white pieces
        setPiece(new Position(7, 0), new Rook(Color.WHITE, new Position(7, 0)));
        setPiece(new Position(7, 1), new Knight(Color.WHITE, new Position(7, 1)));
        setPiece(new Position(7, 2), new Bishop(Color.WHITE, new Position(7, 2)));
        setPiece(new Position(7, 3), new Queen(Color.WHITE, new Position(7, 3)));
        setPiece(new Position(7, 4), new King(Color.WHITE, new Position(7, 4)));
        setPiece(new Position(7, 5), new Bishop(Color.WHITE, new Position(7, 5)));
        setPiece(new Position(7, 6), new Knight(Color.WHITE, new Position(7, 6)));
        setPiece(new Position(7, 7), new Rook(Color.WHITE, new Position(7, 7)));
        for (int col = 0; col < SIZE; col++) {
            setPiece(new Position(6, col), new Pawn(Color.WHITE, new Position(6, col)));
        }
        
        // Initialize black pieces
        setPiece(new Position(0, 0), new Rook(Color.BLACK, new Position(0, 0)));
        setPiece(new Position(0, 1), new Knight(Color.BLACK, new Position(0, 1)));
        setPiece(new Position(0, 2), new Bishop(Color.BLACK, new Position(0, 2)));
        setPiece(new Position(0, 3), new Queen(Color.BLACK, new Position(0, 3)));
        setPiece(new Position(0, 4), new King(Color.BLACK, new Position(0, 4)));
        setPiece(new Position(0, 5), new Bishop(Color.BLACK, new Position(0, 5)));
        setPiece(new Position(0, 6), new Knight(Color.BLACK, new Position(0, 6)));
        setPiece(new Position(0, 7), new Rook(Color.BLACK, new Position(0, 7)));
        for (int col = 0; col < SIZE; col++) {
            setPiece(new Position(1, col), new Pawn(Color.BLACK, new Position(1, col)));
        }
    }
    
    public int getSize() {
        return SIZE;
    }
}
```

### Move.java

```java
package com.chess.model;

public class Move {
    private Position from;
    private Position to;
    private Piece piece;
    private Piece capturedPiece;
    private boolean isCastling;
    private boolean isEnPassant;
    private PieceType promotionPiece;
    
    public Move(Position from, Position to, Piece piece) {
        this.from = from;
        this.to = to;
        this.piece = piece;
        this.isCastling = false;
        this.isEnPassant = false;
        this.promotionPiece = null;
    }
    
    public Position getFrom() {
        return from;
    }
    
    public Position getTo() {
        return to;
    }
    
    public Piece getPiece() {
        return piece;
    }
    
    public Piece getCapturedPiece() {
        return capturedPiece;
    }
    
    public void setCapturedPiece(Piece capturedPiece) {
        this.capturedPiece = capturedPiece;
    }
    
    public boolean isCastling() {
        return isCastling;
    }
    
    public void setCastling(boolean castling) {
        isCastling = castling;
    }
    
    public boolean isEnPassant() {
        return isEnPassant;
    }
    
    public void setEnPassant(boolean enPassant) {
        isEnPassant = enPassant;
    }
    
    public PieceType getPromotionPiece() {
        return promotionPiece;
    }
    
    public void setPromotionPiece(PieceType promotionPiece) {
        this.promotionPiece = promotionPiece;
    }
    
    public boolean isCapture() {
        return capturedPiece != null;
    }
    
    @Override
    public String toString() {
        return piece.getType() + " from " + from + " to " + to;
    }
}
```

### GameState.java

```java
package com.chess.model;

import java.util.ArrayList;
import java.util.List;

public class GameState {
    private Color currentPlayer;
    private boolean isCheck;
    private boolean isCheckmate;
    private boolean isStalemate;
    private boolean isGameOver;
    private Color winner;
    private List<Move> moveHistory;
    
    public GameState() {
        this.currentPlayer = Color.WHITE;
        this.isCheck = false;
        this.isCheckmate = false;
        this.isStalemate = false;
        this.isGameOver = false;
        this.winner = null;
        this.moveHistory = new ArrayList<>();
    }
    
    public Color getCurrentPlayer() {
        return currentPlayer;
    }
    
    public void switchTurn() {
        currentPlayer = currentPlayer.opposite();
    }
    
    public void updateState(Board board) {
        isCheck = isInCheck(board, currentPlayer);
        List<Move> validMoves = getAllValidMoves(board, currentPlayer);
        
        if (validMoves.isEmpty()) {
            if (isCheck) {
                isCheckmate = true;
                isGameOver = true;
                winner = currentPlayer.opposite();
            } else {
                isStalemate = true;
                isGameOver = true;
            }
        } else {
            isCheckmate = false;
            isStalemate = false;
        }
    }
    
    public boolean isInCheck(Board board, Color color) {
        Position kingPos = board.getKingPosition(color);
        if (kingPos == null) {
            return false;
        }
        return board.isSquareUnderAttack(kingPos, color.opposite());
    }
    
    public boolean isInCheckmate(Board board, Color color) {
        if (!isInCheck(board, color)) {
            return false;
        }
        return getAllValidMoves(board, color).isEmpty();
    }
    
    public boolean isStalemate(Board board, Color color) {
        if (isInCheck(board, color)) {
            return false;
        }
        return getAllValidMoves(board, color).isEmpty();
    }
    
    public List<Move> getAllValidMoves(Board board, Color color) {
        List<Move> validMoves = new ArrayList<>();
        
        for (int row = 0; row < board.getSize(); row++) {
            for (int col = 0; col < board.getSize(); col++) {
                Position from = new Position(row, col);
                Piece piece = board.getPiece(from);
                
                if (piece != null && piece.getColor() == color) {
                    List<Position> possibleMoves = piece.getPossibleMoves(board, from);
                    
                    for (Position to : possibleMoves) {
                        Move move = new Move(from, to, piece);
                        
                        // Simulate move to check if it leaves king in check
                        Board testBoard = board.copy();
                        testBoard.movePiece(from, to);
                        
                        if (!isInCheck(testBoard, color)) {
                            validMoves.add(move);
                        }
                    }
                }
            }
        }
        
        return validMoves;
    }
    
    public void addMove(Move move) {
        moveHistory.add(move);
    }
    
    public List<Move> getMoveHistory() {
        return new ArrayList<>(moveHistory);
    }
    
    public boolean isCheck() {
        return isCheck;
    }
    
    public boolean isCheckmate() {
        return isCheckmate;
    }
    
    public boolean isStalemate() {
        return isStalemate;
    }
    
    public boolean isGameOver() {
        return isGameOver;
    }
    
    public Color getWinner() {
        return winner;
    }
}
```

### Player.java

```java
package com.chess.player;

import com.chess.model.Board;
import com.chess.model.Color;
import com.chess.model.Move;

public class Player {
    private Color color;
    private String name;
    
    public Player(Color color, String name) {
        this.color = color;
        this.name = name;
    }
    
    public Color getColor() {
        return color;
    }
    
    public String getName() {
        return name;
    }
    
    public void makeMove(Board board, Move move) {
        // Move is executed by the Game class
        // This method can be used for player-specific logic
    }
}
```

### Game.java

```java
package com.chess.game;

import com.chess.model.*;
import com.chess.player.Player;

public class Game {
    private Board board;
    private Player whitePlayer;
    private Player blackPlayer;
    private GameState gameState;
    
    public Game(Player whitePlayer, Player blackPlayer) {
        this.whitePlayer = whitePlayer;
        this.blackPlayer = blackPlayer;
        this.board = new Board();
        this.gameState = new GameState();
        initializeGame();
    }
    
    public void initializeGame() {
        board.initialize();
        gameState = new GameState();
    }
    
    public boolean makeMove(Move move) {
        if (!isValidMove(move)) {
            return false;
        }
        
        // Execute the move
        Piece piece = board.getPiece(move.getFrom());
        Piece capturedPiece = board.getPiece(move.getTo());
        move.setCapturedPiece(capturedPiece);
        
        board.movePiece(move.getFrom(), move.getTo());
        
        // Handle pawn promotion
        if (piece.getType() == PieceType.PAWN && 
            (move.getTo().getRow() == 0 || move.getTo().getRow() == 7)) {
            if (move.getPromotionPiece() != null) {
                promotePawn(move.getTo(), move.getPromotionPiece(), piece.getColor());
            }
        }
        
        // Update game state
        gameState.addMove(move);
        gameState.switchTurn();
        gameState.updateState(board);
        
        return true;
    }
    
    public boolean isValidMove(Move move) {
        // Check if it's the current player's turn
        Piece piece = board.getPiece(move.getFrom());
        if (piece == null || piece.getColor() != gameState.getCurrentPlayer()) {
            return false;
        }
        
        // Check if move is valid for the piece
        if (!piece.isValidMove(board, move.getFrom(), move.getTo())) {
            return false;
        }
        
        // Simulate move to check if it leaves king in check
        Board testBoard = board.copy();
        testBoard.movePiece(move.getFrom(), move.getTo());
        
        if (gameState.isInCheck(testBoard, gameState.getCurrentPlayer())) {
            return false;
        }
        
        return true;
    }
    
    private void promotePawn(Position position, PieceType promotionType, Color color) {
        Piece newPiece = null;
        switch (promotionType) {
            case QUEEN:
                newPiece = new Queen(color, position);
                break;
            case ROOK:
                newPiece = new Rook(color, position);
                break;
            case BISHOP:
                newPiece = new Bishop(color, position);
                break;
            case KNIGHT:
                newPiece = new Knight(color, position);
                break;
            default:
                newPiece = new Queen(color, position); // Default to Queen
        }
        board.setPiece(position, newPiece);
    }
    
    public Player getCurrentPlayer() {
        return gameState.getCurrentPlayer() == Color.WHITE ? whitePlayer : blackPlayer;
    }
    
    public GameState getGameState() {
        return gameState;
    }
    
    public Board getBoard() {
        return board;
    }
    
    public boolean isGameOver() {
        return gameState.isGameOver();
    }
    
    public Color getWinner() {
        return gameState.getWinner();
    }
}
```

## Example Usage

```java
package com.chess;

import com.chess.game.Game;
import com.chess.model.*;
import com.chess.player.Player;

public class ChessGameExample {
    public static void main(String[] args) {
        // Create players
        Player whitePlayer = new Player(Color.WHITE, "Alice");
        Player blackPlayer = new Player(Color.BLACK, "Bob");
        
        // Initialize game
        Game game = new Game(whitePlayer, blackPlayer);
        
        // Make some moves
        Position from = new Position(6, 4); // White pawn at e2
        Position to = new Position(4, 4);   // Move to e4
        Piece piece = game.getBoard().getPiece(from);
        Move move1 = new Move(from, to, piece);
        
        if (game.makeMove(move1)) {
            System.out.println("Move successful: " + move1);
        } else {
            System.out.println("Invalid move");
        }
        
        // Check game state
        GameState state = game.getGameState();
        System.out.println("Current player: " + state.getCurrentPlayer());
        System.out.println("Is check: " + state.isCheck());
        System.out.println("Is game over: " + state.isGameOver());
        
        // Continue game...
    }
}
```

