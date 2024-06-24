# Introduction

The documentation for the `Data Access` building block is organised as follows...

* **Introduction**<br>
  Introduction to the BB - including summary of purpose and capabilities.
* **Getting Started**<br>
  Quick start instructions - including installation, e.g. of a local instance.
* **Design**<br>
  Description of the BB design - including its subcomponent architecture and interfaces.
* **Usage**<br>
  Tutorials, How-tos, etc. to communicate usage of the BB.
* **Administration**<br>
  Configuration and maintenance of the BB.
* **API**<br>
  Details of APIs provided by the BB - including endpoints, usage descriptions and examples etc.


## About `Data Access`

The Data Access building block provides feature-rich and reliable interfaces to geospatial data assets stored in the platform, addressing human and machine users alike.

The builing block combines capabilities from two complementary libraries


## Capabilities

### Data access

- Support for retrieval of features via OGC API Features and OGC WFS
- Support for retrieval of coverages via OGC API Coverages and OGC WCS
- Support for visualization of coverages and features via OGC API Tiles, OGC WMTS, OGC API Maps, OGC WMS
- Support of assets persisted via common storage technologies including S3-comptabile object storage, HTTP, file system
- Support for multidimensional data formats for data access and visualization

### Data administration

- Administration User Interface

### Configuration and integration

- Dynamic specification of which datasets should be delivered with which data access services
- Designed that it can be integrated as embedded capability into the APIs other building blocks