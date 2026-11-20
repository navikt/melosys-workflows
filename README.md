# melosys-workflows

Felles GitHub Actions-workflows for Team Melosys sine apper.

## `rebygg-og-deploy.yml`: jevnlig rebygg med nytt baseimage

Chainguard-baseimagene får jevnlig sikkerhetsoppdateringer uten at taggen endres, og
Chainguard backporter ikke til eldre bygg. Appene må derfor bygges og deployes på nytt
jevnlig, også når det ikke er endringer i egen kode
([Security Playbook](https://sikkerhet.nav.no/docs/verktoy/chainguard-dockerimages)).

Workflowen:

1. Finner siste commit på standardbranchen og commiten for siste publiserte release.
2. Bygger standardbranchen med tester, og deployer til dev-miljøene i rekkefølge. Neste miljø deployes bare hvis forrige gikk bra.
3. Hvis `prod` er valgt og dev gikk bra: bygger siste release på nytt og deployer den til prod.
   Er releasen samme commit som standardbranchen, brukes dev-imaget.
4. Sender Slack-varsel med status for dev og prod.

Kode som ikke er releaset, kommer aldri i prod: prod får alltid siste publiserte release.
Rett før hver deploy sjekkes det at commiten fortsatt er den siste. Har en vanlig deploy
kommet med nyere kode mens vi bygget, hoppes deployen over, så den ikke rulles tilbake.
Imaget får bare den unike datotaggen fra `nais/docker-build-push` og deployes med digest;
sha-taggen peker fortsatt på imaget som ble testet ved merge.

### Ta i bruk i en app

Legg til `.github/workflows/rebygg-og-deploy.yml` i appen og tilpass valgene:

```yaml
name: Rebygg og deploy
on:
  schedule:
    # Mandag morgen, så teamet er på jobb hvis noe feiler. Ikke på hel time: da er
    # lasten høyest, og GitHub kan forsinke eller i verste fall droppe kjøringen.
    - cron: '17 7 * * 1'
      timezone: Europe/Oslo
  workflow_dispatch:
    inputs:
      prod:
        description: Deploy også til prod (dev deployes alltid først)
        type: boolean
        default: false

permissions:
  contents: read
  packages: read
  id-token: write

concurrency:
  group: rebygg-og-deploy
  cancel-in-progress: false

jobs:
  rebygg:
    uses: navikt/melosys-workflows/.github/workflows/rebygg-og-deploy.yml@v1
    with:
      bygg-kommando: ./mvnw clean package -B -e --settings .github/settings.xml
      byggverktoy: maven
      dev-cluster: dev-fss
      dev-vars: |
        nais/vars-q2.json
        nais/vars-q1.json
      prod-cluster: prod-fss
      # Planlagt kjøring har ingen inputs, og skal deploye til prod.
      prod: ${{ github.event_name == 'schedule' || inputs.prod }}
    secrets:
      READER_TOKEN: ${{ secrets.READER_TOKEN }}
      SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
```

Etter merge: kjør den manuelt uten prod (`gh workflow run rebygg-og-deploy.yml`) før
første planlagte kjøring.

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

Appens workflow må gi `contents: read`, `packages: read` og `id-token: write`. Hver jobb i
den felles workflowen får bare de tilgangene den trenger. Kjøres den fra en annen branch
enn standardbranchen, hoppes alt over.

GitHub skrur av planlagte workflows etter 60 dager uten aktivitet i repoet. Kommer det
varsel om det, må workflowen skrus på igjen under Actions i appen.

## Versjoner

Appene bruker `@v1`. Endringer testes ved å peke én app mot en branch, og `v1` flyttes
når de er godkjent. Endringer som krever at appene endrer oppsett, blir `v2`.
