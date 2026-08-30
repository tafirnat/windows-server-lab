# Windows Server Lab

Das ist die Dokumentation zu meinem Windows-Server-Labor auf Proxmox VE. Ich lerne im Bereich Fachinformatiker für Systemintegration und baue parallel dazu eine eigene Testumgebung auf, um die Themen aus dem Unterricht wirklich anzufassen.

Theorie kann man auswendig lernen. Was ein RAID 5 im Ausfall tatsächlich macht, versteht man aber erst, wenn man eine Platte abzieht und zusieht. Deshalb endet hier fast jedes Kapitel damit, dass ich etwas absichtlich kaputt mache und aufschreibe, was passiert ist. Auch die Stellen, an denen ich falsch lag, stehen drin.

> Hinweis: Alle IP-Adressen, Hostnamen und Benutzernamen in dieser Dokumentation stammen aus einer isolierten Testumgebung und wurden für die Veröffentlichung angepasst.

## Umgebung

Alle Server sind virtuelle Maschinen auf einem Proxmox-VE-Host und laufen in einem eigenen, isolierten Netz ohne Verbindung zum privaten Heimnetz. Jede Maschine hat VirtIO-Treiber und den QEMU Guest Agent.

| Hostname | Betriebssystem | Installationsart | IP |
| :--- | :--- | :--- | :--- |
| `SRV25-CORE` | Windows Server 2025 Standard | Server Core | `172.16.10.20` |
| `SRV25-GUI` | Windows Server 2025 Standard | Desktopdarstellung | `172.16.10.21` |
| `SRV25-DC` | Windows Server 2025 Datacenter | Desktopdarstellung | `172.16.10.22` |
| `SRV22-CORE` | Windows Server 2022 Standard | Server Core | `172.16.10.30` |
| `SRV22-GUI` | Windows Server 2022 Standard | Desktopdarstellung | `172.16.10.31` |
| `SRV22-DC` | Windows Server 2022 Datacenter | Desktopdarstellung | `172.16.10.32` |

Zwei Server ohne grafische Oberfläche sind Absicht. Server Core zwingt zum Arbeiten mit PowerShell, und genau das wollte ich üben. Zwei Windows-Versionen nebeneinander, weil in Firmen selten alles auf demselben Stand ist.

Von jeder Maschine gibt es einen Snapshot im sauberen Ausgangszustand. Nach den Ausfalltests weiter unten war das mehrfach nützlich.

## Kapitel

**1. [Lokale Benutzer und Absicherung des Administrator-Kontos](docs/01-benutzerverwaltung.md)**
Eigenen Administrator anlegen, eingebautes Konto deaktivieren. Warum die SID wichtiger ist als der Name und warum man Konten nie löscht. Derselbe Vorgang einmal grafisch und einmal auf Server Core.

**2. [Datenträgerverwaltung: von der virtuellen Platte zum Laufwerk](docs/02-datentraegerverwaltung.md)**
Die zwei Ebenen im Hypervisor und im Gastsystem. MBR gegen GPT, Schnellformatierung, Verkleinern und Erweitern, und die PowerShell-Pipeline, die alles in vier Zeilen erledigt.

**3. [Volume-Typen und Software-RAID](docs/03-raid.md)**
Übergreifende, gestreifte, gespiegelte und RAID-5-Volumes. Jede Variante gebaut, danach eine Platte offline geschaltet und das Ergebnis festgehalten. Dazu ein Versuch zum Erweitern nach links und nach rechts, der mich überrascht hat. 22 Screenshots.

**4. [Speicherpools und Speicherplätze](docs/04-storage-spaces.md)**
Zwölf Platten in einem Pool, Reserveplatten, vier Absicherungsarten nebeneinander und dünne Bereitstellung. Danach der Ausfalltest: Platten im laufenden Betrieb ziehen, bis der ganze Pool zusammenbricht, und wieder zurückstecken. 44 Screenshots.

**5. [Windows Admin Center](docs/05-windows-admin-center.md)**
Server ohne Oberfläche über den Browser verwalten. WinRM, TrustedHosts und die Anmeldung in einer Arbeitsgruppe.

**6. [NIC-Teaming, Failover und Netzwerkbrücke](docs/06-nic-teaming.md)**
Zehn Netzwerkkarten, zwei Teams mit unterschiedlichen Modellen und eine Brücke. Strong Host gegen Weak Host, ein Routingfehler, der lange unbemerkt blieb, und der Vergleich mit Cisco EtherChannel. 21 Screenshots.

**7. [iSCSI](docs/07-iscsi.md)**
Speicher auf Blockebene über das Netzwerk. Zielserver einrichten, die LUN auf dem Volume aus Kapitel 4 ablegen und von einem Core-Server und einem normalen Windows-Rechner einbinden. 12 Screenshots.

## In Arbeit

Diese Themen stehen als Nächstes an:

- IIS als Webserver mit eigener Website und abweichendem Port
- DHCP-Rolle über das Windows Admin Center verteilen
- Active Directory: Gesamtstruktur anlegen, Domänencontroller hochstufen, DNS integrieren
- Server der Domäne beitreten lassen, mit und ohne grafische Oberfläche

## Werkzeuge

Proxmox VE als Hypervisor, Windows Server 2022 und 2025 in der Evaluierungsversion, Windows Admin Center, PowerShell und die üblichen Bordmittel: `diskmgmt.msc`, `ncpa.cpl`, `lusrmgr.msc`, `iscsicpl.exe` und die Serververwaltung.
