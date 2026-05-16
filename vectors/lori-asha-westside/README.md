# Lori Asha - Westside Reference Vector

This directory contains the public Bitneedle picture-record examples generated from the Lori Asha Westside premix source.

Published assets:

- `single45.record.png`: standard 45-style picture record
- `single45.picture-noise.record.png`: 45-style picture record with random-looking RGB picture noise under the encoded spiral
- `single45.ecdc`: expected ECDC payload for the 45 examples
- `lp.record.png`: LP-style picture record
- `lp.ecdc`: expected ECDC payload for the LP example
- `manifest.json`: hashes, profile metadata, geometry, and expected checks

Decoded WAV files are not published here. To verify recovery, decode the PNG records back to ECDC and compare against the published ECDC SHA-256 hashes in `manifest.json`.
