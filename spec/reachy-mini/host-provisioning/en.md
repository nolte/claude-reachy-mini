# Host Provisioning: WiFi, Filesystem Layout, and App Distribution

Status: draft

## Context
This repository (`claude-reachy-mini`) provides toolbox content (skills, agents, specs) for apps such as `reachy-mini-show` (see [`reachy-mini/app-architecture`](../app-architecture/en.md)). Pollen's default path installs Hugging Face Spaces into the **shared venv** `/venvs/apps_venv/` and manages them via the daemon app lock — one app at a time, visible in the Reachy dashboard. This spec covers a **parallel** operations path for self-developed apps that should run independently of Pollen's dashboard — as systemd services in a dedicated filesystem area on the robot. It closes three gaps that neither the app architecture spec nor the on-device spec currently address: (1) WiFi onboarding into the home network, (2) deterministic filesystem layout for self-hosted apps, (3) two coordinated distribution paths — push from the developer notebook via Ansible (with an scp fallback), and pull directly on the robot via git clone plus a systemd timer.

## Goals
- A new Reachy is on the home network and reachable via SSH after a reproducible setup sequence
- Self-developed apps have a clear, FHS-compliant filesystem layout on the robot, separate from Pollen's shared venv
- Two coordinated distribution paths: push (Ansible role in the plugin repo + scp fallback) and pull (git clone + systemd timer for `git pull --ff-only`)
- The app lifecycle runs through systemd system units, not through Pollen's app manager
- Visible provenance: every generated inventory / service / timer file carries a reference back to the plugin

## Non-Goals
- Replacing Pollen's shared-venv path (still valid for HF Spaces installs — only delineated here)
- Firmware flash, hardware bring-up (separate skills planned)
- Cloud deploy pipelines (e.g. CI-to-robot)
- Multi-tenant separation on a single robot
- Auth/TLS on the app WebSockets — stays `localhost`-only (see [`app-architecture`](../app-architecture/en.md))

## Requirements

### WiFi Onboarding
- **MUST** document a headless-capable path (USB console or recovery AP), because a fresh Reachy is not connected to the home network
- **MUST** support WPA2/WPA3-PSK as the minimum profile; open networks MUST be documented as a risk and refused
- **MUST** persist the result as static `nmcli` connection profiles on the Reachy (`/etc/NetworkManager/system-connections/<ssid>.nmconnection`, mode `0600`)
- **MUST** generate a deterministic mDNS address: `reachy-mini-<short-id>.local` (`short-id` derived from serial or MAC hash), so multiple devices remain addressable in parallel
- **MUST** make SSH key-based authentication mandatory; password auth MUST be disabled after onboarding completes (`PasswordAuthentication no`)
- **SHOULD** allow a rollback connection (a stored test WLAN), so a failed WiFi update does not lock the operator out of the robot
- **MUST NOT** write WLAN credentials into the plugin repo, into logs, or into output protocols — inventory variables MUST be sourced via Ansible Vault or an external secret store

### Filesystem Layout
- **MUST** establish the following layout on the Reachy:

  ```
  /opt/reachy-apps/                              # Root for self-developed apps (root:reachy-apps, 2775)
  /opt/reachy-apps/<slug>/                       # One app per slug
  /opt/reachy-apps/<slug>/source/                # Git working copy or rsync target
  /opt/reachy-apps/<slug>/.venv/                 # Per-app venv (uv venv)
  /opt/reachy-apps/<slug>/var/log/               # App logs (rotated via journald or logrotate)
  /opt/reachy-apps/<slug>/var/state/             # Persistent app state, never under VCS
  /opt/reachy-apps/<slug>/etc/                   # App configuration (config.toml, env files)
  /opt/reachy-apps/<slug>/PROVENANCE.md          # Plugin / spec reference
  /etc/systemd/system/reachy-app@.service        # Template unit, instantiated per slug
  /etc/systemd/system/reachy-app-pull@.service   # Pull update service (oneshot)
  /etc/systemd/system/reachy-app-pull@.timer     # Timer for pull update
  ```

