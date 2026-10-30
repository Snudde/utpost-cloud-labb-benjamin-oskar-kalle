# Beslutsdokument: hostingval

*Ett av teamets sex beslutsdokument. Skrivs i M6, hålls levande. Samma mall som `api-stil.md`, `databas.md` och `testing.md`. Det här dokumentet examinerar Cloud 1 (redogöra för hostinglösningar som AWS, Azure, Google Cloud) – skriv så att en läsare utan Azure-konto förstår valet.*

**Datum:**
**Beslut:** *(en eller två meningar: var kör Utpost i staging, var ska den köra i produktion, och vad är skillnaden?)*

## Bakgrund
*Vad har vi när M5 är klar: två images i GHCR taggade med sha, en pipeline som bygger dem, compose lokalt. Vad saknas för att någon utanför teamet ska kunna öppna appen? Vilka krav ställer kursen (M6: automatisk deploy till staging vid merge, secrets ur koden, publik URL)?*

## Vad vi faktiskt hyr
*Fyll i tabellen med era egna värden ur Azure-portalen eller `az containerapp show`. Kolumnen "nivå" = IaaS / PaaS / SaaS / serverless (klippet *Molnet: vad det är och vad man hyr*).*

| Del | Tjänst | Nivå | Vem sköter uppdateringar/skalning | Vad det kostar oss i staging |
|---|---|---|---|---|
| Klient + API | Azure Container Apps (Consumption), appen `utpost-staging` – en app med fyra containrar som sidecars, 1,75 vCPU / 3,5 GiB | PaaS (containrar) | Microsoft sköter servrar, nätverk, TLS och skalning (0–1 replica); vi sköter imagerna och `staging.yaml` | 0 kr hittills i oktober (Cost Management, 30/10). Vi ligger inom gratiskvoten – och kostnadsdata kommer med upp till ett dygns fördröjning. |
| Postgres | `postgres:16` som sidecar i samma app, ingen volym | ingen tjänst – vår egen container | vi själva – ingen backup, databasen seedas om vid varje start | ingår i raden ovan |
| MongoDB | `mongo:8.2` som sidecar i samma app, ingen volym | ingen tjänst – vår egen container | vi själva – ingen backup, seedas om vid varje start | ingår i raden ovan |
| Images | GHCR (`ghcr.io/snudde/…-api`, `…-client`), taggade `sha-<7>` | SaaS | GitHub driver registret; vi bestämmer vad som publiceras (jobbet Image) | 0 kr – ingår i GitHub-kontot |
| Deploy | GitHub Actions → `az containerapp update` med OIDC (inga lösenord) | SaaS | GitHub kör jobbet, vi skriver och underhåller det | 0 kr – ingår i Actions-minuterna för kontot (en deploy ≈ 1,5 min) |

**Förbrukning 30/10** (appen → Metrics): requests och repliker bara under de timmar vi deployade och testade (som mest 2 repliker samtidigt, under ett revisionsbyte), i övrigt 0 – appen sover mellan passen.

**Gratiskvoten räcker till** (Consumption: 180 000 vCPU-sekunder och 360 000 GiB-sekunder per prenumeration och månad):

- vCPU: 180 000 / 1,75 ≈ 102 900 s ≈ **28,5 timmar vaken per månad**. Minnet ger samma siffra (360 000 / 3,5).
- Ensamma i prenumerationen: ~28 h/månad – räcker för lektionspassen, inte för en app som är vaken dygnet runt (720 h).
- Hela klassen delar den: 29 elever har en egen resursgrupp i samma prenumeration → 28,5 h / 29 ≈ **1 timme vaken per person och månad** innan någon börjar betala. Därför skalar vi till noll.

## Alternativ vi jämförde
*Minst tre riktiga alternativ, och minst två leverantörer. Förslag: Azure Container Apps (det vi gjorde) · Azure App Service for Containers · Google Cloud Run · AWS ECS på Fargate (App Runner tar inte längre nya kunder) · en VM (IaaS) med docker compose · Render/Railway (PaaS utanför de tre stora). För varje: vad hade ändrats i repot, vad hade kostat, vad hade vi fått/förlorat?*

| Alternativ | Vad ändras i repot | Kostnad (staging) | För | Emot |
|---|---|---|---|---|
| | | | | |

## Staging och produktion är inte samma sak
*I staging kör databaserna som sidecars utan volym och seedas vid varje kallstart. Vad krävs för produktion? Managed databas (vilken, vad kostar den), backup, en app som inte skalar till noll, egen domän och certifikat, loggar. Skriv vad ni skulle byta och vad som INTE behöver bytas (imagen? pipelinen?).*

## Inlåsning
*Om Azure försvann i morgon: vilka filer i repot ändras, och vilka ändras inte? Räkna upp dem. Det är svaret på hur portabel lösningen är – och argumentet för att bygga på image + pipeline i stället för plattformens egna byggverktyg.*

Ingenting i imagerna vet att de kör i Azure. Det som låser oss är fyra filer och några värden.

**Bara för Azure – byts om vi flyttar** (till exempel till Cloud Run eller ECS):

- `infra/azure/staging.yaml` – appen beskriven i Container Apps format
- `infra/azure/skapa-staging.sh` – skapar resursgrupp, miljö och app med `az`
- `infra/azure/identitet.sh` – OIDC-identiteten i Microsoft Entra
- jobbet `deploy` i `.github/workflows/ci.yml` – `azure/login` och `az containerapp update`

**Likadant i alla moln – ändras inte:**

- `api/Dockerfile` och `client/Dockerfile`
- `client/nginx.conf.template` – nginx-mallen som läser `API_UPSTREAM` vid start
- `.env.example` – listan över vilka värden en miljö måste sätta
- `api/src/` – koden läser allt ur miljövariabler
- jobben Kvalitet, Bygg, Integration och Image – imagerna i GHCR går att köra var som helst

**Värdena som skiljer mellan miljöer** (`git grep` i `compose.yaml`, `infra/`, `client/Dockerfile`, `.env.example`):

| Värde | Lokalt (compose) | Azure staging |
|---|---|---|
| `API_UPSTREAM` | `api:4000` (egen container) | `localhost:4000` (sidecar i samma app) |
| `DATABASE_URL` | `…@postgres:5432/utpost` | `…@localhost:5432/utpost`, lösenord som secret |
| `MONGO_URL` | `…@mongo:27017/…` | `…@localhost:27017/…`, lösenord som secret |
| `JWT_SECRET` | dev-värde i `compose.yaml` | slumpat av `skapa-staging.sh`, bara som secret i Azure |
| `SEED_ON_START` | `1` (harmlöst lokalt) | `1` – tom databas vid varje start |

## Konsekvenser
*Vad kostar valet i tid och pengar? Kallstarten när appen skalat till noll. Gratiskvoten som delas av alla i prenumerationen. Hemligheter som nu lever på två ställen (lokalt `.env`, i molnet secrets) – hur roteras de? Vem i teamet har tillgång till portalen?*

**Kallstarten:** när appen skalat till noll startar första anropet fyra containrar och seedar databaserna. Uppmätt 30/10: ~1 minut (en omstart med nya containrar tog 58 s, en ny revision 41 s). Under tiden kan svaret bli `502`. Deploy-jobbet väntar ut det med 30 försök à 10 s.

**Skrivet av:**
