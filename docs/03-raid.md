# Volume-Typen und Software-RAID in der Datenträgerverwaltung

## Worum es geht

RAID lässt sich in einer Tabelle auswendig lernen: RAID 0 ist schnell, RAID 1 ist sicher, RAID 5 hat Parität. Was diese Sätze wirklich bedeuten, merkt man aber erst, wenn man eine Platte abschaltet und zusieht, was mit den Daten passiert.

Windows Server kann Software-RAID über dynamische Datenträger. Produktiv nimmt man das heute selten, weil Storage Spaces und Hardware-Controller besser sind. Zum Lernen ist es ideal: in Proxmox sind fünf Platten in einer Minute angelegt, jede Variante ist in ein paar Klicks gebaut, und einen Ausfall simuliert man mit einem Rechtsklick.

Ich habe alle Typen einmal gebaut, danach jeweils eine Platte offline geschaltet und dokumentiert, was übrig blieb.

## Vorbereitung

Fünf Platten mit **absichtlich unterschiedlichen Grössen**, weil sich daran später eine wichtige Regel zeigt:

```bash
qm set <VMID> --scsi1 <storage>:10,cache=writeback,discard=on,ssd=1,iothread=1
qm set <VMID> --scsi2 <storage>:10,cache=writeback,discard=on,ssd=1,iothread=1
qm set <VMID> --scsi3 <storage>:10,cache=writeback,discard=on,ssd=1,iothread=1
qm set <VMID> --scsi4 <storage>:20,cache=writeback,discard=on,ssd=1,iothread=1
qm set <VMID> --scsi5 <storage>:5,cache=writeback,discard=on,ssd=1,iothread=1
```

![Virtuelle Platten in der Hardwareuebersicht von Proxmox](../images/raid-proxmox-all-disks-setup-18.png)

Im Gastsystem tauchen sie zuerst offline auf. Die Serververwaltung zeigt das im Bereich Datenträger:

![Neue Datentraeger im Zustand offline](../images/raid-server-manager-datentraeger-offline-01.png)

Nach dem Onlineschalten kommt die Initialisierung. GPT, wie im vorherigen Kapitel beschrieben:

![Initialisierung des Datentraegers mit GPT](../images/raid-diskmgmt-initialisierung-01.png)

## Die Farben in der Datenträgerverwaltung

Ein Detail, das mir das Lesen der Screenshots sehr erleichtert hat: die Datenträgerverwaltung färbt jeden Volume-Typ anders ein. Man erkennt den Typ also, ohne irgendwo nachzusehen.

| Farbe | Typ |
| :--- | :--- |
| Dunkelblau | einfaches Volume auf einer Basisplatte |
| Olivgrün | einfaches Volume auf einer dynamischen Platte |
| Violett | übergreifendes Volume |
| Türkis | Stripesetvolume, RAID 0 |
| Bordeaux | gespiegeltes Volume, RAID 1 |
| Hellblau | RAID-5-Volume |
| Schwarz | nicht zugeordneter Bereich |

## Erstes Experiment: verkleinern, löschen, erweitern

Bevor ich zu RAID kam, wollte ich den Unterschied zwischen Basisplatte und dynamischer Platte verstehen. Der Versuch hat ein Ergebnis geliefert, mit dem ich nicht gerechnet hatte.

Ausgangslage: ein Volume über die ganze 10-GB-Platte. Danach verkleinern.

![Dialog zum Verkleinern des Volumes](../images/raid-diskmgmt-volume-a-verkleinern-dialog-23.png)

Der Dialog zeigt vier Zahlen, und die dritte ist die einzige, die man eingibt. Von rund 10 GB waren etwa 7 GB verfügbar, es blieben gut 3 GB stehen. Aus dem frei gewordenen Bereich habe ich ein zweites Volume gemacht. Zwei ganz normale Partitionen auf einer Basisplatte:

