# Frontend override — `@openmrs/esm-patient-orders-app`

## What's customized (2 commits)

**1. Translatable priority labels** *(fix)* — the general order form's Priority
dropdown labels come from a frozen module constant (`priorityOptions` in
`@openmrs/esm-patient-common-lib`) that runtime `Translation overrides` cannot
reach. Relabelled at render time from the app's own i18n namespace:

| Urgency | en | fr |
|---|---|---|
| `ROUTINE` | Routine | Routine |
| `STAT` | Stat | **Stat (urgent)** |
| `ON_SCHEDULED_DATE` | Scheduled | **Scheduled (planifié)** |

**2. Laterality support** *(feat)* — order types backed by
`org.openmrs.TestOrder` (e.g. our *Examens d'imagerie* order type) get an
optional **Latéralité** select (Gauche / Droite / Bilatéral) in the general
order form. Such orders are posted with the `testorder` REST subtype carrying
`laterality` — stored natively by OpenMRS core (no backend module needed).
Requires the order type's `Java class name` to be `org.openmrs.TestOrder`
(see `distro/configuration/ordertypes/roland-emr/ordertypes.csv`).

## Where the change lives — GitHub fork

Maintained as commits on our fork (intended as upstream OpenMRS contributions):

**Fork:** <https://github.com/cerveaubleu64/openmrs-esm-patient-chart>

| Branch | Base | Purpose |
|---|---|---|
| [`fix/general-order-priority-labels-v12.2.1`](https://github.com/cerveaubleu64/openmrs-esm-patient-chart/tree/fix/general-order-priority-labels-v12.2.1) | tag `v12.2.1` | **Both commits** — matches the version deployed by this distro; build our module from here |
| [`fix/translatable-general-order-priority-labels`](https://github.com/cerveaubleu64/openmrs-esm-patient-chart/tree/fix/translatable-general-order-priority-labels) | `main` | Upstream PR branch (priority labels only, for now) |

`yarn verify` (lint + typecheck + tests) passes on both branches.
`orders-app-customizations.patch` here is a convenience copy of the full diff
(`git diff v12.2.1..HEAD`).

## Build the patched module from the fork

```sh
git clone --depth 1 --branch fix/general-order-priority-labels-v12.2.1 \
  https://github.com/cerveaubleu64/openmrs-esm-patient-chart.git
cd openmrs-esm-patient-chart
yarn install                                          # Node >= 20
yarn workspace @openmrs/esm-patient-orders-app build
# packages/esm-patient-orders-app/dist/ is a drop-in replacement for the
# assembled openmrs-esm-patient-orders-app-<version>/ directory in the SPA
# (same importmap path, no importmap change needed).
```

Once merged and released upstream, drop this override — the stock npm package
will carry it.
