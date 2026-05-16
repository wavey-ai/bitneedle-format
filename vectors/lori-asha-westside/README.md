# Lori Asha - Westside Reference Vector

This directory contains reference Bitneedle picture-record examples generated from the Lori Asha Westside premix source.

Published assets:

- `single45.record.png`: standard 45-style picture record
- `single45.picture-noise.record.png`: 45-style picture record with deterministic RGB picture noise under the encoded spiral
- `single45.stream-first.record.png`: progressive first-preview render
- `single45.stream-last.record.png`: progressive last-preview render
- `single45.stream-final.record.png`: progressive final/capacity render
- `single45.ecdc`: expected ECDC payload for the 45 examples
- `lp.record.png`: LP-style picture record
- `lp.ecdc`: expected ECDC payload for the LP example
- `manifest.json`: hashes, profile metadata, geometry, and expected checks

Decoded WAV files are not published here. To verify recovery, decode the PNG records back to ECDC and compare against the published ECDC SHA-256 hashes in `manifest.json`.
