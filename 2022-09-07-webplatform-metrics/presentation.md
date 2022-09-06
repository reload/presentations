# Monitorering af Site Status

## Webplatformen

* Lader kommunen udvikle på en række Drupal profiler og drive dem på en cloud platform
* Dækker den fulde lifecycle fra udvikling og push til sitet er live og bliver opdateret

![infra](./img/infra.png)

* Infrastrukturmæssigt:
  * En række repositories og workflows på Github
  * Et par repositories og pipelines i Azure DevOps
  * 3 produktions miljøer (dev, test, prod) med Kubernetes og managed services

### Tal

* Dev + test: Ca 120 sites og 20 servere
* Prod: ca 200 sites og 22
* Lige under 1300 containere i prod
* 10 forskellige Drupal profiler



## Sitefactory

Automatiseringen der kører i et miljø der lader mennesker og automatik opretter, opgradere og nedlægge sites

![cluster.excalidraw](./img/cluster.excalidraw.png)

* Sitefactory
  * Admin (NestJS)
  * API (React Admin)
  * Operators (Go, Ansible, Helm)

* Monitorering
  * Prometheus
  * Loki
  * Grafana
  
## Statuscheck v1

Vi leverede SiteFactory med et statuscheck

* Baseret på CronJob
* Skrev output til stdout som man så med lidt god vilje kunne samle op med Loki

```yaml
command: ["/bin/sh", "-c"]
# Run core-requirements twice. First to get a full status and then
# to detect errors. We wrap the entire output in an json object
# for easier parsing downstream.
# Upon error, exit 1 to fail the job/pod.
args:
  - |
    set -e
    full=$(drush core-requirements --format=json 2> /dev/null)
    errors=$(drush core-requirements --severity=2 --format=json 2> /dev/null)
    echo "{"
    echo "\"full\":${full},"
    echo "\"errors\":${errors}"
    echo "}"
    # Fail if we did not get an empty list of errors.
    if [ ! "${errors}" = "[]" ]; then
      exit 1;
    fi
```

Ikke optimalt
* Performede ekstremt dårligt
* Status var i praksis binær (exitcode)

## Statuscheck v2

Mål

* Gør det muligt for en enkelt person at få overblik over alle sites status, hurtigt.

* Støt den daglige fejlsøgning ved at kunne give indbilk i hvorfor ting fejler.
* Byg det på en måde der understøtter at vi gøre mere af det her og kan automatisere mere og mere.

(Se tidligere cluster overblik)

Vi valgte at basere status-checkket på metrics og events, dvs Prometheus og Loki.

Vi kører nu et fast site-status workload i hvert site namespace

![site-status-Page-1](./img/site-status-Page-1.png)



Prometheus scraper informationer, vi pusher status-rapporter til Loki.

![site-status-Page-2](./img/site-status-Page-2.png)



## Resultat

* Er i dev og test
* Performer langt langt bedre
* Har meget mere potentiale
  * Alarmering
  * Automatisering
  * Andre klienter



## Udfordringer

* Hvornår i piplinen tolker man på data?
