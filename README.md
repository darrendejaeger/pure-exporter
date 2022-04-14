![Current version](https://img.shields.io/github/v/tag/PureStorage-OpenConnect/pure-exporter?label=current%20version)

# Pure Storage Prometheus exporter
Prometheus exporter for Pure Storage FlashArrays and FlashBlades.


### Overview

This applications aims to help monitor Pure Storage FlashArrays and FlashBlades by providing an "exporter", which means it extracts data from the Purity API and converts it to a format which is easily readable by Prometheus.

The stateless design of the exporter allows for easy configuration management as well as scalability for a whole fleet of Pure Storage systems. Each time Prometheus scrapes metrics for a specific system, it should provide the hostname and the readonly API token via GET parameters to this exporter.

To monitor your Pure Storage appliances, you will need to create a new dedicated user on your array, and assign read-only permissions to it. Afterwards, you also have to create a new API key.
The exporter is provided as three different options:

- pure-exporter. Full exporter for both FlashArray and FlashBlade in a single bundle
- pure-fa-exporter.  FlashArray exporter
- pure-fb-exporter.  FlashBlade exporter

### Forked Notes

API token and the Pure endpoint (**for pure-fa-exporter only**) have been tweaked to be read from the environment if request args aren't sent, to better align with an alternate usecase. Expected environment variables:

- APITOKEN
- FA_ENDPOINT

### Building and Deploying

The exporter is preferably built and launched via Docker. You can also scale the exporter deployment to multiple containers on Kubernetes thanks to the stateless nature of the application.

---

#### The official docker images are available at Quay.io

```shell
docker pull quay.io/purestorage/pure-exporter:1.2.5a
```

or

```shell
docker pull quay.io/purestorage/pure-fa-exporter:1.2.5a
```
or

```shell
docker pull quay.io/purestorage/pure-fb-exporter:1.2.5a
```
---

To build and deploy the application via Docker, your local linux user should be added to the `docker` group in order to be able to communicate with the Docker daemon. (If this is not possible, you can still use <kbd>sudo</kbd>)

This can be done with this command in the context of your user:
```bash
# add user to group
sudo usermod -aG docker $(whoami)
# apply the new group (no logout required)
newgrp docker
```

The included Makefile's takes care of the necessary build steps:
```bash
make
```

To run a simple instance of the exporter, run:
```bash
make -f Makefile.fa test
make -f Makefile.fb test
make -f Makefile.mk test
```

The Makefile currently features these targets:
- **build** - builds the docker image with preconfigured tags.
- **test** - spins up a new docker container with all required parameters.


### Local development

The application is usually not run by itself, but rather with the gunicorn WSGI server. If you want to contribute to the development, you can run the exporter locally without a WSGI server, by executing the application directly.

The following commands are required for a development setup:
```bash
# it is recommended to use virtual python environments!
python -m venv env
source ./env/bin/activate

# install dependencies
python -m pip install -r requirements.txt

# run the application in debug mode
python pure_exporter.py
```
Use the same approach to modify the FlashArray and/or the FlashBlade exporter, by simply using the related requitements file.

### Scraping endpoints

The exporter application uses a RESTful API schema to provide Prometheus scraping endpoints.

The full exporter understands the following requests:

System | URL | GET parameters | description
---|---|---|---
FlashArray | http://\<exporter-host\>:\<port\>/metrics/flasharray | endpoint, apitoken | Full array metrics
FlashArray | http://\<exporter-host\>:\<port\>/metrics/flasharray/array | endpoint, apitoken | Array only metrics
FlashArray | http://\<exporter-host\>:\<port\>/metrics/flasharray/volumes | endpoint, apitoken | Volumes only metrics
FlashArray | http://\<exporter-host\>:\<port\>/metrics/flasharray/hosts | endpoint, apitoken | Hosts only metrics
FlashArray | http://\<exporter-host\>:\<port\>/metrics/flasharray/pods | endpoint, apitoken | Pods only metrics
FlashBlade | http://\<exporter-host\>:\<port\>/metrics/flashblade | endpoint, apitoken | Full array metrics
FlashBlade | http://\<exporter-host\>:\<port\>/metrics/flashblade/array | endpoint, apitoken | Array only metrics
FlashBlade | http://\<exporter-host\>:\<port\>/metrics/flashblade/clients | endpoint, apitoken | Clients only metrics
FlashBlade | http://\<exporter-host\>:\<port\>/metrics/flashblade/quotas | endpoint, apitoken | Quotas only metrics

In order to authenticate the exporter to the target array REST API the preferred mechanism is to provide the necessary api-token in the http request, by using the HTTP Authorization header of type 'Bearer'. As an alternative, it is possible to provide the api-token as a request argument, using the *apitoken* key.

The FlashArray-only and FlashBlade only exporters use a slightly different schema, which consists of the removal of the flasharray|flashblade string from the path.

**FlashArray**

URL | required GET parameters | description
---|---|---
http://\<exporter-host\>:\<port\>/metrics | endpoint, apitoken | Full array metrics
http://\<exporter-host\>:\<port\>/metrics/array | endpoint, apitoken | Array only metrics
http://\<exporter-host\>:\<port\>/metrics/volumes | endpoint, apitoken | Volumes only metrics
http://\<exporter-host\>:\<port\>/metrics/hosts | endpoint, apitoken | Hosts only metrics
http://\<exporter-host\>:\<port\>/metrics/pods | endpoint, apitoken | Pods only metrics

**FlashBlade**

URL | required GET parameters | description
---|---|---
http://\<exporter-host\>:\<port\>/metrics | endpoint, apitoken | Full array metrics
http://\<exporter-host\>:\<port\>/metrics/array | endpoint, apitoken | Array only metrics
http://\<exporter-host\>:\<port\>/metrics/clients | endpoint, apitoken | Clients only metrics
http://\<exporter-host\>:\<port\>/metrics/quotas | endpoint, apitoken | Quotas only metrics


Depending on the target array, scraping for the whole set of metrics could result into timeout issues, in which case it is suggested either to increase the scraping timeout or to scrape each single endpoint instead.

### Prometheus configuration

There are two options to configure Prometheus for scraping FlashArray and FlashBlade arrays. The preferred option is to use the HTTP Authorization header, by the means of wich is possible to pass the array API token simply setting the authorization credentials parameter for the job configuration section in the config file. The other option consists of some tricky relabeling rules that are applied to leverage the apitoken parameter for the GET request.
The first option forces to define multiple jobs, one for each of the scraped arrays, as authorization parameters do not allow to leverage labeling, but it enables a higher security, especially when the exporter is deployed on kubernetes. In the second option the API key is given as a *meta label*, and will then be placed into the request as a GET parameter before the actual metrics scraping from the exporter happens. Afterwards, the label will be dropped for security reasons so it does not appear in the actual Prometheus database.

Take this full Prometheus configuration as a reference, but consider that, depending on the FlashArray and FlashBlade APIs response time you might have to adapt the configuration to your own environment, pherhaps splitting the job in a multitude of single jobs:

```yaml
global:
  # How often should Prometheus fetch metrics?
  # Recommended: 10s, 30s, 1m or 5m
  scrape_interval: 10s

scrape_configs:
# Job for all Pure Flasharrays
- job_name: 'pure_flasharray'
  metrics_path: /metrics/flasharray
  relabel_configs:
  # meta label of target address --> get parameter "pure_host"
  - source_labels: [__address__]
    target_label: __param_endpoint
  # label of target api token --> get parameter "pure_apitoken"
  - source_labels: [__pure_apitoken]
    target_label: __param_apitoken
  # display the pure host as the instance label
  - source_labels: [__address__]
    target_label: instance
  # point the exporter to the scraping endpoint of the exporter
  - target_label: __address__
    replacement: localhost:9491 # address of the exporter, in debug mode
                                # THIS NEEDS TO BE CHANGED TO YOUR ENVIRONMENT
  
  # Actual pure hosts (without a prometheus endpoint) as targets
  static_configs:
  - targets: [ mypureflasharray-01.lan ]
    labels:
      __pure_apitoken: 00000000-0000-0000-0000-000000000000

  - targets: [ mypureflasharray-02.lan ]
    labels:
      __pure_apitoken: 00000000-0000-0000-0000-000000000000

# Job for all Pure Flashblades
- job_name: 'pure_flashblade'
  metrics_path: /metrics/flashblade/array
  relabel_configs:
  # meta label of target address --> get parameter "pure_host"
  - source_labels: [__address__]
    target_label: __param_endpoint
  # label of target api token --> get parameter "pure_apitoken"
  - source_labels: [__pure_apitoken]
    target_label: __param_apitoken
  # display the pure host as the instance label
  - source_labels: [__address__]
    target_label: instance
  # point the exporter to the scraping endpoint of the exporter
  - target_label: __address__
    replacement: localhost:9491 # address of the exporter, in debug mode
                                # THIS NEEDS TO BE CHANGED TO YOUR ENVIRONMENT
    
  # Actual pure hosts (without a prometheus endpoint) as targets
  static_configs:
  - targets: [ mypureflashblade-01.lan ]
    labels:
      __pure_apitoken: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

  - targets: [ mypureflashblade-02.lan ]
    labels:
      __pure_apitoken: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Job for all Pure Flashblade clients
- job_name: 'pure_flashblade clients'
  scrape_timeout: 20s
  metrics_path: /metrics/flashblade/client
  relabel_configs:
  # meta label of target address --> get parameter "pure_host"
  - source_labels: [__address__]
    target_label: __param_endpoint
  # label of target api token --> get parameter "pure_apitoken"
  - source_labels: [__pure_apitoken]
    target_label: __param_apitoken
  # display the pure host as the instance label
  - source_labels: [__address__]
    target_label: instance
  # point the exporter to the scraping endpoint of the exporter
  - target_label: __address__
    replacement: localhost:9491 # address of the exporter, in debug mode
                                # THIS NEEDS TO BE CHANGED TO YOUR ENVIRONMENT
    
  # Actual pure hosts (without a prometheus endpoint) as targets
  static_configs:
  - targets: [ mypureflashblade-01.lan ]
    labels:
      __pure_apitoken: T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

  - targets: [ mypureflashblade-02.lan ]
    labels:
      __pure_apitoken: T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

If you now check for the <kbd>up</kbd> metric in Prometheus, the result could look something like this:
```
up{job="pure_flasharray",instance="mypureflasharray-01.lan"} 1
up{job="pure_flasharray",instance="mypureflasharray-02.lan"} 1
up{job="pure_flashblade",instance="mypureflashblade-01.lan"} 1
up{job="pure_flashblade",instance="mypureflashblade-02.lan"} 1
```

The <kbd>job</kbd> label is statically configured via Prometheus, and is only used to identify the type of the metrics source. All hosts of the same type should share the same job label. The <kbd>instance</kbd> should be unique for each individual target, usually it is the FQDN.

If you want to provide additional static labels for each host you can add those at each targets "label" section in the Prometheus configuration file.
Good examples for additional labels are:
- location: us
- datacenter: mountain-view-1
- is_production: 1


### Usage example

In a typical production scenario, it is recommended to use a visual frontend for your metrics, such as [Grafana](https://github.com/grafana/grafana). Grafana allows you to use your Prometheus instance as a datasource, and create Graphs and other visualizations from PromQL queries. Grafana, Prometheus, are all easy to run as docker containers.

To spin up a very basic set of those containers, use the following commands:
```bash
# Pure exporter
docker run -d -p 9491:9491 --name pure-exporter quay.io/purestorage/pure-exporter:<version>

# Prometheus with config via bind-volume (create config first!)
docker run -d -p 9090:9090 --name=prometheus -v /tmp/prometheus-pure.yml:/etc/prometheus/prometheus.yml -v /tmp/prometheus-data:/prometheus prom/prometheus:latest

# Grafana
docker run -d -p 3000:3000 --name=grafana -v /tmp/grafana-data:/var/lib/grafana grafana/grafana
```
Please have a look at the documentation of each image/application for adequate configuration examples.


### Bugs and Limitations

* Pure FlashBlade REST APIs are not designed for efficiently reporting on full clients and objects quota KPIs, therefrore it is suggested to scrape the /metrics/flasblade/array preferrably and use the /metrics/flasblade/clients and /metrics/flasblade/quotas individually and with a lower frequency that the other. In any case, as a general rule, it is advisable to do not lower the scraping interval down to less than 30 sec. In case you experience timeout issues, you may want to increase the internal Gunicorn timeout by specifically setting the `--timeout` variable and appropriately reduce the scraping intervall as well.

* By default the number of workers spawn by Gunicorn is set to 2 and this is not optimal when monitoring a relatively large amount of arrays. The suggested approach is therefore to run the exporter with a number of workers that approximately matches the number of arrays to be scraped.


### License

This project is licensed under the Apache 2.0 License - see the [LICENSE.md](LICENSE.md) file for details
