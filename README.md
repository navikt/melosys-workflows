# melosys-workflows

Felles GitHub Actions-workflows for Team Melosys sine apper.

## `rebygg-og-deploy.yml`: jevnlig rebygg med nytt baseimage

Chainguard-baseimagene får jevnlig sikkerhetsoppdateringer uten at taggen endres, og
Chainguard backporter ikke til eldre bygg. Appene må derfor bygges og deployes på nytt
jevnlig, også når det ikke er endringer i egen kode
([Security Playbook](https://sikkerhet.nav.no/docs/verktoy/chainguard-dockerimages)).

Workflowen:

1. Finner siste commit på standardbranchen og commiten for siste publiserte release, og skriver
   oppsummering med commiten som blir erstattet og den nye.
2. Bygger standardbranchen med tester, og deployer til dev-miljøene i rekkefølge. Neste miljø deployes bare hvis forrige gikk bra.
3. Hvis `prod` er valgt: bygger siste release på nytt med tester, og deployer den til prod.
4. Sender Slack-varsel med status for dev og prod.

Kode som ikke er releaset, kommer aldri i prod: prod får alltid siste publiserte release
(også prerelease, som `deploy-prod` i appene, med mindre `uten-prerelease` er satt).
Unntaket er apper med `prod-fra-standardbranch`, som selv deployer standardbranchen til prod
uten release. Der får prod samme image som dev, og bare hvis dev-deployen gikk bra.
Rett før deployen til dev og til prod sjekkes det at commiten fortsatt er den siste. Har en
vanlig deploy kommet med nyere kode mens vi bygget, hoppes deployen over, så den ikke rulles
tilbake.

Dev og prod bygges og deployes uavhengig av hverandre. De bygger ulik kode, og rebygget
endrer bare baseimaget. Prod-bygget kjører testene på release-commiten, og Nais venter på
readiness ved deploy, så en app som ikke starter på det nye baseimaget, blir ikke rullet ut.
En feil eller en hoppet deploy i dev skal derfor ikke holde prod igjen med gammelt baseimage.
Imaget får bare den unike datotaggen fra `nais/docker-build-push` og deployes med digest;
sha-taggen peker fortsatt på imaget som ble testet ved merge.

### Kjente begrensninger

- **Samtidige deployer:** sjekken over gjøres sekunder før deployen, men workflowen deler ikke
  lås med appens egne deploy-workflows. Publiseres en release eller merges noe akkurat i det
  vinduet, eller mellom to dev-miljøer, kan rebygget overskrive den nyere deployen. Den rettes
  ved neste vanlige deploy.
- **Manuell deploy til prod:** workflowen ser ikke hva som faktisk kjører i prod, bare siste
  publiserte release. Er en nyere commit deployet til prod manuelt etter siste release, ruller
  rebygget prod tilbake til releasen.
- **Forrige commit i oppsummeringen:** hentes fra siste vellykkede kjøring av deploy-workflowene
  (og tidligere rebygg for dev), ikke fra clusteret. Manuelle deployer kommer ikke med.

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
        description: Deploy også til prod
        type: boolean
        default: false

permissions:
  contents: read
  packages: read
  id-token: write
  actions: read

concurrency:
  group: rebygg-og-deploy
  cancel-in-progress: false

jobs:
  rebygg:
    uses: navikt/melosys-workflows/.github/workflows/rebygg-og-deploy.yml@main
    with:
      bygg-kommando: ./mvnw clean package -B -e --settings .github/settings.xml
      byggverktoy: maven
      dev-cluster: dev-fss
      dev-vars: |
        nais/vars-q2.json
        nais/vars-q1.json
      prod-cluster: prod-fss
      dev-deploy-workflow: deploy-dev.yaml
      prod-deploy-workflow: deploy-prod.yaml
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
| `node-versjon` | | | Setter opp Node før bygget, f.eks. `lts/*`. `NODE_AUTH_TOKEN` får `READER_TOKEN` |
| `pnpm-versjon` | | | Setter opp pnpm før bygget, f.eks. `10` |
| `dev-cluster` | ja | | f.eks. `dev-fss` eller `dev-gcp` |
| `dev-vars` | ja, uten `uten-vars` | | Én til tre vars-filer, én per linje, i deployrekkefølge |
| `prod-cluster` | ja | | f.eks. `prod-fss` eller `prod-gcp` |
| `prod-vars` | | `nais/vars-prod.json` | |
| `uten-vars` | | `false` | Appen har ingen vars-filer. Dev og prod deployes uten vars |
| `dev-var` | | | Template-variabler for dev, `key=value` kommaseparert (`VAR` i nais/deploy) |
| `prod-var` | | | Template-variabler for prod, `key=value` kommaseparert |
| `nais-resource` | | `nais/nais.yml` | |
| `dev-resource` | | | Nais-ressurs for dev, når den er en annen enn i prod. Overstyrer `nais-resource` |
| `prod-resource` | | | Nais-ressurs for prod, når den er en annen enn i dev. Overstyrer `nais-resource` |
| `team` | | `teammelosys` | |
| `platforms` | | `linux/amd64` | |
| `runner` | | `ubuntu-latest` | Runner for byggejobbene |
| `prod` | | `false` | Deploy også til prod |
| `uten-prerelease` | | `false` | Prod får siste release som ikke er pre-release, for apper der deploy-prod hopper over prereleaser |
| `prod-fra-standardbranch` | | `false` | For apper uten releaser som deployer standardbranchen til både dev og prod. Prod får dev-imaget etter at dev-deployen gikk bra |
| `dev-deploy-workflow` | | | Appens vanlige deploy-workflow til dev. Uten den vises forrige commit i dev bare fra tidligere rebygg |
| `prod-deploy-workflow` | | | Appens vanlige deploy-workflow til prod. Uten den vises forrige commit i prod som ukjent |

| Secret | Beskrivelse |
| --- | --- |
| `READER_TOKEN` | Tilgjengelig i bygget som `READER_TOKEN` og `PASSWORD` |
| `SLACK_WEBHOOK` | Varsler ikke hvis den mangler |

Appens workflow må gi `contents: read`, `packages: read`, `id-token: write` og `actions: read`. Hver jobb i
den felles workflowen får bare de tilgangene den trenger. Kjøres den fra en annen branch
enn standardbranchen, hoppes alt over.

GitHub skrur av planlagte workflows etter 60 dager uten aktivitet i repoet. Kommer det
varsel om det, må workflowen skrus på igjen under Actions i appen.

## Versjoner

Appene bruker `@main`, som i dok-workflows. Endringer gjelder alle appene så snart de er
merget, og `main` krever derfor PR med godkjenning.
