# melosys-workflows

Felles GitHub Actions-workflows for Team Melosys sine apper.

## Ukentlig rebygg med nytt baseimage

Chainguard-baseimagene får jevnlig sikkerhetsoppdateringer uten at taggen endres, og
Chainguard backporter ikke til eldre bygg. Appene må derfor bygges og deployes på nytt
jevnlig, også når det ikke er endringer i egen kode
([Security Playbook](https://sikkerhet.nav.no/docs/verktoy/chainguard-dockerimages)).

| Workflow | Hva den gjør |
| --- | --- |
| `refresh-base-images.yml` | Mandag kl. 07 (Europe/Oslo) starter den `rebygg-og-deploy.yml` i hver app i [`baseimage/apps.json`](baseimage/apps.json), med prod. Kan også startes manuelt for én app. |
| `rebygg-og-deploy.yml` | Felles workflow som appene kaller. Se under. |
| `bygg.yml` | Bygger og tester en commit og publiserer imaget. Brukes av `rebygg-og-deploy.yml`. |
| `deploy.yml` | Deployer et image til ett cluster. Brukes av `rebygg-og-deploy.yml`. |

### Hva `rebygg-og-deploy.yml` gjør

1. Finner siste commit på standardbranchen og commiten for siste publiserte release.
2. Bygger standardbranchen med tester, og deployer til dev-miljøene i rekkefølge. Neste miljø deployes bare hvis forrige gikk bra.
3. Hvis `prod` er valgt og dev gikk bra: bygger siste release på nytt og deployer den til prod.
   Er releasen samme commit som standardbranchen, brukes dev-imaget.
4. Sender Slack-varsel med status for dev og prod.

Rett før hver deploy sjekkes det at commiten fortsatt er den siste. Har en vanlig deploy
kommet med nyere kode mens vi bygget, hoppes deployen over, så den ikke rulles tilbake.
Imaget får bare den unike datotaggen fra `nais/docker-build-push` og deployes med digest;
sha-taggen peker fortsatt på imaget som ble testet ved merge.

Kode som ikke er releaset, kommer aldri i prod: prod får alltid siste publiserte release.

### Ta i bruk i en app

1. Kopier [`eksempel/.github/workflows/rebygg-og-deploy.yml`](eksempel/.github/workflows/rebygg-og-deploy.yml)
   til appen og tilpass valgene under. Filen må hete `rebygg-og-deploy.yml`.
2. Merge, og kjør den manuelt uten prod: `gh workflow run rebygg-og-deploy.yml`.
3. Legg appen til i [`baseimage/apps.json`](baseimage/apps.json).
4. Sjekk at GitHub-appen Melosys CI er installert på repoet (den starter workflowen).

| Valg | Påkrevd | Standard | Beskrivelse |
| --- | --- | --- | --- |
| `bygg-kommando` | ja | | Bygger og tester appen, f.eks. `./mvnw clean package -B` eller `./gradlew build` |
| `byggverktoy` | ja | | `maven` eller `gradle` |
| `java-versjon` | | `21` | |
| `dev-cluster` | ja | | f.eks. `dev-fss` eller `dev-gcp` |
| `dev-vars` | ja | | Én til tre vars-filer, én per linje, i deployrekkefølge |
| `prod-cluster` | ja | | f.eks. `prod-fss` eller `prod-gcp` |
| `prod-vars` | | `nais/vars-prod.json` | |
| `nais-resource` | | `nais/nais.yml` | |
| `team` | | `teammelosys` | |
| `platforms` | | `linux/amd64` | |
| `runner` | | `ubuntu-latest` | Runner for byggejobbene |
| `prod` | | `false` | Deploy også til prod |

| Secret | Beskrivelse |
| --- | --- |
| `READER_TOKEN` | Tilgjengelig i bygget som `READER_TOKEN` og `PASSWORD` |
| `SLACK_WEBHOOK` | Varsler ikke hvis den mangler |

Appens workflow må gi `contents: read` og `id-token: write`.

### Oppsett i dette repoet

- Variabel `MELOSYS_CI_CLIENT_ID` og secret `MELOSYS_CI_PRIVATE_KEY` (GitHub-appen Melosys CI, med `actions: write`)
- Secret `SLACK_WEBHOOK` for varsel når appene ikke kan startes

GitHub skrur av planlagte workflows etter 60 dager uten aktivitet i repoet. Kommer det
varsel om det, må workflowen skrus på igjen under Actions.
