# Access Control

The Data Access building block shares its eoAPI deployment with the [Resource Discovery building block](https://eoepca.readthedocs.io/projects/resource-discovery/). The STAC API of that deployment is protected by [STAC Auth Proxy](https://github.com/developmentseed/stac-auth-proxy), which validates OIDC tokens issued by the IAM building block and enforces collection-level access policies by injecting CQL2 filters into every request.

In brief:

- Collections without a `.` in their ID (e.g. `sentinel-2-l2a`) are **public**: anyone can read them, and only the `stac_editor` service role can write them.
- Collections named `<username>.<collection>` are readable and writable by that user.
- Collections named `<group>.<collection>` are readable and writable by that group's members; a `-ro` group variant grants read-only access.

Since read responses depend on the caller's identity, clients should send their `Authorization: Bearer <token>` header on all STAC API requests, not only on writes. [STAC Manager](https://github.com/developmentseed/stac-manager), the basis of the data administration interface, does this as of `v1.0.0`.

The policy model, enforcement details, and configuration are documented in the Resource Discovery building block: see [Data Catalogue — Access Control](https://eoepca.readthedocs.io/projects/resource-discovery/en/latest/design/data-catalogue/auth/). The policy implementation lives in the [`eoepca-plus` deployment repository](https://github.com/EOEPCA/eoepca-plus/tree/deploy-develop/argocd/eoepca/data-access/parts/stac-auth-proxy).
