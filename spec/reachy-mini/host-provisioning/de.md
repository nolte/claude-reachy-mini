# Host-Provisioning: WiFi, Filesystem-Layout und App-Distribution

Status: draft

## Kontext
Dieses Repository (`claude-reachy-mini`) liefert Toolbox-Inhalte (Skills, Agents, Specs) für Apps wie `reachy-mini-show` (siehe [`reachy-mini/app-architecture`](../app-architecture/de.md)). Pollens Default-Pfad installiert Hugging-Face-Spaces ins **Shared-venv** `/venvs/apps_venv/` und verwaltet sie über den Daemon-App-Lock — eine App pro Zeit, sichtbar im Reachy-Dashboard. Diese Spec deckt einen **parallelen** Betriebspfad für selbstentwickelte Apps, die unabhängig von Pollens Dashboard laufen sollen — als systemd-Services in einem dedizierten Filesystem-Bereich auf dem Roboter. Sie schließt drei Lücken, die heute weder die App-Architektur- noch die On-Device-Spec adressieren: (1) WiFi-Onboarding ins Heimnetz, (2) deterministisches Filesystem-Layout für eigene Apps, (3) zwei abgestimmte Distributionspfade — Push vom Entwickler-Notebook via Ansible (mit scp-Fallback) und Pull direkt auf dem Roboter via Git-Clone plus systemd-Timer.

## Ziele
- Ein neuer Reachy ist mit einer reproduzierbaren Schritt-Folge im Heimnetz und SSH-erreichbar
- Selbstentwickelte Apps haben ein klares, FHS-konformes FS-Layout auf dem Roboter, getrennt von Pollens Shared-venv
- Zwei abgestimmte Distributionspfade: Push (Ansible-Rolle im Plugin-Repo + scp-Fallback) und Pull (Git-Clone + systemd-Timer für `git pull --ff-only`)
- App-Lifecycle läuft über systemd-System-Units, nicht über Pollens App-Manager
- Sichtbare Provenienz: alle generierten Inventory-/Service-/Timer-Dateien tragen einen Verweis auf das Plugin

## Nicht-Ziele
- Pollens Shared-venv-Pfad ersetzen (bleibt für HF-Spaces-Installs gültig — wird hier nur abgegrenzt)
- Firmware-Flash, Hardware-Bringup (eigene Skills geplant)
- Cloud-Deploy-Pipelines (z. B. CI-zu-Roboter)
- Mehrbenutzer-Mandantentrennung auf einem Roboter
- Auth/TLS auf den App-WebSockets — bleibt `localhost`-only (siehe [`app-architecture`](../app-architecture/de.md))

## Anforderungen

### WiFi-Onboarding
- **MUSS [MUST]** einen headless-fähigen Pfad dokumentieren (USB-Konsole oder Recovery-AP), weil ein frischer Reachy nicht mit dem Heimnetz verbunden ist
- **MUSS [MUST]** WPA2/WPA3-PSK als Mindest-Profil unterstützen; offene Netze MÜSSEN dokumentiert als Risiko abgelehnt werden
- **MUSS [MUST]** das Resultat als statische `nmcli`-Connection-Profile auf dem Reachy ablegen (`/etc/NetworkManager/system-connections/<ssid>.nmconnection`, Mode `0600`)
- **MUSS [MUST]** eine deterministische mDNS-Adresse erzeugen: `reachy-mini-<short-id>.local` (`short-id` aus Serial oder MAC-Hash), damit mehrere Geräte parallel adressierbar sind
- **MUSS [MUST]** SSH-Key-basierte Authentifizierung Pflicht sein; Password-Auth MUSS nach Abschluss des Onboardings abgeschaltet sein (`PasswordAuthentication no`)
- **SOLLTE [SHOULD]** eine Rollback-Connection (gespeichertes Test-WLAN) zulassen, sodass ein fehlgeschlagenes WiFi-Update den Roboter nicht aussperrt
- **DARF NICHT [MUST NOT]** WLAN-Credentials ins Plugin-Repo, in Logs oder in Output-Protokolle schreiben — Inventory-Variablen MÜSSEN per Ansible-Vault oder externer Secret-Quelle bezogen werden

