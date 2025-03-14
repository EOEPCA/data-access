# Health Checks

All services provide health checks for the Resource Health BB or other interested parties to ping.

They are listed here:

| API        | Health Endpoint                                  | Success | Auth    |
|------------|--------------------------------------------------|---------|---------|
| Raster API | https://eoapi.develop.eoepca.org/raster/healthz  | 200     | no auth |
| Vector API | https://eoapi.develop.eoepca.org/vector/healthz  | 200     | no auth |
| STAC API   | https://eoapi.develop.eoepca.org/stac/_mgmt/ping | 200     | no auth |