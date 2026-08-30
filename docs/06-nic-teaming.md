# NIC-Teaming, Failover und Netzwerkbrücke

## Worum es geht

Ein Server mit einer Netzwerkkarte hat eine Stelle, an der alles hängt. Fällt die Karte, das Kabel oder der Switchport aus, ist der Dienst nicht mehr erreichbar, obwohl der Server selbst einwandfrei läuft.

NIC-Teaming fasst mehrere Karten zu einer logischen Karte zusammen. Windows nennt das LBFO, Load Balancing and Failover. Die IP-Adresse gehört danach dem Team, nicht mehr den einzelnen Karten. Fällt eine aus, merken die Clients davon nichts.

Ich habe der VM zehn virtuelle Karten gegeben und daraus drei Gruppen gebaut: ein Team mit Reservekarte, ein Team mit allen Karten aktiv und eine Netzwerkbrücke.

```mermaid
flowchart TD
    subgraph HV["Proxmox: 10 virtuelle Karten"]
        N1["net0 - net3"]
        N2["net4 - net7"]
        N3["net8 - net9"]
    end
    subgraph WIN["Windows Server"]
        subgraph G1["Gruppe A: aktiv / standby"]
            T0["Team 0<br/>3 aktiv, 1 in Reserve"]
        end
        subgraph G2["Gruppe B: aktiv / aktiv"]
            T2["Team 2<br/>4 aktiv"]
        end
        subgraph G3["Gruppe C"]
            BR["Netzwerkbruecke<br/>2 Karten"]
        end
    end
    N1 --> T0
    N2 --> T2
    N3 --> BR
```

Ein Hinweis zu den Zahlen, damit sie richtig eingeordnet werden: das sind virtuelle Karten. Die angezeigten 30 oder 40 GBit/s sind das, was Windows aus mehreren 10-GBit/s-Karten zusammenrechnet. Echte Hardware mit dieser Bandbreite habe ich nicht. Testen konnte ich das **Verhalten** bei Ausfällen, und darum ging es mir.

![Zehn virtuelle Netzwerkkarten in der VM-Konfiguration](../images/nic-teaming-proxmox-10-virtio-nics-26.png)

![Alle Adapter in den Netzwerkverbindungen](../images/nic-teaming-ncpa-10-adapters-overview-27.png)

## Zuerst das Problem erzeugen

Bevor ich ein Team gebaut habe, wollte ich den Ausfall selbst sehen. Jede Karte hatte eine eigene Adresse: die erste fest, die übrigen per DHCP.

Auf dem Server eine SMB-Freigabe eingerichtet:

![Konfiguration der SMB-Freigabe auf dem Server](../images/nic-teaming-server-smb-freigabe-config-14.png)

Vom Client aus erreichbar:

![Zugriff des Clients auf die Freigabe](../images/nic-teaming-client-smb-share-access-21-13.png)

Dann die primäre Karte deaktiviert, was einem Kabelbruch entspricht:

![Deaktivierung der primaeren Netzwerkkarte](../images/nic-teaming-ncpa-lan1-deaktiviert-16.png)

Das Ergebnis kam sofort:

![Fehlermeldung beim Zugriff nach dem Ausfall](../images/nic-teaming-spof-netzwerkfehler-dialog-15.png)

Der Server lief weiter, drei weitere Karten waren online, aber der Client kam nicht mehr an seine Freigabe. Über die DHCP-Adresse einer anderen Karte wäre der Zugriff technisch möglich gewesen. Nur: jeder Benutzer müsste diese Adresse kennen, und nach einem Neustart wäre sie vielleicht eine andere.

Das ist der Punkt, an dem Teaming ansetzt. Nicht die Kabel sind das Problem, sondern dass die Adresse an einer einzelnen Karte hängt.

## Team 1: eine Karte in Reserve

Vier Karten, davon eine als Standby.

![Anlegen des Teams mit Standby-Adapter](../images/nic-teaming-neues-team-standby-config-18.png)

Zwei Einstellungen sind hier entscheidend.

