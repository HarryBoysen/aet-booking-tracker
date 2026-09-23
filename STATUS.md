[Uploading STATUS.md…]()
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

## "Add all hotels from bookings"-knap i Hotels-fanen – bygget og testet, AFVENTER Harrys godkendelse
Harry opdagede at Hotels-listen under "Partners & Hotels" var tom (forventet — den er en separat kontaktkort-liste, adskilt fra Allotments) og bad om at få alle hoteller de allerede har booket, listet der, så han kan rette/redigere dem.

- Ny knap "Add all hotels from bookings" øverst til højre, kun synlig når "Hotels"-filteret er valgt. Bruger samme kilde som booking-formularens hotel-dropdown (`knownHotelNames()` — Allotments + eksisterende bookinger), og opretter et nyt CRM Hotels-kort for hvert navn der endnu ikke har et. Viser antal + en kort liste af navne til bekræftelse, før noget oprettes.
- Overskriver eller sletter aldrig noget eksisterende — kan trygt køres flere gange (opretter ikke dubletter, tjekker uafhængigt af store/små bogstaver). Åben for alle (ikke admin-only), da den kun tilføjer, aldrig retter/sletter.
- Testet (Playwright, ny `test_sync_hotels.js` + fuld regressionssuite — ingen regressioner): knap kun synlig på Hotels-visningen; alle manglende hotelnavne fra bookinger/allotments bliver oprettet; et navn der allerede findes i CRM'en springes over; kørt to gange giver ikke dubletter.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## "Mulige duplikat-hoteller"-panel i Hotels-fanen – bygget og testet, AFVENTER Harrys godkendelse
Harry spurgte "kan du tjekker hoteller og sende mig en liste med dem der minder om hinanden?" — jeg har ikke direkte adgang til jeres rigtige Firestore-data herfra, så løsningen er et permanent, selvbetjent panel i appen i stedet for en engangsliste fra mig.

- Nyt advarselspanel øverst i Hotels-fanen (kun synligt når "Hotels"-filteret er valgt), der viser grupper af hotelnavne der ligner hinanden — fx netop "Koh Kood Paradise Beach Resort" / "Koh Kood Paradise Resort" fra Harrys skærmbilleder. Genbruger samme dubletdetektor og samme "husk mine beslutninger"-collection (`dismissedDuplicates`) som det eksisterende panel i Response Stats — men er BREDERE: det tjekker hele `knownHotelNames()` (CRM Hotels + Allotments + bookinger), ikke kun navne der allerede har bookinger. Det er netop det der fangede Harrys eget eksempel, som ellers ikke ville dukke op i Response Stats-panelet (0 bookinger endnu på nogen af de to kort).
- Selve dubletdetektoren (`findPossibleDuplicateHotelNames`, delt af begge paneler) er samtidig gjort en anelse skarpere: den fangede tidligere kun navne der er præfiks/suffiks-varianter af hinanden (fx "Koh Chang Resort" / "Koh Chang Resort & Spa"). Harrys eget eksempel har et ekstra ord midt i navnet ("...Paradise **Beach** Resort" vs "...Paradise Resort"), som den gamle logik ikke fangede. Ny regel: hvis det korteste navn har mindst 3 ord, og ALLE de ord også findes i det andet navn, regnes de som mulige dubletter — testet grundigt for ikke at give falske positiver på hoteller der bare tilfældigt deler 1-2 generiske ord (fx to forskellige "... Resort").
- To knapper pr. gruppe: "Yes, same hotel — merge" (kun admin — bruger samme `renameHotelEverywhere()` som det eksisterende omdøb/flet-værktøj, retter bookinger + allotments + CRM-kort, og lægger evt. flere CRM-kort med samme endelige navn sammen til ét med kombinerede noter) og "No, different hotels" (alle kan bruge denne — gemmer beslutningen, så panelet ikke bliver ved med at foreslå det samme igen; delt med Response Stats-panelet, så en afvisning ét sted også gælder det andet).
- Testet (Playwright, ny `test_crm_duplicate_finder.js` + fuld regressionssuite på 20 testscripts — ingen regressioner): panelet viser en dukket-op-duplikat med 0 bookinger; ikke-admin ser gruppen men intet flet-knap; admin-fletning kollapser to CRM-kort til ét med kombinerede noter og opdaterer bookinger/allotments; en afvist gruppe forsvinder og forbliver væk.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Fejlmeddelelse ved mislykket gem (2026-09-19) – bygget og testet, AFVENTER Harrys godkendelse
Harry rapporterede: ændrede check-out-dato på booking 35851 ("Golden Beach Resort Krabi", 18/01→25/01, altså en gyldig dato-ændring), men "Save Booking" gjorde ingenting — ingen fejl, ingen kvittering. Ved gennemgang af koden var det en reel mangel: hvis selve gemningen fejler (fx udløbet login, midlertidigt netværksproblem, Firestore-afvisning), var der ingen fejlhåndtering — modalen blev bare stående uden forklaring, præcis som Harry oplevede.

