# Speicherpools und Speicherplätze

## Worum es geht

Im vorherigen Kapitel habe ich Software-RAID direkt auf den Platten gebaut. Dabei legt man sich einmal fest: diese vier Platten sind ein RAID 5, fertig. Will man daneben noch einen Spiegel, braucht man weitere Platten.

Storage Spaces dreht das um. Man wirft alle Platten in einen **Pool** und schneidet daraus **virtuelle Datenträger**. Jeder davon darf eine eigene Absicherung haben, obwohl alle aus demselben Plattenvorrat kommen.

```mermaid
flowchart TD
    subgraph P["Physische Platten"]
        D1["Platte 1"]
        D2["Platte 2"]
        D3["..."]
        D4["Platte 12"]
    end
    POOL["Speicherpool<br/>alle Platten, ein Vorrat<br/>plus Reserve-Datentraeger"]
    subgraph V["Virtuelle Datentraeger"]
        V1["Spiegel<br/>2 Kopien"]
        V2["Spiegel<br/>3 Kopien"]
        V3["Paritaet"]
        V4["doppelte Paritaet"]
    end
    subgraph L["Laufwerke in Windows"]
        L1["F:"]
        L2["H:"]
        L3["G:"]
        L4["I:"]
    end
    D1 & D2 & D3 & D4 --> POOL
    POOL --> V1 & V2 & V3 & V4
    V1 --> L1
    V2 --> L2
    V3 --> L3
    V4 --> L4
```

Dazu kommen zwei Dinge, die klassisches Software-RAID nicht kann: **Reserveplatten**, die bei einem Ausfall automatisch einspringen, und **dünne Bereitstellung**, bei der ein Laufwerk erst dann Platz belegt, wenn wirklich geschrieben wird.

Mein Ziel war, alle Absicherungsarten nebeneinander zu bauen und danach Platten im laufenden Betrieb zu ziehen, um zu sehen, welche zuerst aufgibt.

## Vorbereitung

Zwölf virtuelle Platten in verschiedenen Grössen, zusammen rund 128 GB:

![Zwoelf virtuelle SCSI-Platten in der VM-Hardware](../images/storage-proxmox-hardware-12-disks-13.png)

Damit Windows die Platten in einen Pool aufnimmt, müssen sie leer sein. Reste aus früheren Versuchen stören:

```powershell
Update-HostStorageCache
Get-Disk | Where-Object Number -ge 1 | Set-Disk -IsOffline $false
Get-Disk | Where-Object Number -ge 1 | Clear-Disk -RemoveData -RemoveOEM -Confirm:$false
Get-PhysicalDisk | Where-Object DeviceId -ne 0 | Reset-PhysicalDisk
```

Danach stehen alle zwölf online und ohne Partitionen da:

![Alle zwoelf Platten online und leer](../images/storage-datentraeger-alle-12-disks-online-09.png)

Vor dem ersten Pool liegen die Platten im **Primordial-Pool**. Das ist kein richtiger Pool, sondern nur die Liste dessen, was verfügbar wäre. Wichtig ist die Eigenschaft `CanPool`: steht sie auf `False`, hat die Platte noch Reste und wird nicht angeboten.

![Alle Platten im Primordial-Pool](../images/storage-primordial-urpool-12-disks-10.png)

## Den Pool anlegen

Der Assistent liegt in der Serververwaltung unter **Datei- und Speicherdienste**, **Speicherpools**. Zuerst der Name:

![Namensvergabe fuer den Speicherpool](../images/storage-wizard-pool0-name-11.png)

Der zweite Schritt ist der eigentlich interessante. Hier wählt man nicht nur die Platten aus, sondern legt pro Platte fest, wie sie benutzt wird:

![Auswahl der Platten und Zuweisung der Reserve-Datentraeger](../images/storage-wizard-hot-spare-zuordnung-12.png)

