---
marp: true
theme: dracula
---

Upside:
* Convenience of all in one spot
* Own your Data
* price

Downside:
* Initall Cost
* Maintenance
* Setup
---
#humoros first page
---
#picture of a dude in the left corner with an arm to his chin looking back at the right side of the image with the logos of Netflix, Amazon Prime, Hulu, HBO and Disney+    
<!-- Ich nehme hier streaming als Beispiel. Als netflix erschien war es ein Durchbruch und leutet eine neue Ära ein wie wir Filme und Serien Konsumieren. Es zeigte sich das viele die bis anhin auf illegales streaming oder Torrenting seiten setzen dies vor allem aus bequemlichkeit taten. 
Netflix wuchs und die Konsumenten liebten es, bequem einfach gute inhalte alle an einem einzigen ort für einen vernünftigen preis. Wie wir wissen blieb das nicht lange so und andere wollten ein Stück vom kuchen ab haben. Das einzige bequeme one stop model für filme wurde zu einer zerstückelten landschaft in der man etwa 4-5 verschiedene Abos Jonglieren muss um auf alle Serien und Inhalte zugreifen zu können die man möchte. Nicht nur ist dies sehr teuer sondern auch nervig auf dem laufenden zu bleiben welche serie gerade wo verfügbar ist. Die lösung, eine eigene Zentralisierte Kurierte Medienbibliothek mit eigenenm Streaming -->
--- 
# bild von jellyfin und plex
## link zum eigenen streaming
<!-- Jellyfin ist ein Kostenloser Open source medienserver der ermöglicht filme, serien, bücher und musik selbst zu verwalten und diese auch freunden und Familie zur verfügung zu stellen. Keine 5 Services und Abos mehr alles in einer Platform. -->
---
# Bild von workflop, prowlar -> sonar -> deluge -> jellyfin

<!-- Erkläre workflow -->
---
# Bild von Owncloud und Paperless
## link zu demo von paperless 
## link zu demo von owncloud
<!-- erkläre die services.... Dies sind nur einige beispiele von diensten welche man bequem selber hosten kann. Was jedoch die meisten hier interesieren dürfte, was kostet das ganze und wieviel aufwand ist dies. -->
---
<!-- Ich werde hier ein kleines setup erklären das ähnlich dem meinigen ist. -->
# Beginner setup preis: 
* Pc mit einem i7 4770k  und 16gb ddr3 (ca 11J alt) 200.-
* Festplatten 2Stk WD RED PLUS 4tb = 220.- (digitec)

Total : 420.- to get started

Advanced:
* 2 Pc's 400.-
* 8 Festplatten 870.-
* USV Eaton Ellipse ECO 800 150.-
Total:1420

---
# Betriebskosten Server
* Internet abo: (u got that anyhow)
* Strom: 60W/h * 24 * 365 / 1000(kW/h) * 0.329(ewz 2024 WTF!) = 172.-

<!-- https://www.tagesanzeiger.ch/stromkrise-schlaegt-auf-die-preise-durch-strom-im-kanton-zuerich-wird-teurer-ueber-250-franken-im-jahr-366164551232 -->
---
## Preise Streaming:
<!-- 
Netflix Basic: 12.9/m
Netflix Premium(4k): 27.9/m
Amazon Prime: 9.99/m
Disney+ Standart: 12.9/m
Disney+ Premium(4k): 17.9/m
Apple TV: 10.9/m -->
| Service                      | Price (per month) |
|------------------------------|--------------------|
| Netflix Premium (4K) / Basic | 27.90 / 12.90 CHF |
| Amazon Prime                 | 9.99 CHF          |
| Disney+ Premium (4K) / Standard | 17.90 / 12.90 CHF |
| Apple TV                     | 10.90 CHF         |

4K = 12*(27.9+9.99+17.9+10.9) = 800 / y
basic = 12*(12.9+9.99+12.9+10.9) = 560 / y

---
# Preise Cloud 500gb
<!-- Proton unlimited black firday: 6.5
Proton regular: 9.99
Swisscom cloud hat nur 2tb oder 250gb: 9.9 (2tb)
Mega pro: 7.78 (2tb) -->

| Service                           | Price (per month) | Storage      |
|-----------------------------------|-------------------|--------------|
| Proton Unlimited (Black Friday)  | $6.50             | 500GB        |
| Proton Regular                   | $9.99             | 500GB        |
| Swisscom Cloud                   | $9.90             | 2TB          |
| Mega Pro                         | $7.78             | 2TB          |


AVG: (6.5+9.99+9.9+7.78) / 4 = 8.5 / m
Yearly: 12*8.5 = __102__
---
# Vergleich
* kosten y1
* kosten y2
* kosten y3

<!-- Ich werde jetzt erklären wie man solch ein setup bequem mit docker erstellen kann. -->





