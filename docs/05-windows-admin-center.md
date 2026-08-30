# Windows Admin Center

## Worum es geht

In meinem Lab laufen zwei Server ohne grafische Oberfläche. Das ist die empfohlene Installationsart: Server Core braucht weniger Arbeitsspeicher, weniger Plattenplatz, deutlich weniger Neustarts nach Updates und bietet weniger Angriffsfläche, weil vieles gar nicht erst installiert ist.

Für mich als Anfänger war das am Anfang unbequem. Auf einem Server Core sieht man nichts. Man muss wissen, wonach man fragt.

Windows Admin Center schliesst diese Lücke. Es wird auf einem Server installiert und ist danach über den Browser erreichbar. Von dieser einen Stelle aus verwaltet man alle anderen Server mit, auch die ohne Oberfläche. Auf den verwalteten Maschinen muss nichts installiert werden, die Verbindung läuft über WinRM, das ohnehin vorhanden ist.

```mermaid
flowchart LR
    B["Browser<br/>auf meinem Rechner"]
    subgraph GW["Server mit Oberflaeche"]
        WAC["Windows Admin Center<br/>Gateway"]
    end
    C1["Server Core 1"]
    C2["Server Core 2"]
    C3["weitere Server"]
    B -->|"HTTPS"| WAC
    WAC -->|"WinRM, TCP 5985"| C1
    WAC -->|"WinRM"| C2
    WAC -->|"WinRM"| C3
```

Wichtig zu verstehen: der Browser spricht nur mit dem Gateway. Das Gateway spricht mit den Servern. Es ist kein Agent auf den Zielsystemen, sondern eine Fernsteuerung über Bordmittel.

## Einrichtung

Das Setup ist ein normaler Installer mit zwei Entscheidungen:

- **Port.** Voreingestellt ist 6516. Man kann auch 443 nehmen, dann muss man beim Aufruf keine Portnummer eintippen. Voraussetzung ist, dass auf dem Server keine andere Anwendung schon auf 443 hört.
- **Zertifikat.** Im Lab reicht ein selbst signiertes Zertifikat, das der Installer erzeugt. Der Browser zeigt dann bei jedem Aufruf eine Warnung, weil niemand dieses Zertifikat bestätigt. In einer echten Umgebung würde man ein Zertifikat der internen Zertifizierungsstelle hinterlegen.

Bei der ersten Anmeldung bin ich in einen Fehler gelaufen, der bei Arbeitsgruppen typisch ist. Ich habe nur `adm_local` eingegeben und bekam "Zugriff verweigert". Windows sucht das Konto dann an der falschen Stelle. Richtig ist:

```
.\adm_local
```

Der Punkt mit Backslash bedeutet "auf diesem Computer". Alternativ schreibt man den Rechnernamen davor, also `SRV25-GUI\adm_local`.

Falls das Konto trotzdem abgewiesen wird, fehlt die Mitgliedschaft in der zuständigen Gruppe:

```powershell
Add-LocalGroupMember -Group "Windows Admin Center Administrators" -Member "adm_local"
```

## Server ohne Domäne anbinden

Meine Maschinen stehen in einer Arbeitsgruppe, nicht in einer Domäne. Deshalb gibt es keine gemeinsame Vertrauensstellung, und das Gateway darf sich nicht einfach bei einem anderen Server anmelden.

