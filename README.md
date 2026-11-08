# GitLab Administration: gitlab-public-demo-server

Anleitung und Protokoll für die Upgrades des öffentlichen GitLab-Demo-Servers
`gitlab-pub.comquent.academy` (Hetzner, Ubuntu 22.04, GitLab CE als Linux-Paket).

## Aktueller Stand (08.11.2026)

| | |
|---|---|
| GitLab | 19.5.3 CE (Paket auf `hold`) |
| PostgreSQL | 17.11 (gebündelt) |
| Server | 4 vCPU, 7,6 GB RAM, 75 GB Platte |
| Nächste Pflichtstopps | 19.8, 19.11 |

Das Upgrade von 16.11.10 auf 19.5.3 lief am 08.11.2026 in zehn Schritten ohne Fehler.

## Dateien

- [`CLAUDE.md`](CLAUDE.md): Arbeitsanweisung für Upgrades. Sie enthält den Upgrade-Pfad, den Ablauf je Schritt
  (Vorprüfung, Backup, Installation, Nachprüfung), die Regeln und die Anlässe zum Anhalten. Claude Code liest
  sie automatisch. Sie taugt aber auch als Checkliste für manuelle Upgrades.
- [`update.log`](update.log): Protokoll aller Upgrade-Schritte mit Zeiten, Backups, Warnungen und Prüfergebnissen.
  Am Ende stehen die offenen Empfehlungen.

## Voraussetzungen

- SSH-Zugang als `root` mit Schlüssel. Der Host-Alias `gitlab-demo` muss in `~/.ssh/config` stehen:
  ```
  Host gitlab-demo
      HostName gitlab-pub.comquent.academy
      User root
      IdentityFile ~/.ssh/id_ed25519
      IdentitiesOnly yes
  ```
- Der eigene öffentliche Schlüssel muss in `/root/.ssh/authorized_keys` auf dem Server eingetragen sein.

## Nächstes Upgrade durchführen

1. In diesem Verzeichnis Claude Code starten und `führe CLAUDE.md aus` eingeben, oder die Schritte aus
   `CLAUDE.md` von Hand abarbeiten.
2. Vorher `CLAUDE.md` prüfen: Stimmen Ausgangslage und Upgrade-Pfad noch?
3. Danach `update.log` und die Ausgangslage in `CLAUDE.md` aktualisieren und committen.

## Offene Punkte

Die Empfehlungen aus dem letzten Upgrade stehen am Ende von [`update.log`](update.log), unter anderem:

- Runner prüfen und auf 19.x aktualisieren oder löschen
- Selbstregistrierung abschalten (Spam-Konten)
- `initial_root_password` aus `gitlab.rb` entfernen
- Backups auf ein externes Ziel kopieren
- Alte PostgreSQL-Datenverzeichnisse und ältere Backups aufräumen

## Hinweis

Das Repository enthält keine Zugangsdaten und darf auch keine enthalten. Passwörter, Tokens, `gitlab-secrets.json`
und Backups gehören nicht hierher.
