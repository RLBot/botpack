# AGENTS.md — botpack submissions

This file has two jobs.

1. Review a pull request to this repository and decide whether the bot is ready to merge.
2. Take a bot that runs in RLBot and make it ready to submit here.

Read [README.md](README.md) first. It holds the policy: open source, a valid `bot.toml` or `script.toml`, a working `bob.toml`, a skill limit, and a permissive license. This file holds the mechanics that the README leaves out.

The recurring source of failed submissions is `bob.toml` and the build setup. Bob is small enough that people assume it works like other build tools. It does not. PR #14 failed because the config skipped the examples and used invented keys.

AI assistance is fine. The config still has to follow a template, compile with `bob build bob.toml`, and play a match in RLBot. Every config in this repository follows one of the templates below.

---

## How bob works

Bob reads the root `bob.toml`:

```toml
dependencies = ["./Botpack/**/bob.toml"]
```

Each bot submodule has its own `bob.toml`. Bob runs one Docker build per `[[config]]` entry, then assembles `bob_build/<project_name>/`.

The build has three stages.

1. **Build.** Bob renders a Dockerfile from `builder_config`, runs `docker build`, then `docker run`. The container must write a tar archive to **stdout**. Bob extracts it into `bob_build/<project_name>/`.
2. **Rewrite configs.** Bob finds the built Windows PE (`.exe`) and Linux executable (ELF, name not starting with `lib`, not ending in `.so`) in the extracted files. It overwrites `run_command` and `run_command_linux` in each `bot_configs` file with paths to those binaries. It also copies the logo and loadout from the bot config's directory into the project root.
3. **Split and publish.** `bob split` copies each project's root files and its `*x86*linux*` / `*x86*windows*` folders into `bob_build_x86_64-linux/` and `bob_build_x86_64-windows/`. The workflows compress those two trees into the release tarballs.

Three consequences matter for review.

- Bob picks a Linux binary by MIME type and file name. If the archive contains both a launcher and a Torch-linked core, the **last matching ELF in tar order** becomes `run_command_linux`. Use `tar --sort=name` so `000-runtime/GGLBotCore` sorts before `GGLBot`.
- Bob overwrites `run_command`. The value in the source `bot.toml` only has to be valid for local development. It is not what ships.
- Bob skips the logo and loadout copy when the path does not resolve from the config's directory. A custom Dockerfile can stage those files at the project root itself, so a non-relative path is not always a missing asset. Check the Dockerfile before flagging it.
- `bob split` copies only files at the project root and files under platform folders. Anything nested elsewhere in the project is dropped. Models, DLLs, scripts, and data must be inside the platform folder or copied there by the Dockerfile.

Bob hashes the project directory and skips rebuilds when the hash matches `buildinfo.toml`. That is why PR CI can build only changed bots. It is also why a submodule pointer must be updated after a fix: the root repository pins one commit, and nothing else sees the newer code.

Bob's source is at https://github.com/RLBot/bob. The builder examples are at https://github.com/swz-git/bob-example. The newer botpack-specific GGL template is at https://github.com/SubparN0va/GGLBot/tree/bob.

---

## Hard requirements

### The bot repository

- [ ] Public repository. All source and all models are available. Prebuilt binaries are not accepted; bob builds from source.
- [ ] License that permits redistribution. MIT is the default when the repo has none.
- [ ] A `bob.toml` at the repository root. The file name is exact.
- [ ] A `bot.toml` or `script.toml`. The `#:schema https://rlbot.org/schemas/agent.json` header is optional.
- [ ] Everything the bot needs at runtime is in the repository. No private files, no machine-local paths, no network downloads at bot startup.

### `bot.toml` / `script.toml`

Check the `[settings]` table:

- [ ] `agent_id` is unique and follows `author/bot-name/version` or `author/bot-name`.
- [ ] `name` is set.
- [ ] `run_command` and `run_command_linux` are optional in the source config. Bob overwrites both with the built binary paths; the values only matter when running the bot straight from the repository.
- [ ] `logo_file` is either a real path or omitted. An empty `logo_file = ""` breaks bob: it resolves to the config's parent directory and `fs::copy` fails (PR #6). Bob falls back to `logo.png` when the key is absent.
- [ ] `loadout_file`, when present, points to a file that exists.
- [ ] `hivemind` is set deliberately. `true` means one process controls all cars with that agent id.

Check the `[details]` table. The README requires every field:

- [ ] `description`, `fun_fact`, `source_link`, `developer`, `language`. `tags` is required for bots and optional for scripts.
- [ ] When present, `tags` uses known values. The GUI defines the list in `frontend/src/pages/RocketHost.svelte` of `RLBot/gui`: `1v1`, `2v2`, `3v3`, `teamplay`, `goalie`, `hoops`, `dropshot`, `snow-day`, `rumble`, `spike-rush`, `heatseeker`, `memebot`. `soccar` is a game mode, not a tag. Reject `ML`, `teamwork`, `3v2`, `2v1` and other invented tags.
- [ ] `teamplay` means the bot handles a random number of teammates or matches larger than 3v3. Do not use it for a bot that only plays in a fixed 1v1, 2v2, or 3v3 lineup.

