# Grok Bot skills

Official skill files for two Grok Bot templates:

- **Second Opinion Council** (`council/`)
- **Fleet Doctor** (`fleet-doctor/`)

Template imports don't carry skills yet, so each bot asks its new owner one plain question on first chat ("Want me to finish setting up?"). On yes it downloads its skills from a pinned commit of this repo, checks every file against `PINS.txt` (sha256 and size), and installs only exact matches. Nothing is retyped and nothing installs on a mismatch.

`PINS.txt` format: `sha256 bytes name path`, one skill per line. The template description pins the commit and the sha256 of `PINS.txt`.