- `saveBooking()` er nu pakket ind i try/catch. Fejler gemningen af en hvilken som helst årsag, vises nu en tydelig rød besked øverst i modalen ("Could not save: ... — try refreshing the page and logging in again, or contact Harry"), og den fulde tekniske fejl logges i browserens konsol (F12) til fejlsøgning.
- Retter IKKE i sig selv årsagen til Harrys konkrete problem (den kender jeg stadig ikke, uden adgang til jeres rigtige data) — men gør fremtidige mislykkede gemninger synlige i stedet for stille, så det kan diagnosticeres med det samme.
- Testet (Playwright, ny `test_save_error_visible.js` + fuld regressionssuite på 21 testscripts — ingen regressioner): en simuleret mislykket gemning viser nu fejlbeskeden i modalen i stedet for at lukke stille ned.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav). Afventer desuden Harrys faktiske fejltekst fra browser-konsollen for at finde selve root cause.**

## Lomprayah Tickets + Train Tickets – to nye faner, REVIDERET 2026-09-23 – bygget og testet, AFVENTER Harrys godkendelse
Første udkast blev sendt til Harry som preview (2026-09-23); han svarede med 6 konkrete forbedringspunkter, som er implementeret. Hans stand-ud prioritet var: "booking-tilknytning, opfølgning i My Action List og billetvedhæftning". Billetvedhæftning blev efterfølgende fravalgt igen (se note nederst) — de to andre prioriteter er med.

