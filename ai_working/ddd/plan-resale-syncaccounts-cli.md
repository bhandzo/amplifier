# Plan: Resale Sync Accounts CLI

## Overview

Add CLI commands to `tradedesk_api_client_resale` for managing sync accounts, following the patterns established in the POS CLI.

## Goal

Create a typer-based CLI that:
1. Lists sync accounts with filtering options
2. Creates new sync accounts
3. Outputs full JSON request/response bodies for debugging

## Reconnaissance Summary

### Existing Patterns (POS CLI)

From `extra-point-data/extra_point/tradedesk_api_client_pos/cli.py`:
- Uses `typer.Typer()` with `no_args_is_help=True`
- Console output via `rich.console.Console`
- Commands: `update-tags`, `update-face-values`, `get`
- Has `--dry-run` mode for safe testing
- CSV file input handling for batch operations
- Clean error handling with typed exceptions

### Resale Client Architecture

From `extra-point-data/extra_point/tradedesk_api_client_resale/`:
- **client.py**: `_make_api_request()` handles auth, retries, pagination
- **auth.py**: `refresh_access_token(config, force)` - OAuth2 two-stage auth
- **config.py**: `TradeDeskConfig.from_env()` loads credentials from env vars:
  - `TD_CONSUMER_KEY`
  - `TD_USERNAME`
  - `TD_PASSWORD`
- Uses base URL from `defaults.py`

### Sync Accounts API (from Swagger)

**GET /syncaccounts/V2** - List accounts
- Query params: `searchterm`, `status`, `page`, `pageSize`
- Status values: `NotProcessed`, `Processing`, `CompletedViaProcessor`, `CompletedViaDirectInsert`, `ErrorFailedValidation`, `ErrorFatal`
- Returns: `SyncAccountResponseResponse` (items array + errors)

**POST /syncaccounts/V2** - Create accounts
- Body: Array of `SyncAccountsPostRequest`
- Required: `username`, `password`, `system`
- Optional: `nickName`, `attributes`
- System object: `{ id: int, displayName?: string }`

**Response Schema (SyncAccountResponse)**:
```json
{
  "id": 12345,
  "userName": "username@email.com",
  "nickName": "nickname",
  "status": "CompletedViaProcessor",
  "dates": { ... },
  "system": { "id": 12345, "displayName": "Display Name" },
  "attributes": ["AutoSync", "AutoPDFDownload"],
  "detail": { ... }
}
```

## Proposed Architecture

### File Structure

```
extra-point-data/extra_point/tradedesk_api_client_resale/
├── __init__.py          # Add CLI entrypoint export
├── client.py            # Existing - add sync account methods
├── cli.py               # NEW - Typer CLI commands
├── auth.py              # Existing - no changes
├── config.py            # Existing - no changes
└── defaults.py          # Existing - no changes
```

### CLI Design

```python
# cli.py
app = typer.Typer(
    name="tradedesk-resale",
    help="Trade Desk Resale API CLI - Sync Account management",
    no_args_is_help=True,
)

# Commands:
# 1. list - List sync accounts with optional filters
# 2. create - Create a new sync account
```

### Command: `list`

```
tradedesk-resale list [OPTIONS]

Options:
  --search TEXT        Filter by username, nickname, or system name
  --status TEXT        Filter by status (NotProcessed, Processing, etc.)
  --page INT           Page number (default: 1)
  --page-size INT      Results per page (default: 50, max: 200)
  --output-json        Output raw JSON response (for debugging)
```

### Command: `create`

```
tradedesk-resale create [OPTIONS]

Required:
  --username TEXT      Account username (email)
  --password TEXT      Account password
  --system-id INT      System ID

Optional:
  --nickname TEXT      Account nickname
  --attributes TEXT    Comma-separated attributes (e.g., "AutoSync,AutoPDFDownload")
  --output-json        Output raw JSON request/response (for debugging)
```

## Implementation Plan

### Phase 1: Add Client Methods

Add to `client.py`:

```python
def list_sync_accounts(
    config: TradeDeskConfig,
    search: str | None = None,
    status: str | None = None,
    page: int = 1,
    page_size: int = 50,
) -> dict:
    """List sync accounts with optional filtering."""
    params = {}
    if search:
        params["searchterm"] = search
    if status:
        params["status"] = status
    params["page"] = str(page)
    params["pageSize"] = str(page_size)

    return _make_api_request(
        config=config,
        endpoint="/syncaccounts/V2",
        method="GET",
        params=params,
    )

def create_sync_account(
    config: TradeDeskConfig,
    username: str,
    password: str,
    system_id: int,
    nickname: str | None = None,
    attributes: list[str] | None = None,
) -> dict:
    """Create a new sync account."""
    request_body = [{
        "username": username,
        "password": password,
        "system": {"id": system_id},
    }]
    if nickname:
        request_body[0]["nickName"] = nickname
    if attributes:
        request_body[0]["attributes"] = attributes

    return _make_api_request(
        config=config,
        endpoint="/syncaccounts/V2",
        method="POST",
        json_data=request_body,
    )
```

