# Choice of libraries

A comprehensive list of open source software supporting data access and visualization services is available at
the [OSgeo.org website](https://www.osgeo.org/projects/), including GeoServer, degree, MapServer, pygeoapi, EOxServer amongst others.
The website [stacindex.org](https://stacindex.org/ecosystem?category=Visualization) lists server implementations 
supporting visualization based on STAC. Implementations such as eoAPI, TiTiler, titiler-pgstac, STAC GeoTools Raster Storage (Geoserver) are
listed, which all base on a STAC API or a database with STAC collections and items.


## Decision drivers

### Simplify interdependency by relying on a shared database

The individual building blocks can often not be designed alone,
as all components need to be coordinated with each other to provide homogeneous and performant overall
platform system. As such, the catalogue for EO data with millions of records is designed to be
highly available and scalable with a dedicated focus on STAC metadata. This includes links to all files for
STAC items as assets, which can be used for data access services. To not duplicate millions of database
records, the data access services should rely the same database.

eoAPI has successfully managed millions of assets, indicating its capacity to handle extensive data volumes efficiently.
eoAPI components, including TiTiler, TiPG, and pgSTAC, have been employed in significant projects with
organizations such as NASA (VEDA, CSDA, MAAP), Microsoft's Planetary Computer, and AWS
Sustainability (ASDI). This adoption across high-profile projects highlights eoAPI's role as a preferred
solution for accessing raster and vector data, marking its evolution into a standard approach within the field.

As the STAC Item/Collection structure is very similar to the internal data models of View Server, 
View Server can use pgstac/STAC API as its data backend. This way, the
advanced rendering mechanisms of View Server can easily be ported to a context were a STAC API or
pgstac instance is available, removing a potentially error prone separate database system for View Server
alone.

### Offer performant dynamic tiling

eoAPI offers support for both vector and raster data visualization, underpinned by a modular structure that
integrates various components to facilitate comprehensive data access. These components include TiPG 
for vector data access, TiTiler for raster data, stac-fastapi for accessing STAC metadata, and pgSTAC or
other STAC backends for enhanced metadata management. A notable feature of eoAPI is its capability to
dynamically render metadata queries into detailed mosaics, presenting a visual aggregation of assets for
improved data interpretation and analysis.

eoAPI components, including TiTiler, TiPG, and pgSTAC, have been employed in significant projects with
organizations such as NASA (VEDA, CSDA, MAAP), Microsoft's Planetary Computer, and AWS
Sustainability (ASDI). This adoption across high-profile projects highlights eoAPI's role as a preferred
solution for accessing raster and vector data, marking its evolution into a standard approach within the field.

### Provide a comprehensive set of OGC APIs

The Data Access building block provides the most common OGC API standard interfaces:

* Support for retrieval of features via OGC API Features / OGC WFS
* Support for retrieval of coverages via OGC API Coverages / OGC WCS
* Support for visualization of coverages and features via OGC API Tiles / OGC WMTS, OGC API Maps / OGC WMS

Some of the services are provided by eoAPI with TiTiler and TiPg, in particular raster and vector tiles. 
Coverages is a different concept that goes beyond the simple tile API standards, which does not match the logic
of eoAPI, but instead EOX View Server, such that this libary is included in the stack.

View Server provides a technical baseline for rich data exploitation which forms the basis of data access in
the original EOEPCA system. It allows to create rich visualizations of upstream data, user uploaded data as
well as scientific data products. These visualizations and data renderings are configurable and are exposed
as coverages or layers via a range of OGC compliant web services (WMS/WFS). In addition, an
OpenSearch interface allows to browse the collections and records in an efficient manner. View Server has
its own database model, split into Collections, Products, Coverages and a range of supporting tables. This
is very versatile, and allowed the wide range of features View Server provides and allows to connect to
various storage backends such as S3, FTP, HTTP and others.