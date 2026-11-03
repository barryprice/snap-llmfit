# llmfit (snap)

A Snap package of [llmfit](https://llmfit.axjns.dev/) (GitHub:
[AlexsJones/llmfit](https://github.com/AlexsJones/llmfit)) — a tool that
detects your RAM, CPU and GPU and tells you which open-source LLMs will
actually run well on your machine — available from the Snap Store here:

https://snapcraft.io/llmfit-bp

> **Community / unofficial.** This is an unofficial, community-maintained snap.
> The snap `name` is `llmfit-bp`; the `title` is `llmfit` so the software is
> recognised by its real name.

## What's in the snap

The snap installs one binary from upstream, with a suggested alias:

| Binary       | Alias    | Purpose                                                  |
| ------------ | -------- | -------------------------------------------------------- |
| `bin/llmfit` | `llmfit` | TUI / CLI / web dashboard for matching models to hardware |

To enable that alias, after installing:

  ```sh
  sudo snap alias llmfit-bp.llmfit llmfit
  ```

The binary is built with Cargo (Rust stable, via `rustup`) from the source of
a [pinned upstream release](https://github.com/AlexsJones/llmfit/releases),
with the web dashboard (`llmfit-web`) built first and embedded, as upstream's
own release workflow does. The pinned tag lives in `snap/snapcraft.yaml`;
Renovate bumps it via a pull request whenever AlexsJones/llmfit publishes a
new release, so upgrades are reviewed rather than picked up silently on the
next build.

`lspci` (from `pciutils`) is bundled so llmfit can name GPUs.

## Installing

On a snap that has been promoted to `stable`:

```sh
snap install llmfit-bp
```

From the `edge` track:

```sh
snap install llmfit-bp --edge
```

### Interfaces / hardware access

The snap runs under **strict confinement** and declares the following plugs on
the app:

- `hardware-observe` — read access to `/sys` (PCI devices, `/sys/class/drm`),
  which is how llmfit finds GPUs and reads AMD VRAM sizes inside the snap.
  This is **not** auto-connected; without it llmfit sees only CPU and RAM.
  Enable it with:

  ```sh
  sudo snap connect llmfit-bp:hardware-observe
  ```

- `network` — outbound network access. This is **auto-connected**, and is
  what lets llmfit refresh its model list from Hugging Face (`llmfit update`),
  download models, and talk to local providers over HTTP (Ollama, LM Studio,
  vLLM, Docker Model Runner, llama-server).
- `network-bind` — listen on network sockets. This is **auto-connected**, and
  is needed for the web dashboard / REST API (`llmfit serve`).
- `home` — read/write access to `$HOME`. This is **auto-connected** on
  install, and is what lets llmfit find models already downloaded by LM Studio
  or Ollama and read/write files you point it at.

Models llmfit downloads itself are kept in
`~/snap/llmfit-bp/common/.cache/llmfit/models` (`XDG_CACHE_HOME` is set to
`$SNAP_USER_COMMON/.cache`), so they are kept across snap refreshes rather
than copied per revision. Override it with `LLMFIT_MODELS_DIR` if you need
it elsewhere.

> **Confinement limits.** Strict confinement hides host binaries from the
> snap, so llmfit cannot run `nvidia-smi`, `rocm-smi`, `llama-cli` /
> `llama-server`, `docker`, `ramalama` or `hf` from the host. Consequences:
>
> - GPUs are detected via sysfs + `lspci` instead. NVIDIA VRAM is then
>   estimated from the card's model name rather than measured, and free-VRAM
>   readings are unavailable. Use `--memory` to override the detected VRAM if
>   the estimate is wrong.
> - Providers that are driven by spawning a host binary (llama.cpp's CLI,
>   Docker Model Runner's `docker model pull`, RamaLama) are unavailable.
>   HTTP-based providers (Ollama, LM Studio, vLLM, a running llama-server)
>   work.
> - The TUI's copy-to-clipboard action is unavailable (no display access).

## Building locally

Requires `snapcraft` (install with `sudo snap install snapcraft --classic`):

```sh
snapcraft
```

This builds `llmfit-bp_<version>_<arch>.snap` in the project root. You can
also run a local lint:

```sh
snapcraft lint llmfit-bp_*.snap
```

## Continuous integration

[`.github/workflows/snap.yml`](.github/workflows/snap.yml) is CI-only:

- Builds the snap on every push / pull request.
- Lints the built snap with `snapcraft lint` in the same job (a hard gate:
  lint failures fail the build).

This workflow is a **merge gate only** — it never publishes. Building **and**
publishing to `latest/edge` is handled by snapcraft.io, which schedules its
own builds of this snap and pushes them to the `edge` track automatically.
Promoting to a `stable` track remains a manual decision (via the Snap Store
dashboard). This repo is intentionally never tagged — the upstream `llmfit`
repo is where the release tags live.

## License

The packaged software (llmfit) is licensed under the MIT license, as recorded
in `snap/snapcraft.yaml`.