### Filesystem-Layout
- **MUSS [MUST]** folgendes Layout auf dem Reachy festlegen:

  ```
  /opt/reachy-apps/                              # Wurzel für selbstentwickelte Apps (root:reachy-apps, 2775)
  /opt/reachy-apps/<slug>/                       # Eine App pro Slug
  /opt/reachy-apps/<slug>/source/                # Git-Working-Copy oder rsync-Ziel
  /opt/reachy-apps/<slug>/.venv/                 # Per-App-venv (uv venv)
  /opt/reachy-apps/<slug>/var/log/               # App-Logs (rotiert via journald oder logrotate)
  /opt/reachy-apps/<slug>/var/state/             # Persistenter App-State, kein VCS
  /opt/reachy-apps/<slug>/etc/                   # App-Konfiguration (config.toml, env-files)
  /opt/reachy-apps/<slug>/PROVENANCE.md          # Plugin-/Spec-Verweis
  /etc/systemd/system/reachy-app@.service        # Template-Unit, instanziiert per Slug
  /etc/systemd/system/reachy-app-pull@.service   # Pull-Update-Service (oneshot)
  /etc/systemd/system/reachy-app-pull@.timer     # Timer für Pull-Update
  ```

- **MUSS [MUST]** die Group `reachy-apps` anlegen; der Default-User (`pollen`) MUSS Mitglied sein
- **MUSS [MUST]** Permissions setzen: `/opt/reachy-apps` als `2775` (setgid), Files als `0644`, Directories als `2755`, venv-Inhalte mit uv-Defaults
- **MUSS [MUST]** eine systemd-Template-Unit `reachy-app@.service` definieren, die `<slug>` als Instance-Name nimmt und unter dem `pollen`-User läuft
- **MUSS [MUST]** das Per-App-venv über `uv venv` (nicht `python -m venv`) anlegen — konsistent mit der App-Architektur-Spec
- **DARF NICHT [MUST NOT]** in `/venvs/apps_venv/` (Pollens Shared-venv) schreiben — kollisionsfreie Trennung ist die zentrale Begründung dieses Layouts
- **DARF NICHT [MUST NOT]** App-Quellen oder venvs unter `/home/pollen/` ablegen, weil OS-Reinstalls dort destruktiv sein können

