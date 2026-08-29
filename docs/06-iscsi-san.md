# iSCSI: Speicher über das Netzwerk bereitstellen

## Warum ich das gemacht habe

Eine SMB-Freigabe kannte ich schon. Der Server verwaltet dort das Dateisystem und der Client bekommt Zugriff auf Ordner und Dateien.

Bei iSCSI ist es anders. Der Server gibt keine Dateien heraus, sondern rohe Blöcke. Auf dem Client erscheint das Ergebnis als ganz normale Festplatte, die man selbst initialisieren und formatieren muss. Der Client sieht nicht, dass die Platte in Wirklichkeit über das Netzwerk läuft.

Das wollte ich einmal selbst aufbauen, weil dieser Unterschied in der Praxis wichtig ist. Datenbanken, Exchange und Failover-Cluster brauchen Zugriff auf Blockebene und funktionieren auf einer normalen SMB-Freigabe nicht oder nur eingeschränkt.

| | SMB-Freigabe | iSCSI |
| :--- | :--- | :--- |
| Ebene | Datei | Block |
| Port | 445 | 3260 |
| Dateisystem verwaltet | der Server | der Client |
| Der Client sieht | einen Netzwerkordner | eine Festplatte |
| Anmeldung über | Benutzer und Kennwort | IQN, IP-Filter oder CHAP |

Der Speicherbereich, den der Server herausgibt, heisst LUN. Technisch ist das bei Windows einfach eine VHDX-Datei.

## Serverseite

Die Rolle heisst "iSCSI-Zielserver" und liegt unter den Datei- und Speicherdiensten.

![Installation der Rolle iSCSI-Zielserver](../images/iscsi-01-rolle-zielserver.png)

```powershell
Install-WindowsFeature -Name FS-iSCSITarget-Server -IncludeManagementTools
```

Als Ablageort habe ich das Volume genommen, das ich im vorherigen Kapitel mit Storage Spaces und doppelter Parität gebaut hatte. So liegt die Netzwerkplatte auf einem Speicher, der zwei Plattenausfälle verträgt.

![Auswahl des Speicherorts für die virtuelle iSCSI-Festplatte](../images/iscsi-02-speicherort.png)

Bei der Grösse habe ich 15 GB und "dynamisch erweiterbar" gewählt. Die VHDX-Datei belegt am Anfang nur wenige Megabyte und wächst mit den Daten.

![Grösse und dynamische Zuweisung](../images/iscsi-03-groesse-dynamisch.png)

Zum Schluss legt man ein Ziel an und sagt, wer sich verbinden darf. Ich habe die Zuweisung über die IP-Adresse gemacht, weil das im Lab am einfachsten ist. Man kann stattdessen auch den IQN des Clients eintragen oder CHAP mit Benutzername und Kennwort verwenden.

Dieser Schritt ist wichtiger, als er aussieht. Ohne Einschränkung darf sich jeder im Netz mit dem Ziel verbinden, und da der Client die Platte als seine eigene ansieht, wäre das keine gute Idee.

## Clientseite auf Server Core

Mein Client war ein Core-Server ohne grafische Oberfläche, also lief alles über PowerShell. Der Dienst für den Initiator ist standardmässig aus:

```powershell
Start-Service -Name MSiSCSI
Set-Service -Name MSiSCSI -StartupType Automatic

New-IscsiTargetPortal -TargetPortalAddress "172.16.10.21"
$Target = Get-IscsiTarget
Connect-IscsiTarget -NodeAddress $Target.NodeAddress -IsPersistent $true
```

Wichtig ist `-IsPersistent`. Ohne diesen Schalter ist die Verbindung nach einem Neustart weg, und ein Server, der seinen Datenspeicher nach jedem Reboot verliert, ist nicht zu gebrauchen.

Danach taucht die Platte im System auf und wird wie jede neue Festplatte behandelt:

```powershell
Get-Disk | Where-Object BusType -eq "iSCSI"
```

Initialisieren, Partition anlegen, mit NTFS formatieren. Genau die Schritte aus dem Kapitel über die Datenträgerverwaltung, nur dass die Platte diesmal in einem anderen Server steckt.

Das Ergebnis habe ich über das Windows Admin Center kontrolliert. Man sieht dort den Core-Server, die angebundene Netzwerkplatte und das fertige Volume:

![Die iSCSI-Platte auf dem Core-Server im Windows Admin Center](../images/wac-01-core-datentraeger.png)

## Der gleiche Speicher auf einem normalen Windows-Client

Damit ich sehe, dass daran nichts Serverspezifisches ist, habe ich dasselbe Ziel von einem gewöhnlichen Windows-Rechner aus verbunden. Dafür gibt es `iscsicpl.exe`, den iSCSI-Initiator, der in jedem Windows enthalten ist.

Ziel-IP eintragen, verbinden, in der Datenträgerverwaltung formatieren. Danach steht die Platte im Explorer wie eine lokale Festplatte:

![Die iSCSI-Platte als Laufwerk im Explorer](../images/iscsi-05-laptop-laufwerk.png)

Zum Abschluss ein Schreibtest, weil ich es sonst nicht geglaubt hätte:

![Neuer Ordner auf der iSCSI-Platte](../images/iscsi-06-laptop-schreibtest.png)

## Was ich dabei gelernt habe

Am meisten geholfen hat mir zu verstehen, wer eigentlich das Dateisystem besitzt. Bei einer SMB-Freigabe können viele Benutzer gleichzeitig arbeiten, weil der Server die Zugriffe koordiniert. Bei iSCSI passiert das nicht. Der Client denkt, die Platte gehöre ihm allein.

Wenn man dieselbe LUN gleichzeitig an zwei Rechner gibt, die nichts voneinander wissen, schreiben beide in ihre eigene Vorstellung des Dateisystems und beschädigen es. Genau dafür gibt es Cluster-Dateisysteme. iSCSI kümmert sich um den Transport der Blöcke, nicht um die Koordination.

Der zweite Punkt war die Fehlersuche. Als die Verbindung zuerst nicht zustande kam, lag es nicht am iSCSI, sondern an der Firewall und daran, dass die beiden Maschinen in verschiedenen Netzen standen. Port 3260 muss offen und die Gegenstelle normal erreichbar sein. Ein einfacher Ping vorher spart hier viel Zeit.
