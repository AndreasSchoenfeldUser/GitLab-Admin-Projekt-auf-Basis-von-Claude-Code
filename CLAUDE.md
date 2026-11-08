# GitLab-Upgrade: gitlab-public-demo-server

## Ziel

Die GitLab-Instanz auf dem Server `gitlab-public-demo-server` bei neuen Releases schrittweise auf die aktuellste
Version bringen, unter Einhaltung aller Pflicht-Zwischenstopps (Required Upgrade Stops).

## Zugang

- Der Server ist per SSH als Host-Alias `gitlab-demo` erreichbar (`~/.ssh/config`: HostName `gitlab-pub.comquent.academy`,
  Benutzer `root`, Schlüssel `~/.ssh/id_ed25519`).
- Alle Befehle auf dem Server werden über `ssh gitlab-demo '<befehl>'` ausgeführt. Der Benutzer ist `root`, `sudo` ist nicht nötig.
- Zugangsdaten (Passwörter, Schlüssel) niemals in Dateien, Logs oder Ausgaben schreiben.

## Ausgangslage (Stand 08.11.2026, nach dem Upgrade 16.11.10 → 19.5.3)

- Betriebssystem: Ubuntu 22.04 (jammy), gehostet bei Hetzner, 4 vCPU, 7,6 GB RAM, kein Swap, 75 GB Platte
- Edition: **GitLab Community Edition** (Paket `gitlab-ce`, Linux-Paket/Omnibus, kein Docker)
- Installierte Version: **19.5.3-ce.0**, Paket steht auf `hold`
- PostgreSQL **17.11** (gebündelt), Redis 7.2.16
- Bisheriger Weg und alle Details: siehe `update.log` in diesem Verzeichnis
- Das APT-Repository `/etc/apt/sources.list.d/gitlab_gitlab-ce.list` nutzt `signed-by=/usr/share/keyrings/gitlab_gitlab-ce-archive-keyring.gpg`.
  Falls `apt update` erneut `EXPKEYSIG` meldet:
  `curl -fsSL https://packages.gitlab.com/gitlab/gitlab-ce/gpgkey | gpg --dearmor > /usr/share/keyrings/gitlab_gitlab-ce-archive-keyring.gpg`
- `gitlab.rb` enthält eine bewusste Anpassung: `alertmanager['flags'] = { 'cluster.advertise-address' => '127.0.0.1:9094' }`
  (behebt einen Crash-Loop, weil der Server keine private IP hat). Nicht entfernen.
- Es handelt sich um einen öffentlich erreichbaren Demo-Server.

## Upgrade-Pfad

Pflichtstopps liegen seit 17.5 bei `x.2`, `x.5`, `x.8` und `x.11`. Jeweils den **neuesten Patch-Release** der
Minor-Version verwenden. Die konkreten Patch-Versionen vor jedem Schritt nach `apt-get update` mit
`apt-cache madison gitlab-ce` ermitteln.

| Schritt | Zielversion | Hinweis |
|---|---|---|
| 1 | 19.8.x | |
| 2 | 19.11.x | Vor 20.0: Upgrade Notes für GitLab 20 prüfen (PostgreSQL-, OS- und Konfig-Anforderungen) |
| 3 | weitere Stopps laut Dokumentation | |

Vor dem Start den Pfad gegen die offizielle Dokumentation prüfen. Bei Abweichungen gilt die Dokumentation:

- https://docs.gitlab.com/update/upgrade_paths/
- https://docs.gitlab.com/update/versions/gitlab_19_changes/
- https://docs.gitlab.com/update/background_migrations/

## Ablauf je Upgrade-Schritt

1. **Vorprüfung**
   - `gitlab-ctl status` — alle Dienste müssen `run:` zeigen
   - `df -h / /var/opt/gitlab /var/opt/gitlab/backups` — mindestens 10 GB frei, sonst stoppen und melden
   - `free -h` — bei weniger als 1 GB verfügbarem RAM nicht starten, sondern melden
   - `dpkg --audit` muss leer sein; kein laufendes `apt`/`dpkg`/`unattended-upgrade`, das die dpkg-Sperre hält
   - Background Migrations müssen abgeschlossen sein (alle Werte `0`):
     ```bash
     gitlab-rails runner -e production 'puts Gitlab::Database::BackgroundMigration::BatchedMigration.queued.count'
     gitlab-rails runner -e production 'puts Gitlab::Database::BackgroundMigration::BatchedMigration.with_status(:failed).count'
     gitlab-psql -At -c "SELECT count(*) FROM batched_background_migrations WHERE status NOT IN (3,6);"
     ```
     Übersicht: `gitlab-rake gitlab:background_migrations:list`. Ab 19.5 wartet
     `gitlab-rake gitlab:background_migrations:wait` blockierend, bis alle fertig sind (ohne eigenes Timeout).
     `Gitlab::BackgroundMigration.remaining` gibt es ab GitLab 18.5 nicht mehr.
     Sind die Werte nicht `0`: im Abstand von 5 Minuten erneut prüfen. Nach 60 Minuten ohne Fortschritt stoppen und melden.
   - Bei fehlgeschlagenen Batched Migrations (Status `failed`) nicht fortfahren, sondern melden.
   - Upgrade Notes der Zielversion lesen und relevante Punkte im Log festhalten.

2. **Backup** (vor jedem Major-Sprung und vor jedem PostgreSQL-Upgrade zwingend, sonst empfohlen)
   ```bash
   gitlab-backup create
   cp -p /etc/gitlab/gitlab.rb /root/gitlab-config-backup/gitlab.rb.<VERSION>-<ZEITSTEMPEL>
   cp -p /etc/gitlab/gitlab-secrets.json /root/gitlab-config-backup/gitlab-secrets.json.<VERSION>-<ZEITSTEMPEL>
   ```
   (Dateien mit Versionssuffix ablegen, nichts überschreiben.)