- **Automatisch:** normale Datenplatte, zählt zur Kapazität.
- **Reserve-Datenträger:** liegt bereit, trägt keine Daten und zählt nicht zur Kapazität. Fällt eine aktive Platte aus, zieht das System die Reserve automatisch heran und stellt die Redundanz wieder her, ohne dass jemand eingreifen muss.
- **Manuell:** wird nur benutzt, wenn man sie beim Anlegen eines virtuellen Datenträgers ausdrücklich angibt.

Ich habe zwei Platten als Reserve markiert. Von 128 GB brutto bleiben dadurch rund 113 GB nutzbar. Nach Abzug der Verwaltungsdaten zeigt der fertige Pool etwa 122 GB an:

![Kapazitaet des fertigen Pools](../images/storage-pool0-kapazitaet-122g-14.png)

Genau dafür ist eine Reserve da: bei klassischem Software-RAID muss jemand die defekte Platte tauschen, bevor die Wiederherstellung beginnen kann. Hier startet sie sofort.

## Die Entscheidungen beim virtuellen Datenträger

Bei jedem virtuellen Datenträger fragt der Assistent dasselbe: Anordnung, Absicherung, Bereitstellung, Grösse. Diese vier Entscheidungen bestimmen alles Weitere, deshalb hier einmal ausführlich.

### Speicheranordnung

![Auswahl der Speicheranordnung](../images/storage-speicheranordnung-mirror-layout-16.png)

| Anordnung | Prinzip | Verträgt | Platten mindestens |
| :--- | :--- | :--- | :--- |
| Einfach | verteilt, keine Kopie | keinen Ausfall | 1 |
| Spiegelung | mehrere vollständige Kopien | 1 oder 2 Platten | 2 bzw. 5 |
| Parität | Daten plus Prüfsumme | 1 oder 2 Platten | 3 bzw. 7 |

### Zwei oder drei Kopien

![Vergleich Zwei-Wege- und Drei-Wege-Spiegelung](../images/storage-resilienz-zwei-drei-wege-vergleich-17.png)

Die Zwei-Wege-Spiegelung hält zwei Kopien und verträgt den Ausfall einer Platte. Die Drei-Wege-Spiegelung hält drei Kopien und verträgt zwei.

Die Mindestanzahl von **fünf** Platten bei drei Kopien hat mich zuerst gewundert. Drei Kopien, fünf Platten, das passt rechnerisch nicht. Der Grund liegt nicht bei den Daten, sondern bei den Verwaltungsinformationen: auch nach dem Ausfall von zwei Platten muss noch eine eindeutige Mehrheit übrig sein, die entscheiden kann, welcher Stand der richtige ist. Dieses Mehrheitsprinzip wird später beim Ausfalltest sehr wichtig.

### Feste oder dünne Bereitstellung

![Auswahl zwischen duenner und fester Bereitstellung](../images/storage-bereitstellung-duenn-fest-auswahl-18.png)

```mermaid
flowchart LR
    subgraph T["Duenn"]
        T1["Angelegt: 100 GB"] --> T2["Belegt im Pool: nur was<br/>wirklich geschrieben wurde"]
        T2 --> T3["Rest bleibt fuer andere<br/>Datentraeger verfuegbar"]
    end
    subgraph F["Fest"]
        F1["Angelegt: 15 GB"] --> F2["Belegt im Pool: sofort 15 GB"]
        F2 --> F3["Reserviert, auch wenn leer"]
    end
```

**Fest** reserviert den Platz sofort. Vorteil: der Platz ist sicher da und die Leistung ist etwas besser, weil nicht ständig nachbelegt werden muss.

**Dünn** reserviert nur ein paar Megabyte für Verwaltungsdaten und holt Platz nach, sobald geschrieben wird. Vorteil: man kann grosszügig planen und muss nicht raten, wie gross ein Laufwerk in zwei Jahren sein soll.

Der Haken bei dünner Bereitstellung: man kann mehr vergeben, als der Pool hat. Läuft der Pool voll, stehen alle Laufwerke gleichzeitig, obwohl im Explorer noch Platz angezeigt wird. Wer dünn arbeitet, muss den Pool überwachen.

Ich habe alle vier Datenträger dünn angelegt, damit sie sich nicht gegenseitig den Platz wegnehmen.

## Datenträger 1: Spiegel mit zwei Kopien

