# Factory GNOME workstation setup

This branch adds a separate workstation package lifecycle for
`deadspark@spark-9667`. The NVIDIA OS, GPU stack, desktop, SSH, Docker, and
existing APT terminal packages keep their existing ownership.

## Scope

The pinned Home Manager configuration supplies the package environment. This
operator activates only that package output through an isolated Nix profile at
`~/.local/state/sparkwerx/profile`. It does **not** execute Home Manager's
file/service activation script or alter the normal `~/.nix-profile`. Dotfile
ownership remains with the existing terminal setup; the two reference files
under `terminal/` document the initial selection, not a new deployment.

The initial packages are the existing Sparkwerx fleet base (`ncdu`,
`lazydocker`, `devbox`) plus `nh`, `nix-output-monitor`, `nix-search-tv`, `fzf`,
and the `spark` management launcher. Dependencies come from the committed
lock file. Workstation package updates have a separate `nixpkgs-workstation`
input; they do not update the root or pilot package pins.

The unchanged upstream Nix installer still reads `fleet/hosts.json`. Its
`armen` logical key is used only as a compatibility mapping to `deadspark`;
all personal overlay and Codex flags are false. The actual workstation profile
is defined separately in `workstations/hosts.json`. Neither the existing
headless `converge` nor the pilot-specific Home activation operator applies to
this workstation. The initial installer retains its receipt, snapshots,
checksums, factory-service checks, and timed rollback.

## Setup

The checkout must be at `~/Development/DGX-setup` for the installed launcher.
Run the commands on the declared Spark, as `deadspark`, without prefixing the
whole command with sudo. Bootstrap requests sudo internally.

```bash
./scripts/dgx-workstation status
./scripts/dgx-workstation bootstrap
```

Open a fresh SSH session after Nix bootstrap so its commands enter PATH. Then:

```bash
./scripts/dgx-workstation build
./scripts/dgx-workstation plan
```

After reviewing and saving the configuration, activate and add the
conditional package PATH entry to the existing Zsh local overrides file:

```bash
./scripts/dgx-workstation save
./scripts/dgx-workstation apply
./scripts/dgx-workstation shell
exec zsh
spark
```

The terminal menu uses arrow keys or j/k, Enter, and q. Long-running operations
show their live output. Changes to package selection and pins only edit the
working tree; they require review, a build, and a commit before activation.
The CLI is available for every action. `spark --help` lists them. The menu's
save action commits only the package selection and lock file, refuses unrelated
changes, and uses a local `sparkwerx@localhost` author. It never pushes.

`nh` provides package search and build inspection, and `nix-search-tv` provides
search data for fzf/Television. Use the Sparkwerx operator to activate or roll
back this profile: an unrelated `nh home switch` would use a different profile
and activation contract. There is no generation-deletion or garbage-collection
action in the menu.

## Recovery

```bash
spark generations
spark rollback 1
```

The number is an example: select an actual retained generation from the list.
Activation checks the candidate's host/account manifest, serializes operations,
and writes a recovery journal before changing the profile. The profile switch
uses Nix's atomic generation selection. If command checks fail, the prior
profile is restored immediately. If the process is forcibly killed, the
journal survives; after reconnecting run:

```bash
./scripts/dgx-workstation recover
```

To deactivate all packages from this isolated profile while keeping recovery
history:

```bash
spark disable
```

Use the checkout's script to restore a generation after disabling its launcher.
Disabling removes only the active profile symlink; previous generation roots
remain. The conditional Zsh hook becomes inactive. Its original file is backed
up before the first edit, and unrelated overrides are preserved.

These package generations are not an OS snapshot or an application-data backup.
The Nix installer has its own receipt-based recovery; do not treat package
rollback as uninstalling Nix. No OS reinstall or factory reset is part of this
workstation lifecycle.

## Checks

```bash
python3 -m unittest discover -s dev -p test_workstation.py
python3 scripts/check-docs.py
```

The failure tests cover first-install rollback, update rollback, interrupted
operation recovery, concurrent operations, symlink refusal, shell-file
preservation, and wrong-host refusal. See [validation](VALIDATION.md) for real
Nix-profile lifecycle results and the current host installation hold.

Existing tool references:
[nh](https://github.com/nix-community/nh),
[nix-search-tv](https://github.com/3timeslazy/nix-search-tv),
[Nixmate](https://github.com/daskladas/nixmate).
Nixmate's NixOS rebuild workflow is not the workstation's Ubuntu lifecycle.
