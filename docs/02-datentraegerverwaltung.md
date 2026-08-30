# Datenträgerverwaltung: von der virtuellen Platte zum Laufwerk

## Worum es geht

Bevor ich mit RAID und Speicherpools anfangen konnte, musste ich verstehen, was zwischen einer neuen Platte und einem nutzbaren Laufwerk alles passiert. In einer virtuellen Umgebung sind das zwei Ebenen: erst der Hypervisor, dann das Gastsystem.

```mermaid
flowchart TD
    subgraph HV["Proxmox VE"]
        A["Speicher des Hosts"] --> B["Virtuelle Platte<br/>qcow2, an SCSI-Bus angehaengt"]
    end
    subgraph WIN["Windows Server"]
        C["Datentraeger erscheint<br/>offline und nicht initialisiert"]
        D["Online schalten"]
        E["Initialisieren mit GPT"]
        F["Volume anlegen"]
        G["Mit NTFS formatieren<br/>Laufwerksbuchstabe"]
        C --> D --> E --> F --> G
    end
    B --> C
```

Jeder dieser Schritte kann schiefgehen, und die Fehlermeldung sagt nicht immer, auf welcher Ebene das Problem liegt. Deshalb lohnt es sich, beide Seiten zu kennen.

## Die Hypervisor-Seite

Eine Platte in Proxmox hinzuzufügen dauert eine Minute, aber die Optionen dabei sind nicht egal. Ich habe für alle Lab-Platten dieselben Werte verwendet:

| Option | Wert | Warum |
| :--- | :--- | :--- |
| Bus | `SCSI` | `scsi0` ist das Betriebssystem, alles ab `scsi1` sind Datenplatten |
| Controller | `VirtIO SCSI single` | paravirtualisiert, deutlich schneller als eine emulierte Platte |
| Format | `qcow2` | wächst mit den Daten und kann Snapshots |
| Cache | `Write back` | beste Schreibleistung, im Lab vertretbar |
| Discard | an | gibt gelöschte Blöcke an den Host zurück, sonst wächst die Datei immer weiter |
| SSD-Emulation | an | Windows behandelt die Platte als SSD und defragmentiert nicht |
| IO Thread | an | eigener Thread pro Platte, wichtig bei mehreren Platten gleichzeitig |

Über die Weboberfläche geht das alles auch, aber bei mehreren Platten ist die Shell schneller:

```bash
qm set <VMID> --scsi1 <storage>:10,cache=writeback,discard=on,ssd=1,iothread=1
```

Der Punkt, den ich mir gemerkt habe: **Discard und SSD-Emulation gehören zusammen.** Ohne Discard meldet Windows zwar gelöschte Blöcke, der Host gibt sie aber nie frei, und die Image-Datei wächst dauerhaft. Bei einem Test, bei dem ich mehrfach formatiert habe, war das gut sichtbar.

## Die Windows-Seite

Eine neu angehängte Platte ist im Gastsystem zuerst **offline** und **nicht initialisiert**. Das ist kein Fehler, sondern eine Schutzmassnahme: Windows fasst fremde Platten nicht von allein an.

Drei Schritte in `diskmgmt.msc`:

1. **Online schalten.** Rechtsklick auf den Datenträger.
2. **Initialisieren.** Hier fragt Windows nach dem Partitionsstil. Ich nehme immer **GPT**.
3. **Volume anlegen.** Rechtsklick auf den schwarzen, nicht zugeordneten Bereich, dann durch den Assistenten: Grösse, Laufwerksbuchstabe, Dateisystem, Bezeichnung.

### MBR oder GPT

Diese Frage kommt bei jeder neuen Platte, deshalb einmal ausführlich:

| | MBR | GPT |
| :--- | :--- | :--- |
| Maximale Plattengrösse | 2 TB | praktisch unbegrenzt |
| Partitionen | 4 primäre, mehr nur über erweiterte | 128 |
| Partitionstabelle | einmal vorhanden | mehrfach vorhanden, mit Prüfsumme |
| Start | BIOS | UEFI |

GPT ist heute der Normalfall. MBR braucht man nur noch, wenn ein sehr altes System die Platte lesen soll. Da meine VMs mit UEFI starten, wäre MBR ohnehin die falsche Wahl.

