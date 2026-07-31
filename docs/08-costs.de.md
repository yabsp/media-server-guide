# Kostenanalyse: Eigenbetrieb vs. Streaming

Ist es tatsächlich günstiger, einen eigenen Media-Server zu bauen und zu
betreiben, als einfach für Streaming-Abos zu bezahlen? Dieses Kapitel
dokumentiert die realen Kosten dieses Aufbaus und vergleicht sie mit den
Streaming-Diensten, die nötig wären, um ihn zu ersetzen.

Das Referenzszenario ist ein Haushalt, in dem **bis zu fünf Personen
gleichzeitig streamen** möchten. Alle Beträge sind in **Schweizer Franken
(CHF)**.

!!! note "Preisstand"
    Die Streaming-Preise wurden zuletzt im **Juli 2026** anhand aktueller
    Schweizer Endkundenpreise überprüft. Anbieter ändern ihre Pläne und Preise
    häufig; betrachte diese Zahlen daher als Momentaufnahme und nicht als feste
    Wahrheit.

## Selbst gehosteter Media-Server

### Einmalige Kosten

| Position | Kosten (CHF) |
| --- | ---: |
| 2× Seagate IronWolf 12 TB (April 2025) | 492.00 |
| Seagate IronWolf 4 TB (August 2024) | 99.00 |
| Media-Server-Eigenbau (Jan. 2026) | 577.60 |
| Plex Pass (Lifetime, Dezember 2024 + Test-Monatsabos aus den Monaten davor) | 108.63 |
| Englischer Indexer | 40.00 |
| **Total** | **1 317.23** |

### Wiederkehrende monatliche Kosten

| Position | Kosten (CHF) |
| --- | ---: |
| Deutscher Indexer (CHF 15/Jahr) | 1.25 |
| Usenet-Zugang | 5.78 |
| VPS | 3.25 |
| Strom (≈ 60 W → 31 kWh × CHF 0.33/kWh) ¹ | 10.23 |
| **Total** | **20.51** |

¹ Dies wurde über einen Zeitraum von mehr als einem Monat gemessen. In diesem Zeitraum hatte der Server wenig Leerlaufzeit, weshalb die Messung als Worst-Case-Schätzung betrachtet werden kann.

## Streaming-Alternative

Um die Inhalte und die Anforderung von fünf gleichzeitigen Streams abzudecken,
wären die folgenden Abos nötig. Netflix und Disney+ erreichen fünf gleichzeitige
Streams nur, indem zusätzlich zu ihren Premium-Plänen mit vier Streams ein
kostenpflichtiges **Zusatzmitglied** hinzugefügt wird; die übrigen Dienste
kommen auf maximal drei bis vier Streams.

| Dienst | Abo (für bis zu 5 gleichzeitige Streams) | Monatlich (CHF) |
| --- | --- | ---: |
| Netflix | Premium (4 Streams) + 1 Zusatzmitglied | 30.80 |
| Disney+ | Premium (4 Streams) + 1 Zusatzmitglied | 28.80 |
| Prime Video | Standard (3 Streams) | 9.99 |
| Sky Show | Premium (4 Streams) | 17.90 ² |
| Paramount+ | Premium (4 Streams) | 17.90 |
| **Total** | | **105.39** |

² Sky Show Premium sinkt nach den ersten sechs Monaten um CHF 4.00 auf
**CHF 13.90/Monat**, wodurch sich das Total danach auf **CHF 101.39/Monat**
reduziert.

## Break-Even-Analyse

Die einmalige Investition von CHF 1 317.23 amortisiert sich, sobald das für
Streaming *nicht* ausgegebene Geld die laufenden Kosten des Servers übersteigt.
Wie lange das dauert, hängt davon ab, wie viele Streaming-Dienste der Server
tatsächlich ersetzt.

| Ersetzte Streaming-Dienste | Break-Even |
| --- | ---: |
| Nur Netflix | ≈ 10,7 Jahre |
| Netflix + Disney+ | ≈ 2,8 Jahre |
| Alle fünf Dienste | ≈ 1,3 Jahre |

Mit anderen Worten: Ersetzt der Server nur ein einziges Netflix-Abo, sind die
Einsparungen marginal. Als Ersatz für das gesamte Paket an Diensten, das nötig
ist, um fünf gleichzeitige Zuschauer zu bedienen, amortisiert er sich jedoch in
etwas mehr als einem Jahr.

## Effektive monatliche Kosten nach Laufzeit

Die Amortisation der einmaligen Investition über die Lebensdauer des Servers
ergibt seine tatsächlichen effektiven monatlichen Kosten (einmalige Kosten
verteilt auf die Laufzeit plus die wiederkehrenden monatlichen Kosten). Je länger
er läuft, desto günstiger wird er:

| Laufzeit | Effektive monatliche Kosten (CHF) |
| --- | ---: |
| 2 Jahre | 75.39 |
| 3 Jahre | 57.10 |
| 5 Jahre | 42.46 |
| 7 Jahre | 36.19 |

Selbst bei einer pessimistischen Lebensdauer von zwei Jahren bleiben die
effektiven monatlichen Kosten unter der Streaming-Rechnung von CHF 105.39; über
fünf Jahre sinken sie auf deutlich weniger als die Hälfte.

## Zusätzlicher Kommentar

Wir müssen hier erwähnen, dass sich die Preise der Komponenten und des Plex-Abos
geändert haben. Die Komponentenpreise müssen hingenommen werden, das Plex-Abo ist
jedoch nicht nötig, da es mit [Jellyfin](https://jellyfin.org/) eine Alternative
gibt. Was wir aber im Hinterkopf behalten müssen: Wenn du einen Server mit einer
guten CPU und genügend schnellem Arbeitsspeicher baust, sind weit mehr als 5
gleichzeitige Streams und auch 4K-Streams möglich – all diese Funktionen müssten
bei Streaming-Diensten zusätzlich bezahlt werden. Die Hauptinvestition ist deine
Zeit, aber sobald der Server läuft und automatisiert ist, gibt es für dich nicht
mehr viel zu tun.
