# Obsidian Vault Migration Script

## Purpose
Migrate Flameweeder Obsidian vault from old structure to new flat architecture.

## Module Structure

```
migrate_vault/
├── __init__.py           # Public interface
├── README.md            # This file
├── orchestrator.py      # Main migration orchestrator
├── frontmatter.py       # Frontmatter operations
├── file_ops.py          # File/folder operations
├── resources.py         # Resources migration
├── projects.py          # Projects migration
├── templates.py         # Templates migration
└── report.py            # Migration reporting
```

## Contract

### Inputs
- `vault_path`: Path to Obsidian vault root
- `dry_run`: Boolean flag (preview only)
- `backup`: Boolean flag (create backup)

### Outputs
- Migrated vault structure
- Migration report (success/failure counts)
- Error log for skipped files

### Side Effects
- Creates new folder structure
- Moves/renames files
- Updates frontmatter
- Optional: Creates backup in `vault_path/.migration-backup/`

### Dependencies
- Python 3.11+
- pyyaml: For frontmatter parsing
- pathlib: For file operations
- click: For CLI interface

## Usage

```bash
# Dry run (preview changes)
python -m migrate_vault --vault ~/Developer/flameweeder --dry-run

# Execute migration with backup
python -m migrate_vault --vault ~/Developer/flameweeder --backup

# Execute migration without backup
python -m migrate_vault --vault ~/Developer/flameweeder
```

## Testing

```bash
pytest tests/migrate_vault/
```

## Regeneration

This module can be regenerated from this specification alone.
Key invariants:
- Public CLI interface signatures
- Migration phase ordering
- Frontmatter schema
- Error handling behavior
