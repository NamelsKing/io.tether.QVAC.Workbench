# Beta updater

Proposes a pull request against `beta` when QVAC publishes a newer, verified
staging Flatpak. A human reviews and merges it. The updater never merges and
never enables auto-merge. The production manifest on `master` is only read.

## How it works

`.github/workflows/beta-updater.yml` runs every 4 hours and on manual dispatch,
from the default branch only. Code is taken from the trusted default-branch
commit; the `beta` checkout is data (read as text, never executed or
YAML-parsed). The only credential is the job's `GITHUB_TOKEN`
(`contents: write`, `pull-requests: write`); no secrets are used.

`update_beta.py` (standard library only):

1. Lists `https://static.tether.dev/qv.ac/` (the same listing the production
   checker reads) and keeps `beta-X.Y.Z-rc.N.B/` directories. The newest by
   numeric order wins (`rc.10` > `rc.9`); `beta/`, `latest/` and stable
   `X.Y.Z/` are ignored, so stable discovery is untouched.
2. Does nothing (exit 0) when that version is already in beta (`duplicate`),
   older than beta (`older`), already has an updater PR (`proposed`), is not
   listed (`no-publication`), or any required file is still missing
   (`incomplete`, retried next run). Only the newest version is considered.
3. Verifies `beta-<v>/flatpak/` for x64 and arm64: `QV.AC-<arch>-flatpak.tar.gz.sha512`
   is `<sha512>  <name>` and `io.tether.qvac.source-<arch>.yml` is the canonical
   snippet pointing at that immutable URL with the same digest. Then it downloads
   the x64 archive, re-hashes it, and lists entry names (nothing is extracted)
   to confirm `<app-id>.desktop`, `.metainfo.xml`, `.png`, `run.sh` and `main/`
   exist. The manifest must reference the same names, so an app-ID change that
   is not matched in the archive stops the update. The arm64 archive is not
   downloaded or consumed.
4. Rewrites only the `qvac-workbench` archive `url`, `sha512` and `x-checker-data`
   (line-based, verified by re-reading). A verification failure exits 1 and the
   run is red; nothing is written.

**Bootstrap:** if the beta manifest has no `beta-<v>/flatpak/` source URL yet
(today's obsolete packaging), the base is the production manifest on `master`
(runtime, wrapper and everything else unchanged). Afterwards the base is the
beta manifest itself, so beta-only human edits survive.

The workflow commits only the manifest to `beta-update/<version>`, opens a PR
to `beta`, closes older open `beta-update/*` PRs, and always uploads the diff as
a `beta-update-<version>` artifact.

## Operating

- Repository settings: Actions workflow permissions must allow creating pull
  requests. PRs opened with `GITHUB_TOKEN` do not trigger other workflows, so
  start Flathub test builds the usual way. Scheduled workflows are disabled after
  60 days without repository activity.
- Run now: Actions > Beta updater > Run workflow. Choose `patch` to get only the
  artifact (no branch, no PR). Set the repository variable `BETA_UPDATER_MODE=patch`
  to make scheduled runs patch-only, e.g. if Flathub policy objects to bot PRs.
- Local check, no writes: `python3 -I .github/beta-updater/update_beta.py --master io.tether.QVAC.Workbench.yml --beta-dir <beta checkout> --dry-run`
- Tests (offline, no network, no real archive):
  `python3 -I -m unittest discover -s .github/beta-updater/tests -v`

## Contract with QVAC

QVAC uploads immutable `qv.ac/beta-<X.Y.Z-rc.N.B>/flatpak/` with, per arch,
`QV.AC-<arch>-flatpak.tar.gz`, its `.sha512` and `io.tether.qvac.source-<arch>.yml`,
then promotes the managed `beta` link. Changing any name or format means changing
`update_beta.py` and its tests together.

## AI assistance

The updater, its tests, the workflows and this document were written with AI
assistance and are subject to maintainer review in the pull request that adds
them. No manifest content is AI-authored: the beta manifest is the human-authored
production manifest with only `url`, `sha512` and `x-checker-data` replaced by
deterministic code from a verified publication. No manifest or template is
committed here.
