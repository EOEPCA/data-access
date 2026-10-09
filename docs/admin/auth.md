# STAC API Access Control

The STAC API decides, collection by collection, who may read and who may write. This is done by [STAC Auth Proxy](https://github.com/developmentseed/stac-auth-proxy) using logins from the IAM building block. Public collections can be read without logging in.

This only applies to the STAC API, not to the raster, vector and multidim APIs.

Clients should send their token (`Authorization: Bearer <token>`) with every STAC request, not just when writing. [STAC Manager](https://github.com/developmentseed/stac-manager) does this automatically.

The STAC API is shared with the [Resource Discovery building block](https://eoepca.readthedocs.io/projects/resource-discovery/), which documents the details:

- [Who can access what](https://eoepca.readthedocs.io/projects/resource-discovery/en/latest/design/data-catalogue/auth/)
- [How to deploy it](https://eoepca.readthedocs.io/projects/deploy/en/latest/building-blocks/data-access/#stac-api-access-control)
- [How to scale the proxy](autoscaling.md#stac-auth-proxy)
