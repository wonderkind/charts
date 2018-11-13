# Airflow / Celery

[Airflow](https://airflow.apache.org/) is a platform to programmatically author, schedule and
monitor workflows.


## Install Chart

To install the Airflow Chart into your Kubernetes cluster :

```bash
helm install --namespace "airflow" --name "airflow" incubator/airflow
```

After installation succeeds, you can get a status of Chart

```bash
helm status "airflow"
```

If you want to delete your Chart, use this command:

```bash
helm delete  --purge "airflow"
```

### Helm ingresses

The Chart provides ingress configuration to allow customization the installation by adapting
the `config.yaml` depending on your setup. Please read the comments in the value.yaml file for more
detail on how to configure your reverse proxy or load balancer.

### Prefix

This Helm automatically prefixes all names using the release name to avoid collisions.

### URL prefix

This chart exposes 2 endpoints:

- Airflow Web UI
- Flower, a debug UI for Celery

Both can be placed either at the root of a domain or at a sub path, for example:

```
http://mycompany.com/airflow/
http://mycompany.com/airflow/flower
```

NOTE: Mounting the Airflow UI under a subpath requires an airflow version > 1.9.x. For the moment
(March 2018) this is **not** available on official package, you will have to use an image where
airflow has been updated to its current HEAD. You can use the following one:
`stibbons31/docker-airflow-dev:2.0dev`

Please also note than Airflow UI and Flower do not behave the same:

- Airflow Web UI behave transparently, to configure it one just need to specify the
  `ingress.web.path` value.
- Flower cannot handle this scheme directly and requires to use an URL rewrite mechanism in front
  of it. In short, it is able to generate the right URLs in the returned HTML file but cannot
  respond to these URL. It is commonly found in software that wasn't intended to work under
  something else than a root URL or localhost port. To use it, see the `value.yaml` in detail on how
  to configure your ingress controller to rewrite the URL (or "strip" the prefix path).

### Airflow configuration

`airflow.cfg` configuration can be changed by defining environment variables in the following form:
`AIRFLOW__<section>__<key>`.

See the
[Airflow documentation for more information](http://airflow.readthedocs.io/en/latest/configuration.html?highlight=__CORE__#setting-configuration-options)

This helm chart allows you to add these additional settings with the value key `airflow.config`.

### Worker Statefulset

Celery workers uses StatefulSet instead of deployment.
It is used to freeze their DNS using a Kubernetes Headless Service, and allow the webserver to
requests the logs from each workers individually.
This requires to expose a port (8793) and ensure the pod DNS is accessible to the web server pod,
which is why StatefulSet is for.

## DAGs Deployment

### Mount a Persistent Volume

You can store your DAG files on an external volume, and mount this volume into the relevant Pods
(scheduler, web, worker). In this scenario, your CI/CD pipeline should update the DAG files in the 
PV.
Since all Pods should have the same collection of DAG files, it is recommended to create just one PV
that is shared. This ensures that the Pods are always in sync about the DagBag. 

To share a PV with multiple Pods, the PV needs to have accessMode 'ReadOnlyMany' or 'ReadWriteMany'.
If you are on AWS, you can use [Elastic File System (EFS)](https://aws.amazon.com/efs/). 
If you are on Azure, you can use [Azure File Storage (AFS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv).


### Embedded DAGs

If you want more control on the way you deploy your DAGs, you can use embedded DAGs, where DAGs
are burned inside the Docker container deployed as Scheduler and Workers.

Be aware this requirement more heavy tooling than using git-sync, especially if you use CI/CD:

- your CI/CD should be able to build a new docker image each time your DAGs are updated.
- your CI/CD should be able to control the deployment of this new image in your kubernetes cluster

Example of procedure:

- Fork the [puckel/docker-airflow](https://github.com/puckel/docker-airflow) repository
- Place your DAG inside the `dags` folder of the repository, and update `/requirements.txt` to install new dependencies if needed (see below)
- Add build script connected to your CI that will build the new docker image
- Change the value of airflow.image in values.yaml
- Deploy on your Kubernetes cluster

### Python dependencies

If you want to add specific python dependencies to use in your DAGs, you need to mount a
`/requirements.txt` file at the root of the image.
See the
[docker-airflow readme](https://github.com/puckel/docker-airflow#install-custom-python-package) for
more information.

## Helm chart Configuration

Full and up-to-date documentation can be found in the comments of the `values.yaml` file.