![Zwei Basisdatenpartitionen nach dem Verkleinern](../images/raid-diskmgmt-volume-a-b-split-result-24.png)

Jetzt der eigentliche Versuch. Ich habe zwei Varianten getestet.

```mermaid
flowchart TD
    S["Eine Platte, zwei Volumes: A links, B rechts"]
    S --> V1["Variante 1<br/>A loeschen, Luecke liegt LINKS von B"]
    S --> V2["Variante 2<br/>B loeschen, Luecke liegt RECHTS von A"]
    V1 --> R1["B erweitern<br/>Windows wandelt die Platte in DYNAMISCH um<br/>ein Laufwerk, zwei getrennte Bereiche"]
    V2 --> R2["A erweitern<br/>Platte bleibt BASIS<br/>ein zusammenhaengender Block"]
```

### Variante 1: freier Platz links

Zuerst das linke Volume gelöscht. Windows warnt hier deutlich, weil dabei Daten verloren gehen:

![Warnung beim Loeschen des Volumes](../images/raid-diskmgmt-volume-a-loeschen-warning-25.png)

Danach steht der freie Bereich links, das verbleibende Volume rechts davon. Ich habe versucht, es nach links zu erweitern:

![Erweitern mit freiem Bereich links](../images/raid-diskmgmt-volume-b-erweitern-left-unallocated-26.png)

Windows fragt nicht lange und wandelt die Platte in eine **dynamische** Platte um. Das Ergebnis sieht im Explorer aus wie ein einziges Laufwerk mit rund 10 GB, in der Datenträgerverwaltung liegt es aber in zwei getrennten Bereichen, und die Platte ist olivgrün:

![Dynamische Platte mit zwei getrennten Bereichen](../images/raid-diskmgmt-volume-b-dynamic-split-result-27.png)

### Variante 2: freier Platz rechts

Derselbe Aufbau, nur andersherum. Der freie Bereich liegt direkt rechts neben dem Volume:

![Freier Bereich rechts neben dem Volume](../images/raid-diskmgmt-volume-a-erweitern-right-unallocated-28.png)

Der Assistent bietet den anschliessenden Bereich an:

![Assistent zum Erweitern des Volumes](../images/raid-diskmgmt-volume-a-erweitern-wizard-29.png)

Diesmal keine Umwandlung. Die Platte bleibt eine Basisplatte, das Volume ist ein durchgehender dunkelblauer Block:

![Zusammenhaengendes Volume auf einer Basisplatte](../images/raid-diskmgmt-volume-a-single-basis-result-30.png)

### Warum das so ist

Eine Partition auf einer Basisplatte wird durch Startsektor und Länge beschrieben. Nach hinten wachsen heisst: die Länge erhöhen. Nach vorne wachsen hiesse: den Startsektor verschieben, und dann läge jeder Block der vorhandenen Daten an der falschen Stelle.

Die dynamische Platte arbeitet anders. Sie führt eine eigene Datenbank, in der ein Volume aus mehreren Bereichen zusammengesetzt sein darf. Deshalb kann sie die Lücke links anhängen, und deshalb muss Windows umwandeln.

Wichtig dabei: **die Umwandlung ist eine Einbahnstrasse.** Zurück zu einer Basisplatte kommt man nur, indem man alle Volumes löscht.

## Übergreifendes Volume

Zwei Platten mit 10 und 5 GB werden zu einem Laufwerk mit 15 GB zusammengefasst. Die Auswahl im Assistenten:

![Auswahl der Platten fuer das uebergreifende Volume](../images/raid-diskmgmt-spanned-select-disks-04.png)

Die Daten laufen linear: erst wird die erste Platte gefüllt, dann die nächste.

![Fertiges uebergreifendes Volume](../images/raid-diskmgmt-spanned-data-label-result-09.png)

