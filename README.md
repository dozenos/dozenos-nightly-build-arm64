# dozenos-nightly-build-arm64

Weekly arm64 image builds for DozenOS — the arm64/BlueField-2 sibling of
[`dozenos-nightly-build`](https://github.com/dozenos/dozenos-nightly-build)
(which stays amd64-only and untouched). Implements
`BF2-FLOWTABLE-OFFLOAD-PLAN.md` §8/§9.0.

## Flow

`nightly.yml`, Mondays 05:17 UTC (+ `workflow_dispatch`):

1. **check-changes** — snapshot every `dozenos/*` mirror `rolling` HEAD
   plus both rebrand toolkits (`dozenos-rebrand`, `dozenos-rebrand-arm64`);
   build only when the fingerprint differs from `mirror-state.json`.
2. **build-packages** — full C2 + linux-kernel matrix on GitHub native
   arm64 runners (`ubuntu-24.04-arm`), inside
   `ghcr.io/dozenos/dozenos-build-arm64:rolling`. Each leg checks out
   `dozenos-build` @ `rolling`, applies
   `dozenos-rebrand-arm64/apply-patches.sh` (BF2 kernel config + OFED
   deltas) with a CI-local throwaway commit, then probes/stores the
   arm64-only deb cache (`dozenos-deb-cache-arm64`).
3. **kernel-config gate** (plan §9.0) — the linux-kernel leg fails
   immediately if any BF2 offload/BlueField option is missing from the
   built kernel (`MLX5_CLS_ACT`, `MLX5_TC_CT`, `MLXBF_*`, …).
4. **build-image** — one leg per flavor toml in
   `dozenos-rebrand-arm64/flavors/`, `build-dozenos-image --architecture
   arm64`, ephemeral localhost-HTTP apt repo from this run's own debs.
5. **publish** — minisign every artifact, one GitHub Release per version,
   commit `version.json` + `mirror-state.json` back as the change-gate
   baseline. Then prune (deb cache keep-3, releases keep-15).

`build-container.yml` (Mondays 02:30 UTC) builds the arm64 build container
from the `dozenos-build` mirror's own `docker/` and pushes
`ghcr.io/dozenos/dozenos-build-arm64:rolling`.

## Deliberate round-1 gaps

- **No QEMU smoketest gate** — needs an arm64 test VM environment; wired in
  a later round (plan §8). Publish gates only on every flavor building.
- **No BFB wrap job** — the image→`.bfb` packaging (plan §7) lands as a
  job here once `dozenos-rebrand-arm64/bfb/` tooling is functional.
- **No Secure Boot / shim / MOK** — amd64-only; BF2 boots via NVIDIA
  ATF/UEFI.
- **Copy, not reusable workflow** — this file deliberately duplicates the
  amd64 `nightly.yml` shape so the amd64 pipeline is never touched;
  merging both into one reusable workflow is a later cleanup.

## Verify a download

```sh
minisign -Vm <artifact> -p minisign.pub
```
