# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
make install          # Install dependencies from poetry.lock
make test             # Run all tests (pytest + coverage)
make lint             # Run all linters and formatters via pre-commit
make pre-commit hook=black  # Run a single pre-commit hook (e.g. black, isort, mypy)
```

Run a single test file or test:
```bash
poetry run pytest tests/signing_test.py
poetry run pytest tests/signing_test.py::test_l1_action_signing_matches
```

## Setup Notes

- Requires **Poetry v1.x** (not v2). Install with: `curl -sSL https://install.python-poetry.org | POETRY_VERSION=1.4.1 python3 -`
- Development requires **Python 3.10 exactly**. Point poetry to it: `poetry env use /path/to/python3.10`
- `vcrpy` (used for HTTP cassette recording in tests) requires Python 3.10.10 exactly per `pyproject.toml`.

## Architecture

The SDK has three main public classes, all living in `hyperliquid/`:

### `API` (`api.py`)
Base HTTP client. Wraps `requests.Session` for POST calls to the Hyperliquid REST endpoints. Raises `ClientError` (4xx) or `ServerError` (5xx) from `utils/error.py`.

### `Info(API)` (`info.py`)
Read-only query interface. On init it fetches spot and perp metadata and builds three lookup dicts:
- `name_to_coin`: human name → internal coin identifier
- `coin_to_asset`: coin identifier → integer asset index
- `asset_to_sz_decimals`: asset index → size decimal precision

All queries hit the `/info` endpoint via POST with a `type` field. Also manages WebSocket subscriptions through `WebsocketManager`. Pass `skip_ws=True` to skip WebSocket initialization (required for tests).

**Asset index offsets**: spot assets start at `10000`; builder-deployed perp DEXs start at `110000` with each DEX occupying a block of `10000`.

### `Exchange(API)` (`exchange.py`)
Write interface for trading actions. Internally creates an `Info` instance at construction. Every action follows this pattern:
1. Build an action dict with a `"type"` field
2. Sign it using `sign_l1_action` (most actions) or a user-signed variant
3. Submit via `_post_action` → POST `/exchange`

**Two signing modes** in `utils/signing.py`:
- **L1 actions** (`sign_l1_action`): Action is msgpack-serialized and keccak-hashed, then signed as an EIP-712 "phantom agent". Used for orders, cancels, leverage updates, etc. Supports `vault_address` and `expires_after`.
- **User-signed actions** (e.g. `sign_usd_transfer_action`, `sign_spot_transfer_action`): EIP-712 signed directly with `HyperliquidSignTransaction` domain. These include `signatureChainId = "0x66eee"` in the action and **do not support `expires_after`**.

Market orders (`market_open`, `market_close`) are implemented as aggressive IoC limit orders priced with configurable slippage (default 5%) against the current mid price.

### `WebsocketManager` (`websocket_manager.py`)
Runs as a daemon `threading.Thread`. Maintains a dict of active subscriptions keyed by channel identifier strings (e.g. `"l2Book:btc"`). Sends pings every 50 seconds. Subscriptions that arrive before the connection is ready are queued and replayed on open.

## Testing

Tests use [`pytest-recording`](https://github.com/kiwicom/pytest-recording) with VCR cassettes. HTTP interactions are recorded in `tests/cassettes/` as YAML files and replayed on subsequent runs—tests never make real network calls. Add `@pytest.mark.vcr()` to a test and run once with `--record-mode=once` to capture a new cassette.

`signing_test.py` tests contain hardcoded expected signature values against known keys and actions; if signing logic changes, these expected values must be updated to match.

## Code Conventions

- Line length: **120 characters** (enforced by black/flake8).
- `hyperliquid/utils/types.py` is excluded from pyupgrade — keep older `TypedDict` syntax there.
- Addresses are lowercased before use (see `.lower()` calls throughout `exchange.py`).
- Floats are converted to wire format strings via `float_to_wire()` for prices/sizes in order actions.
- `pyproject.toml` contains configuration for black, isort, mypy, pylint, and pytest.
