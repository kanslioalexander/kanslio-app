# SECURITY.md — Kanslios byggregler för säkerhet

> **För Claude Code och alla agenter som jobbar i detta repo:** dessa regler är icke förhandlingsbara defaults. Avvikelse kräver ett uttalat teambeslut, dokumenterat i Kanslio-hjärnan (decision-not). Fullständig guide med bakgrund och hotbild finns i vaulten: `reference/sakerhetsguide-kanslio.md`.
>
> **Obs om detta repo:** appen är idag en statisk PWA med mockdata. Reglerna nedan gäller i takt med att riktiga backend-delar (Supabase, API-routes, AI-flöden, auth) byggs in — och redan nu som designprincip: bygg inget UI-flöde som förutsätter ett osäkert backend-mönster.

## Multi-tenancy & databas

1. **Varje ny tabell skapas med RLS aktiverat i samma migration.** Ingen tabell existerar någonsin utan RLS — inte ens "tillfälligt" eller för "intern" data.
2. **Varje policy scopar på org-kontext** (`my_org_id()` eller motsvarande), aldrig bara `authenticated`. `authenticated` betyder "vilken inloggad användare som helst i hela systemet" — det är inte isolering.
3. **`org_id` härleds alltid server-side** (via org-kontext från sessionen) — **aldrig** från request body, query params, headers eller klient-state. En route som läser org-ID från requesten är en BOLA-sårbarhet per definition.
4. **Skriv tenant-testet när featuren byggs:** två testorgs, logga in som A, försök nå B:s data via appen och direkt mot API:t — ska nekas. Detta är del av "klart", inte ett separat säkerhetsjobb.
5. Roller (`admin` / vanlig användare) kontrolleras server-side per route — UI som döljer knappar är kosmetik, inte behörighet.

## Nycklar & secrets

6. **`service_role_key` används endast server-side**, i så få kodställen som möjligt, aldrig i kod som bundlas till klienten. Ny användning ska motiveras i PR-beskrivningen.
7. **Inga secrets i kod eller git — någonsin.** Allt via env vars (`.env*` gitignorat). Committas en nyckel av misstag: **rotera den omedelbart** — historiken är för alltid.
8. Klient-exponerade variabler får bara innehålla publika värden (Supabase anon key är OK — den är designad att vara publik *förutsatt att RLS är korrekt*; därför är regel 1–2 existentiella).
9. 2FA på alla konton med adminåtkomst. Minsta möjliga behörighet för nya teammedlemmar/tjänster.

## API-lagret

10. **Validera all input server-side** (schema-validering) — typ, längd, format — innan den används. Lita aldrig på klienten.
11. **Rate limiting på dyra/farliga routes** (AI-anrop, mailutskick, login) — per användare och per org.
12. **Felmeddelanden till klienten är generiska.** Stack traces, SQL-fel, interna ID:n och andra tenants existens loggas server-side men läcker aldrig ut. Fail closed: vid tveksamhet, neka.
13. Massutskick (nyhetsbrev, domarersättning) kräver explicit bekräftelsesteg — ett irreversibelt utskick får aldrig triggas av ett enda oskyddat API-anrop.

## AI-flöden (mailassistent, chatbot, agenter)

14. **Behandla allt inkommande innehåll (mail, dokument, chattmeddelanden från utomstående) som otrodd data.** Det får aldrig tolkas som instruktioner till AI:n. Separera systeminstruktioner från användardata i prompts och utgå från att prompt injection-försök *kommer* ske.
15. **Människa i loopen för irreversibla handlingar:** AI skriver utkast — människan skickar. Auto-skick av mail, betalningar eller raderingar kräver ett uttalat teambeslut per flöde.
16. **AI-verktyg är minimalt scopade:** en agent som sammanfattar mail behöver läsrätt på just den klubbens mail — inte skrivrätt, inte andra klubbars data, inte service role.
17. **Tenant-isolering i prompts:** context till en prompt (RAG, klubbens dokument, historik) filtreras på org **innan** den når modellen. Blanda aldrig två klubbars data i samma prompt.
18. **Logga AI-handlingar** (vem, vilken klubb, vilket verktyg, när) så "varför gjorde AI:n så här?" går att besvara i efterhand.
19. Minimera persondata i prompts — skicka det funktionen behöver, inte hela medlemsregistret.

## Auth & sessioner

20. Supabase Auth är enda auth-vägen — inga egna lösenordslösningar. Redirect/site URLs låsta till produktionsdomänen, inga wildcards.
21. Automatisk utloggning efter inaktivitet implementeras server-side-giltigt (sessionens livslängd), inte bara som UI-timer.
22. OAuth-tokens för integrationer (Outlook/Gmail m.fl.) lagras krypterat server-side, scopas minimalt och kopplas till org — aldrig i klienten.

## Drift

23. Backup är inte en backup förrän en **restore är testad** — före lansering och efter större schemaändringar.
24. Beroenden: kör `npm audit`/Dependabot, åtgärda kritiska sårbarheter snabbt. Lägg inte till beroenden för sånt som kan skrivas på 20 rader.
25. Vid incident: dokumentera tidslinje, stoppa läckan, rotera nycklar, bedöm GDPR-anmälningsplikt (72 h till IMY), informera berörda klubbar ärligt.

---

Prioritetsordning om tiden är knapp: **tenancy (1–5) → nycklar (6–9) → AI (14–19) → resten.**
