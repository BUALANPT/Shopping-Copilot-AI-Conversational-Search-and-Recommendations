# Final Submission Checklist

## GitHub

- [ ] The final repository is set to Public;
- [ ] The default branch contains clean commits;
- [ ] `git status --short` produces no output;
- [ ] Do not commit `data/catalog.jsonl`, Dense/BGE indexes, model caches, `.venv`, `.vscode`, or keys;
- [ ] No personal absolute paths or experiment temporary directories appear in the GitHub file listing;
- [ ] After a fresh clone from GitHub, installation, index building, testing, and three demonstration rounds are completed successfully.

## Default Entry Point

- [ ] `agent:Agent` points to `solution.agent.Agent`;
- [ ] `starter.agent:Agent` points to `solution.agent.Agent`;
- [ ] `evaluator/local_evaluator.py` remains unmodified;
- [ ] Running `python -m evaluator.local_evaluator` executes the final solution;
- [ ] The original weak baseline is preserved in `starter.baseline_agent:Agent`.

## Devpost

- [ ] Replace GitHub links in `docs/DEVPOST_DESCRIPTION.md`;
- [ ] Replace with YouTube Public video links;
- [x] Team members and equal contributions are documented for LIU BOANG, JI CHENGYU, and LI XIZI;
- [ ] List Python, SQLite FTS5, NumPy, FastEmbed, Ollama/Qwen, and the local web frontend;
- [ ] Explain the measured reason for Qwen being disabled by default;
- [ ] Clearly label metrics as dev150 without claiming parity on a private set;
- [ ] Specify data sources, known limitations, and future improvements.

## Demo

- [ ] The system displays `System Ready` after the first load;
- [ ] All three script rounds pass completely;
- [ ] The input box becomes available again after each round;
- [ ] The terminal `demo/cli_demo.py` can serve as a backup;
- [ ] The video does not display keys, personal paths, notifications, or unauthorized trademarked materials.