**Teammodus.** Ich habe **Switchunabhängig** gewählt. Der Switch weiss dann nichts vom Team und muss nicht konfiguriert werden. Der Vorteil: die Karten dürfen sogar an verschiedenen Switches hängen, was gegen den Ausfall eines ganzen Switches schützt. Die Alternative wäre LACP, das aber eine passende Konfiguration auf dem Switch verlangt.

**Lastenausgleich.** Hier musste ich von der Empfehlung abweichen. Windows schlägt normalerweise **Dynamisch** vor. In einer VM funktioniert das nicht, weil dem Gastsystem die nötige Hardwareunterstützung fehlt, und Windows quittiert es mit einem Fehler. In virtuellen Umgebungen nimmt man **Adresshash**: aus IP-Adressen und Ports wird ein Hashwert gebildet, der die Karte bestimmt. Das war der erste Fehler, in den ich gelaufen bin.

Nach dem Anlegen zeigt die Übersicht die Rollenverteilung, eine Karte steht auf Standby:

![Uebersicht des Teams mit Standby-Adapter](../images/nic-teaming-teamvorgang-standby-overview-19.png)

![Statusfenster des ersten Teams](../images/nic-teaming-ncpa-team0-status-30gbps-21.png)

Die Standby-Karte trägt im Normalbetrieb keine Daten. Deshalb zeigt das Team die Summe von drei Karten, nicht von vier. Fällt eine aktive Karte aus, springt die Reserve ein und die Bandbreite bleibt gleich.

## Team 2: alle Karten aktiv

Beim zweiten Team gibt es keinen Standby, alle vier tragen Daten:

![Konfiguration des zweiten Teams ohne Standby](../images/nic-teaming-team2-active-active-config-28.png)

![Statusfenster des zweiten Teams](../images/nic-teaming-ncpa-team2-status-40gbps-29.png)

Auch hier habe ich im laufenden Betrieb eine Karte deaktiviert. Das Team ist nicht ausgefallen, die verbleibenden drei haben weitergemacht, nur die angezeigte Bandbreite sank entsprechend.

Der Unterschied zwischen den beiden Modellen ist eine Abwägung:

| | mit Reserve | alle aktiv |
| :--- | :--- | :--- |
| Bandbreite im Normalbetrieb | geringer, eine Karte wartet | maximal |
| Bandbreite nach einem Ausfall | unverändert | sinkt |
| Sinnvoll bei | gleichbleibender Leistung, auch im Fehlerfall | maximalem Durchsatz |

Die IP-Konfiguration des zweiten Teams, bewusst **ohne Standardgateway**:

![IPv4-Konfiguration des zweiten Teams](../images/nic-teaming-ncpa-team2-ipv4-config-30.png)

Windows hat mich beim Eintragen sogar gewarnt, dass im selben Subnetz bereits ein Standardgateway existiert. Zwei Standardgateways im selben Netz führen zu widersprüchlichen Routen, und der Rechner entscheidet dann anhand von Metriken, was schwer nachvollziehbar wird. Ein Gateway pro System, alles andere nur mit Adresse.

![Beide Teams nebeneinander in der Serververwaltung](../images/nic-teaming-server-manager-team0-team2-overview-31.png)

## Zwei Teams im selben Netz

Das zweite Team bekam die Adresse `172.16.10.45`, also dasselbe Subnetz wie das erste mit `172.16.10.21`. Der Client konnte das erste Team anpingen, das zweite aber nicht, obwohl beide online waren und die Konfiguration richtig aussah.

Der Grund ist das **Strong-Host-Modell**, das Windows standardmässig verwendet:

```mermaid
sequenceDiagram
    participant C as Client
    participant T2 as Team 2
    participant T0 as Team 0
    C->>T2: Ping an die Adresse von Team 2
    Note over T2: Anfrage kommt an
    T2->>T0: Antwort soll ueber das Gateway raus,<br/>das liegt an Team 0
    Note over T0: Strong Host: Antwort mit fremder<br/>Absenderadresse wird verworfen
    Note over C: keine Antwort
```

Beim Strong-Host-Modell gehört eine Adresse fest zu einer Schnittstelle. Ein Paket darf nur über die Schnittstelle hinausgehen, zu der seine Absenderadresse gehört. Die Antwort von Team 2 will aber den Weg über das Gateway nehmen, das an Team 1 hängt, und wird deshalb verworfen.