- **MUST** create the `reachy-apps` group; the default user (`pollen`) MUST be a member
- **MUST** set permissions: `/opt/reachy-apps` as `2775` (setgid), files as `0644`, directories as `2755`, venv contents using uv defaults
- **MUST** define a systemd template unit `reachy-app@.service` that takes `<slug>` as the instance name and runs under the `pollen` user
- **MUST** create the per-app venv via `uv venv` (not `python -m venv`) — consistent with the app architecture spec
- **MUST NOT** write into `/venvs/apps_venv/` (Pollen's shared venv) — collision-free separation is the central justification for this layout
- **MUST NOT** place app sources or venvs under `/home/pollen/`, because OS reinstalls can be destructive there

### systemd Service Contract
- **MUST** define a template unit `reachy-app@.service` that, for `<slug>`, sets `ExecStart` to `/opt/reachy-apps/<slug>/.venv/bin/python -m <slug_underscore>.main`
- **MUST** set `Restart=on-failure`, `RestartSec=5`, `User=pollen`, `Group=reachy-apps`
- **MUST** set `EnvironmentFile=-/opt/reachy-apps/<slug>/etc/env` (the `-` makes the file optional)
- **MUST** set `WorkingDirectory=/opt/reachy-apps/<slug>/source`
- **MUST** apply systemd hardening: `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `ReadWritePaths=/opt/reachy-apps/<slug>/var`, `PrivateTmp=true`
- **SHOULD** set CPU/memory quotas per app (e.g. `MemoryMax=512M`), since the RPi 4 CM4 has limited resources
- **MUST NOT** name a systemd unit identically to a Pollen-owned unit (`reachy-mini-daemon.service`) or override its `ExecStart`

### Push Path: Ansible
- **MUST** ship an Ansible role `reachy-app` in the plugin repo under `ansible/roles/reachy-app/`
- **MUST** ship an inventory template at `ansible/inventory.example.yml` declaring the variables `app_slug`, `app_repo_url`, `app_branch`, `app_python_version`, `wifi_ssid` (Vault), `wifi_psk` (Vault), `ssh_pubkey`
- **MUST** structure the role into the following tasks: `network` (WiFi profiles), `users` (`reachy-apps` group, SSH key), `filesystem` (directory layout, permissions), `runtime` (uv install, venv bootstrap), `app` (rsync the source, `pip install -e .`), `service` (systemd unit render, `daemon-reload`, `enable`, `start`)
- **MUST** be idempotent — a second run without source changes produces zero tasks in the `changed` state
- **MUST** encrypt secrets (`wifi_psk`, optional `git_deploy_key`) via Ansible Vault; the vault password is sourced externally (`ANSIBLE_VAULT_PASSWORD_FILE` or `--vault-password-file`)
- **SHOULD** offer a `--check`-capable dry-run variant (`ansible-playbook ... --check --diff`)
- **SHOULD** reference the scp fallback (see next section) from the role documentation — the role itself uses rsync via `ansible.builtin.synchronize`
- **MUST NOT** have the role modify Pollen-owned paths (`/venvs/apps_venv/`, `reachy-mini-daemon.service`)

### Push Path: scp Fallback
- **MUST** document a minimal scp/rsync fallback that works without Ansible — as a copy-pasteable snippet, not as a second pipeline
- **MUST** have the fallback hit the same permissions and the same path (`/opt/reachy-apps/<slug>/source`) so subsequent Ansible runs recognize the state
- **MUST** have the fallback explicitly document which steps it does **not** cover (WiFi setup, group setup, vault secrets) — it is app sync only, not full bootstrap
- **MUST NOT** advertise the fallback as the "preferred path" — Ansible is the default

### Pull Path: Git on the Robot
- **MUST** make the initial `git clone` part of the push role as the Ansible task `app/git_clone.yml` (even when subsequent updates run device-locally) — bootstrap once, then self-sustaining
- **MUST** provide a systemd template unit `reachy-app-pull@.service` as `oneshot`: it runs `git fetch && git reset --hard origin/<branch>` (hard reset against the branch HEAD, since a robot filesystem must not retain local commits), `uv pip install -e .` if `pyproject.toml` changed, and `systemctl restart reachy-app@<slug>` if sources or the venv changed
- **MUST** provide a timer unit `reachy-app-pull@.timer` with default interval `OnUnitActiveSec=5min` and `RandomizedDelaySec=30s` (jitter, in case multiple robots on the same network pull simultaneously)
- **MUST** have the pull path use a read-only deploy key (GitHub or Gitea) for auth, **never** a user token with write rights
- **MUST** install the deploy key via Ansible Vault into `/opt/reachy-apps/<slug>/etc/deploy.key` with mode `0400`, owner `pollen`
- **MUST** set the branch pin in `etc/env` (`APP_GIT_BRANCH`) exclusively to `main`; `develop` and feature branches MUST be delivered via the push path (Ansible/scp), not via the pull timer
- **MUST** have the pull service run a disk pre-check before `git fetch`: when less than 1 GiB is free under `/opt/reachy-apps/`, run `journalctl --vacuum-time=14d` and set a marker for the next run that rebuilds the venv from scratch (`rm -rf /opt/reachy-apps/<slug>/.venv` followed by `uv venv`)
- **MUST** have the pull service read the battery SoC via the Pollen daemon REST API on Wireless and skip the pull run below 30 % SoC — log line `pull skipped: battery <n>%`, no `git fetch`, no service restart
- **MUST** have the pull service check `GET /api/daemon/robot-app-lock-status` before every pull — when the Pollen daemon holds an app in its lock, the pull is skipped and logged to journald as `pull skipped: pollen-app-lock <holder>`; a currently running Reachy-app service is **not** touched
- **MUST** ensure a pull failure (network down, auth error, merge conflict) does **not** tear down the running service — only a failed pull-run log line in journald
- **SHOULD** log pull successes as a reduced log volume (`success: <sha-short>`), not the full `git pull` output
- **MAY** retrofit a webhook trigger for higher freshness (separate later spec, explicitly a non-goal here)
- **MUST NOT** allow local edits on the robot in the pull path — the hard reset against origin is contract, not a bug

### Coexistence with Pollen's Shared-Venv Path
- **MUST** explicitly document that self-developed apps in this layout do **not** appear in the Reachy dashboard and do **not** hold Pollen's app lock — they run alongside the Pollen daemon, not in its app slot
- **MUST** document that behaviors in these apps still talk to the hardware via the Pollen daemon REST API (port 8000) and the ReachyMini SDK connection — they remain SDK consumers, just outside the Pollen app framework
- **MUST** include a "When to use this path instead of HF Spaces install?" section: fast iteration without HF push, multiple parallel apps, hard-wired configuration, Ansible-driven home setup
- **MUST NOT** position this path as a replacement for a concurrently running Pollen app — when Pollen's daemon holds an app in its lock, a parallel self-hosted behavior collides on the hardware connection; the operator must know this

### Logging and Diagnostics
- **MUST** route app logs through journald (`journalctl -u reachy-app@<slug>`), consistent with the Pollen daemon logs
- **MUST** also have the pull service log to journald (`journalctl -u reachy-app-pull@<slug>`), readable separately
- **MUST** make `PROVENANCE.md` in the app directory and the header of the systemd units carry a plugin reference, so an operator on the machine can see where the state came from
- **MUST NOT** include tokens, vault values, WiFi PSKs, or deploy keys in logs

### Security
- **MUST** configure the SSH server (`/etc/ssh/sshd_config.d/10-reachy-apps.conf`) for key-only auth, no root login, optionally a non-default port
- **MUST** ensure the default user `pollen` needs neither NOPASSWD-sudo nor permanent sudo to deploy an app — the Ansible role escalates in a controlled fashion
- **MUST** use a dedicated SSH key pair (from the notebook) for each Reachy, no shared keys across multiple devices
- **MUST** never let vault values (`wifi_psk`, `git_deploy_key`) appear in plain inventory files — `ansible-vault encrypt_string` is mandatory
- **SHOULD** activate Fail2Ban or equivalent against repeated SSH auth failures
- **MAY** recommend a separate VLAN / network segmentation — guidance, not a MUST

### Dependency Lifecycle
- **MUST** keep responsibility for dependency updates fully in the respective app repository (Renovate / Dependabot live there, not in the plugin); the robot passively pulls the state pinned in `pyproject.toml`
- **MUST** have the pull service call only `uv pip install -e .` and never `uv lock --upgrade` or `pip install -U` — updates flow exclusively through repo commits, not through robot-local re-resolution
- **MUST NOT** have the plugin repo dictate a Renovate or Dependabot configuration for app repos; a recommendation template MAY emerge in a later spec, but not here

### Decommission
- **MUST** ship an idempotent Ansible playbook `ansible/playbooks/reachy-app-decommission.yml` in the plugin repo
- **MUST** have the playbook execute the following steps, in this order: `systemctl disable --now reachy-app-pull@<slug>.timer`, `systemctl disable --now reachy-app-pull@<slug>.service`, `systemctl disable --now reachy-app@<slug>.service`, delete `/opt/reachy-apps/<slug>/`, remove the SSH public key from `~pollen/.ssh/authorized_keys` (when this robot was the sole consumer), remove the host from the inventory file
- **MUST** have the playbook require a `pause` / `prompt` confirmation before each destructive step — an accidental wipe must not be triggered by an inventory typo
- **MUST** have the playbook be runnable in `--check --diff` mode and report every planned change without executing it
- **MUST** have the playbook expose a variable `purge_wifi_profile` (default `false`) — when `true`, the `nmconnection` file is removed; when `false`, the robot stays reachable on the home network (standard case: app-only decommission, not full device decommission)
- **MUST NOT** have the playbook touch Pollen-owned paths (`/venvs/apps_venv/`, `reachy-mini-daemon.service`, `/etc/systemd/system/reachy-mini-daemon.service`) — it cleans up plugin state exclusively

## Acceptance Criteria
- [ ] Ansible role `reachy-app` exists at `ansible/roles/reachy-app/` with the six task files (`network`, `users`, `filesystem`, `runtime`, `app`, `service`)
- [ ] Inventory template `ansible/inventory.example.yml` is in the repo, the `ansible-vault` marker is documented
- [ ] systemd templates `reachy-app@.service`, `reachy-app-pull@.service`, `reachy-app-pull@.timer` are present as Jinja2 templates in the role
- [ ] A second Ansible run without source changes produces zero `changed` tasks
- [ ] The WLAN profile is persisted after setup as an `nmconnection` file with mode `0600`
- [ ] Password auth is disabled in sshd after bootstrap
- [ ] The app filesystem at `/opt/reachy-apps/<slug>/` exists with the prescribed subdirs and permissions
- [ ] `journalctl -u reachy-app@<slug>` shows app logs; `journalctl -u reachy-app-pull@<slug>` shows pull logs
- [ ] The pull timer fires every 5 min ± 30 s and performs a hard reset against the origin branch
- [ ] A pull failure (e.g. network down) does **not** tear down the app service
- [ ] `PROVENANCE.md` in the app directory links back to the plugin
- [ ] A written delineation against the Pollen shared-venv route exists, clarifying when to choose which path
- [ ] WiFi PSK and deploy key only appear in the repo as vault strings
- [ ] The scp fallback is documented as a complementary snippet, not as a second pipeline
- [ ] The mDNS address `reachy-mini-<short-id>.local` resolves from the notebook
- [ ] The systemd hardening options (`NoNewPrivileges`, `ProtectSystem`, `ProtectHome`, `ReadWritePaths`, `PrivateTmp`) are set in the template unit
- [ ] The `reachy-apps` group exists, `pollen` is a member, `/opt/reachy-apps` is `2775`
- [ ] `APP_GIT_BRANCH` is set to `main` on every robot; the develop state reaches the robot only via the push path
- [ ] The pull service skips below 30 % battery SoC on Wireless with a `pull skipped: battery <n>%` log line in journald
- [ ] The pull service skips while the Pollen app lock is held, with a `pull skipped: pollen-app-lock <holder>` log line in journald
- [ ] The pull service vacuums journald logs when less than 1 GiB is free under `/opt/reachy-apps/` and rebuilds the venv on the next run
- [ ] The pull service never calls `uv lock --upgrade` or `pip install -U`
- [ ] `ansible/playbooks/reachy-app-decommission.yml` exists, runs idempotently, has a confirmation step before destructive actions, runs in `--check --diff` mode
- [ ] The decommission playbook touches no Pollen-owned paths
- [ ] The decommission playbook leaves the `nmconnection` file in place by default (`purge_wifi_profile=false`)

## Sources
- App architecture (against which this spec delineates itself): [`spec/reachy-mini/app-architecture/en.md`](../app-architecture/en.md)
- On-device test agent (related lifecycle context): [`spec/claude/reachy-mini-on-device/en.md`](../../claude/reachy-mini-on-device/en.md)
- NetworkManager `nmconnection` format: <https://networkmanager.dev/docs/api/latest/ref-settings.html>
- systemd template units (section "TEMPLATES"): <https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html>
- systemd hardening cheat sheet: <https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html>
- Ansible Vault: <https://docs.ansible.com/ansible/latest/vault_guide/>
- Pollen's Reachy Mini apps documentation (the shared-venv path being delineated against): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- uv (for the per-app venv): <https://docs.astral.sh/uv/>

## Open Questions
- ~~Which branches are valid pull targets?~~ **Answered:** `main` exclusively. `develop` and feature branches travel via the push path — see section "Pull Path: Git on the Robot".
- ~~Should the Ansible role honor Renovate/Dependabot guidance for `pyproject.toml` updates on the robot?~~ **Answered:** no, it stays a repo-only concern. The pull service installs only the state pinned in `pyproject.toml` and never calls `uv lock --upgrade` — see section "Dependency Lifecycle".
- ~~How is a robot decommissioned from the inventory?~~ **Answered:** via the idempotent playbook `ansible/playbooks/reachy-app-decommission.yml` with confirmation steps before destructive actions — see section "Decommission".
- ~~What minimum disk size is realistic on a Wireless before we need auto-cleanup?~~ **Answered:** the pull service vacuums journald logs and rebuilds the venv as soon as less than 1 GiB is free under `/opt/reachy-apps/` — see section "Pull Path: Git on the Robot".
- ~~Should the pull timer react to battery state of charge?~~ **Answered:** yes, on Wireless. Pull is skipped below 30 % SoC; additional skip when the Pollen daemon holds an app in its lock — see section "Pull Path: Git on the Robot".

*(No open questions at this time. New questions will be appended here as they surface.)*
