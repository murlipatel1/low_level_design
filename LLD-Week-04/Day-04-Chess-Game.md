# Day 4 - Chess Game

**Theme:** Build a maintainable chess rules engine with legal-move validation, stateful special rules, and robust undo/history.

---

## Learning goals

By the end of this document, you should be able to:

- Separate board model, move validation, and game orchestration cleanly.
- Implement command-based move execution with undo.
- Track special-move state (castling, en passant, promotion).
- Explain check and checkmate evaluation at interview depth.

---

## 1) Clarifying decisions

For this design, we choose:

1. Human-vs-human core.
2. Promotion allows explicit choice (default queen fallback).
3. Include check/checkmate/stalemate; advanced draw rules as extension.
4. Single-step undo via command stack and snapshots.
5. Single-threaded game loop.

---

## 2) Core entities

| Entity | Responsibility |
|--------|----------------|
| `Game` | Turn control, move execution, status updates |
| `Board` | 8x8 piece placement and lookup |
| `Cell` / `Position` | Coordinate model and occupancy |
| `Piece` hierarchy | Movement geometry by piece type |
| `Move` | Action data (from/to/promotion/special flags) |
| `MoveValidator` | Legal move check with state constraints |
| `CheckService` | Detect check/checkmate/stalemate |
| `MoveCommand` | Execute/undo one move |
| `GameSnapshot` | Memento for reliable rollback |
| `MoveHistory` | Replay/UI/audit trail |

---

## 3) Pattern mapping

| Pattern | Role |
|---------|------|
| Command | `MoveCommand.execute/undo` encapsulates reversible mutation |
| Memento | `GameSnapshot` captures board/context for undo/save |
| Template Method | Base `Piece` flow with subtype move generation |
| Iterator | Board/ray traversal for attack checks |
| Strategy (extension) | AI move pickers (`Random`, `Minimax`) |

---

## 4) Board and piece model

```java
public final class Position {
    private final int row;
    private final int col;

    public Position offset(int dr, int dc) {
        return new Position(row + dr, col + dc);
    }

    public boolean isOnBoard() {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }
}

public final class Board {
    private final Cell[][] grid = new Cell[8][8];

    public Piece getPiece(Position p) { return grid[p.row()][p.col()].piece(); }
    public void setPiece(Position p, Piece piece) { grid[p.row()][p.col()].set(piece); }
    public void movePiece(Position from, Position to) {
        Piece p = getPiece(from);
        setPiece(from, null);
        setPiece(to, p);
    }
}
```

```java
public abstract class Piece {
    protected final Color color;
    protected final PieceType type;

    public final List<Position> candidateMoves(Board board, Position from) {
        return getRawMoves(board, from).stream()
            .filter(Position::isOnBoard)
            .toList();
    }

    protected abstract List<Position> getRawMoves(Board board, Position from);
    public abstract Piece copy();
}
```

---

## 5) Move validation and check logic

```java
public final class MoveValidator {
    private final CheckService checkService;

    public boolean isLegal(Move move, Board board, Color sideToMove, GameContext context) {
        Piece piece = board.getPiece(move.from());
        if (piece == null || piece.color() != sideToMove) return false;
        if (!isGeometryOrSpecialMoveLegal(move, board, context)) return false;
        return !checkService.wouldLeaveKingInCheck(board, move, sideToMove, context);
    }
}
```

`CheckService` owns king-in-check and hypothetical-move evaluation, not `Game` directly.

Checkmate rule:

- `inCheck(side)` and no legal moves -> checkmate.
- `not in check` and no legal moves -> stalemate.

---

## 6) Special move state

Store in `GameContext`:

- Castling rights flags
- En passant target square (one-turn validity)
- Promotion choice from move data

```java
public final class GameContext {
    private CastlingRights castlingRights = CastlingRights.initial();
    private Optional<Position> enPassantTarget = Optional.empty();
}
```

---

## 7) Command + Memento undo

```java
public interface GameCommand {
    void execute();
    void undo();
}

public final class MoveCommand implements GameCommand {
    private final Game game;
    private final Move move;
    private GameSnapshot before;

    public MoveCommand(Game game, Move move) {
        this.game = game;
        this.move = move;
    }

    @Override
    public void execute() {
        before = game.takeSnapshot();
        game.applyMoveInternal(move);
        game.switchTurn();
        game.updateStatus();
    }

    @Override
    public void undo() {
        game.restoreSnapshot(before);
        game.switchTurn();
        game.updateStatus();
    }
}
```

Undo for captures is simplest and safest with snapshot restore in interview scope.

---

## 8) Game orchestration

```java
public final class Game {
    private Board board;
    private GameContext context = new GameContext();
    private Color sideToMove = Color.WHITE;
    private GameStatus status = GameStatus.ONGOING;
    private final Deque<GameCommand> undoStack = new ArrayDeque<>();
    private final MoveValidator validator;
    private final CheckService checkService;
    private final MoveHistory history = new MoveHistory();

    public Result makeMove(Move move) {
        if (status.isTerminal()) return Result.fail("game over");
        if (!validator.isLegal(move, board, sideToMove, context)) return Result.fail("illegal move");
        MoveCommand cmd = new MoveCommand(this, move);
        cmd.execute();
        undoStack.push(cmd);
        history.record(move);
        return Result.ok();
    }

    public void undo() {
        if (undoStack.isEmpty()) return;
        undoStack.pop().undo();
    }
}
```

---

## 9) Extensions

- Chess clock with timeout result.
- AI strategy plug point via `MovePicker`.
- Threefold repetition and 50-move draw tracking.
- Network/multiplayer synchronization model.

---

## 10) Self-check with answers

1. **Why Memento if Command undo exists?**  
   Memento simplifies undo for complex move side effects and also supports save/load snapshots.

2. **Template Method vs piece-rule strategy trade-off?**  
   Template is straightforward for classic piece hierarchy; strategy is better for runtime-variant rule sets.

3. **Risk of one static validator for everything?**  
   God-method complexity, poor testability, and brittle extension handling.

---

## 11) First tests

1. Illegal move that leaves own king in check is rejected.
2. Capture followed by undo restores full board/context.
3. Castling disallowed after king/rook movement.

---

## Day 4 checkpoint

- [x] Board, piece, validator, and check responsibilities are separated.
- [x] Special move context state is explicit.
- [x] Command + snapshot undo path is defined.
- [x] Check/checkmate logic and test plan are covered.
