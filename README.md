# etterlatte-gcp-migrering

Verktøy for å gjennomføre databasemigrering mellom Google Cloud SQL databaser på en sikker måte.

Med denne fremgangsmåten er det også mulig å velge spesifikke tabeller som skal migreres. Dette i motsetning til andre verktøy som kun tilbyr
migrering av hele databaser.

Obs merk at denne applikasjonen er begrenset til maks 512 mb.
Om man deployer med en ugydlig config må man sjekke loggene til applikasjonsressursen evt serviceressursen som blir opprettet i
tillegg til podden.

## Plan for databasemigrering

Skal gjøres i dev før produksjon - alltid!

## Forarbeid

1. Flytt all logikk og trafikk til målappen. Sørg for at alle apper går via den nye appen før databasen migreres. Dette deler migreringen i to uavhengige steg og gjør rollback enklere.
2. Eksporter skjemadefinisjoner for tabellene som skal flyttes
    ```shell
    pg_dump -h localhost -p 5432 -U <BRUKER> -d <KILDEDATABASE> --schema-only --exclude-table-data=flyway_schema_history
    ```
3. Få opprettet skjema-innhold i den databasen man skal flytte data inn i (helst via Flyway)

   Merk: Det kan være lurt å tenke på om man vi vil legge på indeksene i etterkant da
   insert med hele tabellen sammen med indeksering kan ta lang tid. Sjekk indekstørrelse opp mot tabellstørrelse for
   å avgjøre dette evt en test i dev/lokalt.
4. Lag servicebruker i GCP med SQL admin-tilganger (hvis den ikke allerede finnes): https://console.cloud.google.com/iam-admin/serviceaccounts/create
5. Legg til servicebrukeren manuelt i Cloud Console for BEGGE databaser:

   Gå til: ```SQL instances → Users → Add user account```

   OBS: Sjekk hva eksisterende bruker  heter:
    * Eksempel: 
      * Fullt navn``migrering-user@<PROSJEKT_ID>.iam.gserviceaccount.com``
      * Kort navn:``migrering-user@<PROSJEKT_ID>.iam``
6. Grant tilganger på kildedatabasen (helst via Flyway):
    ```postgresql
    GRANT USAGE ON SCHEMA public TO "cloudsqliamserviceaccount";   
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO "cloudsqliamserviceaccount";
    GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO "cloudsqliamserviceaccount";
   ```
7. Grant tilganger på måldatabasen (helst via Flyway):
    ```postgresql
    GRANT USAGE ON SCHEMA public TO "cloudsqliamserviceaccount";
    GRANT INSERT ON ALL TABLES IN SCHEMA public TO "cloudsqliamserviceaccount";
    GRANT UPDATE ON ALL SEQUENCES IN SCHEMA public TO "cloudsqliamserviceaccount";
    ```
8. Verifiser at brukeren har riktige tilganger med kommandoen \dp i psql.

### Forarbeid - migreringspod

1. Klon repo:
   ```shell
   git clone https://github.com/navikt/etterlatte-gcp-migrering
   ```
2. Generer nøkkel for servicebruker
    * Gå til IAM → Service Accounts → Keys → Add key → JSON i GCP.
    * Lim inn JSON-nøkkelen i secret.yaml.
3. Apply secret for servicebruker:
   ```shell
   kubectl apply -f secret.yaml
   ```
4. Start migreringspod:
   ```shell
   kubectl apply -f gcloud.yaml
   ```
5. Oppdater network policy med riktige IP-adresser for BEGGE databaser. IP-adresser finner man i oversikten over databaseinstanser i GCP Console. Oppdater network-dev.yaml eller network-prod.yaml med riktige public IP-adresser.
6. Apply network policy
   ```shell
   kubectl apply -f network-<dev/prod>.yaml
   ```
7. Restart pod slik at den leser inn secret og network policy
    * ``kubectl scale deployment gcloud --replicas=0``
    * Vent til pod er slått av, kjør deretter:
    * ``kubectl scale deployment gcloud --replicas=1``

## Selve migreringen
1. Scale ned target apper (kun nødvendig hvis vi faktisk skal flytte data):
    ```shell
   kubectl scale deployment <KILDEAPP> --replicas=0
   ```
2. Skru av audit-logging på databasen vi skriver til, slik at vi ikke spammer loggen. DETTE SKAL SKRUS PÅ IGJEN UANSETT UTFALL !!
3. Ta backup av begge databaser før du begynner. Dette kan du gjøre i GCP.
4. Finn navn på migreringspod:
   ```shell
   kubectl get pods | grep gcloud
   ```
5. Exec inn i pod:
   ```shell
   kubectl exec -it <PODNAVN> -c gcloud -- sh
   ```
6. Autentiser servicebrukeren:
   ```shell
   gcloud auth activate-service-account <SERVICE_USER_FULLT_NAVN> --key-file /var/run/secrets/nais.io/migration-user/user
   ```