- **1. Booking-tilknytning**: nyt felt "Booking" øverst i formularen — et **søgbart tekstfelt** (ikke en dropdown-liste, som ville være ubrugelig med flere hundrede bookinger). Skriv et bookingnummer eller gæstenavn, og browseren viser matchende forslag live; vælg ét og Gæst + Partner udfyldes automatisk OG låses (gråt, kan ikke rettes ved en fejl). Det manuelle "Booking Reference"-felt skjules til fordel for bookingens egen reference. **Inkluderer også arkiverede bookinger** (mærket "(Archived)" i søgeresultatet), da en billet stadig kan mangle at blive booket selvom hotel-bookingen allerede er arkiveret. Én booking kan sagtens have flere billetter (flere færge-/tog-ben) — det er bare flere billetter der peger på samme booking-id. Ryd feltet igen for at låse Gæst/Partner op og udfylde dem frit (til grupper uden en oprettet booking endnu).
- **2. Booking Opens + Follow-up Date**: to nye datofelter. "Booking Opens" er informativt (rejsen kan ikkes bookes før denne dato). "Follow-up Date" er den der driver "My Action List" — en billet dukker automatisk op under en ny sektion "Transport tickets to book" i den ansvarlige agents Action List fra og med denne dato, indtil billetten markeres Booked. Ingen fast regel for alle billetter — datoen sættes manuelt pr. billet, præcis som Harry bad om. Rød/gul severity efter hvor tæt afrejsen er (≤3 dage = rød).
- **3. Booking/betaling/afsendelse adskilt**: Booked, Paid og Sent to partner er nu tre uafhængige afkrydsningsfelter (både i modalen og direkte i listen) — hver med sin egen stempling. Sent to partner gemmer nu også HVEM der sendte (agentnavn) og hvornår, ikke kun ja/nej. Payment Due Date er sit eget datofelt, ikke bundet til om Paid er sat.
- **4. Rigtige billetdetaljer**: nyt felt Ticket No(s). (fritekst, flere billetnumre kan skrives i ét felt). For tog: også Wagon (Vogn) og Seat/Berth (Plads/køje), skjult på Lomprayah-fanen. (Selve filvedhæftningen af billetten er IKKE med — se note nederst.)
- **5. Klarere datoer og pax**: Departure/Arrival er nu splittet i egne dato- OG tidsfelter (Arrival Date er separat, til overnatningstog — defaulter til samme dato som afrejse, men kan ændres). Datoer vises i listen som "11 Jan 2027" (dag måned år udskrevet), så dag/måned aldrig kan forveksles. Adults og Children (med alder pr. barn) er separate felter i stedet for ét fritekstfelt, med automatisk sum ("Total pax: 4") vist live.
- **6. Listen som arbejdsoverblik**: filterknapper tilføjet — **Upcoming** (standard: i dag eller senere), **Not booked**, **Not sent**, **Next 7 days**, **History** (alt før i dag). Plus en "ansvarlig agent"-dropdown til at filtrere listen. Ældre ture forsvinder automatisk fra standardvisningen til History, i stedet for at rode den daglige liste til.
- **Lille rettelse**: Train-fanens eksempeltekst i Fra/Til viser nu togstationer ("Chiang Mai"/"Bangkok") i stedet for den gamle "Koh Samui Pier"-færgeplaceholder.
- **Fravalgt igen (2026-09-23)**: billetvedhæftning (upload af PDF/billede) blev først bygget via Firebase Storage, men Harry vurderede at de ikke har brug for det — fjernet helt igen (ingen Storage-import, intet upload-felt, ingen `attachmentName`/`attachmentUrl` på billet-dokumentet). Det betyder også at Blaze-plan-opgraderingen IKKE er nødvendig for denne funktion længere — kun for den separate, endnu ikke godkendte e-mail-påmindelses-funktion nedenfor, hvis/når den bygges.
- **Ikke bygget i denne runde**: en egentlig import-guide fra de gamle Excel-ark med gennemgang af usikre/tvetydige linjer (Harry nævnte dette som "vigtigt tjek ved import", men det lå uden for hans top-prioritet). Hvis I ønsker de gamle Excel-data migreret i bulk til trackeren, kan det bygges som en separat, isoleret funktion — sig til.
- Begge faner deler stadig samme Firestore-collection (`transportTickets`, med et `type`-felt) og samme kode.
- Testet (Playwright, `test_transport_tickets.js` omskrevet fra bunden + fuld regressionssuite på 22 testscripts — ingen regressioner): booking-tilknytning udfylder og låser Gæst/Partner korrekt, adults+children-sum beregnes live, Arrival Date defaulter til Departure Date, Booked/Paid/Sent er uafhængige og stempler korrekt (inkl. hvem der sendte), alle fire filtre + agent-filter + Upcoming/History-opdeling virker, og "Transport tickets to book" dukker korrekt op i My Action List (med "Mark booked"-genvej der fjerner den igen).
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Fælles team-note på forsiden (My Action List) – bygget og testet, AFVENTER Harrys godkendelse
Harry spurgte om man kan skrive en note der kan ses på forsiden. Afklaret: alle medarbejdere skal kunne skrive/redigere, og det skal være ÉN fælles note (ikke en liste af flere) — overskrives når nogen ændrer den.

- Ny boks øverst på "My Action List"-fanen (appens forside efter login). Er der ingen note, vises kun en lille "+ Add a note for the team"-knap — fylder intet når den ikke bruges. Er der en note, vises den med gul baggrund + hvem der sidst skrev den + dato ("Harry · 2026-09-23"), og en rediger-knap i hjørnet.
- Alle logget-ind medarbejdere kan skrive/redigere/rydde noten — ikke admin-only, som Harry bad om. Gemning overskriver hele noten (ikke en log/liste) — helt bevidst, for at holde det simpelt som en whiteboard-besked.
- Gemt i Firestore som ét dokument (`settings/teamNote`: `{text, updatedBy, updatedDate}`), samme collection som de eksisterende follow-up-tærskler bruger — ingen ny collection eller regelændring nødvendig.
- Testet (Playwright, ny `test_team_note.js` + fuld regressionssuite på 23 testscripts — ingen regressioner): tom tilstand viser kun tilføj-knappen, gemt note viser tekst + forfatter + dato, redigering forudfylder den eksisterende tekst, Cancel gemmer intet, at gemme igen overskriver hele noten (ikke tilføjer), Clear rydder den tilbage til tom tilstand for alle.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Notes-kolonne direkte i Lomprayah/Train Tickets-listen – bygget og testet, AFVENTER Harrys godkendelse
Harry bekræftede at billettrackeren virker live, og bad om at kunne skrive en kort bemærkning direkte på hver linje i listen (fx "Can book to Nov"), uden at skulle åbne redigeringsvinduet.