Die Lösung ist eine Liste vertrauenswürdiger Gegenstellen auf dem Gateway:

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "172.16.10.*" -Concatenate -Force
Get-Item WSMan:\localhost\Client\TrustedHosts
```

`-Concatenate` hängt den Eintrag an die bestehende Liste an, statt sie zu überschreiben. Ohne diesen Schalter löscht man sich die vorherigen Einträge weg, was ich beim ersten Mal genau so gemacht habe.

Beim Hinzufügen eines Servers muss man dann **Anmeldeinformationen separat angeben** wählen und wieder `.\adm_local` eintragen. Ohne diesen Haken versucht das Gateway, die eigene Anmeldung weiterzureichen, und das funktioniert in der Arbeitsgruppe nicht.

Sobald die Maschinen später in einer Domäne sind, fällt dieser ganze Teil weg. Dann übernimmt Kerberos die Vertrauensstellung und der Server wird einfach über seinen Namen hinzugefügt.

## Was man damit macht

Der praktische Nutzen war bei mir beim Speicher am grössten. Hier verwalte ich einen Core-Server komplett über den Browser und sehe eine Festplatte, die über das Netzwerk angebunden ist:

![Datentraegerverwaltung eines Core-Servers im Windows Admin Center](../images/iscsi-initiator-wac-core-datentraeger1-iscsi-data-f-10.png)

Auf einen Blick sichtbar: der verwaltete Server, die eingebundene Netzwerkplatte mit ihrem Typ, das fertige NTFS-Volume mit Grösse und freiem Platz. Auf einem Core-Server müsste ich diese Angaben sonst mit `Get-Disk`, `Get-Partition` und `Get-Volume` einzeln zusammensuchen und mir selbst zusammenreimen.

Weitere Bereiche, die ich regelmässig benutzt habe:

| Bereich | Wofür |
| :--- | :--- |
| Rollen und Features | Rollen nachinstallieren, ohne die genauen Featurenamen zu kennen |
| Dienste | Dienst starten, stoppen, Starttyp ändern |
| Ereignisanzeige | Fehler suchen, ohne die Protokollnamen auswendig zu wissen |
| PowerShell | eine Sitzung direkt im Browserfenster, ohne RDP |
| Dateien und Freigaben | SMB-Freigaben anlegen und Berechtigungen setzen |
| Netzwerk | Adressen und Adapter ansehen |

Der eingebaute PowerShell-Bereich ist dabei fast der nützlichste. Man arbeitet grafisch, findet die richtige Stelle, und wenn man dann doch einen Befehl braucht, ist die Konsole einen Klick entfernt.

## Wenn es nicht geht

Drei Fehler sind mir mehrfach begegnet:

| Symptom | Ursache | Lösung |
| :--- | :--- | :--- |
| Zugriff verweigert bei der Anmeldung | Konto ohne `.\` eingegeben oder nicht in der WAC-Gruppe | `.\adm_local` verwenden, Gruppenmitgliedschaft prüfen |
| WinRM-Fehler beim Hinzufügen eines Core-Servers | WinRM lauscht nicht oder Gegenstelle nicht in TrustedHosts | auf dem Zielserver `winrm quickconfig`, dann TrustedHosts prüfen |
| Zertifikatswarnung im Browser | selbst signiertes Zertifikat | im Lab über "Erweitert" fortfahren, produktiv ein richtiges Zertifikat hinterlegen |

Fast immer lag es bei mir an WinRM oder an der TrustedHosts-Liste, nicht am Windows Admin Center selbst.

## Was ich dabei gelernt habe

**Windows Admin Center ersetzt PowerShell nicht.** Alles, was man wiederholt oder auf mehreren Servern gleichzeitig macht, gehört in ein Skript. Die Oberfläche macht keine Schleifen und dokumentiert nichts.

**Wofür es wirklich hilft:** einen Server anschauen, den man nicht täglich benutzt. Vorher musste ich bei einem Core-Server raten, welcher Befehl mir den Zustand zeigt. Jetzt klicke ich mich einmal durch, sehe die Lage und weiss danach, wonach ich in PowerShell suchen muss. Für das Lernen ist das genau die richtige Reihenfolge.

**Der Gateway-Server ist eine kritische Stelle.** Er darf sich bei allen anderen Servern anmelden. Wer ihn übernimmt, hat Zugriff auf die ganze Umgebung. Deshalb gehört er abgesichert wie ein Verwaltungsserver, und deshalb war es sinnvoll, vorher das eingebaute Administratorkonto zu deaktivieren.