3. **Installation**
   - Lang laufende Befehle in einer `tmux`-Session auf dem Server starten, damit ein SSH-Abbruch oder Tool-Timeout
     den Vorgang nicht unterbricht. Backup und Installation verketten, damit die Installation nur nach
     erfolgreichem Backup startet:
     ```bash
     ssh gitlab-demo "apt-mark unhold gitlab-ce; tmux new -d -s upgrade 'gitlab-backup create > /root/backup-<ALT>.log 2>&1 && echo BACKUP_OK >> /root/backup-<ALT>.log && DEBIAN_FRONTEND=noninteractive apt-get install -y gitlab-ce=<VERSION> > /root/upgrade-<VERSION>.log 2>&1; echo EXIT=\$? >> /root/upgrade-<VERSION>.log'"
     ```
   - Fortschritt über `tail -n 50 /root/upgrade-<VERSION>.log` verfolgen, bis `EXIT=` erscheint.
   - Danach `apt-mark hold gitlab-ce`, damit kein versehentliches `apt upgrade` Versionen überspringt.

4. **Nachprüfung**
   - `cat /opt/gitlab/embedded/service/gitlab-rails/VERSION` — Zielversion bestätigt
   - `gitlab-ctl status` — alle Dienste `run:`
   - `curl -s http://localhost/-/readiness` — direkt nach der Installation kann kurz `502` kommen, bis Puma gestartet ist
   - `gitlab-rake gitlab:check SANITIZE=true` — Fehler im Log dokumentieren
   - Den Block `Warnings:` im Install-Log lesen. Meldet er `Please restart redis`, nach Abschluss der
     Migrationen `gitlab-ctl restart redis` ausführen und die Redis-Version prüfen (`gitlab-redis-cli INFO server`).
   - Background Migrations wie in Schritt 1 abwarten, bevor der nächste Schritt beginnt
   - Die Warnung `git_data_dirs has been deprecated` ist hier ein Fehlalarm (nicht explizit gesetzt), keine Aktion nötig.

5. **Protokoll** in `update.log` (lokal, in diesem Verzeichnis) fortschreiben: Datum/Uhrzeit, Version vorher/nachher,
   Dauer, Auffälligkeiten, Warnungen aus `gitlab:check`.

## PostgreSQL

- Version prüfen: `gitlab-psql -c "SELECT version();"`
- Das Linux-Paket aktualisiert PostgreSQL auf diesem Single-Node-Server bei Bedarf **automatisch** während der
  Paketinstallation (bisher bei 17.11 auf 16 und bei 18.11 auf 17). Deshalb muss vor solchen Schritten ein erfolgreiches
  Backup vorliegen. Im Install-Log auf `Upgrading PostgreSQL to` und `Upgrade has completed` achten.
- Verlangen die Upgrade Notes eine neuere PostgreSQL-Version und erledigt das Paket es nicht selbst: nur nach erfolgreichem
  Backup und auf der zuletzt installierten Version **vor** dem Sprung `gitlab-ctl pg-upgrade -V <version>` ausführen.
- Die vom Upgrade zurückgelassenen alten Datenbankverzeichnisse (`/var/opt/gitlab/postgresql/data.<alt>`) **nicht** löschen,
  auch wenn das Install-Log `rm -rf` vorschlägt. Den Pfad im Log notieren.

## Verbindliche Regeln

- Immer nur **einen** Schritt des Upgrade-Pfads auf einmal. Niemals Pflichtstopps überspringen.
- Niemals `apt upgrade`, `apt full-upgrade` oder `apt install gitlab-ce` ohne festgelegte Version ausführen.
- Niemals `/etc/gitlab/skip-auto-backup` anlegen. Schlägt das automatische Pre-Install-Backup fehl, Ursache ermitteln
  (meist PostgreSQL nicht gestartet: `gitlab-ctl start`, `gitlab-ctl tail postgresql`) und melden.
- Keine Backups, Logs oder Datenbankverzeichnisse löschen.
- Kein Upgrade des Betriebssystems (`do-release-upgrade`) und keine Änderungen an Firewall, SSH-Konfiguration oder Benutzern.
- Änderungen an `/etc/gitlab/gitlab.rb` nur, wenn die Upgrade Notes es ausdrücklich verlangen oder der Betreiber es
  freigibt. Vorher Kopie anlegen, Änderung im Log begründen, danach `gitlab-ctl reconfigure`.
- Ist für eine benötigte Version kein Paket für `jammy` verfügbar, stoppen und melden.

## Wann anhalten und nachfragen

Sofort anhalten und den Sachstand berichten bei:

- Fehlern bei `apt`, `dpkg`, `gitlab-ctl reconfigure` oder `pg-upgrade`
- fehlgeschlagenen oder hängenden Background Migrations
- Diensten, die nach dem Upgrade nicht in den Status `run:` kommen
- Breaking Changes in den Upgrade Notes, die Konfigurationsänderungen erfordern
- zu wenig Speicherplatz oder Arbeitsspeicher (OOM-Kills in `dmesg`, sehr hohe Last)
- jeder Unklarheit über den nächsten Schritt

## Abschluss

Nach dem letzten Schritt in `update.log` zusammenfassen: Endversion, PostgreSQL-Version, Ergebnis von
`gitlab:check`, offene Warnungen sowie Empfehlungen (z. B. Runner-Updates, Aufräumen alter PostgreSQL-Daten und Backups),
die manuell erledigt werden sollten. Offene Empfehlungen aus dem letzten Upgrade stehen am Ende von `update.log`.