Das **Weak-Host-Modell** erlaubt genau das. Umgestellt wird es pro Schnittstelle:

```powershell
Set-NetIPInterface -InterfaceAlias "NIC-Team2" -WeakHostSend Enabled -WeakHostReceive Enabled
Set-NetIPInterface -InterfaceAlias "NIC-Team0" -WeakHostSend Enabled -WeakHostReceive Enabled
```

Danach war beides erreichbar:

![Erfolgreicher Ping auf beide Teams](../images/nic-teaming-client-ping-team0-team2-success-32.png)

Auf dem Client musste ich noch `arp -d *` ausführen, weil dort die alten Einträge im Zwischenspeicher standen und der Rechner weiterhin die alte Zuordnung benutzte.

Erwähnenswert: Strong Host ist die sicherere Voreinstellung, weil sie bestimmte Angriffe über gefälschte Absenderadressen erschwert. Man schaltet sie nicht ohne Grund ab. In einem Testnetz mit zwei Teams im selben Subnetz ist es vertretbar, in einem echten Netz würde man die Teams eher in getrennte Subnetze legen.

## Wie das Team technisch funktioniert

Die IP-Adresse gehört jetzt dem Team:

![IPv4-Konfiguration des ersten Teams](../images/nic-teaming-ncpa-nic-team0-ipv4-config-22.png)

Interessanter ist, was mit den Mitgliedskarten passiert. In deren Eigenschaften ist IPv4 nicht mehr aktiviert, stattdessen steht dort das **Multiplexorprotokoll**:

![Eigenschaften einer Mitgliedskarte mit Multiplexorprotokoll](../images/nic-teaming-ncpa-lan1-multiplexor-protocol-24.png)

Der Multiplexor ist ein Treiber, der sich zwischen die physischen Karten und den Netzwerkstapel legt. Nach aussen meldet er eine einzige Karte. Die physischen Karten sind nur noch Transportwege ohne eigene Identität.

Dasselbe von der Kommandozeile aus:

![ipconfig-Ausgabe des Team-Adapters](../images/nic-teaming-ipconfig-nic-team0-multiplexor-23.png)

Die einzelnen Karten tauchen nicht mehr mit eigener Adresse auf, nur der Team-Adapter mit einer IP und einem Gateway.

## Ein Fehler, den ich fast übersehen hätte

Nach dem Anlegen des ersten Teams blieb eine ungültige zweite Standardroute mit dem Ziel `0.0.0.0` im System zurück, ein Rest der alten Konfiguration.

Das Tückische war, dass zunächst alles zu funktionieren schien. Der Ping auf den Router im selben Subnetz ging durch. Weg war nur alles andere: kein Internet, keine RDP-Verbindung aus einem anderen Subnetz.

Der Grund ist, dass ein Ziel im eigenen Subnetz direkt über ARP erreicht wird. Die Routingtabelle wird dafür gar nicht befragt. Ein erfolgreicher Ping auf den eigenen Router beweist deshalb nur, dass ARP funktioniert, und sagt nichts über das Routing.

Geprüft habe ich es so:

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0" |
    Format-Table ifIndex,InterfaceAlias,NextHop,RouteMetric -AutoSize