Name und Beschreibung:

![Name und Beschreibung des ersten virtuellen Datentraegers](../images/storage-vdisk-name-beschreibung-15.png)

Grösse 20 GB:

![Groesse des ersten virtuellen Datentraegers](../images/storage-vdisk-groesse-20g-19.png)

Ein virtueller Datenträger ist noch kein Laufwerk. Er erscheint im System als neue Platte und muss ganz normal partitioniert und formatiert werden. Der Assistent dafür startet direkt im Anschluss:

![Auswahl von Server und Datentraeger im Volume-Assistenten](../images/storage-volume-server-datentraeger-auswahl-20.png)

![Groesse des Volumes](../images/storage-volume-groesse-20g-21.png)

![Zuweisung des Laufwerkbuchstabens](../images/storage-volume-laufwerkbuchstabe-f-22.png)

![Dateisystem NTFS und Bezeichnung](../images/storage-volume-ntfs-raid1-2mirror-23.png)

Danach liegt das Laufwerk im Explorer wie jedes andere:

![Das fertige Laufwerk im Explorer](../images/storage-explorer-dieser-pc-f-drive-24.png)

Eine Sache wollte ich nachsehen: auf welchen physischen Platten liegt dieser Spiegel eigentlich? Bei klassischem RAID 1 wären es genau zwei. Hier nicht:

```powershell
Get-VirtualDisk -FriendlyName "vDisk0" | Get-PhysicalDisk
```

![Physische Platten hinter dem virtuellen Datentraeger](../images/storage-powershell-vdisk0-physical-disks-25.png)

Die Ausgabe listet **alle zehn aktiven Platten** des Pools. Storage Spaces spiegelt nicht Platte gegen Platte, sondern arbeitet mit kleinen Abschnitten, sogenannten Slabs. Jeder Abschnitt wird zweimal abgelegt, aber die Paare sind über den ganzen Pool verteilt. Deshalb ist die Wiederherstellung nach einem Ausfall auch schneller: es lesen viele Platten gleichzeitig, statt dass eine einzige Partnerplatte alles liefern muss.

Die beiden Reserveplatten tauchen in der Liste nicht auf. Sie warten.

## Datenträger 2: Spiegel mit drei Kopien

Gleicher Ablauf, andere Entscheidung bei der Absicherung.

![Name des zweiten virtuellen Datentraegers](../images/storage-vdisk1-name-dialog-26.png)

![Anordnung Spiegelung](../images/storage-vdisk1-mirror-layout-27.png)

![Auswahl der Drei-Wege-Spiegelung](../images/storage-vdisk1-drei-wege-spiegelung-28.png)

![Duenne Bereitstellung](../images/storage-vdisk1-duenn-bereitstellung-29.png)

![Groesse 30 GB](../images/storage-vdisk1-groesse-30g-30.png)

30 GB angelegt bedeutet hier 90 GB tatsächlichen Verbrauch, sobald das Laufwerk voll ist. Wegen der dünnen Bereitstellung wird davon zunächst nichts belegt.

## Datenträger 3: einfache Parität

![Name des Paritaets-Datentraegers](../images/storage-vdisk-parity-name-dialog-31.png)

![Anordnung Paritaet](../images/storage-vdisk-parity-layout-selection-32.png)

![Vergleich einfache und doppelte Paritaet](../images/storage-vdisk-parity-einzel-dual-vergleich-33.png)

Der Vergleichsdialog nennt beide Varianten samt Mindestanzahl: einfache Parität ab drei Platten, doppelte ab sieben. Mit zehn aktiven Platten ging beides.

![Duenne Bereitstellung](../images/storage-vdisk-parity-duenn-bereitstellung-34.png)

![Groesse 15 GB](../images/storage-vdisk-parity-groesse-15g-35.png)

![Formatierung mit NTFS](../images/storage-volume-ntfs-raid5-label-36.png)

## Datenträger 4: doppelte Parität

![Name des Datentraegers mit doppelter Paritaet](../images/storage-vdisk-dual-parity-name-39.png)

