# LRGP-py

Python implementation of the **Lightweight Reticulum Gaming Protocol (LRGP)** — a compact, session-based protocol for multiplayer games over [LXMF](https://github.com/markqvist/LXMF) / [Reticulum](https://github.com/markqvist/Reticulum) mesh networks.

LRGP enables turn-based and real-time multiplayer games to run over LoRa radios, WiFi, TCP, and any other medium Reticulum supports. Game moves are encoded as tiny msgpack envelopes that fit in a single encrypted packet — no link setup needed.

LRGP now has thirteen built-in apps. Tic-Tac-Toe, Four in a Row, Checkers,
and the new **Hex**, **Gomoku**, **Reversi**, **Dots and Boxes**,
**Mancala**, **Ultimate Tic-Tac-Toe**, **Go**, **Battleship**, and
**Quoridor** apps are dependency-free; Chess is available through the
optional `chess` dependency.

## Quick Start

```bash
# Install (zero dependencies for core library)
pip install -e .

# Add Chess support (pulls in python-chess)
pip install -e ".[chess]"

# Run tests
pip install -e ".[chess,dev]"
pytest

# Play Tic-Tac-Toe locally (no network needed)
python examples/ttt_local.py

# Inspect the built-in Four in a Row manifest (no extra dependency)
python -c "from lrgp.apps import FourInARowApp; print(FourInARowApp().get_manifest())"

# Inspect the built-in Checkers manifest (no extra dependency)
python -c "from lrgp.apps import CheckersApp; print(CheckersApp().get_manifest())"

# Walk Scholar's Mate locally
python examples/chess_local.py

# Check wire budget for all TTT actions
python examples/envelope_sizes.py
```

## How It Works

LRGP encodes game sessions as LXMF custom fields:

```python
fields[0xFB] = "lrgp.v1"                       # protocol marker
fields[0xFD] = {                               # envelope
    "a": "ttt.1",                              # app_id.version
    "c": "move",                               # command
    "s": "a1b2c3d4e5f60718",                   # session_id
    "p": {"i": 4, "b": "____X____", ...},      # payload (game-specific)
    "n": b"\\xde\\xad\\xbe\\xef\\xc0\\xff\\xee\\x01",  # 8-byte CSPRNG nonce
}
```

The LXMF `content` field carries fallback text (e.g., `"[LRGP TTT] Move 3"` or `"[LRGP Chess] e2e4"`) for non-LRGP clients.

Four in a Row uses `four_in_a_row.1`. Its move intent is just `{"c": 0}`;
the canonical wire move is `{"c": 0, "n": 1, "x": ""}`. Both peers apply
gravity and reconstruct the 42-cell board and next turn locally, so neither a
board snapshot nor a turn identity is sent with a move. A winning move alone
adds `"w"`, set to the authenticated mover.

Checkers uses `checkers.1`. Its move intent is a space-separated square path
such as `{"m": "a5 c3 e1"}`; the canonical wire move is
`{"m": "a5 c3 e1", "n": 12, "x": ""}`. Each peer replays the path history to
rebuild the board, so neither the board nor the next turn is sent. See
[docs/checkers.md](docs/checkers.md).

Hex (`hex.1`), Gomoku (`gomoku.1`), and Reversi (`reversi.1`) share one wire
contract: a move intent is a single square such as `{"m": "h8"}` (Hex also
accepts `{"m": "swap"}` as White's first move) and the wire move is
`{"m": "h8", "n": 0, "x": ""}`, plus `"w"` on a win. Dots and Boxes
(`dots_and_boxes.1`) and Mancala (`mancala.1`) use the same contract with a
line (`{"m": "c3h"}`) or a pit letter (`{"m": "d"}`) as the move; a player who
completes a box or ends in their store simply moves again. See
[docs/placement-games.md](docs/placement-games.md).

The newer single-token games use the same contract; SPEC.md Appendix G
defines each one. Ultimate Tic-Tac-Toe (`ultimate_ttt.1`) moves are a square
such as `{"m": "e5"}` on its 9x9 board. Go (`go.1`) moves are a point such as
`{"m": "e5"}` or `{"m": "pass"}`.

Battleship (`battleship.1`) is the first game with hidden information. Each
player commits to a fleet with a SHA-256 hash of the fleet and a random salt
(`{"h": ...}` on challenge and accept); a shot `{"s": "e5", "a": "h", ...}`
carries the answer to the opponent's last shot, and both fleets are revealed
at the end so each side checks the other's answers. The local intents are
`{"f": fleet, "k": salt}` (both optional) to challenge or accept, `{"s": "e5"}`
to shoot, and `{}` to concede or reveal. SPEC.md Appendix H defines it.

Quoridor (`quoridor.1`) moves step the pawn (`{"m": "e2"}`) or place a wall
(`{"m": "c3h"}`, `{"m": "c3v"}`).

All envelopes are msgpack-serialized and fit within LXMF's 295-byte OPPORTUNISTIC delivery limit — no link setup needed, single encrypted packet.

### Replay protection

Every outbound envelope carries an 8-byte CSPRNG nonce under key `n`. The
router first probes without recording, authorizes the authenticated sender,
then atomically checks and records before application mutation. The cache is
scoped by receiving identity and session, bounded to 512 nonces per scope and
1024 scopes, with a 10-minute absolute TTL. This keeps unauthenticated traffic
from consuming or evicting replay state. Terminal sessions retain replay
entries through their normal TTL; only explicit session removal may drop that
identity/session scope.

Use `LrgpRouter` as the protocol boundary. It validates canonical wire data,
rejects trailing bytes and duplicate map keys in byte-oriented decoding, binds
each session to its authenticated participant, enforces global session-ID
uniqueness across apps, and caps unsolicited pending challenges at 16 per
participant and 128 per receiving identity. Incoming authenticated sender and
receiving-identity identifiers, and outgoing local identity/recipient
identifiers, must all be non-empty.

The optional `LrgpTransport` forwards an LXMF `source_hash` only after LXMF
reports a validated message signature. Custom integrations must likewise pass
`dispatch_incoming` a sender derived from authenticated transport metadata,
never a display name, fallback text, or envelope value.

Inbound dispatch changes live game state before the caller commits its durable
session/action transaction. Snapshot first; if that commit fails, call
`rollback_incoming(app_id, session_id, identity_id, envelope["n"], snapshot)`.
This restores the session and releases only the accepted scoped nonce, so the
transport's exact retransmission can be applied. If durable recording of an
authenticated `remote_error` fails, call
`forget_incoming_nonce(identity_id, session_id, envelope["n"])` instead: that
result consumed a nonce but did not mutate game state. SQLite `save_session`
is insert-only; use its allowlisted `update_session` method for existing
records.

## Project Structure

```
src/lrgp/
  constants.py     # Protocol constants
  errors.py        # Error hierarchy
  envelope.py      # Pack/unpack/validate envelopes
  dedup.py         # Receiving-identity/session replay cache
  session.py       # Session state machine
  app_base.py      # Abstract GameBase for games
  router.py        # App registry and dispatch
  store.py         # SQLite persistence
  transport.py     # LXMF bridge (optional, requires lrgp[rns])
  apps/
    checkers.py    # Checkers (English draughts, square-path wire format)
    _placement.py  # Shared engine for the single-token games below
    hex.py         # Hex (11x11, swap rule)
    gomoku.py      # Gomoku (15x15 freestyle)
    reversi.py     # Reversi (8x8, automatic passing)
    dots_and_boxes.py # Dots and Boxes (5x5 boxes, extra turns)
    mancala.py     # Mancala (Kalah, six pits of four seeds)
    ultimate_ttt.py # Ultimate Tic-Tac-Toe (nine small boards)
    go.py          # Go (9x9, area scoring, superko)
    battleship.py  # Battleship (10x10, committed fleets)
    quoridor.py    # Quoridor (9x9, ten walls each)
    four_in_a_row.py # Four in a Row (7x6 gravity, reconstructed state)
    tictactoe.py   # Tic-Tac-Toe reference game
    chess.py       # Chess (python-chess engine, UCI wire format; lrgp[chess])
```

## Writing a Game

Implement the `GameBase` class:

```python
from lrgp.app_base import GameBase

class MyGame(GameBase):
    app_id = "mygame"
    version = 1
    display_name = "My Game"
    session_type = "turn_based"
    validation = "both"
    actions = ["challenge", "accept", "decline", "move"]
    # ... implement abstract methods ...
```

## Protocol Spec

See [SPEC.md](SPEC.md) for the formal protocol specification — implementable without seeing the Python code.
The game-specific design is summarized in
[docs/four-in-a-row.md](docs/four-in-a-row.md).

## Network Usage

For LXMF transport (requires Reticulum):

```bash
pip install -e ".[rns]"
python examples/ttt_cli.py
```

## See Also

- [lrgp-rs](../lrgp-rs) — Rust implementation (wire-compatible)

## License

MIT — see [LICENSE](LICENSE).