```

Dort darf nur eine Zeile stehen. Bei mir waren es zwei, eine davon mit `0.0.0.0` als nächstem Ziel. Sauber bekommen habe ich es, indem ich die Adresse in einem Rutsch neu gesetzt habe:

```cmd
netsh int ip set address "NIC-Team0" static 172.16.10.21 255.255.255.0 172.16.10.1
```

## Netzwerkbrücke

Aus den letzten beiden Karten habe ich eine Brücke gebaut. Man markiert beide Karten und wählt **Verbindungen überbrücken**:

![Aktion Verbindungen ueberbruecken](../images/nic-teaming-ncpa-verbindungen-ueberbruecken-action-33.png)

Es entsteht ein neuer Adapter namens Netzwerkbrücke:

![Status der Netzwerkbruecke](../images/nic-teaming-ncpa-netzwerkbruecke-status-20gbps-34.png)

![IPv4-Konfiguration der Netzwerkbruecke](../images/nic-teaming-ncpa-netzwerkbruecke-ipv4-config-35.png)

Die Brücke bekam `172.16.10.46`, wieder ohne Standardgateway. Damit hat der Server jetzt drei Adressen im selben Subnetz: `.21` am ersten Team, `.45` am zweiten und `.46` an der Brücke.

Auch hier trat das Strong-Host-Problem wieder auf, und die Lösung war dieselbe:

```powershell
Set-NetIPInterface -InterfaceAlias "*brücke*" -WeakHostSend Enabled -WeakHostReceive Enabled
Get-NetConnectionProfile -InterfaceAlias "*brücke*" | Set-NetConnectionProfile -NetworkCategory Private
```

Das Netzwerkprofil auf **Privat** zu setzen war nötig, weil die Firewall im öffentlichen Profil eingehende Pings blockiert.

![Erfolgreicher Ping auf die Netzwerkbruecke](../images/nic-teaming-client-ping-bridge-success-36.png)

### Brücke ist nicht Teaming

Das Ergebnis sieht ähnlich aus, der Zweck ist ein völlig anderer.

| | Teaming | Brücke |
| :--- | :--- | :--- |
| Zweck | Ausfallsicherheit für den Server selbst | zwei Netzsegmente verbinden |
| Rolle des Servers | Endgerät | Weiterleitung, wie ein kleiner Switch |
| Lastverteilung | über Hashwerte, gezielt | keine, es wird nur weitergeleitet |
| Verschiedene Medien | nein, nur gleiches Ethernet | ja, etwa LAN und WLAN |
| CPU-Last | sehr gering, im Treiber | höher, jedes Paket wird geprüft |
| Schleifengefahr | keine | vorhanden |

Der letzte Punkt ist der wichtigste. Wer zwei Karten im selben VLAN überbrückt und im Netz kein Spanning Tree hat, baut eine Schleife. Broadcasts laufen dann im Kreis und können das Segment lahmlegen. Eine Brücke gehört zwischen **verschiedene** Segmente.

## Vergleich mit Cisco

Aus dem Netzwerkunterricht kannte ich EtherChannel und LACP. Der Vergleich hat mir geholfen, das Windows-Teaming einzuordnen:

| | Cisco EtherChannel / LACP | Windows LBFO, switchunabhängig |
| :--- | :--- | :--- |
| Wo konfiguriert | auf dem Switch | im Betriebssystem |
| Switch muss mitspielen | ja, zwingend | nein |
| Karten an verschiedenen Switches | nur mit Stacking | ohne Weiteres möglich |
| Aushandlung | über LACP-Pakete | keine, der Switch merkt nichts |

Beides erreicht dasselbe Ziel auf verschiedenen Ebenen. Der switchunabhängige Modus ist flexibler und einfacher, LACP kann dafür den eingehenden Verkehr besser verteilen, weil beide Seiten Bescheid wissen.

## Test und Verifikation

Beim Netzwerk reicht "es geht" nicht, weil ein Teil funktionieren kann, während ein anderer stillschweigend nicht funktioniert. Genau das ist mir mit der `0.0.0.0`-Route passiert. Deshalb prüfe ich in vier Stufen. Ausgaben von `SRV25-GUI`.

### Stufe 1: Stimmt der Aufbau der Teams?

```powershell
Get-NetLbfoTeam | Select-Object Name, Members, TeamingMode, LoadBalancingAlgorithm, Status
```

```
Name      Members                     TeamingMode       LoadBalancingAlgorithm Status
----      -------                     -----------       ---------------------- ------
NIC-Team0 {LAN1, LAN2, LAN3, LAN4}    SwitchIndependent AddressHash            Up
NIC-Team2 {Ether1, Ether2, Ether3,    SwitchIndependent AddressHash            Up
           Ether4}
```

Die Rollenverteilung innerhalb eines Teams ist eine eigene Abfrage:

```powershell
Get-NetLbfoTeamMember -Team "NIC-Team0" |
    Select-Object Name, AdministrativeMode, OperationalStatus
