# global-certs

## Purpose

This chart creates the shared wildcard Certificates, the internal CA Certificate,
and the InfisicalPushSecret that exports the CA key pair. It uses its own
templates. It has no upstream chart, dependency archive, or CRDs. `appVersion`
is the resource bundle version, not a cert-manager version.

## Before first install

1. Install cert-manager, the ClusterIssuers, the Infisical operator, and reflector.
   This chart does not install these controllers.
2. Make sure `universal-auth-credentials` exists in namespace `infisical`.
   Do not put credentials or certificate private keys in chart values.
3. Review `cluster-config/apps/dev/global-certs.yaml`. It selects namespace-bound
   Secret names, domains, issuers, and the Infisical connection.
4. Build and publish the chart through CI. Replace `PENDING_CI_VERSION` in
   `cluster-config/argo/dev/global-certs.yaml` with the first CI chart version.
5. Use manual sync for the Application handover. Review its resource list before
   sync. The new Application uses project `platform`, the existing destination
   `cert-manager`, and release name `global-certs`.

No InfisicalSecret-to-StaticSecret migration is needed. The chart retains the
existing InfisicalPushSecret name. Do not orphan-delete that push resource.

## Values reference

| Value | Use |
| --- | --- |
| `internalCA.enabled` | Enable the CA Certificate; default `false`. |
| `internalCA.name` | Fixed cluster-wide default: `raulstechtips-internal-ca`. |
| `internalCA.secretName` | Fixed cluster-wide default: `raulstechtips-internal-ca-key-pair`. |
| `internalCA.commonName` | Fixed cluster-wide default: `k8s-internal-ca.cluster.local`. |
| `internalCA.issuer` | Fixed cluster-wide ClusterIssuer default: `selfsigned-issuer`. |
| `internalCA.pushToInfisical` | Export the CA Secret; requires the CA to be enabled. |
| `internalCA.secretPath` | Export path; default `/certificates/internal`. |
| `wildcardCertificates` | List of `name`, `secretName`, `domain`, `issuer`; empty by default. |
| `secretTemplate.annotations` | Reflector annotations copied to all certificate Secrets. |
| `infisical.projectSlug`, `environmentSlug` | Destination project and environment. |
| `infisical.push.hostAPI` | Infisical API URL. |
| `infisical.push.credentialsRef.name`, `namespace` | Universal-auth Secret reference. |

Each wildcard entry requests its domain and `*.<domain>`. The internal CA keeps
ECDSA 256. All issuer references use `ClusterIssuer`. The export keeps
`includeAllSecrets: true`, `updatePolicy: Replace`, and `resyncInterval: 1m`.
The chart does not set a deletion policy; it retains the live API default.

## Upgrades

Run the shared local checks before publishing:

```sh
export TMPDIR=/tmp/ulw-wrappers/global-certs/tmp
mkdir -p "$TMPDIR"
/Users/raul/Desktop/Projects/HomeLab/.omo/wrapper-charts/tools/checks.sh \
  /Users/raul/Desktop/Projects/HomeLab/global-certs \
  /tmp/ulw-wrappers/global-certs
```

Use `/opt/homebrew/bin/helm`. Render with the environment values and compare with
the live Application before manual sync. Do not change CA key settings or
certificate names as part of an adoption.

## Adoption notes

Verified local result: 19 tests in four suites pass. Kubeconform reports three
valid resources and one skipped InfisicalPushSecret for the all-on values.
The default render is empty. Development parity:
`SUMMARY SAME=0 CHANGED=7 NEW=0 PRUNE=0 HOOK=0 UNKNOWN-KIND=0`.
All changes are the label additions and project selector change listed below.

- All six live Certificates are included. `dev-raulstechtips` is Ready but not
  tracked by an Application. Its local old-folder manifest confirms the intent
  to manage it here. The chart adopts it without a specification change.
- All seven resources receive `app.kubernetes.io/name: global-certs`,
  `app.kubernetes.io/instance: global-certs`, and
  `app.kubernetes.io/managed-by: Helm`. These label additions are expected
  `CHANGED` results for the CA, live, prod, stage, uat, and dev Certificates,
  and the internal CA InfisicalPushSecret. They do not change certificate data.
- The internal CA export uses `destination.projectSlug: k3s-ntpo` instead of
  live `projectId: a86484a5-93d4-4443-a85f-105e822d5973`. The live CRD accepts
  both fields. The live Jenkins export already uses this slug and reports a
  successful reconcile. This is an intended connection-format change to match
  the Jenkins pattern. Confirm the project during handover. Host, credentials,
  environment, source Secret, and destination path retain their live values.
- `infisical-push-secret-live-tls` and `infisical-push-secret-prod-tls` in
  namespace `default` are not adopted. No Application tracks them. They read
  the reflected TLS Secrets in namespace `keycloak` and write
  `KEYCLOAK_LIVE_TLS_CRT`, `KEYCLOAK_LIVE_TLS_KEY`, `KEYCLOAK_PROD_TLS_CRT`, and
  `KEYCLOAK_PROD_TLS_KEY` to `/keycloak/tls`. They are Keycloak-specific export
  adapters, not shared certificate definitions. Leave them unchanged and
  review their ownership with the Keycloak chart separately.
- The new Application removes automated prune and self-heal, uses project
  `platform`, and keeps history limit 10. Live has no sync options or
  ignoreDifferences to preserve. There is no `crds/` folder, so `skipCrds` is
  not needed. There is no ingress in this resource-only chart.
- No cluster changes, Secret-value reads, or Git writes were used. The evidence
  directory `.omo/evidence/wrappers/global-certs/` contains the local checks,
  rendered resources, parity output, and the adoption report.