7. Sett prosjekt (ved behov, kjør ``gcloud projects list``):
    ```shell
    gcloud config set project <PROJECT_ID>
    ```
8. Finn connection name for kildedatabasen:
   ```shell
   gcloud sql instances describe <KILDEINSTANS> --format="get(connectionName)" --project <PROJECT_ID>
   ```
   Eksempel dev:
   ```shell
   gcloud sql instances describe etterlatte-vedtaksvurdering --format="get(connectionName)" --project <PROJECT_ID>
   ```
9. Start proxy mot kildedatabasen:
   ```shell
   cloud_sql_proxy -enable_iam_login -instances=<CONNECTION_NAME>=tcp:5432 &
   ```
10. Test tilkobling mot kildedatabasen (valgfritt):
   ```shell
   psql -h localhost -p 5432 -U <SERVICEBRUKER_KORTNAVN> -d <KILDEDATABASE>
   ```
11. Dump data fra kildedatabasen:
   ```shell
   pg_dump --format=custom -h localhost -p 5432 -U <USER> -d vedtaksvurdering -f /data/dump-custom.sql --data-only --exclude-table-data=flyway_schema_history && echo "data er dumpet"
   ```
12. Drep cloud_sql_proxy for å frigjøre port 5432:
    ```shell
    ls -l /proc/*/exe
    ```
    Finn PID fra path (eks. /proc/123/exe betyr PID er 123), kjør deretter:
    ```shell
    kill -9 <PID>
    ```
13. Finn connection name for måldatabasen:
    ```shell
    gcloud sql instances describe <MÅLINSTANS> --format="get(connectionName)" --project <PROJECT_ID>
    ```
14. Start proxy mot måldatabasen:
    ```shell
    cloud_sql_proxy -enable_iam_login -instances=<CONNECTION_NAME>=tcp:5432 &
    ```
15. Last opp data til måldatabasen:
    ```shell
    pg_restore -h localhost -U <USER> -d sakogbehandlinger -p 5432 --data-only /data/dump-custom.sql
    ```
16. Slett SQL-dump fra pod:
    ```shell
    rm /data/dump.sql
    ```
17. Scale opp kilde- og målapper igjen:

    Dette fordi vi får overvåkningsfeil hvis kildeappen ikke kjører
    ```shell
    kubectl scale deployment <KILDEAPP> --replicas=1
    kubectl scale deployment <MÅLAPP> --replicas=1
    ```
18. Skru på igjen audit-logging
    Merge PR med audit-logging-properties på i dev|prod.yaml, og (for å få riktig filtrering på hva som skal auditlogges):
    ```shell
    nais postgres enable-audit <MÅLAPP>
    ```

## Verifisering

Sjekk at audit-logging ble skrudd på:
```shell
nais postgres verify-audit MÅLAPP> --team etterlatte
```
Sjekk størrelser på tabeller og indekser i måldatabasen.
Sjekk at tegnsett ble riktig, at æ, ø og å er som forventet.

Det beste er å sjekke mot egne verdier, men Copilot anbefaler noe som dette:
```postgresql
SELECT relname AS table_name, pg_size_pretty(pg_total_relation_size(relid)) AS "Total Size", pg_size_pretty(pg_indexes_size(relid)) AS "Index Size", pg_size_pretty(pg_relation_size(relid)) AS "Actual Size" FROM pg_catalog.pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC;
```

## Opprydding
Når migrering er fullført og verifisert:
```shell
kubectl delete -f gcloud.yaml
kubectl delete -f secret.yaml
kubectl delete -f network-<dev/prod>.yaml

# OBS: Slett secrets som er lagt inn for service-bruker i GCP-console 
# ENDA MERE OBS: SKRU PÅ AUDITLOGGING!
```

## Kjente feil

#### Feil: "error: invalid command \N"

Dette er ikke en faktisk \N-feil. Det kan være en tilgangsfeil som skjer tidlig i kjøringen.
Det du egentlig vil se i loggen er noe slikt:

ERROR: permission denied for table behandling_versjon

Løsning: Gi cloudsqliamserviceaccount tilgang til tabellen som feiler. Se Forarbeid - Databaser

#### Feil: "password authentication failed"

Betyr at servicebrukeren ikke er lagt til i database-instansen.
Løsning: Se Forarbeid - Databaser steg 4.

#### Feil: "Number of retries exceeded while attempting to acquire PostgreSQL advisory lock"

Skjer hvis to pods prøver å kjøre Flyway-migrering samtidig.
Løsning: Konfigurer lockRetryCount eller sørg for at kun én pod kjører av gangen.

#### Feil: Ikke tilgang til nye tabeller selv om tilgang er gitt tidligere

Nye tabeller arver ikke eksisterende GRANT. Tilganger må gis på nytt.
Løsning: Kjør GRANTkommandoene på nytt. Se Forarbeid - Databaser steg 5 og 6.