```

```
Name AdministrativeMode OperationalStatus
---- ------------------ -----------------
LAN1 Standby            Standby
LAN2 Active             Active
LAN3 Active             Active
LAN4 Active             Active
```

Hier sieht man den Unterschied zwischen den beiden Modellen als Zahl: drei aktive Karten bei Team 0, vier bei Team 2.

### Stufe 2: Wer hat welche Adresse?

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object InterfaceAlias -match "Team|brücke" |
    Select-Object InterfaceAlias, IPAddress, PrefixLength
```

```
InterfaceAlias  IPAddress     PrefixLength
--------------  ---------     ------------
NIC-Team0       172.16.10.21            24
NIC-Team2       172.16.10.45            24
Netzwerkbrücke  172.16.10.46            24
```

Die zehn physischen Karten tauchen hier bewusst **nicht** auf. Sie haben keine eigene Adresse mehr, das ist der Multiplexor in einer Ausgabe.

### Stufe 3: Ist das Routing sauber?

Das ist die Prüfung, die ich beim ersten Mal übersprungen habe:

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0" |
    Format-Table ifIndex, InterfaceAlias, NextHop, RouteMetric -AutoSize
```

Fehlerhafter Zustand nach dem Anlegen des Teams:

```
ifIndex InterfaceAlias NextHop      RouteMetric
------- -------------- -------      -----------
     22 NIC-Team0      172.16.10.1          256
     14 LAN1           0.0.0.0              256
```

Richtiger Zustand nach der Bereinigung:

```
ifIndex InterfaceAlias NextHop      RouteMetric
------- -------------- -------      -----------
     22 NIC-Team0      172.16.10.1          256
```

Es darf genau eine Zeile geben. Eine zweite mit `0.0.0.0` als nächstem Ziel ist ein Rest der alten Konfiguration und sorgt dafür, dass Ziele ausserhalb des Subnetzes nicht mehr erreichbar sind.

### Stufe 4: Erreichbarkeit von aussen, in der richtigen Reihenfolge

Vom Client aus, jede Zeile prüft etwas anderes:

```powershell
Test-NetConnection -ComputerName 172.16.10.21   # Team 0, Hauptadresse
Test-NetConnection -ComputerName 172.16.10.45   # Team 2
Test-NetConnection -ComputerName 172.16.10.46   # Netzwerkbrücke
Test-NetConnection -ComputerName 172.16.10.1    # Router, Ziel im selben Subnetz
Test-NetConnection -ComputerName 1.1.1.1        # Ziel ausserhalb, prueft das Routing
```

Die letzte Zeile ist die entscheidende. Ein Ziel im eigenen Subnetz wird über ARP erreicht und braucht die Routingtabelle gar nicht. Erst ein Ziel ausserhalb beweist, dass die Standardroute stimmt. Genau deshalb sah bei mir lange alles gesund aus, obwohl es das nicht war.

Vor dem Weak-Host-Umbau antwortete nur `172.16.10.21`, danach alle drei Adressen. Der Screenshot dazu steht weiter oben im Abschnitt über die beiden Teams.

### Der Failover-Test selbst

```powershell
# Eine aktive Karte abschalten, waehrend ein Dauerping laeuft
Disable-NetAdapter -Name "LAN2" -Confirm:$false
Get-NetLbfoTeamMember -Team "NIC-Team0" |
    Select-Object Name, AdministrativeMode, OperationalStatus
```

```
Name AdministrativeMode OperationalStatus
---- ------------------ -----------------
LAN1 Standby            Active
LAN2 Active             Failed
LAN3 Active             Active
LAN4 Active             Active
```

Die Reservekarte steht weiterhin auf `Standby` als eingestellter Modus, ihr Betriebszustand ist aber auf `Active` gewechselt. Der Dauerping lief ohne Unterbrechung weiter, die Adresse `172.16.10.21` blieb erreichbar. Das ist der ganze Zweck der Übung.

## Automatisierung

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, LinkSpeed, Status

New-NetLbfoTeam -Name "NIC-Team0" -TeamMembers "LAN1","LAN2","LAN3","LAN4" `
    -TeamingMode SwitchIndependent -LoadBalancingAlgorithm AddressHash -Confirm:$false

