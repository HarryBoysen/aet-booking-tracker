[STATUS.md](https://github.com/user-attachments/files/32335709/STATUS.md)
# AET Booking Tracker – STATUS

Sidst opdateret: 2026-09-17

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
- **Google Sign-In / auth stadig uløst** – appen bruger kun et lokalt brugernavn (localStorage), ingen rigtig Firebase Auth, Firestore-regler er reelt åbne. Sikkerhedsspørgsmål, ikke glemt.

## Åbent spørgsmål (afventer svar)
- Sold-Out Calendar-artifact (separat værktøj, ikke denne tracker): forslag om at tilføje en note under "Allotment Availability" om at et hotels STOP SALE report betyder tabt allotment, selvom en tip stadig viser grøn. Ikke lagt ind endnu – afventer "ja/nej".

## Arbejdsregler (fra Harry, gælder fremover)
- Læs denne fil + seneste ændringer før noget nyt bygges
- Én opgave ad gangen, sig konkret hvad der laves nu
- Beslutninger skrives ind her med det samme de tages
- Mindst mulig ændring pr. opgave
- Efter en ændring: kort opsummering af hvad der blev lavet / ikke rørt / hvad der bør testes manuelt
- Opdatér denne fil efter hver rigtig ændring
- Sikkerhedsspørgsmål markeres åbne, glemmes ikke
- Samme fejl rettet 2-3 gange uden held → stop, sig det højt, find rodårsag
- Standard-model (Sonnet) til det meste; tungere model kun til svære strukturelle beslutninger (fx login-løsning en gang for alle)

## Teknisk note (til mig selv / næste session)
- Repo: harryboysen/aet-booking-tracker, fil aet_booking_tracker.html, deployes via GitHub Pages
- Kan ikke pushe direkte til repo herfra – filer leveres, Harry redeployer manuelt
- Test lokalt med Playwright + fbstub (se tidligere test-scripts) før levering
