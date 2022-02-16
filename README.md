![Current version](https://img.shields.io/github/v/tag/PureStorage-OpenConnect/pure-fb-openmetrics-exporter?label=current%20version)

# Pure Storage FlashBlade OpenMetrics exporter
OpenMetrics exporter for Pure Storage FlashBlade.

## Support Statement
This exporter is provided under Best Efforts support by the Pure Portfolio Solutions Group, Open Source Integrations team.
For feature requests and bugs please use GitHub Issues.
We will address these as soon as we can, but there are no specific SLAs.
##

### Overview

This application aims to help monitor Pure Storage FlashBlades by providing an "exporter", which means it extracts data from the Purity API and converts it to the OpenMetrics format, which is for instance consumable by Prometheus.

The stateless design of the exporter allows for easy configuration management as well as scalability for a whole fleet of Pure Storage systems. Each time Prometheus scrapes metrics for a specific system, it should provide the hostname via GET parameter and the API token as Authorization token to this exporter.

To monitor your Pure Storage appliances, you will need to create a new dedicated user on your array, and assign read-only permissions to it. Afterwards, you also have to create a new API key.


### Building and Deploying

The exporter is preferably built and launched via Docker. You can also scale the exporter deployment to multiple containers on Kubernetes thanks to the stateless nature of the application.

---

#### The official docker images are available at Quay.io

```shell
docker pull quay.io/purestorage/pure-fb-ome:<release>
```

where the release tag follows the semantic versioning.

---

### Local development
If you want to contribute to the development or simply build the package locally you should use python virtualenv

The following commands describe how to run a typical build using the basic virtualenv package:
```shell

python -m venv pure-fb-ome-build
source ./pure-fb-ome-build/bin/activate

# install dependencies
python -m pip install --upgrade pip
pip install build

# clone the repository
git clone git@github.com:PureStorage-OpenConnect/pure-fb-openmetrics-exporter.git

# modify the code and build the package
cd pure-fb-openmetrics-exporter
...
python -m build

```

The newly built package can be found in the <kbd>./dist</kbd> directory.

Tests can be run using tox. You need a running FlashBlade you can use for executing the tests. Modify the tests/conftest.py file accordingly with your FlashBlade management endpoint and API token.

### Docker image

The provided dockerfile can be used to generate a docker image of the exporter. It accepts the version of the python package as the build parameter, therefore yoo can build the image using docker as follows

```shell

VERSION=<version>
docker build --build-arg exporter_version=$VERSION -t pure-fb-ome:$VERSION .
```

### Scraping endpoints

The exporter uses a RESTful API schema to provide Prometheus scraping endpoints.

**Authentication**

Authentication is used by the exporter as the mechanism to cross authenticate to the scraped appliance, therefore for each array it is required to provide the REST API token for an account that has a 'readonly' role. The api-token must be provided in the http request using the HTTP Authorization header of type 'Bearer'. This is achieved by specifying the api-token value as the authorization parameter of the specific job in the Prometheus configuration file.

The exporter understands the following requests:


URL | GET parameters | description
---|---|---
http://\<exporter-host\>:\<port\>/metrics | endpoint | Full array metrics
http://\<exporter-host\>:\<port\>/metrics/array | endpoint | Array metrics
http://\<exporter-host\>:\<port\>/metrics/clients | endpoint | Clients metrics
http://\<exporter-host\>:\<port\>/metrics/usage | endpoint | Quotas usage metrics


Depending on the target array, scraping for the whole set of metrics could result into timeout issues, in which case it is suggested either to increase the scraping timeout or to scrape each single endpoint instead.

### Usage examples

In a typical production scenario, it is recommended to use a visual frontend for your metrics, such as [Grafana](https://github.com/grafana/grafana). Grafana allows you to use your Prometheus instance as a datasource, and create Graphs and other visualizations from PromQL queries. Grafana, Prometheus, are all easy to run as docker containers.

To spin up a very basic set of those containers, use the following commands:
```bash
# Pure exporter
docker run -d -p 9491:9491 --name pure-fb-ome quay.io/purestorage/pure-fb-ome:<version>

# Prometheus with config via bind-volume (create config first!)
docker run -d -p 9090:9090 --name=prometheus -v /tmp/prometheus-pure.yml:/etc/prometheus/prometheus.yml -v /tmp/prometheus-data:/prometheus prom/prometheus:latest

# Grafana
docker run -d -p 3000:3000 --name=grafana -v /tmp/grafana-data:/var/lib/grafana grafana/grafana
```
Please have a look at the documentation of each image/application for adequate configuration examples.

A simple but complete example to deploy a full monitoring stack on kubernetes can be found in the [examples](examples/config/k8s) directory  


### Bugs and Limitations

* Pure FlashBlade REST APIs are not designed for efficiently reporting on full clients and objects quota KPIs, therefrore it is suggested to scrape the "array" metrics preferably and use the "clients" and "usage" metrics individually and with a lower frequency than the other.. In any case, as a general rule, it is advisable to do not lower the scraping interval down to less than 30 sec. In case you experience timeout issues, you may want to increase the internal Gunicorn timeout by specifically setting the `--timeout` variable and appropriately reduce the scraping intervall as well.

* By default the number of workers spawn by Gunicorn is set to 2 and this is not optimal when monitoring a relatively large amount of arrays. The suggested approach is therefore to run the exporter with a number of workers that approximately matches the number of arrays to be scraped.


### License

This project is licensed under the Apache 2.0 License - see the [LICENSE.md](LICENSE.md) file for details