### Phase 2: Create CLI

Create `cli.py`:

```python
"""
ABOUTME: CLI for Trade Desk Resale API sync account operations.
ABOUTME: Provides commands to list and create sync accounts with JSON debugging output.
"""
import json
import typer
from rich.console import Console

from .client import list_sync_accounts, create_sync_account
from .config import TradeDeskConfig

app = typer.Typer(
    name="tradedesk-resale",
    help="Trade Desk Resale API CLI - Sync Account management",
    no_args_is_help=True,
)

console = Console()

@app.command()
def list(
    search: str | None = typer.Option(None, "--search", help="Filter by username, nickname, or system"),
    status: str | None = typer.Option(None, "--status", help="Filter by status"),
    page: int = typer.Option(1, "--page", help="Page number"),
    page_size: int = typer.Option(50, "--page-size", help="Results per page (max 200)"),
    output_json: bool = typer.Option(False, "--json", help="Output raw JSON response"),
):
    """List sync accounts with optional filtering."""
    config = TradeDeskConfig.from_env()

    response = list_sync_accounts(
        config=config,
        search=search,
        status=status,
        page=page,
        page_size=page_size,
    )

    if output_json:
        console.print_json(json.dumps(response))
    else:
        # Pretty table output
        items = response.get("items", [])
        console.print(f"Found {len(items)} sync accounts")
        for item in items:
            console.print(f"  [{item['id']}] {item['userName']} - {item['status']}")

@app.command()
def create(
    username: str = typer.Option(..., "--username", help="Account username (email)"),
    password: str = typer.Option(..., "--password", help="Account password"),
    system_id: int = typer.Option(..., "--system-id", help="System ID"),
    nickname: str | None = typer.Option(None, "--nickname", help="Account nickname"),
    attributes: str | None = typer.Option(None, "--attributes", help="Comma-separated attributes"),
    output_json: bool = typer.Option(False, "--json", help="Output raw JSON request/response"),
):
    """Create a new sync account."""
    config = TradeDeskConfig.from_env()

    attr_list = attributes.split(",") if attributes else None

    # Build request for debugging output
    request_body = [{
        "username": username,
        "password": "***REDACTED***",  # Don't log password
        "system": {"id": system_id},
    }]
    if nickname:
        request_body[0]["nickName"] = nickname
    if attr_list:
        request_body[0]["attributes"] = attr_list

    if output_json:
        console.print("[bold]Request:[/bold]")
        console.print_json(json.dumps(request_body))

    response = create_sync_account(
        config=config,
        username=username,
        password=password,
        system_id=system_id,
        nickname=nickname,
        attributes=attr_list,
    )

    if output_json:
        console.print("\n[bold]Response:[/bold]")
        console.print_json(json.dumps(response))
    else:
        items = response.get("items", [])
        if items:
            console.print(f"[green]Created sync account:[/green] {items[0]['id']}")
        errors = response.get("errors", [])
        if errors:
            console.print(f"[red]Errors:[/red] {errors}")

if __name__ == "__main__":
    app()
```

### Phase 3: Wire Up Entry Point

Update `pyproject.toml` or `__init__.py` to expose CLI:

```toml
[project.scripts]
tradedesk-resale = "extra_point.tradedesk_api_client_resale.cli:app"
```

## Philosophy Alignment Check

| Principle | Compliance |
|-----------|------------|
| Ruthless simplicity | Two focused commands, no over-engineering |
| Start minimal, grow as needed | List + Create first, expand later |
| Use libraries as intended | Direct typer usage, following POS CLI patterns |
| Preserve architectural patterns | Reuses existing auth, config, client patterns |
| Clear error responses | JSON output for debugging, pretty output for humans |
| No stubs or placeholders | All code will be functional |

## Success Criteria

1. `tradedesk-resale list` returns sync accounts from API
2. `tradedesk-resale list --json` outputs full JSON response
3. `tradedesk-resale create --username X --password Y --system-id Z` creates account
4. `tradedesk-resale create --json` shows request and response bodies
5. All commands use existing auth flow (env vars)

## Open Questions

1. **Should we add a `--verbose` flag for request headers?** - Could help debug auth issues
2. **Error handling granularity** - Should we parse and format specific error codes?
3. **System ID lookup** - Should we add a command to list available systems?

## Next Steps After Approval

1. Implement client methods in `client.py`
2. Create `cli.py` with typer commands
3. Add entry point to project configuration
4. Test with real API
5. Document usage in `/docs/docs/` (MkDocs site)
