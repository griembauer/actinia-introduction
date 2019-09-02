*Main page of this document: See [https://neteler.gitlab.io/actinia-introduction/](https://neteler.gitlab.io/actinia-introduction/)*

# actinia tutorial at Geostat 2019

Author: Markus Neteler, mundialis GmbH & Co. KG, Bonn

*Last update: 2 Sep 2019*

TODO:

* set up n geostat users properly
* check geo data
* Definition of deployment
* endpoints

## Abstract

Actinia ([https://actinia.mundialis.de/)](https://actinia.mundialis.de/)) is an open source REST API for scalable, distributed, high performance processing of geographical data that uses mainly GRASS GIS for computational tasks. Core functionality includes the processing of single and time series of satellite images, of raster and vector data. With the existing (e.g. Landsat) and Copernicus Sentinel big geodata pools which are growing day by day, actinia is designed to follow the paradigm of bringing algorithms to the cloud stored geodata. Actinia is an OSGeo Community Project since 2019. In this course we will briefly introduce some Geo and EO data basics and give a short introduction to REST API and cloud processing concepts. This is followed by an introduction to actinia processing along with hands-on to get more familiar with the topic by exercises.

## Required software for this tutorial

* Chrome/Chromium browser
* RESTman extension: [https://chrome.google.com/webstore/detail/restman/ihgpcfpkpmdcghlnaofdmjkoemnlijdi](https://chrome.google.com/webstore/detail/restman/ihgpcfpkpmdcghlnaofdmjkoemnlijdi)

Note: We will use the demo actinia server at [https://actinia.mundialis.de/](https://actinia.mundialis.de/) - hence Internet connection is required.

# Geostat 2019 tutorial

Planned tutorial time: 2:30 hs = 150 min

## Introduction

(10 min)

For this tutorial we assume working knowledge concerning geospatial and Earth observation. The tutorial includes, however, a brief introduction to REST (Representational State Transfer) API and cloud processing related basics.

<!--
### Geo and EO basics

* geodata layers
    * raster
    * vector
    * timeseries of both
    * image data (aerial, drone, satellite, ...)
* single and multispectral data (2-D arrays of reflectance values)
-->

### Why cloud computing ?

With the tremendous increase of available geospatial and Earth observation lately driven by the Copernicus programme (esp. Sentinel satellites) and increasing availability of open data the need for computational resources is growing in a non-linear way.

Cloud technology offers a series of **advantages**:

* scalable, distributed, and high performance processing
* large quantities of Earth Observation (EO) and geodata provided in dedicated cloud infrastructures
* addressing the paradigm of computing next to the data
* no need to bother yourself with the low-level management of petabytes of data

Still, some critical **issues** have to be addressed:

* lack of Analysis-Ready-data (ARD) available for consumption in the cloud
* lack of compatibility between different data systems
     * we are on it: openEO H2020 project[openEO H2020 project](https://openeo.org)
 * lack of cloud abstraction, for easier move between vendors and providers

### Overview actinia

Actinia ([https://actinia.mundialis.de/)](https://actinia.mundialis.de/)) is an **open source REST API for scalable, distributed, high performance processing of geospatial and Earth observation data** that uses mainly GRASS GIS for computational tasks. Core functionality includes the processing of single and time series of satellite images, of raster and vector data. With the existing (e.g. Landsat) and Copernicus Sentinel big geodata pools which are growing day by day, actinia is designed to follow the paradigm of bringing algorithms to the cloud stored geodata. Actinia is an OSGeo Community Project since 2019. The source code is available on GitHub at [https://github.com/mundialis/actinia_core](https://github.com/mundialis/actinia_core). It is written in Python and used Flask, Redis, and other components.

**Functionality beyond GRASS GIS**

While at time actinia is mainly a REST interface to GRASS GIS it offers through wrapping the possibility to extend its functionality with other software (ESA SNAP, GDAL, ...). Extensions are added by writing a GRASS GIS Addon Python script which then includes the respective function calls of the software to be integrated.

**Persistent and ephemeral databases**

With **persistent storage** we consider a data storage which keeps data also in case of shutoff as well as keeping them without a scheduled deletion time. In the Geo/EO context, persistent storage is used to provide, e.g. the base cartography, i.e. elevation models, street networks, building footprints, etc.
The **ephemeral storage** is used for on demand computed results including user generated data and temporary data as occurring in processing chains. In an ephemeral storage data are only kept for a limited period of time (e.g., for 24 hs).

In the cloud computing context this is relevant as cost incurs when storing data.

Accordingly, actinia offers two modes of operation: persistent and ephemeral processing. In particular, the **actinia server** is typically deployed on a server with access to a persistent GRASS GIS database (PDB) and optionally to one or more GRASS GIS user databases (UDB).

The actinia server has access to compute nodes (**actinia nodes**; separate physically distinct machines) where the actual computations are performed.The actinia server acts as a **load balancer**, distributing jobs to actinia nodes. Results are either stored in GRASS UDBs in GRASS native format or directly exported to a different data format (see Fig. 1).

<center>
<a href="img/actinia_PDB_UDB.png"><img src="img/actinia_PDB_UDB.png" width="60%"></a><br>
Fig. 1: Architecture of an actinia deployment
</center>

**Deployment**

In a nutshell, deployment means to launch software, usually in an automated way on a computer node. A series of technologies exist for that but importantly virtualization plays an important role which helps towards a higher level of abstraction instead of a high dependency on hardware and software specifics.

An aim is to operate **Infrastructure as Code** (IaC), i.e. to have a set of scripts which order the needed computational resources in the cloud, setup the network and storage topology, connect to the nodes, install them with the needed software (usually docker based, i.e. so-called containers are launched from prepared images) and processing chains. Basically, the entire software part of a cloud computing infrastructure is launched "simply" through scripts with the advantage of restarting it easily as needed, maintain it and migrate to other hardware. **CI/CD** systems (continuous integration/continuous deployment) allow to define dependencies, prevent from launching broken software and allow the versioning of the entire software stack. 

In terms of actinia, **various ways of deployment** are offered: local installation, docker, docker-compose, docker-swarm, Openshift, and kubernetes.

**Architecture of actinia**

Several **components** play a role in a cloud deployment of actinia (for an example, see Fig. 2):

* analytics: this are the workers of GRASS GIS or wrapped other software,
* external data sources: import providers for various external data sources,
* interface layer:
    * most importantly the **REST API**,
    * [openEO GRASS GIS driver](https://github.com/Open-EO/openeo-grassgis-driver),
    * ace - [actinia command execution](https://github.com/mundialis/actinia_core/blob/master/scripts/README.md) (to be run from a GRASS GIS session),
* metadata management: interface to GNOS, managed through [actinia-GDI](https://github.com/mundialis/actinia-gdi/)
* database system:
    * job management in a Redis database
    * the GRASS GIS database (here are the geo/EO data!)
* connection to OGC Web services for output
   * Geoserver integration

<center>
<a href="img/actinia_architecture_FTTH.png"><img src="img/actinia_architecture_FTTH.png" width="60%"></a><br>
Fig. 2: Architecture of an actinia deployment
</center>

## REST API and geoprocessing basics

(20 min)

### What is REST: intro

An **API** (Application Programming Interface) defines a way of communicating between different software applications. There have been developed many different ways to implement APIs. A RESTful API (Representational State Transfer - REST, for details see [https://en.wikipedia.org/wiki/Representational_state_transfer](https://en.wikipedia.org/wiki/Representational_state_transfer)) is a web API for creating web services that communicate with web resources.

In detail, a REST API uses URL arguments to specify what information shall be returned through the API. This is not much different from requesting a Web page in a browser but through the REST API we can **execute commands remotely and retrieve the results**.

Each URL is called a **request** while the data sent back to the user is called a **response**, after some **processing** was performed.

<!--
###  Concepts of service URL, resources, request, response...

Looking in further detail into REST calls, we see that an API request consists of three parts (source: [https://www.earthdatascience.org/courses/earth-analytics/get-data-using-apis/intro-to-programmatic-data-access-r/](https://www.earthdatascience.org/courses/earth-analytics/get-data-using-apis/intro-to-programmatic-data-access-r/)):
*   Data **REQUEST**: through this you try to access an URL using your browser that specifies a particular subset of data.
*   Data **processing:** A web server somewhere uses that URL to query a specified dataset.
*   Data **RESPONSE**: The web server then sends you back some content.
-->

A **request** consists of four parts (see also [1]):

* the endpoints
* the header
* the data (or body)
* the methods

**Endpoint:**

An endpoint is the URL you request for. It follows this structure: https://api.some.server 
The final part of an endpoint is query parameters. Using query parameters you can modify your request with key-value pairs, beginning with a question mark (`?`). With an ampersand (`&`) each parameter pair is separated, e.g.:

`?query1=value1&query2=value2`

As an example, we check the repos of a GitHub user, in sorted form:

[https://api.github.com/users/neteler/repos?sort=pushed](https://api.github.com/users/neteler/repos?sort=pushed)

**Header & Body:**

* Both requests and responses have two parts: a header, and optionally a body
* Response headers contain information about the response.
* In both requests & responses, the body contains the actual data being transmitted (e.g., population data)

**Methods and Response Codes**

(source: [2])

* Request Methods:
    * In REST APIs, every request has an HTTP method type associated with it.
    * The most common HTTP methods include:
    * GET | A GET request asks to receive a copy of a resource
    * POST | A POST request sends data to a server in order to create a new resource
    * PUT | A PUT request sends data to a server in order to modify an existing resource
    * DELETE | A DELETE request is sent to delete a resource
* Response Codes:
    * HTTP responses don't have methods, but they do have status codes: HTTP status codes are included in the header of every response in a REST API. Status codes include information about the result of the original request.
    * Selected status codes (see also [https://httpstatuses.com)](https://httpstatuses.com)):
        * 200 - OK | All fine
        * 404 - Not Found | The requested resource was not found
        * 401 - Unauthorized | The request is not authorized to be completed
        * 500 - Internal Server Error | Something went wrong while the server was processing your request

**JSON format**

JSON is a structured, machine readable format (while also human readable at the same time; in contrast to XML, at least for many people).

```bash
# this command line...
GRASS 7.9.dev (nc_spm_08):~ > v.buffer input=roadlines output=roadbuf10 distance=10 --json
```

looks like this in JSON:

```json
{
  "module": "v.buffer",
  "id": "v.buffer_1804289383",
  "inputs":[
     {"param": "input", "value": "roadlines"},
     {"param": "layer", "value": "-1"},
     {"param": "type", "value": "point,line,area"},
     {"param": "distance", "value": "10"},
     {"param": "angle", "value": "0"},
     {"param": "scale", "value": "1.0"}
   ],
  "outputs":[
     {"param": "output", "value": "roadbuf10"}
   ]
}
```

Hint: When writing JSON files, some linting (validation) might come handy, e.g. using [https://jsonlint.com/](https://jsonlint.com/).

## First Hand-on: working with REST API requests

(50min)

### Step by step...

* Step 1: get your credentials (for authentication) from the trainer (or use the "demouser" with "gu3st!pa55w0rd")
* Step 2: launch cURL or RESTman or ...
    * choose your REST client:
        * a) cURL on command line: [https://curl.haxx.se/docs/manpage.html](https://curl.haxx.se/docs/manpage.html)
        * b) RESTman in Browser
    * Try this call: [https://actinia.mundialis.de/api/v1/locations](https://actinia.mundialis.de/api/v1/locations)
* Step 3: Explore the existing actinia data
    * i.e., available GRASS locations, mapsets, raster, vector, and space-time datasets
    * Check the [list of data](https://github.com/mundialis/actinia_core/blob/master/scripts/README.md#available-data) currently available on the actinia server 
    * e.g.
        * [https://actinia.mundialis.de/api/v1/locations](https://actinia.mundialis.de/api/v1/locations)
        * [https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets](https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets)
        * [https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets/landsat/raster_layers](https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets/landsat/raster_layers)
        * [https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets/landsat/raster_layers/lsat5_1987_10](https://actinia.mundialis.de/api/v1/locations/nc_spm_08/mapsets/landsat/raster_layers/lsat5_1987_10)
    * process_results are ordered alphabetically, not thematically
* Step 4: Submit a compute job and check its status
    * Examples incl. Spatio-Temporal sampling: [https://github.com/mundialis/actinia_core/blob/master/scripts/curl_commands.sh](https://github.com/mundialis/actinia_core/blob/master/scripts/curl_commands.sh)

### Exploring the API

The actinia REST API documentation at [https://redocly.github.io/redoc/?url=https://actinia.mundialis.de/api/v1/swagger.json](https://redocly.github.io/redoc/?url=https://actinia.mundialis.de/api/v1/swagger.json) comes with a series of examples.

Check out the various sections.

### Further Examples

Here we use the command line and the `curl` software:

```bash
# set credentials and REST server URL
export ACTINIA_USER='demouser'
export ACTINIA_PASSWORD='gu3st!pa55w0rd'
export actinia="https://actinia.mundialis.de"

# show locations
curl -u "$ACTINIA_USER:$ACTINIA_PASSWORD" -X GET ${actinia}/api/v1/locations

# show (raster) map details
curl -u "$ACTINIA_USER:$ACTINIA_PASSWORD" -X GET ${actinia}/api/v1/locations/latlong_wgs84/mapsets/modis_ndvi_global/strds/ndvi_16_5600m

# Get a list or raster layers from a STRDS
curl "$ACTINIA_USER:$ACTINIA_PASSWORD"  -X GETi "${actinia}/api/v1/locations/ECAD/mapsets/PERMANENT/strds/precipitation_1950_2013_yearly_mm/raster_layers?where=start_time>2013-05-01"

# query point value, including JSON in request
curl -u "$ACTINIA_USER:$ACTINIA_PASSWORD" -X POST -H "content-type: application/json" 'https://actinia.mundialis.de/latest/locations/latlong_wgs84/mapsets/modis_ndvi_global/strds/ndvi_16_5600m/sampling_sync_geojson' -d '{"type":"FeatureCollection","crs":{"type":"name","properties":{"name":"urn:ogc:def:crs:EPSG::4326"}},"features":[{"type":"Feature","properties":{"cat":1},"geometry":{"type":"Point","coordinates":[7,50]}}]}'

# store query in JSON file, then send it to server:
echo '{"type":"FeatureCollection","crs":{"type":"name","properties":{"name":"urn:ogc:def:crs:EPSG::4326"}},"features":[{"type":"Feature","properties":{"cat":1},"geometry":{"type":"Point","coordinates":[7,50]}}]}' > /tmp/pc_query_point_.json
curl -u "$ACTINIA_USER:$ACTINIA_PASSWORD" -X POST -H "content-type: application/json" 'https://actinia.mundialis.de/latest/locations/latlong_wgs84/mapsets/modis_ndvi_global/strds/ndvi_16_5600m/sampling_sync_geojson'    -d @/tmp/pc_query_point_.json
```

### Further command line exercise suggestions

- draft -

* compute NDVI from a Landsat scene
* computations using data in the nc_spm_08 location:
    * slope and aspect from a DEM (there are several)
    * flow accumulation with r.watershed from a DEM
    * buffer around hospitals
    * advanced: network allocation with hospitals and streets_wake
* Dealing with workflows
    * Prepare a workflow (processing chain)
    * async versus sync REST API CALLS with processing chains
        * See: [https://github.com/mundialis/actinia_core/blob/master/scripts/curl_commands.sh#L77](https://github.com/mundialis/actinia_core/blob/master/scripts/curl_commands.sh#L77)
    * Submit a workflow (processing chain)

### Controlling actinia from a running GRASS GIS session

Controlling actinia from a running GRASS GIS session:

"ace" - actinia command execution from a GRASS GIS terminal: [https://github.com/mundialis/actinia_core/tree/master/scripts](https://github.com/mundialis/actinia_core/tree/master/scripts)


## Own exercises in actinia

(40 min)

EXERCISE: "Property risks from trees" _<<-- needs finetuning_

* define region of interest
* needed geodata:
    * building footprints
    * download from OSM (via [http://overpass-turbo.eu/](http://overpass-turbo.eu/) | Wizard > building > ok > Export > Geojson)
    * these data are now on your machine and not on the actinia server
    * use "ace importer" or cURL to upload
    * select Sentinel-2 scene
* proposed workflow:
    * actinia "ace" importer for building footprint upload
    * v.buffer of 10m and 30m around footprints
    * select S2 scene, compute NDVI with i.vi
    * filter NDVI threshold > 0.6 (map algebra) to get the tree pixels - more exiting would be a ML approach (with previously prepared training data ;-)) (r.learn.ml offer RF and SVM)
    * on binary tree map (which corresponds to risk exposure)
    * count number of tree pixels in 5x5 moving window (r.neighbors with method "count")
    * compute property risk statistics using buffers and tree count map and upload to buffered building map (v.rast.stats, method=maximum)
    * export of results through REST resources

EXERCISE: "Population at risk near coastal areas"
* used geodata:
    * SRTM 30m (already in actinia - find out location yourself)
    * Global Population 2015 (already in actinia - find out location yourself)
* fetch metadata with actinia interface
* what's important about projections?
* proposed workflow:
    * set computational region to a small subregion and contrain pixel amount through defined user settings
    * buffer SRTM land areas by 5000 m inwards
    * zonal statistics with pop map

## Conclusions and future

(15 min incl discussions)

* integration in own scientific or business processes
* openEO actinia driver
* where is the code and how to contribute: GitHub
    * https://github.com/mundialis/actinia_core/

## See also: openEO resources
* OpenEO Web Editor: [https://open-eo.github.io/openeo-web-editor/demo/](https://open-eo.github.io/openeo-web-editor/demo/)
    * Server: [https://openeo.mundialis.de](https://openeo.mundialis.de)
    * user: <actinia user>
    * pw: <actinia pw>

## References

[1] Zell Liew, 2018: Understanding And Using REST APIs, [https://www.smashingmagazine.com/2018/01/understanding-using-rest-api/](https://www.smashingmagazine.com/2018/01/understanding-using-rest-api/)
[2] Planet 2019: Developer resource center, [https://developers.planet.com/planetschool/rest-apis/](https://developers.planet.com/planetschool/rest-apis/)
## About the trainer

Markus Neteler is partner and general manager at [mundialis](https://www.mundialis.de) GmbH & Co. KG, Bonn, Germany. From 2001-2015 he worked as a researcher in Italy. Markus is co-founder of OSGeo and since 1998, coordinator of the GRASS GIS development (for details, see his private [homepage](https://grassbook.org/neteler/)).

------------------------------------------------------------------------

- Repository of this material on [gitlab](https://gitlab.com/neteler/actinia-intro/tree/master)


*[About](about.md) | [Privacy](https://about.gitlab.com/privacy/)*