### `bob.toml`

```toml
[[config]]
project_name = "MyBot"
bot_configs = ["src/bot.toml"]

[config.builder_config]
builder_type = "pyinstaller"
entry_file = "bot.spec"
```

- [ ] `project_name` is unique in this botpack. It becomes the folder name in `bob_build/`.
- [ ] Every `bot_configs` path is relative to the `bob.toml` directory and stays inside that directory. Bob rejects configs outside the project root.
- [ ] `builder_type` is one of `pyinstaller`, `rust`, `csharp`, `custom`. Unknown keys are ignored by deserialization, so a wrong key such as `[builders.default]` silently produces a broken build (PR #14).
- [ ] Multiple `[[config]]` entries are allowed when one repo ships several bots. See `Botpack/NectoFamily/bob.toml`, `Botpack/Seer-RLBotV5/bob.toml`, and `Botpack/ByteBotpack/bob.toml`.
- [ ] `bob.toml` is committed in the bot repository, not only in this one.

### Skill and PR hygiene

- [ ] The bot is outside the top 100. README.md uses Galactus as the cutoff: a bot equal to Galactus or better is too good. Ask for evidence when the claim is vague.
- [ ] The bot runs on CPU. Bots that need CUDA, a GPU-only wheel, or a local Rocket League install are rejected.
- [ ] No build step needs a Rocket League installation. PR #27 rejected a tool that dumped collision meshes from a local game.
- [ ] The PR adds the bot under `Botpack/` with `git submodule add <url>`, and the submodule points to a pushed commit that exists. PR #13 failed because the bot referenced a RocketSim commit that was not available.
- [ ] The submodule pointer is refreshed after every fix in the bot repository. PR #39 needed several rounds of this.
- [ ] PR CI is green. It runs `bob build bob.toml` on `ubuntu-24.04`.

---

## Choose a builder

| Language            | Builder       | Template to copy                                                |
| ------------------- | ------------- | --------------------------------------------------------------- |
| Python, no ML       | `pyinstaller` | https://github.com/RLBot/python-example                         |
| Python with PyTorch | `pyinstaller` | https://github.com/VirxEC/Elementv5                             |
| C++ / GigaLearn     | `custom`      | https://github.com/SubparN0va/GGLBot/tree/bob                   |
| Rust                | `rust`        | https://github.com/swz-git/bob-example/tree/master/rust_example |
| C#                  | `csharp`      | https://github.com/VirxEC/csharp-example                        |

Use a template rather than writing a config from scratch. For `custom`, the Dockerfile must compile the bot from source and emit a tar to stdout containing at least one `x86*windows*` folder and one `x86*linux*` folder. Bob's Dockerfile contract is documented at https://github.com/RLBot/bob/blob/master/bob-cli/dockerfiles/_README.md.

The `prebuilt_example` folder in bob-example packages ready-made binaries. It demonstrates the Dockerfile contract only. Do not copy it into a botpack submission; the build must compile from source.

---

## Python bots

### The spec file

Generate it with PyInstaller, then commit it. The command from python-example:

```bash
pyinstaller --onefile src/bot.py --paths src
```

For PyTorch bots, use the Elementv5 form:

```bash
pyinstaller --onefile src/bot.py --paths=src \
  --add-data=src/model.p:. \
  --exclude-module=torch \
  --hidden-import=timeit --hidden-import=pickletools \
  --hidden-import=uuid --hidden-import=unittest.mock \
  --hidden-import=ctypes.wintypes
```

- Keep `:.` at the end of `--add-data`. PyInstaller requires it.
- `excludes=['torch']` keeps the 400+ MB torch package out of the exe. The pack supplies torch separately.
- The hidden imports stop PyInstaller from trimming standard-library modules that torch imports at load time. `ctypes.wintypes` is required on Windows; all four Seer specs needed it (Seer PR #2). Apply it to every spec in the repo.
- The shared PyInstaller Dockerfile replaces `\\` with `/` in the spec for the Linux build. Writing paths as `'src\\bot.py'` works on both platforms.
- Check that `datas` points at files that exist. TensorBot needed a fix because its spec referenced the wrong model files (commit `89dcca5` in this repo).

### Importing torch from the pack

Never rely on a plain `import torch` at build time. The pack excludes torch from every exe. Use the Elementv5 snippet in the first module that loads torch:

```python
try:
    import torch
except ImportError:
    import sys
    sys.path.insert(0, "../../torch-archive")
    import torch
```

The path is relative to the **working directory**, which is the bot's project folder in the pack (`bob_build/<project_name>/`), not the platform subfolder. `../../../torch-archive` is wrong (PR #39). `../../torch-archive` is right.

The archive ships these packages: `torch`, `filelock`, `fsspec`, `jinja2`, `markupsafe`, `mpmath`, `networkx`, `sympy`, `torchgen`, `typing_extensions`. It does **not** ship numpy. A dependency outside that list must be bundled by the spec or removed. TensorBot had to drop `onnx2pytorch` for this reason.

### Models and `torch.load`

- [ ] Every model is listed in `datas` and exists at that path.
- [ ] Frozen code resolves models through `sys._MEIPASS`. Use a helper like Seer's `model_path()`:

```python
def model_path(filename: str) -> str:
    if getattr(sys, "frozen", False) and hasattr(sys, "_MEIPASS"):
        base = Path(sys._MEIPASS)
    else:
        base = Path(__file__).resolve().parent.parent
    return str(base / filename)
```

CWD-relative paths such as `../36000.p` fail in the pack (PR #39).

- [ ] Every `torch.load` call uses `weights_only=True`. These bots load plain state dicts. The default emits a warning and will change in future torch versions (PR #39).
- [ ] Model files do not require a network download on first run.

### Python size rules

Exclude torch. Do not bundle a second copy of numpy, ONNX Runtime, or LLVM unless the bot truly needs it. `llvmlite` alone adds 53 MB to a Linux exe. Prefer the shared runtime where one exists.

---

## C++ / GigaLearn bots

Start from https://github.com/SubparN0va/GGLBot/tree/bob, not from a main-branch GGL checkout.

- [ ] Build against the shared CPU torch archive. The Dockerfile downloads `botpack_x86_64-linux.tar.xz` and `botpack_x86_64-windows.tar.xz` from the `r-1` release of https://github.com/VirxEC/pytorch-archive, and passes `-DLIBTORCH_ROOT=.../torch-archive/torch`.
- [ ] The output has a launcher (`GGLBot` / `GGLBot.exe`) at the platform root and the Torch-linked core in `000-runtime/`. Copy `launcher/GGLBotLauncher.cpp` and the split CMake targets.
- [ ] The Dockerfile emits `tar --sort=name` so Bob selects the launcher as the entry point.
- [ ] No `libtorch*.so`, `*.dll`, or `libc10.so` in the output. Rain shipped 426 MB of `.so` and 253 MB of DLLs before the switch.
- [ ] Models live in the `rlbot/` folder and keep their name. The GGL template copies loose `*.lt` files flat; Rain has three checkpoint folders and its Dockerfile preserves them.
- [ ] Headers and includes use the exact case used on disk. The build runs in a Linux container, where paths are case sensitive; MSVC on Windows is not (GGLBot README).
- [ ] The Dockerfile uses a Linux base image. Windows containers are not supported in GitHub Actions (PR #13).
- [ ] If the bot needs a different LibTorch version than the shared archive, raise it during review. That normally means updating the shared archive, not bundling a private copy.

---

## Rust and C# bots

Rust:

```toml
[[config]]
project_name = "rust_agent_example"
bot_configs = ["./bot.toml"]

[config.builder_config]
builder_type = "rust"
targets = ["x86_64-pc-windows-gnu", "x86_64-unknown-linux-musl"]
bin_name = "rust_example"
```

- No Dockerfile. The built-in rust builder cross-compiles both targets.
- `bin_name` must match the Cargo binary.

C#:

```toml
[[config]]
project_name = "CSharpExample"
bot_configs = ["example.bot.toml"]

[config.builder_config]
builder_type = "csharp"
# base_dir = "ExampleBot"
```

- `base_dir` is needed only when the `.csproj` is not at the repository root.
- No Dockerfile.

---

## Failure index

These are the failures seen in this repository's PR history. Each one has cost at least one review round.

| Symptom                                                          | Cause                                                                 | Fix                                                     |
| ---------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------- |
| Bob build fails with an unrecognised config shape                | Invented `bob.toml` keys that skip the examples (PR #14)              | Copy the matching template from bob-example             |
| `Fetched in submodule path ... did not contain <sha>`            | Submodule references an unpushed commit (PR #13)                      | Push the commit, then update the pointer in this repo   |
| Docker error about Windows containers                            | `FROM mcr.microsoft.com/windows/...` (PR #13)                         | Use a Linux base image and cross-compile                |
| `ModuleNotFoundError: No module named 'torch'` in the packed bot | Plain `import torch`, or wrong relative archive path (PR #39)         | Use the try/except snippet with `../../torch-archive`   |
| `FileNotFoundError: '../36000.p'`                                | CWD-relative model path (PR #39)                                      | Resolve through `sys._MEIPASS`                          |
| Warning about `weights_only=False`                               | `torch.load` default (PR #39)                                         | Pass `weights_only=True`                                |
| Bot dies on Windows at startup                                   | Missing hidden import (Seer PR #2)                                    | Add `ctypes.wintypes` and the other four hidden imports |
| Spec points at missing files                                     | Generated spec not updated after a model rename (commit `89dcca5`)    | Check every `datas` path against the repo               |
| `fs::copy` fails on the logo                                     | `logo_file = ""` (PR #6)                                              | Remove the key or set a real file                       |
| Bot rejected for size or missing tags                            | Details section incomplete, invented tags (PR #39)                    | Fill all six detail fields; use schema tags             |
| `wine: socket : Function not implemented` in CI                  | Runner upgrade to 26.04 with Docker 29.4.2 and seccomp 0.2.2 (PR #40) | Keep workflows pinned to `ubuntu-24.04`                 |
| Bot crashes with a missing dependency at runtime                 | Dependency not in the torch archive (TensorBot)                       | Bundle it in the spec or remove the dependency          |
| Review rejected: build needs a local game install                | Tool such as `RLArenaCollisionDumper` (PR #27)                        | Remove the tool from the build and runtime              |

---

## Reviewing a PR to this repository

Run these checks in order. Stop and report the first failing check; do not try to fix the bot inside the botpack PR.

1. **Read the PR body.** It should name the bot, its agent id, its strength evidence, and what was tested. Ask for the missing pieces.
2. **Check the diff.** A submission should add one `.gitmodules` entry and one gitlink under `Botpack/`. Anything else needs an explanation. Do not accept a copied source tree in place of a submodule.
3. **Verify the pin.** Run `git ls-remote <url> HEAD` for the submodule URL and compare with `git ls-tree HEAD Botpack/<name>` in the PR. The commit must be reachable from the bot repository's default branch or a pushed branch.
4. **Review the bot repository.** Clone it separately or use `git show <sha>:<path>`. Apply every checklist above: README requirements, `bob.toml`, bot config, builder-specific rules.
5. **Check the CI run.** The PR workflow downloads the last release, builds only changed projects, and runs on `ubuntu-24.04`. A green run proves the Docker build succeeded. It does not prove the bot plays.
6. **Check the built layout** when CI artifacts are available. The project folder must contain `bot.toml`, the logo, and a platform folder with the entry binary. For GGL bots, confirm `000-runtime/GGLBotCore` exists and no large LibTorch files are present.
7. **Check for regressions in workflows.** Do not let a submission PR change `.github/workflows/*` unless the PR is about CI itself.

When a fix is needed, open the PR against the **bot's repository**, not this one. Then tell the submitter to update the submodule pointer and push. This is the loop used in PR #39 and in the Seer and Rain fixes.

---

## Converting a bot that already works in RLBot

1. Fork or clone the bot repository. Work there, not in this repository.
2. Read its `bot.toml`. Fill in every `[details]` field. Set a unique `agent_id`. Add a real logo and loadout. Remove empty or invented values.
3. Pick the builder from the table above. Copy its template `bob.toml` and set `project_name` and `bot_configs`.
4. For Python, generate the spec with the flags in the Python section. Add `ctypes.wintypes` to hidden imports. Exclude torch. Put models in `datas`.
5. Replace every `import torch` with the archive snippet. Use `../../torch-archive`. Add `weights_only=True` to `torch.load`. Replace CWD-relative model paths with a `sys._MEIPASS`-aware helper.
6. For Rust and C#, copy the example configs; no Dockerfile.
7. For GGL/C++, rebase onto the `bob` branch of GGLBot and follow the C++ section.
8. Remove anything that needs a local Rocket League install, a GPU, or a network download at startup.
9. Install bob from https://github.com/RLBot/bob/releases and run `bob build bob.toml` with Docker running. Use Linux containers.
10. Inspect `bob_build/<project_name>/`. Verify both platform folders exist, the entry binary is the expected one, and all models and data are present.
11. Install the built pack into RLBot and run a match. Bob cannot test gameplay; CI does not either.
12. Commit `bob.toml`, the spec or Dockerfile, and all source. Push. Then open a PR here that adds `Botpack/<name>` as a submodule at that commit.

---

## What an agent cannot verify from a PR

- Gameplay, strength, and skill limit. These need a match and evidence from the submitter.
- Windows runtime behavior. Bob's CI builds Windows binaries, but nobody runs them in a match there.
- Runtime torch compatibility when a bot needs a version other than the archive's. The archive in this pack pins torch 2.4.1.
- Whether a dependency is available in the shared archive without inspecting the archive's package list, which is in https://github.com/VirxEC/pytorch-archive.

Report these as open questions instead of assuming they pass.
