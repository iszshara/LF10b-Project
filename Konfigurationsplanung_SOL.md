# Projektbeschreibung

Erstellen einer Backup und Wiederherstellungslösung für ein Debian basiertes System, welches einen Minecraft Server hostet.
Die Backuplösung umfängt das System selbst und die Minecraft welt und dazugehörige Daten des Servers und des Spiels.
Das Monitoring soll überblick über das System und das Spiel schaffen zudem per Browser erreichbar sein.
Als Wiederherstellung soll eine 2VM als PXE Server und Dateiserver (für Backups) dienen

# Persönliche Ziele
- Ansible lernen
- Ein Projekt fertigstellen
- Linux besser kennenlernen
- Serverhärtung lernen
- Automatisierungslösung bereitstellen

# Dienste des Servers und Software
## Game Server
| Dienst | Software |
| --- | --- |
| Backup | BorgBackup (system level), Pterodactyl (Minecraft Server)|
| Monitoring | Pterodactyl, Prometeus, Graphana |
| Datenbank | Maria DB |
| Serverabsicherung | fail2ban |
| Webserver | (Nginx)|
| Minecraft Server | Minecraft Server |

## Backup & PXE Server
| Dienst |
| --- |
| DHCP |
| TFTP |

# Architektur
## Plattform
Für das Projekt sollen Virtuelle Maschinen in VMware Fusion / VMware Workstation genutzt werden.
## Betriebssysteme
- Ubuntu Server
## Virtualisierung
Pterodactyl stellt die Spieleserver per Docker bereit. Isolation, Portfreigaben und Ressourcenlimits werden durch Docker geregelt. Das Dashboard läuft nativ auf der VM.

# Geplante RTO zeit
2 Stunden

# Überwachte Systemeigenschaften
## Pterodactyl
Überwachung des jeweiligen Nodes auf dem der Server läuft

- Server-Status (Online/Offline; Uptime)
- Ressourcenverbrauch des Nodes (CPU-Last; RAM-Verbrauch; Festplattennutzung; Netzwerktraffic)
- Konsolen-Output (echtzeitausgabe des Logs; direkte Befehleingabe /stop /give etc.)
- Server-Dateien (Dateimanager direkt im Panel)

## Prometeus + Grafana
- CPU Nutzung
- Ram Nutzung (Total, used, free, cached)
- Speichernutzung (Speichernutzung, I/O speed)
- Netzwerk (Traffic, offene Verbindungen)
- Uptime
- laufende Prozesse

## Aplication Level Monitoring
Implementierung der Pterodactyl Panel kann mit `pterodactyl-exporter` können die oben genannten

# Automatisierungspläne
Die Wiederherstellung des Servers bspw. durch Ausfall aller Festplatten oder genereller hardware schaden soll mit Ansible Automatisiert werden. Der "frische" und Funktionsfähige Server soll per PXE die Installation von Ubuntu Server starten und ein Custom Image nutzen um Ansible mit zu installieren. Anschließend soll automatisch ein Playbook abgespielt werden, welches den server wieder einrichtet.

# Grober Zeitplan
| Datum | Aufgabe
| --- | --- 
|25.09. | Server Aufsetzen
|29.09. | Monitoring
|30.09. | Backup
|17.11. | Automatisierung
|18.11. | Testen & Optimieren

# Aufgabenverteilung
| Aufgabe | Teammitglied |
| --- | ---
| Ansible Playbook | Luis
| Minecraft & Pterodactyl Server aufgbauen | Robert
| Backup & PXE Server aufbauen | Lugas
| Dokumentation | Luis; Lucas; Robert

# Zu befolgende Tutorials

### Pterodactyl & Nginx
*   **Pterodactyl Installation (Official Docs):** https://pterodactyl.io/panel/1.0/getting_started.html
*   **Pterodactyl Installer Script:** https://pterodactyl-installer.se
*   **Nginx Setup (Official Docs):** https://pterodactyl.io/panel/1.0/webserver_setup.html

### Monitoring
*   **Install Prometheus & Grafana on Ubuntu:** https://www.linode.com/docs/guides/install-prometheus-and-grafana-on-ubuntu/
*   **Pterodactyl Dashboard** https://grafana.com/grafana/dashboards/16575-pterodactyl-server/
*   **Pterodactyl Exporter** https://github.com/LOENS2/pterodactyl_exporter
*   **Minecraft Exporter Standalone (Docker):** https://github.com/dirien/minecraft-prometheus-exporter

### Backups
*   **Borg Backup (German Tutorial)** https://de.linux-terminal.com/?p=4514

### Automation & Security
*   **Ansible Getting Started (Official Docs):** https://docs.ansible.com/ansible/latest/user_guide/index.html
*   **Ansible Beginner's Tutorial:** https://learnansible.dev/
*   **Ubuntu Server Hardening Guide:** https://www.linux-audit.com/ubuntu-server-hardening-guide-a-comprehensive-checklist/
*   **CIS Benchmarks for Ubuntu:** https://www.cisecurity.org/benchmark/ubuntu_linux