### Schnellformatierung oder nicht

Der Assistent bietet **Schnellformatierung** an, und der Haken ist standardmässig gesetzt.

- **Mit Haken:** Es wird nur eine neue, leere Dateizuordnungstabelle geschrieben. Die alten Daten liegen physisch noch da, sind aber nicht mehr auffindbar. Dauert Sekunden.
- **Ohne Haken:** Zusätzlich wird die gesamte Oberfläche auf defekte Sektoren geprüft. Dauert bei grossen Platten sehr lange.

Im Lab immer mit Haken. Ohne Haken ist bei physischen Platten sinnvoll, bei denen man einen Defekt vermutet. Wichtig zu wissen: Schnellformatierung ist **kein sicheres Löschen**. Wer eine Platte weitergibt, muss sie überschreiben.

## Verkleinern und Erweitern

Ein bestehendes Volume lässt sich nachträglich ändern, und beide Richtungen haben ihre eigenen Regeln.

**Verkleinern** geht nur so weit, wie am Ende der Partition zusammenhängend freier Platz vorhanden ist. Windows zeigt oft weniger an, als eigentlich frei ist. Der Grund sind unbewegliche Dateien wie die Auslagerungsdatei oder Schattenkopien, die mitten in der Partition liegen und beim Verkleinern nicht verschoben werden.

**Erweitern** geht nur in den direkt danebenliegenden freien Bereich, und zwar nach rechts. Warum das so ist und was passiert, wenn der freie Platz links liegt, habe ich im [nächsten Kapitel](03-raid.md) mit Screenshots durchgetestet. Das Ergebnis hat mich überrascht.

## Alles auf einmal per PowerShell

Für eine einzelne Platte reicht die Oberfläche. Sobald es mehrere sind, ist die Pipeline schneller und man vertippt sich nicht:

```powershell
Get-Disk | Where-Object PartitionStyle -eq 'RAW' |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -AssignDriveLetter -UseMaximumSize |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "Data" -Confirm:$false
```

Die Kette liest sich wie der Ablauf selbst: alle rohen Platten suchen, als GPT initialisieren, eine Partition über die volle Grösse anlegen, formatieren. Der Filter auf `RAW` ist die Sicherung. Ohne ihn würde der Befehl auch Platten treffen, auf denen schon Daten liegen.

Zum Nachsehen:

```powershell
Get-Disk   | Select-Object Number, FriendlyName, OperationalStatus, PartitionStyle, Size
Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem, Size, SizeRemaining, HealthStatus
```

## Begriffe, die in den nächsten Kapiteln wiederkommen

| Deutsch | Englisch | Bedeutung |
| :--- | :--- | :--- |
| Datenträgerverwaltung | Disk Management | die Konsole `diskmgmt.msc` |
| Basisdatenträger | Basic Disk | normale Platte mit Partitionen |
| dynamischer Datenträger | Dynamic Disk | Voraussetzung für Software-RAID unter Windows |
| Volume | Volume | logische Einheit mit Laufwerksbuchstaben |
| nicht zugeordnet | Unallocated | freier, noch nicht partitionierter Bereich |
| Volume verkleinern | Shrink Volume | Partition verkleinern, freien Platz erzeugen |
| Volume erweitern | Extend Volume | freien Platz an eine Partition anhängen |
| Schnellformatierung | Quick Format | formatieren ohne Oberflächenprüfung |

## Was ich dabei gelernt habe

Der Zustand **offline und nicht initialisiert** hat mich beim ersten Mal ein paar Minuten gekostet. Ich hatte die Platte in Proxmox angelegt, in Windows nachgesehen und nichts gefunden. Sie war da, nur eben nicht angefasst. Seitdem ist der erste Griff bei einer neuen Platte immer `Update-HostStorageCache` und danach ein Blick in `Get-Disk`.

Der zweite Punkt ist die Trennung der Ebenen. Wenn eine Platte in Windows fehlt, kann das am Hypervisor liegen oder am Gastsystem. Es hilft, zuerst zu schauen, ob die Platte in der VM-Hardware überhaupt auftaucht, bevor man in Windows sucht.