![Anordnung Paritaet](../images/storage-vdisk-dual-parity-layout-40.png)

![Auswahl der doppelten Paritaet](../images/storage-vdisk-dual-parity-selection-41.png)

![Duenne Bereitstellung](../images/storage-vdisk-dual-parity-duenn-42.png)

![Groesse 18 GB](../images/storage-vdisk-dual-parity-groesse-18g-43.png)

![Formatierung mit NTFS](../images/storage-volume-ntfs-raid6-label-44.png)

## Zwischenstand

Vier Laufwerke, vier verschiedene Absicherungen, ein einziger Pool:

| Datenträger | Absicherung | Grösse | Verträgt | Laufwerk |
| :--- | :--- | :--- | :--- | :--- |
| vDisk0 | Spiegel, 2 Kopien | 20 GB | 1 Platte | F: |
| vDisk-1Parity | einfache Parität | 15 GB | 1 Platte | G: |
| vDisk1 | Spiegel, 3 Kopien | 30 GB | 2 Platten | H: |
| vDisk-2Parity | doppelte Parität | 18 GB | 2 Platten | I: |

![Alle vier Laufwerke im Explorer](../images/storage-explorer-all-4-volumes-f-g-h-i-45.png)

Im Explorer stehen 83 GB nutzbarer Speicher, obwohl der Pool davon fast nichts belegt hat. Das ist die dünne Bereitstellung in einem Bild.

## Der Ausfalltest

Jetzt der Teil, für den ich das Ganze gebaut hatte. Ich habe im laufenden Betrieb Platten aus der VM entfernt, eine nach der anderen, ohne vorher etwas herunterzufahren.

![Plattenliste in Proxmox vor dem Ausfalltest](../images/storage-fault-injection-proxmox-scsi-list-46.png)

### Erste Platte weg

![Zustand nach dem Entfernen der ersten Platte](../images/storage-fault-step1-scsi12-vdisk0-down-47.png)

- Pool: Warnung
- **Zwei-Wege-Spiegel: ausgefallen**, im Explorer verschwunden
- Drei-Wege-Spiegel, einfache und doppelte Parität: Warnung, aber weiter erreichbar

Der Zwei-Wege-Spiegel verträgt rechnerisch einen Ausfall. Er ist trotzdem als Erster weggebrochen, weil bei einem harten Abziehen nicht sauber abgemeldet wird. Bei nur zwei Kopien fehlt dann die Mehrheit, um zu entscheiden, welche Kopie gültig ist, und Storage Spaces nimmt den Datenträger lieber offline, als möglicherweise falsche Daten auszuliefern.

### Zweite Platte weg

![Zustand nach dem Entfernen der zweiten Platte](../images/storage-fault-step2-scsi11-vdisk2parity-down-48.png)

Jetzt gab auch die doppelte Parität auf. Übrig blieben die einfache Parität und der Drei-Wege-Spiegel, beide noch lesbar.

Dass ausgerechnet die doppelte Parität vor der einfachen ausfiel, war für mich das Überraschendste am ganzen Test. Auf dem Papier verträgt sie mehr. In der Praxis muss sie bei jedem Zugriff aus zwei Prüfsummen zurückrechnen und ist damit empfindlicher, wenn Platten unsauber verschwinden.

### Dritte Platte weg

![Kompletter Ausfall des Pools](../images/storage-fault-step3-pool0-complete-failure-49.png)

Der ganze Pool fiel aus. Nicht einzelne Datenträger, sondern alle vier gleichzeitig. Im Explorer blieb nur noch `C:`.

Der Grund ist wieder das Mehrheitsprinzip, diesmal auf Pool-Ebene. Der Pool führt seine eigene Verwaltungsdatenbank, die mehrfach auf den Platten liegt. Fehlt davon die Mehrheit, weiss der Pool nicht mehr sicher, wie er aufgebaut ist. Statt zu raten, schaltet er sofort ab.

Rechnerisch wären einzelne Datenträger noch zu retten gewesen. Das nützt nichts, wenn die Ebene darunter nicht mehr weiss, wo die Abschnitte liegen.

## Die Wiederherstellung

