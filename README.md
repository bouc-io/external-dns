# ExternalDNS Deployment

This directory captures the Helm values and operational guidance used to run [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) across Bouc.io environments. ExternalDNS watches Kubernetes resources and reconciles DNS records in Google Cloud DNS so that Istio ingress endpoints stay routable via managed hostnames.

## Layout

- `base.values.yaml` – chart defaults vendored from upstream to document the full value surface.
- `lcl.values.yaml` – local overlay. Mirrors the chart schema but is rendered inert via `replicaCount: 0` in the Flux HelmRelease patch.
- `snbx.values.yaml` – sandbox overlay. Adds Cloud DNS specifics, workload identity annotations, and gateway/domain filters.

> **Note:** Flux consumes these files through the Git submodule living under `fluxcd/fluxcdboucio/clusters/components/external-dns/`. Update this directory first, validate, and then bump the submodule to propagate changes.

## Environment Behavior

### Local (`lcl.values.yaml`)

- Provider: `inmemory` (changed from `noop` to a valid provider for local testing).
- Registry: `inmemory`; avoids TXT writes when experimenting locally.
- Sources: `service` and `istio-gateway`.
- Flux patch (`fluxcd/fluxcdboucio/clusters/local/infrastructure/fluxcd-external-dns.yaml`) sets `spec.values.replicaCount: 0`, so no pods run unless explicitly overridden.

To test locally:

1. Temporarily set `replicaCount: 1` (or remove the override) in the Flux patch.
2. Provide credentials (either enable workload identity in your cluster or mount a JSON key via `extraEnv` / `secretConfiguration`).
3. Reconcile Flux or run `helm upgrade --install` manually with the same values.
4. Restore the `replicaCount: 0` override when finished.

### Sandbox (`snbx.values.yaml`)

- Provider: `google`; registry: `txt`.
- TXT ownership: `_external_dns` prefix and `external-dns-boucio` owner ID.
- Domain filter: `bouc.io` reduces API churn.
- Sources: `service` and `istio-gateway` so both Kubernetes Services and Istio Gateways drive DNS records.
- Workload Identity: `serviceAccount.annotations` includes `iam.gke.io/gcp-service-account: external-dns@${GCP_PROJECT_ID}.iam.gserviceaccount.com`. `GCP_PROJECT_ID` is resolved by Flux `postBuild` substitution from the `cluster-vars` ConfigMap, so it must be set on sandbox clusters. If it is unset, Flux renders the annotation blank (`external-dns@.iam.gserviceaccount.com`) and the GKE metadata server rejects token requests with a 400 before any Cloud DNS call is made.
- Extra args: `--google-zone-visibility=public`; extend as needed (e.g., `--google-project` or `--source=ingress`).

Istio gateway annotations for sandbox live in `infrastructure/istio/sandbox/sandbox.ingress-gateway.yaml`. Example:

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.bouc.io,api.bouc.io,www.bouc.io,sso.bouc.io,prometheus.bouc.io,grafana.bouc.io,kiali.bouc.io,chat.bouc.io
```

Ensure every hostname you expect ExternalDNS to manage is listed, or annotate individual `VirtualService` / `Service` resources instead.

## Prerequisites

1. **Google Cloud DNS Zone** – Public zone for `bouc.io` (or your target domain) with Namecheap delegating to Google nameservers.
2. **Workload Identity Binding** – Grant `roles/dns.admin` to the Google service account and bind it to the Kubernetes service account:
   ```bash
   gcloud iam service-accounts add-iam-policy-binding \
     external-dns@bPROJECT_ID.iam.gserviceaccount.com \
     --role roles/iam.workloadIdentityUser \
     --member "serviceAccount:PROJECT_ID.svc.id.goog[external-dns/external-dns]"
   ```
3. **Flux Post-build Substitutions (optional)** – If the project ID changes, abstract the service account email behind a ConfigMap/Secret and reference it via `postBuild.substituteFrom` in the Flux Kustomization.

### Creating the Google Cloud Service Account

If the Google Cloud Service Account (GSA) for ExternalDNS does not exist, you need to create it and grant the necessary permissions.

1.  **Create the Google Cloud Service Account:**

    ```bash
    gcloud iam service-accounts create external-dns --display-name="ExternalDNS service account" --project=PROJECT_ID
    ```

2.  **Grant the DNS Administrator Role to the GSA:**

    ```bash
    gcloud projects add-iam-policy-binding PROJECT_ID --member="serviceAccount:external-dns@PROJECT_ID.iam.gserviceaccount.com" --role="roles/dns.admin"
    ```

3.  **Bind the Kubernetes Service Account to the GSA:**

    ```bash
    gcloud iam service-accounts add-iam-policy-binding external-dns@PROJECT_ID.iam.gserviceaccount.com --role="roles/iam.workloadIdentityUser" --member="serviceAccount:PROJECT_ID.svc.id.goog[external-dns/external-dns-release]"
    ```
    > **Note:** The member in the command above assumes that the Kubernetes service account is named `external-dns-release` and is in the `external-dns` namespace. If you have a different release name or namespace, you will need to adjust the `member` flag accordingly.

## Deployment Flow

1. **Edit values** here (`lcl.values.yaml`, `snbx.values.yaml`) to adjust behavior.
2. **Validate** with `helm template` or `kustomize build --load-restrictor=LoadRestrictionsNone` to ensure the rendered chart accepts the values.
3. **Update submodule** (`fluxcd/fluxcdboucio/clusters/components/external-dns`) to pull in the new values.
4. **Commit** both this directory and the submodule bump.
5. **Flux Reconcile** (`flux reconcile kustomization local-infra` or `sandbox-infra`) to push the changes to the cluster.

## Verification Checklist

- `kubectl -n external-dns get pods` – sandbox shows running pods; local shows zero replicas.
- `kubectl logs -n external-dns deploy/external-dns-helmrelease` – no auth errors; log level JSON INFO per standards.
- `gcloud dns record-sets list --zone=<ZONE_NAME>` (or Cloud console) – expected records exist.
- `dig host.bouc.io` – resolves to the Istio ingress IP.
- TXT records `_external_dns.<hostname>` exist with the configured owner ID.

## Logging & Observability

- ExternalDNS outputs JSON logs (`logFormat: json`). Ensure log collectors parse the fields, especially when tailing via `kubectl logs`.
- Add Prometheus scraping via the chart’s optional `metrics.service` and `serviceMonitor` sections if deeper metrics are required.
- For incident triage, capture canonical summary logs whenever a reconcile loop completes; align with Bouc.io’s logging standards.

## References

- https://github.com/bitnami/charts/blob/main/bitnami/external-dns/README.md
- https://github.com/bitnami/charts/blob/main/bitnami/external-dns/values.yaml
- https://youtu.be/wLHegOz_aR4?si=AjF-2yvyUc52a63p
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/sources/istio.md
- https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/gke/

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