- Ny "Notes"-kolonne i tabellen på både Lomprayah- og Train-fanen, mellem "Sent" og redigér/slet-knapperne — et almindeligt tekstfelt man kan skrive direkte i. Gemmes automatisk når man forlader feltet (samme mønster som de eksisterende Booked/Paid/Sent-afkrydsningsfelter i listen — ingen "Save"-knap nødvendig).
- Genbruger billettens eksisterende `notes`-felt (det samme som "Notes"-feltet nederst i redigeringsvinduet) — det er altså IKKE et nyt, separat felt; skriver man i kolonnen, opdateres den samme note man også ser hvis man åbner billetten.
- Testet (Playwright, `test_transport_tickets.js` udvidet + fuld regressionssuite på 23 testscripts — ingen regressioner): Notes-kolonnen vises på begge faner, tekst skrevet direkte i en række gemmes og overlever en genindlæsning af listen.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Mere kompakt Lomprayah/Train Tickets-liste – bygget og testet, AFVENTER Harrys godkendelse
Harry bad om tre justeringer af listevisningen efter Notes-kolonnen: alt skal stå på én linje (evt. mindre tekst), "Paid" fjernes fra listen for at give plads, og der skal være en tydeligere streg mellem hver booking.

- Skriftstørrelsen i selve billet-tabellen er sat ned (fra 13px til 12px), og hver celle får `white-space:nowrap` så indholdet ikke længere kan brække over flere linjer — hver booking fylder nu præcis én række.
- "Paid"-kolonnen (afkrydsningsfelt) er fjernet fra listevisningen på begge faner for at give mere plads til de andre kolonner. Selve "betalt"-status er ikke fjernet fra systemet — den kan stadig ses og redigeres inde i billetten (rediger-ikonet), kun den hurtige afkrydsning direkte i listen er væk.
- Streg mellem hver linje er gjort tydeligere (fra en meget svag streg til en klart synlig grå streg), så det er lettere at se hvor én booking slutter og den næste starter.
- Disse ændringer påvirker kun selve billet-tabellen (Lomprayah/Train Tickets) — andre tabeller i systemet (fx Response Stats) ser ud som før.
- Testet (Playwright, `test_transport_tickets.js` opdateret + fuld regressionssuite på 23 testscripts — ingen regressioner).
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Den "skubbende" linje i bunden af tabellen – rettet, AFVENTER Harrys godkendelse
Harry sendte et screenshot og spurgte hvorfor der er en grå "skubbe"-linje i bunden af Lomprayah/Train Tickets-tabellen. Det var den vandrette scrollbar til tabellen — den kom fordi tabellen var sat til en fast minimumsbredde (1150px), som var lige akkurat bredere end det tilgængelige plads på siden, så scrollbaren altid var synlig, selv når det ikke var nødvendigt (særligt på Lomprayah-fanen, som har én kolonne mindre end Train og derfor slet ikke behøvede den bredde).

- Fjernet den faste minimumsbredde — tabellen fylder nu kun så meget plads som kolonnerne rent faktisk fylder, i stedet for at blive tvunget bredere end nødvendigt.
- Gjort "Notes"-feltet lidt smallere (170px → 150px) for at give ekstra luft.
- Scrollbaren er stadig der som sikkerhedsnet på meget smalle skærme/vinduer, men er nu tynd og diskret i stedet for den brede, grå bjælke fra før — og vises kun når den faktisk er nødvendig.
- Testet (Playwright, ny `test_ticket_multiroute_and_scroll.js`): tabellen kræver ikke vandret scroll på en normal bærbar-bredde, hverken på Lomprayah- eller Train-fanen. Fuld regressionssuite på 24 testscripts — ingen regressioner.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Flere ruter pr. booking i Lomprayah Tickets – bygget og testet, AFVENTER Harrys godkendelse
Harry påpegede at der typisk er 2-3 Lomprayah-ruter pr. kunde (fx ø-til-ø, så videre til fastlandet), og bad om en måde at oprette flere ruter pr. booking uden besvær.

