# EAGLE

A private, local AI assistant that runs on your own computer. Your database lives on your computer
too, it works without internet, and **no data leaves your device — even when you are online.**
The internet is only used to download EAGLE and the AI models.

EAGLE is the local, privacy-first version of MERCY.

```bash
git clone https://github.com/<you>/eagle && cd eagle
./eagle.sh
```

> Status: **planning**. Nothing runs yet.

- Plan: [docs/PLAN.md](docs/PLAN.md)
- AI models (all local: Ollama, Hugging Face in Jupyter, your own model): [docs/AI_MODELS.md](docs/AI_MODELS.md)
- Runs on Linux (Ubuntu first) and macOS; Windows through WSL2.

## License

Copyright (C) 2026 MD RATUL HOSSEN

EAGLE is free software: you can redistribute it and/or modify it under the terms of the
[GNU Affero General Public License, version 3](LICENSE) (AGPL-3.0-only).

AI models are not part of EAGLE. You download them yourself, and each model has its own license.
