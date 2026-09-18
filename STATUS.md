[STATUS.md](https://github.com/user-attachments/files/32384719/STATUS.md)
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
- Ingen kritiske. Se "accepteret adfærd" under Auth-projekt for en lille, bevidst ikke-rettet detalje.

## Auth-projekt (Firebase Auth + Firestore-regler) – LIVE siden 2026-09-18
- Deployet og bekræftet virkende af Harry: login krævet, alle 15 konti oprettet, Firestore-regler publiceret.
- Login: rigtig Firebase Auth (email/password), ingen offentlig selvregistrering. Kun Harry opretter medarbejderkonti manuelt i Firebase Console.
- Firestore-regler (`firestore.rules` i repoets rod): allowlist af 15 AET-e-mails kan læse/oprette/opdatere/**slette**. Der er IKKE en separat admin-rolle for slet-ret — det var den oprindelige plan, men Harry ombestemte sig 2026-09-18: adgangsstyringen ER allowlisten (kun Harry opretter/fjerner konti i Firebase Console), alle på listen har ellers samme rettigheder. Presence-collection: alle kan læse, kun egen bruger kan skrive eget dokument.
- Identitet: Firebase Auth-e-mailen er den autoritative bruger. Et visningsnavn (fx "Harry", "Koh Chang", "Sales 2") vises i UI og bruges som "oprettet af"/note-forfatter i stedet for rå e-mail – kun kilden til det eksisterende navnefelt er ændret, ingen ny audit-kolonne.
- Nyt: "X online now" øverst på siden (heartbeat hvert 45. sek., regnes som offline efter 2 min. uden heartbeat). Ingen oprydningsjob nødvendigt.
- Testet:
  - Firestore-reglerne kørt mod en rigtig lokal Firestore-emulator (12/12 bestået efter allow-alle-slette-ændringen): inkognito/uautoriseret kan hverken læse eller skrive, godkendt medarbejder (inkl. ikke-Harry) kan læse/oprette/opdatere/slette, presence kan kun skrives af ejeren selv.
  - App-niveau (Playwright): login-skærm vises til data hentes, forkert password afvises, gæstenavne optræder ikke i DOM før login, delete-knapper vises for alle godkendte konti, online-tæller udelukker "stale" heartbeats.
  - Alle tidligere regressionstests kørt igen (Pax, flere værelsestyper, allotment-perioder, gammelt dataformat, dato-validering, stats-totaler, Pax-dedup) – ingen regressioner.
- **Accepteret adfærd (bevidst ikke rettet, Harry sagde nej 2026-09-18)**: "online now"-tallet opdateres ikke øjeblikkeligt ved log ud — presence-dokumentet slettes ikke aktivt ved sign-out, det tæller bare ikke længere med efter det er blevet "gammelt" (op til 2 min.). Harry er okay med den lille forsinkelse. Ret det ikke uden at spørge igen.

## "My Action List" – bygget og testet, AFVENTER Harrys godkendelse af preview (IKKE deployet endnu)
- Ny standard-startside efter login: "My Action List" (før: "Bookings"). Viser kun opgaver hvor den indloggede bruger er ansvarlig agent (matchet via Firebase Auth-e-mail → agentnavn).
- Genbruger eksisterende data (status/sent/reply/deposit/creditcard, `markHotelReplied`, `copyFollowup`) – ingen ny datamodel nødvendig for follow-up/betaling.
- Tre sektioner, sorteret rødt→gult→neutralt og derefter ældste/mest kritiske først:
  - **Follow-up due today / overdue**: se "Regelstyret opfølgning" nedenfor for de nøjagtige tærskler (nu administrerbare, ikke længere hardcoded).
  - **Payment required**: manglende depositum og/eller kreditkort. Rød hvis check-in er ≤3 dage væk eller overskredet; gul hvis agent endnu ikke er informeret; ellers neutral.
  - **Supplier reply pending**: sendt, intet svar, men endnu ikke forfalden.
- Handlinger pr. række: Send follow-up (kopierer eksisterende follow-up-tekst til udklipsholder – sender IKKE selv e-mail), Mark reply received (kalder eksisterende `markHotelReplied`, fjerner rækken hvis intet andet mangler), Escalate (egen modal – vælg/indtast ny ansvarlig agent + valgfri note).
- Escalate sætter nyt felt `h.assignedAgent` (overstyrer `booking.agent` for netop den hotellinje) og tilføjer et spor til nyt array `h.escalations[]` ({to, from, date, note}). Sporet vises også som en lille linje under hotellinjen i den normale Bookings-fane ("↳ Escalated to X by Y on dato — note").
- Ny Firestore-collection `users` (administreres via nyt "(team)"-link i headeren): {email, displayName, agentName, role}. `role: 'admin'` giver KUN adgang til en "Show everyone's tasks"-togglet på My Action List – det er ikke relateret til den tidligere fjernede slette-admin-rolle og ændrer intet ved slette-rettigheder. Kun brian.harry.boysen@gmail.com er admin som standard (hardcoded fallback), indtil/hvis `users`-collection sætter andre roller.
- De rigtige medarbejdernavne (fra Harrys liste) er også rettet ind i den eksisterende `USER_DISPLAY_NAMES`-mapping (headerens "logget ind som"-navn), som før viste generiske pladsholdernavne som "Sales 2", "Koh Chang" osv. Nu: Harry, Kai, New, Natty, Sherly, Benz, Waan, Mi, Hem, Nipon, Bee, Air, Jonas, Winnie, Neung.
- Team-modalen har en "Load default team"-knap, der udfylder alle 15 login-emails → agentnavne i `users`-collection med ét klik (kan køres flere gange uden at lave dubletter, doc-id = e-mail).
- "Bookings"-fanen er urørt/ufiltreret – al eksisterende funktionalitet virker som før, blot ikke længere default-fane.
- Ingen Firestore-regel-ændringer nødvendige – den generiske wildcard-regel dækker automatisk den nye `users`-collection.
- Testet (Playwright, 3 simulerede brugere: Hem, Sherly, Harry-som-admin): egne opgaver kun synlige for én selv, multi-leverandør-booking giver separate opgaver, Mark reply received opdaterer både Action List og Bookings-fane, Escalate flytter opgaven og efterlader synligt spor i Bookings-fanen, admin-toggle viser alles opgaver med agent-tag pr. række. Hele den gamle regressionssuite kørt igen uden fejl (2 ældre testscripts måtte opdateres til først at klikke ind på Bookings-fanen, da den ikke længere er default – ren testfix, ingen app-ændring).
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Regelstyret opfølgning + datakontrol – bygget og testet, AFVENTER Harrys godkendelse af preview (IKKE deployet endnu)
Bygget oven på "My Action List" som én isoleret ændring, per Harrys punkt 3+4-spec (2026-09-18).

**3. Regelstyret opfølgning**
- Følg-op-status beregnes nu udelukkende ud fra `h.sent`/`h.reply`/`h.status` — aldrig et manuelt indtastet ord. Ingen ny statusværdi at holde synkroniseret.
- Tre tempi: 0–1 dage = neutral "Awaiting reply"; 2–3 dage = gul "Follow-up due"; 4+ dage = rød "Overdue — escalation required". Nulstilles automatisk når `reply` sættes. Bekræftet/afvist (`status` = confirmed/cancelled) kan aldrig optræde som "reply pending" — testet eksplicit.
- Alle dag-beregninger bruger nu Thailand-tid (Asia/Bangkok, UTC+7, ingen sommertid) som fælles "i dag" — ikke browserens lokale tid. Så en kollega i Danmark og en i Thailand ser samme dagstal på samme booking. Ny hjælpefunktion: `thailandTodayStr()`.
- Hver række viser både "Sent Xd ago" og en tydelig næste-handling/dato, fx "Next: escalate if no reply by 2026-09-19" eller "Overdue since 2026-09-16 — escalation required".
- "Send follow-up" er uændret: kopierer/åbner stadig kun en kladde, sender aldrig selv.
- Eskalering gemmer stadig tidspunkt + oprindelig/ny ansvarlig + note i `h.escalations[]` (uændret fra "My Action List").
- Tærsklerne (standard 2 og 4 dage) er nu Firestore-indstillinger (`settings/followup`: `amberThresholdDays`, `redThresholdDays`), redigerbare i "(team)"-modalen under "Follow-up thresholds" — ingen kodeændring nødvendig for at justere dem senere.

**4. Datakontrol og kvalitet**
- Søgbar autocomplete (datalist) tilføjet for Agent (kilde: `users`-collection + tidligere brugte agentnavne) og Hotel/leverandør (kilde: Allotments + CRM Hotels + tidligere brugte hotelnavne). Partner er bevidst BEHOLDT som en rigtig `<select>` frem for fritekst+autocomplete — den trækker allerede udelukkende fra den eksisterende Partner-masterdata, så den kan slet ikke give stave-dubletter, hvilket er bedre end at konvertere den til fritekst. Land/destination er ikke tilføjet som nyt felt (markeret "gerne" i specen, og der findes ikke i forvejen et destinationsfelt på selve bookingen at hægte det på).
- Booking kan ikke længere gemmes uden: Bookingnummer, Gæstenavn (var IKKE valideret før!), Ansvarlig agent, Partner (var før "Optional" — nu påkrævet), mindst én hotel-/leverandørlinje, Hotel/leverandør-navn + Check-in-dato pr. linje der er rørt ved, Antal værelser (default 1, altid opfyldt). Fejl vises direkte ved feltet på letforståeligt engelsk, ikke længere via `alert()`.
- Check-out må nu heller ikke være samme dag som check-in (før: kun blokeret hvis FØR). Smart-default er justeret til at foreslå check-in + 1 dag i stedet for samme dag, så defaultværdien ikke længere er ugyldig i sig selv.
- Duplikat-advarsel: samme bookingnummer ELLER samme gæst + delt check-in-dato på tværs af eksisterende bookinger. Viser den eksisterende booking. Kun brugere med `role: 'admin'` (samme smalle rolle som "Show everyone's tasks" i My Action List — IKKE den gamle slette-admin-rolle) kan klikke "Save anyway"; alle andre kan kun gå tilbage og rette.
- Data renses ved gemning: trim + case-insensitiv matching af Agent og Hotel/leverandør-navn mod eksisterende masterdata ("sherly" → "Sherly" hvis "Sherly" allerede findes) — aldrig stille: en toast viser hvad der blev matchet/rettet.
- Ændringshistorik: nyt `booking.history[]` (append-only) med hvem, hvornår og hvad der ændrede sig (top-felter + hotel-linjer tilføjet/fjernet/ændret status/deposit/kreditkort). Vises under hotel-tabellen i den udvidede booking-card i Bookings-fanen (seneste 5, "+N earlier" hvis flere).
- Testet (Playwright): booking kan ikke gemmes med manglende nøglefelter eller ugyldige datoer (inkl. samme-dags check-out); agent-/hotelnavne kan ikke duplikeres pga. stavning/store-små bogstaver; duplikat-advarslen blokerer ikke-admin hårdt men lader admin gemme en reelt separat booking; follow-up-status skifter korrekt ved 2 og 4 dage (og ved brugerdefinerede tærskler) og forsvinder efter registreret svar. Hele den gamle regressionssuite + "My Action List"-testene kørt igen — ingen regressioner (2 ældre testscripts opdateret til at udfylde Gæst+Partner, da de nu er påkrævede — ren testfix).
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Hotel- og værelsestype-dropdown – bygget og testet, AFVENTER Harrys godkendelse af preview (IKKE deployet endnu)
Harrys opfølgende forslag (2026-09-18) oven på punkt 4: gør Hotel/leverandør (og efterfølgende Værelsestype) til en rigtig dropdown i stedet for fritekst+autocomplete, ligesom Partner, så ét hotel aldrig kan ende med at være stavet 2 forskellige steder.

- **Hotel / Supplier Name**: nu en rigtig `<select>` (som Partner) i stedet for fritekst med datalist. Kilden er `knownHotelNames()` (Allotments + CRM Hotels + tidligere brugte hotelnavne, case-insensitiv dedupe). Nederst i listen: "+ Add new hotel...", som viser et fritekstfelt til det nye navn. Et nyt hotel valgt denne vej bliver automatisk gemt som en ny, uafhængig CRM Hotels-post ved gemning (samme skema som at oprette et hotel manuelt i CRM-fanen) — det er selve mekanismen der "garanterer" at ét hotel ikke kan komme til at stå 2 steder stavet forskelligt, for fremover er det navn en valgbar mulighed for alle. En eksisterende, gammel fritekst-indtastet booking, hvis hotelnavn endnu ikke findes i listen, beholdes uændret som en valgt mulighed ved redigering — intet omskrives stille.
- **Room Type**: samme idé, men pr. hotel-linje — dropdown'en viser kun de værelsestyper der allerede er kendt for netop det valgte hotel (fra dets Allotment + tidligere bookinger); har hotellet ingen historik endnu, falder den tilbage til den fulde tværgående liste. Dropdown'en genopbygges automatisk når man skifter hotel på linjen. **Bevidst forskel fra Hotel**: en ny værelsestype valgt via "+ Add new room type..." bliver IKKE automatisk gemt nogen steder (værelsestyper har ingen selvstændig masterdata-post — de ligger kun inde i en Allotments kvote-liste, og at skrive et nyt `{qty:0, ingen perioder}`-element ind i en eksisterende allotment ved gemning er for risikabelt: det kunne forstyrre "Full/Low"-beregningen og peak-occupancy-logikken uden grundig, dedikeret test af hele den kode-sti). I stedet: samme case-insensitive stave-sikkerhedsnet som hotelnavne (`normalizeAgainstList`) ved gemning, så "deluxe"/"Deluxe" stadig samles til én stavning.
- Testet (Playwright, ny `test_roomtype_dropdown.js` + alle 16 eksisterende testscripts opdateret og kørt igen — ingen regressioner): hotel-dropdown tilbyder kendte hoteller + "+ Add new hotel..."; nyt hotel bliver en ny CRM-post; et hotel der matches via normalisering (fx "rawi warin resort" → "Rawi Warin Resort") bliver IKKE også oprettet som ny CRM-post; værelsestype-dropdown er begrænset til det valgte hotels egne typer, falder tilbage til fuld liste for et helt nyt hotel, en ny/matchet værelsestype ændrer aldrig Allotments-arrayet.
- **Status: GODKENDT af Harry (2026-09-18), fil sendt til upload. Afventer bekræftelse på at den faktisk er lagt live i GitHub Pages.**

## Admin: omdøb/flet/slet hoteller og værelsestyper – bygget og testet, AFVENTER Harrys godkendelse af preview (IKKE deployet endnu)
Harrys opfølgende spørgsmål (2026-09-18): kan han som admin rette/slette hvis der allerede findes 2 af samme hotel (fx fra før dropdown-løsningen fandtes)? Besvaret med 3 afklarende spørgsmål — svar: (1) ja, en omdøbning skal automatisk rette alle eksisterende bookinger så det bliver en reel fletning, (2) kun admin må gøre det, (3) sletning/fjernelse skal vise en advarsel med antal berørte bookinger, men admin kan vælge at fortsætte.

- **Hotelnavn (CRM Hotels-fanen)**: "Name"-feltet er nu låst (gråt, ikke redigerbart) for alle undtagen admin, når man redigerer et EKSISTERENDE hotel — med en tydelig note om hvorfor. Admin kan omdøbe frit; ved gem sammenlignes bogstavret (ikke kun ignorerer store/små bogstaver, så selv en ren stave-variant udløser fletningen). Systemet viser altid en advarsel med antal berørte bookinger/allotments før noget rettes, og admin skal bekræfte. Omdøbningen retter navnet på ALLE eksisterende bookinger og allotments der bruger den gamle stavning (matcher uafhængigt af store/små bogstaver) — det er det der gør det til en reel fletning. Rammer omdøbningen et navn der allerede findes som et andet hotel-kort, bliver de to kort automatisk lagt sammen til ét (noter fra begge bevares), og det overflødige kort slettes. Hver berørt booking får en linje i sin ændringshistorik ("Hotel renamed (admin merge): ... → ...").
- **Hotelnavn (Allotments-fanen)**: samme låsning/admin-krav gælder feltet "Hotel Name" når man redigerer en eksisterende allotment — omdøbning her kaskaderer på samme måde til bookinger og andre allotments/aliases (dog uden CRM-fletning, det håndteres i CRM-fanen). Ikke-admin kan stadig se og bruge "Also known as" til at tilføje en stavevariant uden at skulle omdøbe noget.
- **Værelsestype (Allotments-fanen, pr. hotel)**: samme princip — feltet er låst for ikke-admin på en allerede-gemt værelsestype (nye, aldrig gemte linjer kan alle stadig tilføje/fjerne frit). Admin kan omdøbe, med samme "X bookinger bliver opdateret"-advarsel, og omdøbningen retter navnet på matchende værelsestype-linjer i alle bookinger for netop det hotel (både det nye `roomTypes[]`-format og det gamle flade format).
- **Sletning**: at slette et hotel-CRM-kort eller fjerne en værelsestype-linje er nu også admin-only, med en klar advarsel hvis navnet stadig er i brug (viser antal bookinger/allotments) — men blokerer ikke, admin kan bevidst fortsætte (fx hvis kortet reelt er overflødigt). At tilføje et NYT hotel eller en NY værelsestype er stadig åbent for alle, uændret.
- Testet (Playwright, ny `test_hotel_roomtype_merge.js`, 4 scenarier + fuld regressionssuite kørt igen — ingen regressioner): ikke-admin kan ikke omdøbe (felt låst); admin-omdøbning fletter en stave-duplikat-hotel til ét CRM-kort med kombinerede noter og retter bookingens hotelnavn + historik; admin-omdøbning af en værelsestype retter alle matchende bookinger for det hotel; ikke-admin kan ikke fjerne en allerede-gemt værelsestype-linje.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Åbent spørgsmål (afventer svar)
- Sold-Out Calendar-artifact (separat værktøj, ikke denne tracker): forslag om at tilføje en note under "Allotment Availability" om at et hotels STOP SALE report betyder tabt allotment, selvom en tip stadig viser grøn. Ikke lagt ind endnu – afventer "ja/nej".

## Arbejdsregler
Se CLAUDE.md i repoets rod.

## Teknisk note (til mig selv / næste session)
- Repo: harryboysen/aet-booking-tracker, fil aet_booking_tracker.html, deployes via GitHub Pages
- Kan ikke pushe direkte til repo herfra – filer leveres, Harry redeployer manuelt
- Test lokalt med Playwright + fbstub (se tidligere test-scripts) før levering