- Ny knap "Save & add another route" i redigeringsvinduet for Lomprayah-billetter (vises kun for Lomprayah, ikke Train, da tog normalt kun er én strækning). Knappen gemmer den aktuelle rute som sin egen billet-linje, men holder vinduet åbent og beholder booking, gæstenavn, partner, agent og antal personer — kun rute/dato/tid/status/billetdetaljer nulstilles, så man straks kan skrive den næste rute ind.
- Hver rute gemmes fortsat som sin egen linje i listen (som i dag), så Booked/Sent/Notes stadig kan følges rute for rute — det er kun *indtastningen* der er blevet hurtigere, ikke selve datastrukturen.
- En grøn bekræftelse ("Route saved. Enter the next route below...") vises i vinduet efter hver gemt rute, så det er tydeligt at det virkede.
- Almindelig "Save Ticket"-knap virker som altid og lukker vinduet efter sidste rute.
- Testet (Playwright, ny `test_ticket_multiroute_and_scroll.js`): opretter 2 ekstra ruter for samme booking via "Save & add another route", bekræfter at booking/gæst/booking ref er ens på alle 3 linjer, og at knappen er skjult for Train-billetter. Fuld regressionssuite på 24 testscripts — ingen regressioner.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Notes-feltet er nu en ren hvid boks + ruter samlet i én boks pr. kunde – bygget og testet, AFVENTER Harrys godkendelse
To justeringer efter forrige runde: (1) eksempel-teksten "e.g. Can book to Nov" i Notes-feltet skulle væk, så det bare er en almindelig hvid boks. (2) Når samme kunde/booking har flere Lomprayah-ruter, skal de ligge samlet i én boks — samme princip som når en booking har flere hoteller på Bookings-fanen (én kort/boks pr. booking, med hver hotel som en linje inde i boksen).

- Notes-feltet i listen viser ikke længere eksempel-tekst — det er nu bare en tom, hvid boks man kan skrive direkte i (gælder både Lomprayah og Train).
- Lomprayah Tickets-fanen viser nu én boks pr. kunde/booking (samme visuelle stil som en booking-boks på Bookings-fanen — hvid kasse med afrundede hjørner). Bookingnummer, gæstenavn, antal personer, agent og partner står én gang i boksens "header" i stedet for at blive gentaget på hver linje. Inde i boksen er hver rute sin egen linje med Afgang, Rute, Booked, Sent og Notes — præcis som Booked/Paid/Sent er styret rute for rute i dag. Har kunden 2+ ruter, viser boksen fx "2 routes" i headeren, så det er tydeligt der er flere.
- Ruter grupperes efter den bookingen de er linket til; er en billet ikke linket til en booking, grupperes den efter gæstenavn i stedet (samme kunde skrevet med samme navn havner stadig i samme boks).
- Train Tickets-fanen er IKKE ændret — den viser stadig én flad liste, da tog normalt kun har én strækning pr. kunde.
- Testet (Playwright, `test_ticket_multiroute_and_scroll.js` udvidet): opretter en kunde med 3 ruter og bekræfter de vises i ÉN boks (ikke 3 separate), at boksen nævner "3 routes", og at Notes-feltet ikke har eksempel-tekst. Fuld regressionssuite på 25 testscripts — ingen regressioner.
- **Status: klar til deploy, men afventer Harrys eksplicitte godkendelse af preview/screenshot først (hans krav).**

## Åbent spørgsmål (afventer svar)
- Sold-Out Calendar-artifact (separat værktøj, ikke denne tracker): forslag om at tilføje en note under "Allotment Availability" om at et hotels STOP SALE report betyder tabt allotment, selvom en tip stadig viser grøn. Ikke lagt ind endnu – afventer "ja/nej".

## Arbejdsregler
Se CLAUDE.md i repoets rod.

## Teknisk note (til mig selv / næste session)
- Repo: harryboysen/aet-booking-tracker, fil aet_booking_tracker.html, deployes via GitHub Pages
- Kan ikke pushe direkte til repo herfra – filer leveres, Harry redeployer manuelt
- Test lokalt med Playwright + fbstub (se tidligere test-scripts) før levering
