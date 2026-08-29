# Windows Server Lab

Das ist die Dokumentation zu meinem Windows-Server-Labor. Ich lerne im Bereich Fachinformatiker für Systemintegration und habe mir dafür eine eigene Testumgebung auf Proxmox VE aufgebaut.

Der Grund ist einfach: Theorie kann man auswendig lernen, aber vieles versteht man erst, wenn man es einmal selbst kaputt gemacht hat. Deshalb geht es in fast jedem Kapitel hier darum, etwas einzurichten und danach absichtlich einen Fehler zu erzeugen, um zu sehen, wie das System reagiert.

Ich schreibe hier auf, **warum** ich etwas so gemacht habe, nicht jeden einzelnen Klick. Die Screenshots zeigen den Rest.

> Hinweis: Alle IP-Adressen, Hostnamen und Benutzernamen in dieser Dokumentation stammen aus einer isolierten Testumgebung und wurden für die Veröffentlichung angepasst.

## Umgebung

Alle Server laufen als virtuelle Maschinen auf Proxmox VE in einem eigenen, isolierten Netz ohne Verbindung zu meinem privaten Heimnetz. Jede Maschine hat VirtIO-Treiber und den QEMU Guest Agent, damit Snapshots sauber funktionieren.

| Hostname | Betriebssystem | Installationsart | IP |
| :--- | :--- | :--- | :--- |
| `SRV25-CORE` | Windows Server 2025 Standard | Server Core | `172.16.10.20` |
| `SRV25-GUI` | Windows Server 2025 Standard | Desktopdarstellung | `172.16.10.21` |
| `SRV25-DC` | Windows Server 2025 Datacenter | Desktopdarstellung | `172.16.10.22` |
| `SRV22-CORE` | Windows Server 2022 Standard | Server Core | `172.16.10.30` |
| `SRV22-GUI` | Windows Server 2022 Standard | Desktopdarstellung | `172.16.10.31` |
| `SRV22-DC` | Windows Server 2022 Datacenter | Desktopdarstellung | `172.16.10.32` |

Zwei Server ohne grafische Oberfläche sind Absicht. Server Core zwingt einen dazu, mit PowerShell zu arbeiten, und genau das wollte ich üben.

Von jeder Maschine gibt es einen Snapshot im Ausgangszustand. Nach den Ausfalltests weiter unten war das mehr als einmal nützlich.

## Themen

1. [Lokale Benutzer und Absicherung des Administrator-Kontos](docs/01-benutzerverwaltung.md)
   Eigener Administrator statt des eingebauten Kontos, GUI und Server Core im Vergleich, und warum man Konten deaktiviert statt sie zu löschen.

2. [Datenträger, Volumes und Software-RAID](docs/02-datentraeger-und-raid.md)
   Basis- und dynamische Datenträger, übergreifende und gespiegelte Volumes, RAID 0 und RAID 5 mit Ausfalltest und Rebuild.

3. [Speicherpools und Speicherplätze](docs/03-storage-spaces.md)
   Storage Spaces mit zwölf Platten, Hot-Spare, die verschiedenen Absicherungsarten und ein Ausfalltest, bei dem ich Platten im laufenden Betrieb gezogen habe.

4. [Windows Admin Center](docs/04-windows-admin-center.md)
   Server ohne grafische Oberfläche über den Browser verwalten, WinRM und TrustedHosts.

5. [NIC-Teaming, Failover und Netzwerkbrücke](docs/05-nic-teaming.md)
   Ausfallsicherheit im Netzwerk, zwei Teams parallel betreiben, Strong Host gegen Weak Host und ein Routingfehler, der lange unbemerkt geblieben ist.

6. [iSCSI](docs/06-iscsi-san.md)
   Speicher auf Blockebene über das Netzwerk, Zielserver einrichten und die Platte von einem Core-Server und einem normalen Windows-Client aus einbinden.

## In Arbeit

Diese Themen stehen bei mir als Nächstes an und kommen dazu, sobald ich sie fertig habe:

- IIS als Webserver mit einer eigenen Website und einem anderen Port
- DHCP-Rolle über das Windows Admin Center verteilen
- Active Directory: Gesamtstruktur anlegen, Domänencontroller hochstufen und DNS integrieren
- Server der Domäne beitreten lassen, mit und ohne grafische Oberfläche

## Werkzeuge

Proxmox VE als Hypervisor, Windows Server 2022 und 2025 in der Evaluierungsversion, Windows Admin Center, PowerShell und die üblichen Bordmittel wie `diskmgmt.msc`, `ncpa.cpl` und die Serververwaltung.
