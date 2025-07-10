# OGC API - Maps

[OGC API - Maps](https://ogcapi.ogc.org/maps/) is the successor to the popular [OGC Web Map Service (WMS)](https://www.ogc.org/standards/wms/) standard. It complements the OGC API - Tiles, OGC WMTS and XYZ map tile standards, to provide an interface for extracting a user-defined map image with applied styling for a specific area of interest, as a single returned image.


## Implementation in the EOEPCA Data Acccess Building Block

We evaluated different approaches to implement the OGC API - Maps standard in the EOEPCA Data Access Building Block. The following options were considered:

1. Extending the existing TiTiler implementation to support OGC API - Maps.
2. Implementing a separate service that plugs into public APIs of TiTiler and STAC.

As the OGC API - Maps specification is rather new (1.0.0 in June 2024), we decided to follow the second approach and work on an [integration into TiTiler](https://github.com/developmentseed/titiler/issues/753) as the standard matures and more clients emerge.

The code for the OGC API - Maps implementation is available in the EOEPCA GitHub repository at <https://github.com/EOEPCA/eoapi-maps-plugin>.

It is using [pygeoapi](https://pygeoapi.io/) to implement the OGC API - Maps interface, and TiTiler to handle the map tile generation.
