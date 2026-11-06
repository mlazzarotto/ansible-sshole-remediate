# sshhole-remediate

[![lint](https://img.shields.io/github/actions/workflow/status/mlazzarotto/ansible-sshole_remediate/lint.yml?branch=main&label=lint)](https://github.com/mlazzarotto/ansible-sshole_remediate/actions/workflows/lint.yml)
[![license](https://img.shields.io/github/license/mlazzarotto/ansible-sshole_remediate?v=1)](https://github.com/mlazzarotto/ansible-sshole_remediate/blob/main/LICENSE)
[![tag](https://img.shields.io/github/v/tag/mlazzarotto/ansible-sshole_remediate)](https://github.com/mlazzarotto/ansible-sshole_remediate/tags)
[![last commit](https://img.shields.io/github/last-commit/mlazzarotto/ansible-sshole_remediate)](https://github.com/mlazzarotto/ansible-sshole_remediate/commits/main)
![Alpine](https://img.shields.io/badge/platform-Alpine-0D597F?logo=alpinelinux&logoColor=white)
![ansible-core](https://img.shields.io/badge/ansible--core-2.16%2B-EE0000?logo=ansible&logoColor=white)
[![CVE-2026-123158](https://img.shields.io/badge/CVE--2026--123158-security-critical)](https://nvd.nist.gov/vuln/detail/CVE-2026-123158)

Ansible playbook that upgrades OpenSSH on Alpine Linux to close CVE-2026-123158
("sshhole"), then runs the detection script again and fails if the host is
still exposed.

## The problem

On musl libc without PAM, `sshd` accepts any password longer than 256 bytes for
accounts whose shadow field is exactly `*`. That is not every passwordless
account: busybox `adduser -D` writes `!`, and the script only counts `*`. Check
what your hosts actually have with the detection script before assuming.
The fix landed in OpenSSH `10.6p2`, which is in edge while stable is still on
`10.0p1`.

## Requirements

- `ansible-core` 2.16 or newer on the controller.
- Alpine 3.16 or newer on the targets.
- Python 3 on the targets. The first play installs `python3` when missing; set
  `-e sshole_bootstrap_python=false` to manage it yourself.
- Root or `become` on the targets.
- The inventory must define an `alpine` group, see `inventory.example.ini`.
- The controller needs network access to GitHub, to fetch the detection script.
- Runs under AWX, AAP 2.5+ and Semaphore UI, see Platform compatibility below.

## Platform compatibility

The playbook is a plain `ansible-playbook` with no roles, no collections and no
interactive prompts, so it runs unchanged under AWX, Red Hat Ansible Automation
Platform and Semaphore UI.

| Platform | Minimum | Notes |
| --- | --- | --- |
| AWX | execution environment with ansible-core 2.16+ | recent `awx-ee` is fine, verify with `ansible --version` |
| Ansible Automation Platform | 2.5 or newer (ships ansible-core 2.16) | 2.4 ships 2.14 and fails on `meta: end_host` |
| Semaphore UI | runner with ansible-core 2.16+ | the official image has lagged before, pin or install it |

### Notes for all three

- Put the targets in an `alpine` group.
- Pass `mode`, `sshole_sha256` and `alpine_reboot` as survey or extra variables.
- The machine credential needs privilege escalation for `become`.
- `delegate_to: localhost` runs inside the execution environment or runner
  container, not on your laptop. Anything written there is ephemeral, so rely
  on the job log instead of a file.
- The runner needs egress to GitHub for the detection script and to the Alpine
  mirror for packages. Override `sshole_script_url` and `edge_repo` for
  air-gapped setups.
- The detection output contains the `*` account names, so it lands in the job
  log and may be visible to more people than you expect.

### Platform setup

- AWX and AAP: create a job template, attach a machine credential with become,
  and add `mode` as a survey variable.
- Semaphore UI: create an inventory with the `alpine` group, an environment with
  the SSH key, and a task template that passes `-e mode=openssh`.

## Usage

```sh
cp inventory.example.ini inventory.ini   # then edit it
ansible-playbook -i inventory.ini sshhole-remediate.yml -e mode=openssh
```

## Modes

### openssh

`mode=openssh` (default) upgrades only the installed `openssh*` packages from
edge. It appends one line to `/etc/apk/repositories`:

```
@edge https://dl-cdn.alpinelinux.org/alpine/edge/main
```

That tag pins the openssh packages to edge. In apk, a tagged package passes the
tag to its dependencies, so a dependency that the untagged repositories cannot
satisfy can also come from edge. On Alpine 3.22 in practice only the six
`openssh*` packages move, because the stable libraries satisfy them. Only
`edge/main` is added; a dependency that lives in `edge/community` would make
the upgrade fail loudly.

Without the pin the next `apk upgrade --available` pulls the stable version back
and silently reopens the hole. The pin lives in `/etc/apk/world` once `apk add`
runs, so it survives reboots and later upgrades.

### full

`mode=full` rewrites every uncommented repository line to `edge`, refreshes the
index and runs `apk upgrade --available`. This is a one way upgrade: there is no
supported downgrade path back to stable, only a reinstall. A previous `@edge`
line and the world entry are left in place; they are redundant once every repo
points to edge, but harmless.

## What changes on the host

- `/etc/apk/repositories`: the `@edge` line, or every line in `full` mode.
- `/etc/apk/world`: the openssh packages become pinned to `@edge`.
- Packages: openssh only, or the whole system.
- Host keys: `ssh-keygen -A` runs after the upgrade. OpenSSH 10.6 adds an
  ML-DSA host key type, and a restart without the key in place fails the key
  exchange while `sshd -t` still returns 0.
- `sshd` is restarted, not reloaded. After an in-place upgrade a SIGHUP makes
  it re-exec the deleted old binary through `/proc/self/exe`, so new
  connections break. A restart keeps established sessions, only the listener
  is replaced. The play then checks that the listener process runs the
  upgraded binary, and fails if it does not.

`/etc/ssh/sshd_config` is never touched. The playbook does not lock accounts or
disable password authentication.

## Idempotency

A second run with no package changes reports `ok` and does not restart `sshd`,
unless the listener is still running an older binary, in which case it is
restarted and reported as changed.

## Rollback

`full` mode keeps a copy before touching anything:

```sh
cp -a /etc/apk/repositories.pre-edge /etc/apk/repositories
```

`openssh` mode is not undone by restoring the repositories file alone. `apk add
openssh@edge` records `openssh@edge` in `/etc/apk/world`, so dropping the only
`@edge` repository definition leaves a world constraint with nowhere to resolve
from. The order matters:

1. Remove the `@edge` suffix from the openssh entries in `/etc/apk/world`.
2. Remove the `@edge` line from `/etc/apk/repositories`.
3. `apk update`.
4. Reinstall the stable versions explicitly. `apk` does not downgrade on its
   own, so the version has to be pinned, for example
   `apk add --upgrade 'openssh=10.0_p1-r11'`.

Test the downgrade on a canary before relying on it.

## Detection script

The playbook does not vendor the script. It downloads it at run time from
[oz9un/sshole](https://github.com/oz9un/sshole), the upstream repository:

```
https://raw.githubusercontent.com/oz9un/sshole/main/check.sh
```

It is fetched on the controller, never on the targets, and executed as root on
every target.

### Exit codes

- `0`: exposed
- `1`: not exposed
- `2`: can't tell

A `2` produces a warning, not a failure. Any other exit code fails the play.

### Pinning the script

The default tracks `main`. Pin it when you want to be sure of what you are
executing:

```sh
ansible-playbook -i inventory.ini sshhole-remediate.yml \
  -e mode=openssh \
  -e sshole_sha256=<sha256 of check.sh>
```

## Dry runs

`--check` runs the target validation, the package inventory, and the exposure
check on the targets, simulates the repository edits, and skips the package
upgrade and the post-check. It is a partial preview, not a full simulation.

## After the run

`mode=full` usually pulls a new kernel. Set `-e alpine_reboot=true` to reboot at
the end with `ansible.builtin.reboot`, which waits for the host to come back.

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `mode` | `openssh` | `openssh` or `full` |
| `alpine_branch` | `edge` | branch written into the repositories |
| `edge_repo` | `https://dl-cdn.alpinelinux.org/alpine/edge/main` | repo pinned as `@edge` |
| `openssh_min_version` | `10.6_p2` | compared with `apk version -t` |
| `sshole_script_url` | upstream `check.sh` on `main` | detection script source |
| `sshole_sha256` | empty | optional checksum pin for the script |
| `sshole_bootstrap_python` | `true` | install `python3` when missing |
| `alpine_reboot` | `false` | reboot at the end |

## License

MIT, see `LICENSE`. The detection script is not part of this repository and
keeps its own license upstream.

## Lint

```sh
ansible-lint
```