Der einzige Vorteil ist Kapazität, und zwar die volle Summe, auch bei ungleichen Platten. Es gibt keine höhere Geschwindigkeit und keinerlei Sicherheit. Fällt eine der beiden Platten aus, ist das gesamte Volume verloren, auch der Teil, der auf der intakten Platte liegt, denn die Verwaltungsstrukturen des Dateisystems sind verteilt.

## Stripesetvolume, RAID 0

Hier werden die Daten blockweise abwechselnd auf beide Platten geschrieben. Zwei Platten schreiben gleichzeitig, das bringt Tempo.

![Fertiges Stripesetvolume in Tuerkis](../images/raid-diskmgmt-striped-cyan-result-12.png)

An dieser Stelle ist mir zum ersten Mal die **Regel der kleinsten Platte** begegnet. Ich habe eine 10-GB- und eine 12-GB-Platte kombiniert und statt 22 GB nur 20 GB bekommen. Von jeder Platte wird immer nur so viel benutzt, wie die kleinste hergibt, denn die Streifen müssen auf allen Platten gleich gross sein. Die restlichen 2 GB bleiben als nicht zugeordneter Bereich liegen.

Dann der Ausfalltest. Eine Platte offline geschaltet:

![Stripesetvolume nach dem Ausfall einer Platte](../images/raid-diskmgmt-striped-failure-test-13.png)

Das Volume steht sofort auf **Fehler**, das Laufwerk ist im Explorer verschwunden. Bei RAID 0 gibt es keine halben Zustände: jeder zweite Block fehlt, damit ist keine einzige Datei mehr vollständig.

Wichtig für die Einordnung: RAID 0 erhöht das Ausfallrisiko sogar. Zwei Platten haben zusammen eine höhere Ausfallwahrscheinlichkeit als eine einzelne, und hier reicht eine davon.

## Gespiegeltes Volume, RAID 1

Zwei Platten, identischer Inhalt. Man bezahlt mit der Hälfte der Kapazität.

Ich habe eine 15-GB- und eine 20-GB-Platte genommen. Der Spiegel richtet sich nach der kleineren, auf der grösseren bleiben rund 5 GB übrig:

![Gespiegeltes Volume mit uebrig gebliebenem Bereich](../images/raid-diskmgmt-mirror-result-unallocated-5g-16.png)

Interessant ist, was mit diesem Rest passieren darf. Bei einem Hardware-Controller wäre er verloren. Windows dagegen verwaltet die Platte weiter, der freie Bereich lässt sich als eigenes Volume nutzen oder in eine andere Konstruktion einbauen.

Der Ausfalltest, gleiche Vorgehensweise wie bei RAID 0, aber ein völlig anderes Ergebnis:

![Gespiegeltes Volume im Zustand fehlerhafte Redundanz](../images/raid-diskmgmt-mirror-degraded-fault-tolerance-17.png)

Das Volume meldet **Fehlerhafte Redundanz**, das Laufwerk bleibt aber im Explorer und die Daten sind normal lesbar. Genau das ist der Zweck.

Der Zustandsname ist gut gewählt: nicht die Daten sind fehlerhaft, sondern die Redundanz. Die Sicherheitsreserve ist aufgebraucht. Fällt jetzt die zweite Platte aus, sind die Daten weg.

## RAID 5

Vier Platten mit verteilter Parität. Die Parität ist eine XOR-Verknüpfung: aus den vorhandenen Blöcken lässt sich der fehlende zurückrechnen. Sie liegt nicht auf einer eigenen Platte, sondern reihum verteilt, damit keine Platte zum Flaschenhals wird.

```mermaid
flowchart TD
    subgraph R5["RAID 5 ueber vier Platten"]
        D1["Platte 1<br/>Daten / Daten / Paritaet"]
        D2["Platte 2<br/>Daten / Paritaet / Daten"]
        D3["Platte 3<br/>Paritaet / Daten / Daten"]
        D4["Platte 4<br/>Daten / Daten / Paritaet"]
    end
    R5 --> N["Nutzbar: Anzahl minus 1<br/>mal Groesse der kleinsten Platte"]
```

