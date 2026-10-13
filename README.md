# AIPOCH Open-Science use cases

This repository collects completed Open-Science case exports and their matching case posters.

Each case is stored as a folder at the repository root, named with its English case title, with two files:

- `<English case title>/<English case title>.science` — the native Open-Science session export.
- `<English case title>/<English case title>.png` — the matching case poster.

The `.science` files are preserved from the supplied exports. Posters use the AIPOCH OPEN-SCIENCE case-poster format. Synthetic demonstrations and unresolved findings remain identified in the source sessions; the presence of a package does not imply general model performance.

## Cases

The repository includes the existing completed cases from the source archive and the newly packaged cases from the current Open-Science case collection. Each case folder is self-contained.

## Category index and package sources

[Browse 94 cases by category](CATEGORIES.md): Curiosity Lab (48), AI Research (46), and Academic Research (reserved). `categories.json` records category membership without changing the publication manifest schema. Each case has a same-named poster and introduction. Existing local session exports are retained; thirteen additional exports are referenced through fixed Hugging Face revision URLs in `manifest.json`.