Die entfernten Platten lagen in Proxmox noch als nicht zugewiesene Platten herum, die Daten waren also unverändert:

![Nicht zugewiesene Platten in Proxmox](../images/storage-recovery-proxmox-unused-disks-50.png)

Ich habe sie in umgekehrter Reihenfolge zurückgesteckt.

**Erste Platte zurück:**

![Zustand nach der ersten zurueckgesteckten Platte](../images/storage-recovery-step1-scsi10-attached-51.png)

Der Pool hob die Notabschaltung sofort auf. Die einfache Parität und der Drei-Wege-Spiegel waren augenblicklich wieder im Explorer. Die Mehrheit für die Verwaltungsdatenbank war wiederhergestellt, und damit lief die Ebene darunter wieder.

**Zweite Platte zurück:**

![Zustand nach der zweiten zurueckgesteckten Platte](../images/storage-recovery-step2-scsi11-attached-52.png)

Doppelte Parität und Zwei-Wege-Spiegel kamen zurück, die doppelte Parität blieb aber auf Warnung, weil noch eine Platte fehlte.

**Dritte Platte zurück:**

![Hinzufuegen der letzten Platte in Proxmox](../images/storage-recovery-proxmox-add-scsi12-dialog-53.png)

![Alle Datentraeger wieder fehlerfrei](../images/storage-recovery-step3-all-volumes-restored-healthy-54.png)

Alle vier Datenträger meldeten wieder **fehlerfrei**, alle Laufwerke waren da, kein Datenverlust. Die Synchronisierung lief von allein, ich musste nichts reaktivieren.

Das ist ein deutlicher Unterschied zum Software-RAID aus dem vorherigen Kapitel. Dort musste ich das Volume von Hand reaktivieren. Hier erkennt der Pool die zurückkehrenden Platten und repariert selbständig.

## Dasselbe per PowerShell

Über die Oberfläche versteht man die Entscheidungen besser, für den zweiten Aufbau nimmt man ein Skript:

```powershell
# Platten, die in einen Pool dürfen
$disks = Get-PhysicalDisk -CanPool $True

# Pool anlegen
New-StoragePool -FriendlyName "Auto_Pool" `
    -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks $disks

# Virtuellen Datenträger anlegen
$vDisk = New-VirtualDisk -StoragePoolFriendlyName "Auto_Pool" -FriendlyName "Auto_Mirror" `
    -ResiliencySettingName Mirror -NumberOfDataCopies 2 -ProvisioningType Thin -Size 20GB

# Und daraus ein Laufwerk machen
$vDisk | Get-Disk |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -AssignDriveLetter -UseMaximumSize |
    Format-Volume -FileSystem ReFS -NewFileSystemLabel "Auto_ReFS" -Confirm:$false
```

Zum Aufräumen nach einem Testlauf:

```powershell
Get-VirtualDisk | Remove-VirtualDisk -Confirm:$false
Get-StoragePool -IsPrimordial $false | Remove-StoragePool -Confirm:$false
```

Die Reihenfolge ist Pflicht. Solange virtuelle Datenträger existieren, lässt sich der Pool nicht entfernen.

## Test und Verifikation

Storage Spaces hat drei Ebenen, und jede hat ihren eigenen Zustand. Bei einer Störung muss man wissen, welche davon meldet. Alle Ausgaben von `SRV25-GUI` (`172.16.10.21`).

### Ebene 1: der Pool

```powershell
Get-StoragePool -IsPrimordial $false |
    Select-Object FriendlyName, HealthStatus, OperationalStatus, `
                  @{n='GrossGB';e={[math]::Round($_.Size/1GB,1)}}, `
                  @{n='FreiGB';e={[math]::Round(($_.Size-$_.AllocatedSize)/1GB,1)}}
```

```
FriendlyName HealthStatus OperationalStatus GrossGB FreiGB
------------ ------------ ----------------- ------- ------
Pool0        Healthy      OK                  122.0  119.4
```

