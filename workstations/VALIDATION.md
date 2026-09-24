# Workstation validation, 2026-09-24

Target: `spark-9667`, ARM64, user `deadspark`, factory GNOME.
Upstream base: `fd5cf97b50dacac4692f270aeb76cd6acd4b6d5d`.

## Completed checks

- Thirteen workstation tests passed, including rollback, recovery, locking,
  host checks, shell preservation, and saving only the intended Git files.
- The complete Python suite passed in the disposable Nix container:
  55 tests, one existing environment-dependent skip.
- The real Nix 2.35.2 profile integration test passed in that x86_64 container.
  It uses real store paths, `nix-env` generations, and executable checks,
  without mocked profile switches. It covers failed first installation,
  failed update, recovery from a persisted interrupted transaction, disabling,
  reactivation, and verifies retained generations appear among Nix GC roots.
- The curses menu rendered and quit successfully in PTYs at 90x24 and 40x8.
  The latter exercises its terminal-too-small message.
- The ARM64 workstation derivation evaluated successfully. This alone does
  not establish an ARM64 package build or runtime pass.
- `nix flake check --no-build --all-systems --no-write-lock-file` passed for
  the complete flake. These are evaluations, not executions of its systemd
  container tests.
- Modified Python passed Ruff lint/format; modified Nix passed nixfmt.
  Documentation and whitespace checks passed.

The first wider-suite attempt on the local host lacked `pre-commit`. The first
container attempt needed a per-process Git ownership exception for the
read-only source mount. Both environment issues were resolved in the disposable
container; no development dependencies were installed on the DGX host.

## Live host observations and installation hold

SSH, Docker, GDM, and NVIDIA persistence were active. The live GPU reported
NVIDIA GB10, driver 580.178.04. Nix was absent from PATH and no workstation
profile had been activated.

Systemd reported degraded because NVIDIA's
`nvidia-remove-gnome-software-once.service` failed during first boot when another
APT process held the dpkg lock. Its script retries at a subsequent boot and
uses `apt-get -y --auto-remove purge gnome-software gnome-contacts`.

A read-only APT simulation proposed eleven removals, including `gnome-core`,
GNOME Software, Contacts, and `nvidia-firmware-580` version 580.82.09. That
firmware package is older than the separately installed
`nvidia-firmware-580-580.178.04` matching the active driver. No claim is made
that the proposed removal would break GPU operation; it nevertheless changes
factory packages and requires an explicit maintenance decision.

The failed service has not been restarted, disabled, or reset. The upstream
Nix bootstrap requires a healthy systemd state, so host installation is held.
Do not clear the failure merely to pass that check. Either deliberately keep
these applications and disable their pending one-time removal, or review and
complete NVIDIA's cleanup; then recheck host health before bootstrap.

No OS snapshots, host reboot, Nix installation, or package-profile activation
are claimed by these tests. Existing terminal dotfiles remain managed by their
previous deployment.

## Later host checkpoint: desktop retention selected

The owner explicitly chose to keep the desktop applications. They executed
the reviewed `~/keep-desktop.sh` through their own sudo terminal. It recorded
the unit's previous state under `~/.local/state/terminal-setup/`, disabled only
the one-time package-removal service, and acknowledged that unit's failure.
Independent SSH checks then confirmed:

- systemd status `running`;
- the one-time service `disabled`, `inactive`, result `success`;
- GNOME Software and Contacts still installed;
- GDM, SSH, Docker, NVIDIA persistence, Dashboard, and Dashboard Admin active.

The original health hold is resolved. No packages were removed and no reboot
occurred. Re-enabling this unit would restore its pending removal at a future
boot; that would reverse the owner's selected policy and is not automatic
recovery behavior.

The official installer freshness check found a separate hold:
the repository pins 2.35.1, while the currently published installer is 2.35.2
(ARM64 SHA-256 `a1b35e56da5adadbc117c3cf17b83948ac657f3c0bd79d47bbe0aa70832b5c8e`).
The check verified the downloaded artifact and returned its documented
update-needed exit status. The pin has not been changed and host Nix remains
absent. Updating it affects the existing bootstrap lifecycle evidence and
must receive matching installation/uninstall/recovery tests before activation.

## Native ARM64 package validation

The separate `sparkwerx-workstation-validation` Docker container built the
workstation package successfully on the Spark. Its only host bind mount was
the checkout, read-only; it had no GPU access, Docker socket mount, privileged
mode, host Nix store, or host home-directory mount.

Image: `nixos/nix:2.35.2`, manifest digest
`sha256:7a007c766426c1877758ddc5cb87a965ac131fc78c582ce0083d922d51ae945c`.
Docker inspected its selected architecture as `arm64`.

Built derivation:
`/nix/store/w46bph3llxr3yb5wam67s4g527503aia-sparkwerx-deadspark.drv`.
Output:
`/nix/store/19ppl4g73746z67i8xwnd0apfdnrz8yp-sparkwerx-deadspark`.

Version checks passed for ncdu 2.9.2, lazydocker 0.25.2, Devbox 0.18.0,
nh 4.4.2, and fzf 0.72.0. Help/startup checks passed for nix-output-monitor
2.2.0 and nix-search-tv 2.2.7; their versions came from the built manifest.
nix-search-tv does not implement `--version`, so its initially rejected version
flag was replaced with its supported help check. The packaged `spark` launcher
ran after its expected checkout path was provided inside the container.

The real Nix-profile lifecycle test passed again natively on ARM64. In
addition, the actual package output above was activated, disabled, and restored
in a temporary container-only profile, with its executable postflight passing
on both activations. These checks did not create a host package profile.
