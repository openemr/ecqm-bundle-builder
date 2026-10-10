# eCQM measure bundle builder

OpenEMR calculates eCQMs from a **measure bundle** for each performance period:
for every measure, the measure JSON (including its CQL, compiled as ELM) and its
value sets expanded from VSAC. This tool builds those bundles from CMS's eCQM
packages with [cqm-parsers](https://github.com/sjpadgett/cqm-parsers), in a
Docker container, so no Ruby setup is needed.

The bundles are published in the
[`openemr/oe-cqm-parsers`](https://github.com/openemr/oe-cqm-parsers) Composer
package, which OpenEMR reads from
`vendor/openemr/oe-cqm-parsers/{year}_reporting_period/json_measures`. This
directory holds only the tool; measure packages and generated bundles are never
committed here.

## What you need

- Docker with Compose v2.24 or later.
- A free [UMLS account](https://uts.nlm.nih.gov/uts/signup-login) and its API
  key (UTS profile page). VSAC uses it to expand value sets.
- The CMS eCQM packages for the period: on the
  [eCQI Resource Center](https://ecqi.healthit.gov/), choose
  **Eligible Clinician** eCQMs for the performance period and download the
  measure packages. Each measure is a zip, for example
  `CMS122-v14.0.000-QDM.zip`.

## Build a performance period

1. Set up once:

   ```bash
   cd contrib/ecqm-bundle-builder
   cp .env.sample .env    # then fill in VSAC_API_KEY
   ```

   To update the published bundles directly, set `BUNDLE_HOST_DIR` in `.env` to
   a checkout of `openemr/oe-cqm-parsers`. Otherwise bundles go to `./bundles`.

2. Put the packages in `{BUNDLE_HOST_DIR}/{year}_reporting_period/cms_measures/`.

3. Build:

   ```bash
   docker compose run --rm builder ruby build_bundle.rb --year 2026
   ```

   Each package becomes `json_measures/{cms_id}/{cms_id}.json` and
   `json_measures/{cms_id}/value_sets.json`. Measures already built are skipped,
   so running again retries only the ones that failed. Failures are listed at
   the end and written to `failed_measures_{timestamp}.log` in the period
   directory. VSAC errors are usually temporary. `--rebuild` builds everything
   again.

4. Generate the measure list for OpenEMR's QRDA screens:

   ```bash
   docker compose run --rm builder ruby generate_ecqm_list.rb --year 2026
   ```

   This writes `ecqm_2026_reporting.sql` next to the period directories,
   guarded with `#IfNotRow` so it can go into an upgrade script as it is.
   Use `--defaults CMS122v,CMS165v,...` to choose which measures are active by
   default.

### Value set profile

Each period's value sets come from the eCQM Annual Update **published the year
before** (the May update is for the next performance period):

| Performance period | VSAC profile |
|---|---|
| 2026 | eCQM Update 2025-05-08 |
| 2025 | eCQM Update 2024-05-02 |
| 2024 | eCQM Update 2023-05-04 |
| 2023 | eCQM Update 2022-05-05 |

`build_bundle.rb --help` prints the table the script uses. For a new period, add
its line to `VSAC_PROFILES` in `build_bundle.rb`, or pass `--profile`. A wrong
profile still builds, but its value sets are wrong, so results are silently
wrong.

### Empty value sets

cqm-parsers fails a measure when VSAC expands one of its value sets to
nothing. Some packages still declare a retired value set they no longer use:
CMS157v14 (2026) declares the legacy ONC Administrative Sex
(`2.16.840.1.113762.1.4.1`) beside its replacement, Federal Administrative Sex,
and the 2025-05-08 profile expands it to nothing. Value sets listed in
`ALLOWED_EMPTY_VALUE_SETS` in `build_bundle.rb`, or passed with
`--allow-empty-value-set OID`, are kept empty instead. The build still fails the
measure if its logic (any `ValueSetRef` in its ELM) uses an empty value set, so
an empty value set can never silently drop data.

### Directory names

OpenEMR loads a measure from `json_measures/{option_id}/{option_id}.json`, and
the measure list's `option_id` is the measure's `cms_id` (for example
`CMS122v14`). Package file names changed in 2026 (`CMS122-v14.0.000-QDM.zip`),
so directories are always named by `cms_id`, never by the package file. To fix a
period built before this rule:

```bash
docker compose run --rm builder ruby build_bundle.rb --year 2026 --rename-existing
```

## Ship it

1. Commit the new `{year}_reporting_period` directory to `openemr/oe-cqm-parsers`
   and tag a release.
2. In OpenEMR, bump `openemr/oe-cqm-parsers` in `composer.json` and run
   `composer update openemr/oe-cqm-parsers`.
3. Add the list SQL to the current `sql/*-upgrade.sql` and to `sql/database.sql`.

## Verify it

- `vendor/bin/phpunit -c phpunit-isolated.xml tests/Tests/Isolated/Services/Qdm/MeasureCalculationMethodTest.php`
  checks every measure of every bundle.
- Parity with the cqm-execution reference: generate test fixtures for the
  period, then run `tests/Tests/Isolated/Cqm/Parity/PhpEngineParityTest.php`.
- End to end on a test database:
  `OPENEMR_ENABLE_CQM_TEST_PATIENTS=1 php bin/console openemr:cqm-load-test-patients --year=2026 --verify`.
- Cypress test decks for the period, checked with `tests/Tests/ECQM/MeasureResultsTest.php`.

## Notes

- Only the builder container is needed. cqm-parsers builds measures and value
  sets in memory, so no database is involved.
- `Gemfile.lock` pins every gem, including the cqm-parsers commit, so a measure
  rebuilt from the same package gives the same measure JSON apart from record
  ids, which are generated on each build.
- If you change `build_bundle.rb` or `generate_ecqm_list.rb`, run
  `docker compose build` to rebuild the image.
