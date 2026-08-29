# Windows Admin Center für die Verwaltung von Server Core

## Warum ich das gemacht habe

In meinem Lab laufen zwei Server ohne grafische Oberfläche. Das ist realistisch, denn Server Core braucht weniger Ressourcen und hat weniger Angriffsfläche. Für mich als Anfänger war es aber am Anfang unbequem: alles läuft über PowerShell, und man muss die Befehle kennen.

Windows Admin Center ist der Mittelweg. Es wird auf einem Server installiert und ist danach über den Browser erreichbar. Von dort verwaltet man die anderen Server mit, auch die ohne Oberfläche. Auf den verwalteten Servern muss man nichts installieren, die Verbindung läuft über WinRM.

## Einrichtung

Das Setup selbst ist ein normaler Installer. Zwei Punkte muss man entscheiden:

- **Port:** Standard ist 6516, man kann auch 443 nehmen.
- **Zertifikat:** Im Lab reicht ein selbst signiertes Zertifikat. Der Browser meckert dann bei jedem Aufruf, das ist normal und in einer echten Umgebung würde man ein richtiges Zertifikat verwenden.

Bei der Anmeldung habe ich zuerst einen Fehler gemacht. Ich habe nur `adm_local` eingegeben und bekam "Zugriff verweigert". Der Server sucht das Konto dann in der falschen Ebene. Richtig ist `.\adm_local` mit Punkt und Backslash davor, damit ist das lokale Konto gemeint.

## Server Core anbinden

Meine Server sind noch nicht in einer Domäne, sie stehen in einer Arbeitsgruppe. Deshalb vertraut der Gateway den anderen Maschinen nicht automatisch. Man muss sie vorher eintragen:

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "172.16.10.*" -Concatenate -Force
```

Danach im Browser den Server hinzufügen, "Anmeldedaten separat angeben" auswählen und wieder `.\adm_local` verwenden.

Später, wenn die Maschinen in einer Domäne sind, fällt dieser Schritt weg, weil Kerberos die Vertrauensstellung übernimmt.

## Was ich damit gemacht habe

Der praktische Nutzen war für mich beim Storage am grössten. Auf dem folgenden Bild verwalte ich einen Core-Server komplett über den Browser und sehe eine iSCSI-Festplatte, die über das Netzwerk angebunden ist:

![Datenträgerverwaltung eines Core-Servers im Windows Admin Center](../images/wac-01-core-datentraeger.png)

Man sieht den verwalteten Server, die eingebundene Netzwerkplatte und das fertige NTFS-Volume mit Grösse und freiem Platz. Das ist genau die Information, die ich sonst mit mehreren PowerShell-Befehlen einzeln zusammensuchen müsste.

Ausserdem nützlich im Alltag:

- Rollen und Features nachinstallieren, ohne `Install-WindowsFeature` auswendig zu können
- Dienste starten und stoppen
- Ereignisanzeige lesen
- eine PowerShell-Sitzung direkt im Browserfenster öffnen

## Was ich dabei gelernt habe

Windows Admin Center ersetzt PowerShell nicht, und das sollte es auch nicht. Für alles, was man wiederholt oder für mehrere Server gleichzeitig macht, ist ein Skript besser.

Wofür es aber wirklich hilft: einen Server anschauen, den man nicht täglich benutzt. Bei einem Core-Server ohne Oberfläche musste ich früher raten, welchen Befehl ich brauche. Jetzt klicke ich mich einmal durch, sehe den Zustand und weiss danach, wonach ich in PowerShell suchen muss.

Der häufigste Fehler bei mir war übrigens nicht die Software, sondern WinRM. Wenn ein Core-Server sich nicht verbinden liess, half fast immer `winrm quickconfig` auf der Gegenseite und ein Blick in die TrustedHosts-Liste.