Set-NetLbfoTeamMember -Name "LAN1" -AdministrativeMode Standby

New-NetIPAddress -InterfaceAlias "NIC-Team0" -IPAddress "172.16.10.21" `
    -PrefixLength 24 -DefaultGateway "172.16.10.1"

Get-NetLbfoTeam
```

## Fehlersuche

| Symptom | Ursache | Lösung |
| :--- | :--- | :--- |
| Team lässt sich nicht anlegen, Fehler beim Lastenausgleich | `Dynamisch` in einer VM nicht unterstützt | `AddressHash` verwenden |
| Ping auf den Router geht, Internet nicht | zweite Standardroute mit `0.0.0.0` | `Get-NetRoute -DestinationPrefix "0.0.0.0/0"`, danach `netsh int ip set address` |
| Zweites Team antwortet nicht auf Ping | Strong-Host-Modell | `Set-NetIPInterface -WeakHostSend Enabled -WeakHostReceive Enabled` |
| Nach dem Umstellen weiterhin keine Antwort | alter Eintrag im ARP-Zwischenspeicher des Clients | auf dem Client `arp -d *` |
| Netzwerkbrücke antwortet nicht | Firewallprofil steht auf Öffentlich | `Set-NetConnectionProfile -NetworkCategory Private` |
| Mitgliedskarte hat noch eine eigene IP | Karte war vor dem Team konfiguriert | Adresse an der Karte entfernen, sie gehört dem Team |
| Bandbreite niedriger als erwartet | eine Karte steht auf Standby | `Get-NetLbfoTeamMember` prüfen |
| Verbindung bricht bei Kartenausfall ab | Team im falschen Modus oder Switch erwartet LACP | `SwitchIndependent` prüfen |
| Broadcast-Sturm nach dem Überbrücken | beide Karten im selben VLAN, kein Spanning Tree | Brücke auflösen, nur getrennte Segmente überbrücken |

## Begriffe

| Deutsch | Englisch | Bedeutung |
| :--- | :--- | :--- |
| NIC-Teaming | LBFO | mehrere Karten als eine logische Karte |
| switchunabhängiger Modus | Switch Independent | Team ohne Konfiguration am Switch |
| Adresshash | Address Hash | Verteilung über einen Hashwert aus Adressen und Ports |
| Standby-Adapter | Standby Adapter | Reservekarte, trägt im Normalbetrieb keine Daten |
| Multiplexortreiber | Multiplexor Driver | Treiber, der die Karten zusammenfasst |
| Netzwerkbrücke | Network Bridge | Verbindung zweier Segmente auf Layer 2 |
| Multihoming | Multihoming | mehrere Adressen oder Schnittstellen an einem System |
| Strong- / Weak-Host-Modell | Strong / Weak Host | ob eine Antwort die Schnittstelle wechseln darf |

## Was ich dabei gelernt habe

**Das Teaming selbst war der unproblematischste Teil.** Es hat den Ausfall genau so aufgefangen, wie es soll. Die Zeit ging für zwei Dinge drauf, die nichts mit Teaming zu tun hatten.

**Ein erfolgreicher Ping beweist wenig.** Die Sache mit der `0.0.0.0`-Route hat mir das deutlich gezeigt. Ein Ping auf ein Gerät im eigenen Subnetz sagt nur, dass ARP funktioniert. Ob das Routing stimmt, sieht man erst bei einem Ziel ausserhalb. Seitdem teste ich Netzwerkprobleme immer in dieser Reihenfolge: erst im eigenen Subnetz, dann ausserhalb, dann mit Namen statt Adressen.

**Strong Host gegen Weak Host hätte ich sonst nie verstanden.** Solange ein Server nur eine Adresse hat, merkt man davon nichts. Erst mit zwei Teams im selben Netz wird sichtbar, dass Windows Adressen fest an Schnittstellen bindet.

**Empfehlungen gelten nicht überall.** Der Lastenausgleich **Dynamisch** ist auf physischer Hardware richtig und in einer VM schlicht nicht verfügbar. Bei einer Empfehlung lohnt die Frage, unter welchen Voraussetzungen sie gilt.