Die Plattenauswahl im Assistenten, bewusst mit vier verschiedenen Grössen:

![Auswahl der Platten fuer RAID 5](../images/raid-diskmgmt-raid5-select-disks-18.png)

Die Kapazitätsanzeige zeigt das Ergebnis der Regel der kleinsten Platte:

![Kapazitaetsanzeige im RAID-5-Assistenten](../images/raid-diskmgmt-raid5-wizard-capacity-19.png)

Bei 8, 12, 6 und 12 GB rechnet Windows mit 6 GB pro Platte. Nutzbar sind also drei mal 6 GB, rund 18 GB, obwohl 38 GB verbaut sind. Der Rest bleibt liegen:

![Fertiges RAID-5-Volume mit ungenutzten Bereichen](../images/raid-diskmgmt-raid5-light-blue-unallocated-result-20.png)

Der Verschnitt ist auf den Platten gut zu sehen. Bei gleich grossen Platten wäre er null, und die Effizienz steigt mit der Anzahl:

| Platten | Nutzbar | Effizienz |
| :--- | :--- | :--- |
| 3 mal 10 GB | 20 GB | 67 % |
| 4 mal 10 GB | 30 GB | 75 % |
| 5 mal 10 GB | 40 GB | 80 % |

Danach Testdaten geschrieben und eine Platte offline geschaltet:

![RAID 5 im Zustand fehlerhafte Redundanz](../images/raid-diskmgmt-raid5-degraded-offline-test-21.png)

Wie beim Spiegel: Warnung ja, Datenverlust nein. Das Laufwerk zeigt weiterhin die volle Kapazität und die Dateien lassen sich öffnen. Windows rechnet die fehlenden Blöcke bei jedem Zugriff aus der Parität zurück. Das kostet Leistung, aber der Betrieb läuft.

## Der Rebuild

Beim Wiedereinschalten übernimmt Windows die Platte nicht von allein. Sie ist wieder da, das Volume bleibt aber im Fehlerzustand, bis man **Volume reaktivieren** wählt.

![Reaktivierung und Synchronisierung des RAID-5-Volumes](../images/raid-diskmgmt-raid5-reactivate-rebuild-22.png)

Danach läuft die Synchronisierung mit einer Prozentanzeige, und erst wenn sie durch ist, steht wieder **Fehlerfrei**.

## Was ich dabei gelernt habe

**Der Rebuild ist die kritische Phase, nicht der Ausfall.** Zwischen "Platte wieder online" und "Fehlerfrei" liegt die Synchronisierung, und in dieser Zeit gibt es weiterhin keine Redundanz. Bei RAID 5 wird dabei jede andere Platte vollständig gelesen, also genau die Belastung, bei der eine schwächelnde Platte ausfällt. Bei grossen Platten dauert das Stunden.

**Ein degradiertes RAID sieht völlig normal aus.** Im Explorer ist nichts zu sehen: gleiche Kapazität, alle Dateien da. Nur die Datenträgerverwaltung zeigt den Zustand. Ohne Überwachung merkt man den ersten Ausfall gar nicht und erfährt vom Problem erst beim zweiten, wenn es zu spät ist.

**Die Regel der kleinsten Platte kostet echtes Geld.** Bei meinem RAID 5 aus 8, 12, 6 und 12 GB waren 14 GB ungenutzt, mehr als ein Drittel. Wer RAID plant, kauft gleich grosse Platten.

**RAID ist kein Backup.** Es schützt gegen Plattenausfall, sonst gegen nichts. Eine gelöschte Datei ist auf beiden Spiegelhälften gleichzeitig gelöscht. Verschlüsselungstrojaner, Bedienfehler und ein Defekt am Controller treffen alle Platten zusammen.
