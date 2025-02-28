# odf-grafana
Playbook and dashboards to help with ODF performance analysis.  

This project uses Ansible, and the community grafana operator to create a dedicated Grafana instance connected to Openshift's prometheus instance. The playbook also loads any available dashboards from this project into the Grafana instance, so you don't have to start from scratch!

## Installation
before you start you'll need to satisfy the dependencies.

### OpenShift Deployment
you will need;  
* ansible
* oc (*binaries are [here](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)*)
* pwgen

Once you have these packages/binaries in place, download this repo to your machine. The repo provides the following playbooks

| Filename | Purpose |
|----------|---------|
| deploy-grafana.yml | creates a new namespace and deploys Grafana with default dashboards|
| add-dashboard.yml | Load a dashboard to an existing Grafana deployment |
| purge-grafana.yml | deletes your grafana namespace (sledgehammer!) |

### Local Deployment
Requirements:
* ansible
* pwgen
* docker
* docker-compose

Podman and podman-compose can be used instead of their docker equivalents, but the containers will not survive a reboot.

Once you have these packages/binaries in place, download this repo to your machine. The repo provides the following playbooks
| Filename | Purpose |
|----------|---------|
| deploy-local.yml | Creates a local deployment using a provided tarball of prometheus metrics (use `-e prom_tarball=<PATH TO TARBALL>`) |
| purge-local.yml | Destroys a local deployment and deletes extracted prometheus files |

To obtain a prometheus tarball from an OpenShift Cluster, details can be found in [this article](https://access.redhat.com/solutions/5482971)

## Usage
The `group_vars/all.yml` file defines a number of parameters that can be used to tweak a deployment, but normally you should need to change anything.


### Deploying Grafana
Run the deploy-grafana.yml playbook
```
# ansible-playbook deploy-grafana.yml
```
In less than a minute you'll have a functional Grafana instance :smile:  
At the end of the run you'll see a summary of the settings defined for the Grafana instance.

```
Deployment Summary
------------------

Grafana Details
  Namespace : mygrafana
  User      : grafana
  Password  : mysupersecretpassword
  Login URL : https://grafana-route-mygrafana.apps.cuznerp-odf-test.aws.mycompany.org

Prometheus Datasource Connectivity
  Monitoring namespace : openshift-monitoring
  Prometheus URL       : https://thanos-querier.openshift-monitoring.svc.cluster.local:9091
  Service Account Used : grafana-serviceaccount
  Token used           : <service account token>

```

NB. For later reference, the `deploy-grafana.yml` playbook creates an `odf-grafana-credentials` file in your home directory,
which contains a record of the password used/generated for the grafana login.

Use the grafana information to login to your instance and get started! The screen capture below shows you the Prometheus data source definition and the default ODF dashboard.

![grafana UI](assets/grafana-dashboard.gif)

#### Resource Requirements
The grafana operator supports tuning the resources allocated to the grafana pod. The
defaults are normally fine (1/2 core and 1GB of memory), but if your pod is in a pending
state due to resources, you can specify reduced requirements by updating your `all.yml`
file. The *"Resource Limits"* section contains some defaults for cpu and memory that have
been confirmed to work in smaller OCP environments.

### Removing the Grafana deployment

Once your done with your grafana instance either delete the namespace manually to tidy things up or run the purge playbook.

```
# ansible-playbook purge-grafana.yml -e grafana-namespace=mygrafana
```

NB. When you remove your grafana configuration, remember to use `oc project default` to switch you local CLI from the old project.

### Adding a dashboard

If you need to add dashboards to your instance after a deployment, you can use the `add-dashboard.yml` playbook and specify the path to the dashboard file either from the all.yml file or as an extra-vars on the playbook command line.

```
# ansible-playbook add-dashboard.yml -e dashboard_path=<insert your path here>
```
NB. For the dashboard to work, it must define it's datasource as $datasource. This is a simple way to make your dashboards portable.


### Adding a datasource (needs further testing!)
If you need to attach your grafana instance to a secondary cluster's prometheus instance, you can use the `add-datasource.yml` playbook. To
set this up correctly you will need to set the following variables in `group_vars/all.yml'

* `prometheus_route`: the external route to main thanos-querier instance within the openshift-monitoring namespace
* `prometheus_datasource_name`: the name to use inside your grafana instance for this datasource
* `token`: the token from a suitable serviceaccount in the other cluster that permits access to prometheus data

```
# ansible-playbook add-datasource.yml
```

## Included Dashboards

The deployment installs several dashboards to help provide OCP and ODF performance insights out-of-the-box.

### OCP Overview
The OCP overview provides a high level overview of the OCP platform.  


![ocp overview](assets/OCP%20Overview.png)

### ODF Performance Analysis
There are many different facets to ODF, so the ODF dashboard is split into multiple rows covering the following types of information;

- Ceph Overview
- Noobaa Overview
- radosgw Overview
- Data distribution
- Physical Disk Activity - by host and by OSD
- Network load (with NIC speed and MTU size)
- CPU and Memory analysis for Ceph and Noobaa daemons
  
For 'bonus points', if you have the mgr/prometheus rbd_stats_pools defined, you can also see per PVC performance.

It's a lot!

Here's a screenshot with all rows expanded (they're collapsed by default), to give you a sense of the type of data that is visualised.

![odf performance analysis](assets/odf%20performance%20analysis.png)


## Configurations Tested

The playbooks have been tested against the following config

| Host OS | ansible | pwgen | oc | OCP | Cloud | Grafana operator| Grafana |
|---------|---------|-------|----|-----|-------|------|---------|
| Fedora 36 | 5.9 | 2.08 | 4.11 | 4.10, 4.11 | AWS | 4.5.1 | 9.0.7 |
| Fedora 37 | 7.0 | 2.08 | 4.11 | 4.11 | AWS | 4.8.0 | 9.3.1 |
| RHEL 8.6 | 2.9 | 2.08 (EPEL) | 4.11 | 4.10, 4.11 | AWS, Bare-metal | 4.5.1 | 9.0.7 |


## Design Notes

1. The playbook uses the `oc` command instead of the ansible `k8s` module, primarily to reduce dependencies. 