Die Spalte `FreiGB` ist bei dünner Bereitstellung die wichtigste Kennzahl überhaupt. Angelegt waren Laufwerke über 83 GB, belegt sind davon nur wenige. Läuft dieser Wert gegen null, stehen **alle** Laufwerke gleichzeitig, obwohl der Explorer noch Platz anzeigt.

### Ebene 2: die virtuellen Datenträger

```powershell
Get-VirtualDisk | Select-Object FriendlyName, ResiliencySettingName, `
                                @{n='GB';e={[math]::Round($_.Size/1GB,0)}}, `
                                HealthStatus, OperationalStatus
```

```
FriendlyName  ResiliencySettingName GB HealthStatus OperationalStatus
------------  --------------------- -- ------------ -----------------
vDisk0        Mirror                20 Healthy      OK
vDisk1        Mirror                30 Healthy      OK
vDisk-1Parity Parity                15 Healthy      OK
vDisk-2Parity Parity                18 Healthy      OK
```

Nach dem Entfernen der ersten Platte sah dieselbe Abfrage so aus:

```
FriendlyName  ResiliencySettingName GB HealthStatus OperationalStatus
------------  --------------------- -- ------------ -----------------
vDisk0        Mirror                20 Unhealthy    Detached
vDisk1        Mirror                30 Warning      Degraded
vDisk-1Parity Parity                15 Warning      Degraded
vDisk-2Parity Parity                18 Warning      Degraded
```

Der Unterschied zwischen `Degraded` und `Detached` ist genau der Unterschied zwischen "läuft ohne Reserve weiter" und "im Explorer nicht mehr vorhanden". Drei Laufwerke lieferten weiter Daten aus, der Zwei-Wege-Spiegel nicht mehr.

### Ebene 3: die physischen Platten

```powershell
Get-PhysicalDisk | Select-Object DeviceId, Usage, HealthStatus, OperationalStatus, `
                                 @{n='GB';e={[math]::Round($_.Size/1GB,0)}}
```

Hier sieht man auch, welche Platten als Reserve dienen: sie tragen `Usage = HotSpare` statt `Auto-Select` und zählen nicht zur nutzbaren Kapazität.

### Wo liegt ein Datenträger wirklich?

```powershell
Get-VirtualDisk -FriendlyName "vDisk0" | Get-PhysicalDisk |
    Select-Object DeviceId, Usage, HealthStatus
```

Die Antwort waren **alle zehn** aktiven Platten, nicht zwei. Der Screenshot dazu steht weiter oben beim ersten Datenträger. Das ist der wichtigste Unterschied zum klassischen RAID 1: dort wäre ein Spiegel genau zwei Platten, hier verteilt er sich in kleinen Abschnitten über den gesamten Pool.

### Der Ausfall aus Sicht des Dateisystems

```powershell
Get-Volume | Where-Object DriveLetter -in 'F','G','H','I' |
    Select-Object DriveLetter, FileSystemLabel, HealthStatus, `
                  @{n='FreiGB';e={[math]::Round($_.SizeRemaining/1GB,1)}}
```

Im degradierten Zustand fehlte `F` in dieser Liste, während `G`, `H` und `I` unverändert `Healthy` meldeten und ihre Dateien auslieferten. Nach dem Zurückstecken aller Platten waren alle vier wieder vollständig da.

### Reparatur beobachten

```powershell
Get-StorageJob
Repair-VirtualDisk -FriendlyName "vDisk0"
```

`Get-StorageJob` zeigt laufende Reparaturen mit Fortschritt in Prozent. Solange dort ein Auftrag läuft, ist die Redundanz noch nicht wiederhergestellt, auch wenn der Datenträger schon wieder erreichbar ist.

## Fehlersuche

