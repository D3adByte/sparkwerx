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
