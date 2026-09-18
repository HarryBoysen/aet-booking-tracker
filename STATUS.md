[STATUS.md](https://github.com/user-attachments/files/32381109/STATUS.md)
[STATUS.md](https://github.com/user-attachments/files/32335990/STATUS.md)
# AET Booking Tracker – STATUS

Sidst opdateret: 2026-09-18

## Hvad virker
- Bookinger: ref, agent, gæst, Pax (påkrævet felt), flere hoteller pr. booking
- Hvert hotel kan have flere værelsestyper (fx 1 Standard + 1 Family)
- Check-out defaulter til Check-in-dato ved indtastning; rødt warning + blokeret save hvis Check-out < Check-in; confirm-prompt ved ophold > 14 nætter
- Samme dato-logik under Allotments' "Check period" filter
- Nyt hotel i en multi-stop booking: Check-in defaulter automatisk til forrige hotels Check-out
- Allotments: room types kan have flere sæsonperioder (konkrete datoer, ikke årligt tilbagevendende – skal genindtastes hvert år) med hver sin kvote
- Peak-occupancy beregning (sweep-line) virker pr. værelsestype-linje og pr. sæsonperiode
- Response Stats: Pax-kolonne + totalrække, Room nights-totalrække. Pax i bundtotal tælles 1 gang pr. unik booking (ikke pr. hotel-linje) – bug rettet
- CSV-export inkluderer Pax og alle værelsestyper pr. hotel
- Bagudkompatibel med gamle bookinger/allotments i det gamle flade data-format (ingen migrering nødvendig)

## Kendte åbne problemer
- **Auth: bygget og testet, men IKKE deployet endnu.** Se "Auth-projekt" nedenfor. Live-versionen på GitHub Pages bruger stadig det gamle localStorage-navn og åbne Firestore-regler, indtil Harry deployer den nye version.

## Auth-projekt (Firebase Auth + Firestore-regler) – status 2026-09-18
- Kode skrevet og lokalt testet, men ikke live endnu.
- Login: rigtig Firebase Auth (email/password), ingen offentlig selvregistrering. Kun Harry opretter medarbejderkonti manuelt i Firebase Console.
- Firestore-regler (`firestore.rules` i repoets rod): allowlist af 15 AET-e-mails kan læse/oprette/opdatere/**slette**. Der er IKKE en separat admin-rolle for slet-ret — det var den oprindelige plan, men Harry ombestemte sig 2026-09-18: adgangsstyringen ER allowlisten (kun Harry opretter/fjerner konti i Firebase Console), alle på listen har ellers samme rettigheder. Presence-collection: alle kan læse, kun egen bruger kan skrive eget dokument.
- Identitet: Firebase Auth-e-mailen er den autoritative bruger. Et visningsnavn (fx "Harry", "Koh Chang", "Sales 2") vises i UI og bruges som "oprettet af"/note-forfatter i stedet for rå e-mail – kun kilden til det eksisterende navnefelt er ændret, ingen ny audit-kolonne.
- Nyt: "X online now" øverst på siden (heartbeat hvert 45. sek., regnes som offline efter 2 min. uden heartbeat). Ingen oprydningsjob nødvendigt.
- Testet:
  - Firestore-reglerne kørt mod en rigtig lokal Firestore-emulator (12/12 bestået efter allow-alle-slette-ændringen): inkognito/uautoriseret kan hverken læse eller skrive, godkendt medarbejder (inkl. ikke-Harry) kan læse/oprette/opdatere/slette, presence kan kun skrives af ejeren selv.
  - App-niveau (Playwright): login-skærm vises til data hentes, forkert password afvises, gæstenavne optræder ikke i DOM før login, delete-knapper vises for alle godkendte konti, online-tæller udelukker "stale" heartbeats.
  - Alle tidligere regressionstests kørt igen (Pax, flere værelsestyper, allotment-perioder, gammelt dataformat, dato-validering, stats-totaler, Pax-dedup) – ingen regressioner.
- **Mangler før deploy**: Harry opretter de 15 konti i Firebase Console og aktiverer Email/Password sign-in (se separat guide), og deployer `aet_booking_tracker.html` + `firestore.rules` samtidig.

## Åbent spørgsmål (afventer svar)
- Sold-Out Calendar-artifact (separat værktøj, ikke denne tracker): forslag om at tilføje en note under "Allotment Availability" om at et hotels STOP SALE report betyder tabt allotment, selvom en tip stadig viser grøn. Ikke lagt ind endnu – afventer "ja/nej".

## Arbejdsregler
Se CLAUDE.md i repoets rod.

## Teknisk note (til mig selv / næste session)
- Repo: harryboysen/aet-booking-tracker, fil aet_booking_tracker.html, deployes via GitHub Pages
- Kan ikke pushe direkte til repo herfra – filer leveres, Harry redeployer manuelt
- Test lokalt med Playwright + fbstub (se tidligere test-scripts) før levering