| Symptom | Ursache | Lösung |
| :--- | :--- | :--- |
| Platte wird im Pool-Assistenten nicht angeboten | Reste einer alten Partitionierung, `CanPool = False` | `Clear-Disk -RemoveData -RemoveOEM`, danach `Reset-PhysicalDisk` |
| Drei-Wege-Spiegelung nicht auswählbar | weniger als fünf Platten im Pool | Platten ergänzen oder Zwei-Wege-Spiegelung nehmen |
| Doppelte Parität nicht auswählbar | weniger als sieben Platten im Pool | Platten ergänzen oder einfache Parität nehmen |
| Datenträger `Detached` nach Plattenausfall | zu wenige Kopien für eine eindeutige Mehrheit | Platte zurückholen, Pool erkennt sie selbst |
| Ganzer Pool offline | Mehrheit der Verwaltungsdaten verloren | Platten zurückholen, Pool hebt die Sperre selbst auf |
| Laufwerk voll, obwohl der Explorer Platz zeigt | dünne Bereitstellung, Pool ist erschöpft | Platten ergänzen oder Datenträger verkleinern |
| Reparatur startet nicht von allein | kein freier Platz und keine Reserve | `Repair-VirtualDisk` von Hand, Reserve einplanen |
| Pool lässt sich nicht entfernen | virtuelle Datenträger existieren noch | erst `Remove-VirtualDisk`, dann `Remove-StoragePool` |

## Begriffe

| Deutsch | Englisch | Bedeutung |
| :--- | :--- | :--- |
| der Speicherpool | Storage Pool | Zusammenfassung physischer Platten zu einem Vorrat |
| der Speicherplatz | Storage Space | virtueller Datenträger aus dem Pool |
| der Ur-Pool | Primordial Pool | Liste der Platten, die in einen Pool könnten |
| der Reserve-Datenträger | Hot Spare | Reserveplatte, springt bei Ausfall automatisch ein |
| die Resilienz | Resiliency | Absicherungsart: einfach, Spiegel, Parität |
| die Zwei-Wege-Spiegelung | Two-Way Mirror | zwei Kopien, verträgt eine Platte |
| die Drei-Wege-Spiegelung | Three-Way Mirror | drei Kopien, verträgt zwei Platten |
| die einfache Parität | Single Parity | eine Prüfsumme, verträgt eine Platte |
| die doppelte Parität | Dual Parity | zwei Prüfsummen, verträgt zwei Platten |
| die dünne Bereitstellung | Thin Provisioning | Platz wird erst beim Schreiben belegt |
| die feste Bereitstellung | Fixed Provisioning | Platz wird sofort reserviert |
| der Abschnitt | Slab | kleine Einheit, in der Daten über den Pool verteilt werden |
| das Mehrheitsprinzip | Quorum | Regel, nach der der Pool über Gültigkeit entscheidet |

## Was ich dabei gelernt habe

**Die Reihenfolge der Ausfälle war nicht die erwartete.** Ich hätte auf die doppelte Parität gesetzt, gewonnen hat der Drei-Wege-Spiegel. Ein Spiegel muss beim Lesen nur eine andere Kopie nehmen, Parität muss rechnen. Wenn Platten unsauber wegbrechen, ist die einfachere Lösung die robustere. Genau deshalb setzt Microsoft bei virtuellen Maschinen und Datenbanken auf Spiegelung und empfiehlt Parität eher für Archivdaten.

**Redundanz pro Laufwerk hilft nicht, wenn der Pool kippt.** Alle vier Laufwerke lagen im selben Pool und sind gemeinsam ausgefallen. Der Pool ist eine eigene Fehlerquelle, die man bei klassischem RAID so nicht hat. Für echte Sicherheit braucht es eine Sicherung ausserhalb.

**Das Mehrheitsprinzip erklärt beides.** Warum drei Kopien fünf Platten brauchen, warum der Zwei-Wege-Spiegel als Erster wegbrach und warum der Pool bei der dritten Platte komplett abschaltete: immer dieselbe Regel. Sobald keine eindeutige Mehrheit mehr da ist, entscheidet sich Storage Spaces gegen die Verfügbarkeit und für die Datensicherheit.

**Dünne Bereitstellung ist bequem und gefährlich zugleich.** 83 GB Laufwerke aus einem Pool mit 122 GB sind unproblematisch. Hätte ich mehr vergeben, als vorhanden ist, würden alle Laufwerke gleichzeitig stehen bleiben, sobald der Pool voll ist. Ohne Überwachung merkt man das erst, wenn es passiert ist.