### systemd-Service-Vertrag
- **MUSS [MUST]** eine Template-Unit `reachy-app@.service` definieren, die für `<slug>` als ExecStart `/opt/reachy-apps/<slug>/.venv/bin/python -m <slug_underscore>.main` aufruft
- **MUSS [MUST]** `Restart=on-failure`, `RestartSec=5`, `User=pollen`, `Group=reachy-apps` setzen
- **MUSS [MUST]** `EnvironmentFile=-/opt/reachy-apps/<slug>/etc/env` setzen (`-` macht das File optional)
- **MUSS [MUST]** `WorkingDirectory=/opt/reachy-apps/<slug>/source` setzen
- **MUSS [MUST]** systemd-Hardening anwenden: `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `ReadWritePaths=/opt/reachy-apps/<slug>/var`, `PrivateTmp=true`
- **SOLLTE [SHOULD]** CPU-/Memory-Quotas pro App setzen (z. B. `MemoryMax=512M`), da der RPi 4 CM4 limitierte Ressourcen hat
- **DARF NICHT [MUST NOT]** eine systemd-Unit identisch zu einer Pollen-eigenen Unit (`reachy-mini-daemon.service`) heißen oder ihren `ExecStart` überschreiben

### Push-Pfad: Ansible
- **MUSS [MUST]** eine Ansible-Rolle `reachy-app` im Plugin-Repo unter `ansible/roles/reachy-app/` ausliefern
- **MUSS [MUST]** ein Inventory-Template unter `ansible/inventory.example.yml` mitgeben mit den Variablen `app_slug`, `app_repo_url`, `app_branch`, `app_python_version`, `wifi_ssid` (Vault), `wifi_psk` (Vault), `ssh_pubkey`
- **MUSS [MUST]** die Rolle in folgende Tasks gliedern: `network` (WiFi-Profile), `users` (`reachy-apps`-Gruppe, SSH-Key), `filesystem` (Verzeichnis-Layout, Permissions), `runtime` (uv-Install, venv-Bootstrap), `app` (rsync der Source, `pip install -e .`), `service` (systemd-Unit-Render, `daemon-reload`, `enable`, `start`)
- **MUSS [MUST]** die Rolle idempotent sein — ein zweiter Run ohne Quell-Änderung führt zu null Tasks im `changed`-Status
- **MUSS [MUST]** Geheimnisse (`wifi_psk`, ggf. `git_deploy_key`) per Ansible-Vault verschlüsseln; das Vault-Passwort kommt aus einer externen Quelle (`ANSIBLE_VAULT_PASSWORD_FILE` oder `--vault-password-file`)
- **SOLLTE [SHOULD]** eine `--check`-fähige Trockenlauf-Variante anbieten (`ansible-playbook ... --check --diff`)
- **SOLLTE [SHOULD]** den scp-Fallback (siehe nächste Sektion) in der Rollen-Doku referenzieren — die Rolle selbst nutzt rsync via `ansible.builtin.synchronize`
- **DARF NICHT [MUST NOT]** die Rolle Pollen-eigene Pfade (`/venvs/apps_venv/`, `reachy-mini-daemon.service`) modifizieren

### Push-Pfad: scp-Fallback
- **MUSS [MUST]** einen minimalen scp/rsync-Fallback dokumentieren, der ohne Ansible auskommt — als kopierbares Snippet, nicht als zweite Pipeline
- **MUSS [MUST]** der Fallback dieselben Permissions und denselben Pfad (`/opt/reachy-apps/<slug>/source`) treffen, sodass nachfolgende Ansible-Runs den State erkennen
- **MUSS [MUST]** der Fallback explizit dokumentieren, welche Schritte er **nicht** abdeckt (WiFi-Setup, Gruppen-Setup, Vault-Secrets) — er ist nur App-Sync, nicht voller Bootstrap
- **DARF NICHT [MUST NOT]** der Fallback als „bevorzugter Pfad" beworben werden — Default ist Ansible

### Pull-Pfad: Git auf dem Roboter
- **MUSS [MUST]** der initiale `git clone` als Ansible-Task `app/git_clone.yml` Teil der Push-Rolle sein (auch wenn Folge-Updates rein gerätelokal laufen) — Bootstrap einmalig, dann selbstständig
- **MUSS [MUST]** eine systemd-Template-Unit `reachy-app-pull@.service` als `oneshot` existieren: führt `git fetch && git reset --hard origin/<branch>` (Hard-Reset gegen Branch-HEAD, weil ein Roboter-FS keine lokalen Commits halten soll), `uv pip install -e .` bei Änderungen am `pyproject.toml`, und `systemctl restart reachy-app@<slug>` falls Quellen oder venv geändert wurden
- **MUSS [MUST]** eine Timer-Unit `reachy-app-pull@.timer` mit Default-Intervall `OnUnitActiveSec=5min` und `RandomizedDelaySec=30s` (Jitter, falls mehrere Roboter im selben Netz pullen)
- **MUSS [MUST]** der Pull-Pfad einen Read-Only-Deploy-Key (GitHub oder Gitea) als Auth nutzen, **niemals** einen User-Token mit Schreibrechten
- **MUSS [MUST]** der Deploy-Key per Ansible-Vault eingespielt werden, in `/opt/reachy-apps/<slug>/etc/deploy.key` mit Mode `0400`, Owner `pollen`
- **MUSS [MUST]** der Branch-Pin in `etc/env` als `APP_GIT_BRANCH` ausschließlich auf `main` gesetzt sein; `develop` und Feature-Branches MÜSSEN über den Push-Pfad (Ansible/scp) bedient werden, nicht über den Pull-Timer
- **MUSS [MUST]** der Pull-Service vor `git fetch` einen Disk-Vorcheck durchführen: bei weniger als 1 GiB freiem Speicher unter `/opt/reachy-apps/` `journalctl --vacuum-time=14d` ausführen und für den Folge-Lauf einen Marker setzen, der den venv beim nächsten Pull frisch baut (`rm -rf /opt/reachy-apps/<slug>/.venv` mit anschließendem `uv venv`)
- **MUSS [MUST]** der Pull-Service auf Wireless den Battery-SoC über die Pollen-Daemon-REST-API lesen und den Pull-Lauf bei weniger als 30 % SoC überspringen — Logzeile `pull skipped: battery <n>%`, kein `git fetch`, kein Service-Restart
- **MUSS [MUST]** der Pull-Service vor jedem Pull `GET /api/daemon/robot-app-lock-status` prüfen — hält der Pollen-Daemon eine App im Lock, wird der Pull übersprungen und mit `pull skipped: pollen-app-lock <holder>` in journald geloggt; ein laufender Reachy-App-Service wird **nicht** angefasst
- **MUSS [MUST]** ein Pull-Failure (Netzwerk weg, Auth-Fehler, Merge-Konflikt) den laufenden Service **nicht** abreißen — nur eine fehlerhafte Pull-Run-Logzeile in journald
- **SOLLTE [SHOULD]** Pull-Erfolge als reduzierte Logmenge (`success: <sha-short>`) loggen, nicht den vollen `git pull`-Output
- **KANN [MAY]** für höhere Aktualität ein Webhook-Trigger nachgerüstet werden (separate spätere Spec, hier explizit Nicht-Ziel)
- **DARF NICHT [MUST NOT]** der Pull-Pfad lokale Edits am Roboter zulassen — der Hard-Reset gegen origin ist Vertrag, nicht Bug

### Konfliktfreiheit mit Pollens Shared-venv-Pfad
- **MUSS [MUST]** explizit dokumentieren, dass selbstentwickelte Apps in diesem Layout **nicht** im Reachy-Dashboard erscheinen und **nicht** den Pollen-App-Lock halten — sie laufen parallel zum Pollen-Daemon, nicht in seinem App-Slot
- **MUSS [MUST]** dokumentieren, dass Behaviors in diesen Apps trotzdem über die Pollen-Daemon-REST-API (Port 8000) und über die ReachyMini-SDK-Verbindung mit der Hardware sprechen — sie sind weiterhin SDK-Konsumenten, nur außerhalb des Pollen-App-Frameworks
- **MUSS [MUST]** eine Hinweis-Sektion „Wann diesen Pfad statt HF-Spaces-Install?" enthalten: schnelle Iteration ohne HF-Push, mehrere parallele Apps, fest verdrahtete Konfiguration, Ansible-betriebenes Heim-Setup
- **DARF NICHT [MUST NOT]** dieser Pfad eine simultan laufende Pollen-App ersetzen — wenn der Pollen-Daemon eine App im Lock hat, kollidiert ein parallel laufendes Self-Hosted-Behavior um die Hardware-Verbindung; das muss der Operator wissen

### Logging und Diagnose
- **MUSS [MUST]** App-Logs durch journald gehen (`journalctl -u reachy-app@<slug>`), Konsistenz mit Pollen-Daemon-Logs
- **MUSS [MUST]** der Pull-Service ebenfalls in journald loggen (`journalctl -u reachy-app-pull@<slug>`), separat lesbar
- **MUSS [MUST]** `PROVENANCE.md` im App-Verzeichnis und im Header der systemd-Units einen Plugin-Verweis tragen, sodass jemand auf der Maschine sieht, wo der State herkommt
- **DARF NICHT [MUST NOT]** Logs Tokens, Vault-Werte, WiFi-PSK oder Deploy-Keys enthalten

### Sicherheit
- **MUSS [MUST]** die SSH-Server-Konfiguration (`/etc/ssh/sshd_config.d/10-reachy-apps.conf`) ausschließlich Key-Auth zulassen, kein Root-Login, optional Port-Wechsel
- **MUSS [MUST]** der Default-User `pollen` weder NOPASSWD-sudo noch dauerhaftes sudo brauchen, um eine App zu deployen — die Ansible-Rolle eskaliert kontrolliert
- **MUSS [MUST]** für jeden Reachy ein eigenes SSH-Schlüsselpaar (vom Notebook) verwendet werden, keine geteilten Keys über mehrere Geräte
- **MUSS [MUST]** Vault-Werte (`wifi_psk`, `git_deploy_key`) niemals in Plain-Inventory-Files erscheinen — `ansible-vault encrypt_string` ist Pflicht
- **SOLLTE [SHOULD]** Fail2Ban oder vergleichbar gegen wiederholte SSH-Auth-Fehler aktiviert werden
- **KANN [MAY]** eine separate VLAN/Netz-Segmentierung empfohlen werden — Hinweis, kein MUSS

### Dependency-Lifecycle
- **MUSS [MUST]** die Verantwortung für Dependency-Updates vollständig im jeweiligen App-Repository liegen (Renovate / Dependabot dort, nicht im Plugin); der Roboter zieht passiv den im `pyproject.toml` gepinnten Stand
- **MUSS [MUST]** der Pull-Service ausschließlich `uv pip install -e .` aufrufen und niemals `uv lock --upgrade` oder `pip install -U` — Updates kommen ausschließlich über Repo-Commits, nicht durch Roboter-lokale Re-Resolution
- **DARF NICHT [MUST NOT]** das Plugin-Repo eine Renovate- oder Dependabot-Konfiguration für App-Repos vorgeben; eine Empfehlungs-Vorlage MAY in einer späteren Spec entstehen, aber nicht hier

### Decommission
- **MUSS [MUST]** ein idempotenter Ansible-Playbook `ansible/playbooks/reachy-app-decommission.yml` im Plugin-Repo ausgeliefert werden
- **MUSS [MUST]** der Playbook folgende Schritte ausführen, in dieser Reihenfolge: `systemctl disable --now reachy-app-pull@<slug>.timer`, `systemctl disable --now reachy-app-pull@<slug>.service`, `systemctl disable --now reachy-app@<slug>.service`, Löschen von `/opt/reachy-apps/<slug>/`, Entfernung des SSH-Public-Keys aus `~pollen/.ssh/authorized_keys` (sofern dieser Roboter der einzige Konsument war), Entfernung des Hosts aus dem Inventory-File
- **MUSS [MUST]** der Playbook eine `pause`/`prompt`-Bestätigung vor jedem destruktiven Schritt verlangen — versehentliches Wipe darf nicht durch einen Tippfehler im Inventory ausgelöst werden
- **MUSS [MUST]** der Playbook im `--check --diff`-Modus laufen können und alle geplanten Änderungen ausweisen, ohne sie auszuführen
- **MUSS [MUST]** der Playbook eine Variable `purge_wifi_profile` (Default `false`) tragen — bei `true` wird das `nmconnection`-File entfernt; bei `false` bleibt der Roboter im Heimnetz erreichbar (Standardfall: nur App-Decommission, kein Geräte-Decommission)
- **DARF NICHT [MUST NOT]** der Playbook Pollen-eigene Pfade (`/venvs/apps_venv/`, `reachy-mini-daemon.service`, `/etc/systemd/system/reachy-mini-daemon.service`) anfassen — er räumt ausschließlich Plugin-State ab

## Akzeptanzkriterien
- [ ] Ansible-Rolle `reachy-app` existiert unter `ansible/roles/reachy-app/` mit den sechs Task-Files (`network`, `users`, `filesystem`, `runtime`, `app`, `service`)
- [ ] Inventory-Template `ansible/inventory.example.yml` ist im Repo, der `ansible-vault`-Marker ist dokumentiert
- [ ] systemd-Templates `reachy-app@.service`, `reachy-app-pull@.service`, `reachy-app-pull@.timer` sind als Jinja2-Templates in der Rolle hinterlegt
- [ ] Ein zweiter Ansible-Run ohne Quell-Änderung produziert null `changed`-Tasks
- [ ] WLAN-Profil ist nach Setup als `nmconnection`-File mit Mode `0600` hinterlegt
- [ ] Password-Auth ist nach Bootstrap im sshd deaktiviert
- [ ] Das App-Filesystem unter `/opt/reachy-apps/<slug>/` existiert mit den vorgesehenen Subdirs und Permissions
- [ ] `journalctl -u reachy-app@<slug>` zeigt App-Logs; `journalctl -u reachy-app-pull@<slug>` zeigt Pull-Logs
- [ ] Pull-Timer läuft alle 5 min ± 30 s und führt einen Hard-Reset gegen origin-Branch durch
- [ ] Ein Pull-Failure (z. B. Netz weg) reißt den App-Service **nicht** ab
- [ ] `PROVENANCE.md` im App-Verzeichnis verlinkt auf das Plugin
- [ ] Es gibt eine schriftliche Abgrenzung zur Pollen-Shared-venv-Route, die klärt, wann welcher Pfad zu wählen ist
- [ ] WiFi-PSK und Deploy-Key sind im Repo nur als Vault-Strings vorhanden
- [ ] Der scp-Fallback ist als ergänzendes Snippet dokumentiert, nicht als zweite Pipeline
- [ ] mDNS-Adresse `reachy-mini-<short-id>.local` wird vom Notebook aufgelöst
- [ ] systemd-Hardening-Optionen (`NoNewPrivileges`, `ProtectSystem`, `ProtectHome`, `ReadWritePaths`, `PrivateTmp`) sind in der Template-Unit gesetzt
- [ ] Die Group `reachy-apps` existiert, `pollen` ist Mitglied, `/opt/reachy-apps` ist `2775`
- [ ] `APP_GIT_BRANCH` ist auf jedem Roboter `main`; develop-Stand erreicht den Roboter nur über den Push-Pfad
- [ ] Pull-Service skipt bei < 30 % Battery-SoC auf Wireless mit Logzeile `pull skipped: battery <n>%` in journald
- [ ] Pull-Service skipt bei gehaltenem Pollen-App-Lock mit Logzeile `pull skipped: pollen-app-lock <holder>` in journald
- [ ] Pull-Service vakuumiert journald-Logs bei < 1 GiB freiem Speicher unter `/opt/reachy-apps/` und baut den venv beim Folge-Lauf neu
- [ ] Pull-Service ruft niemals `uv lock --upgrade` oder `pip install -U` auf
- [ ] `ansible/playbooks/reachy-app-decommission.yml` existiert, läuft idempotent, hat einen Bestätigungs-Schritt vor destruktiven Aktionen, läuft im `--check --diff`-Modus
- [ ] Decommission-Playbook fasst keine Pollen-eigenen Pfade an
- [ ] Decommission-Playbook lässt das `nmconnection`-File standardmäßig stehen (`purge_wifi_profile=false`)

## Quellen
- App-Architektur (zu der diese Spec sich abgrenzt): [`spec/reachy-mini/app-architecture/de.md`](../app-architecture/de.md)
- On-Device-Test-Agent (verwandter Lifecycle-Kontext): [`spec/claude/reachy-mini-on-device/de.md`](../../claude/reachy-mini-on-device/de.md)
- NetworkManager `nmconnection`-Format: <https://networkmanager.dev/docs/api/latest/ref-settings.html>
- systemd Template-Units (Section „TEMPLATES"): <https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html>
- systemd-Hardening Spickzettel: <https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html>
- Ansible-Vault: <https://docs.ansible.com/ansible/latest/vault_guide/>
- Pollens Reachy-Mini-Apps-Doku (Shared-venv-Pfad, gegen den abgegrenzt wird): <https://github.com/pollen-robotics/reachy_mini/blob/main/docs/source/SDK/apps.md>
- uv (für Per-App-venv): <https://docs.astral.sh/uv/>

## Offene Fragen
- ~~Welche Branches sind gültige Pull-Targets?~~ **Beantwortet:** ausschließlich `main`. `develop` und Feature-Branches gehen über den Push-Pfad — siehe Sektion „Pull-Pfad: Git auf dem Roboter".
- ~~Soll die Ansible-Rolle Renovate/Dependabot-Hinweise für `pyproject.toml`-Updates auf dem Roboter respektieren?~~ **Beantwortet:** nein, reines Repo-Thema. Der Pull-Service installiert nur den im `pyproject.toml` gepinnten Stand und ruft niemals `uv lock --upgrade` auf — siehe Sektion „Dependency-Lifecycle".
- ~~Wie wird ein Roboter aus dem Inventar entfernt?~~ **Beantwortet:** über den idempotenten Playbook `ansible/playbooks/reachy-app-decommission.yml` mit Bestätigungs-Schritten vor destruktiven Aktionen — siehe Sektion „Decommission".
- ~~Welche Mindest-Disk-Größe ist auf einem Wireless realistisch, bevor wir Auto-Cleanup brauchen?~~ **Beantwortet:** Pull-Service vakuumiert journald-Logs und rebaut den venv, sobald weniger als 1 GiB unter `/opt/reachy-apps/` frei ist — siehe Sektion „Pull-Pfad: Git auf dem Roboter".
- ~~Soll der Pull-Timer auf den Battery-Stand reagieren?~~ **Beantwortet:** ja, auf Wireless. Pull-Skip bei < 30 % SoC; zusätzlich Skip, wenn der Pollen-Daemon eine App im Lock hält — siehe Sektion „Pull-Pfad: Git auf dem Roboter".

*(Aktuell keine offenen Fragen. Neue Fragen werden hier ergänzt, sobald sie auftauchen.)*
