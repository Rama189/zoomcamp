>Previous: [Introduction to Data Engineering](1_intro.md)

>[Back to index](README.md)

>Next: [Data Warehouse](3_data_warehouse.md)

# Data Ingestion

This lesson will cover the topics of _Data Lake_ and _pipelines orchestration with Airflow_.

_[Back to the top](#table-of-contents)_

# Data Lake

_[Video source](https://www.youtube.com/watch?v=W3Zm6rjOq70&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=16)_

## What is a Data Lake?

A ***Data Lake*** is a _central repository_ that holds _big data_ from many sources.

The _data_ in a Data Lake could either be structured, unstructured or a mix of both.

The main goal behind a Data Lake is being able to ingest data as quickly as possible and making it available to the other team members.

A Data Lake should be:
* Secure
* Scalable
* Able to run on inexpensive hardware

## Data Lake vs Data Warehouse

A Data Lake (DL) is not to be confused with a Data Warehouse (DW). There are several differences:

* Data Processing:
  * DL: The data is **raw** and has undergone minimal processing. The data is generally unstructured.
  * DW: the data is **refined**; it has been cleaned, pre-processed and structured for specific use cases.
* Size:
  * DL: Data Lakes are **large** and contains vast amounts of data, in the order of petabytes. Data is transformed when in use only and can be stored indefinitely.
  * DW: Data Warehouses are **small** in comparison with DLs. Data is always preprocessed before ingestion and may be purged periodically.
* Nature:
  * DL: data is **undefined** and can be used for a wide variety of purposes.
  * DW: data is historic and **relational**, such as transaction systems, etc.
* Users:
  * DL: Data scientists, data analysts.
  * DW: Business analysts.
* Use cases:
  * DL: Stream processing, machine learning, real-time analytics...
  * DW: Batch processing, business intelligence, reporting.

Data Lakes came into existence because as companies started to realize the importance of data, they soon found out that they couldn't ingest data right away into their DWs but they didn't want to waste uncollected data when their devs hadn't yet finished developing the necessary relationships for a DW, so the Data Lake was born to collect any potentially useful data that could later be used in later steps from the very start of any new projects.

## ETL vs ELT

When ingesting data, DWs use the ***Export, Transform and Load*** (ETL) model whereas DLs use ***Export, Load and Transform*** (ELT).

The main difference between them is the order of steps. In DWs, ETL (Schema on Write) means the data is _transformed_ (preprocessed, etc) before arriving to its final destination, whereas in DLs, ELT (Schema on read) the data is directly stored without any transformations and any schemas are derived when reading the data from the DL.

## Data Swamp - Data Lakes gone wrong

Data Lakes are only useful if data can be easily processed from it. Techniques such as versioning and metadata are very helpful in helping manage a Data Lake. A Data Lake risks degenerating into a ***Data Swamp*** if no such measures are taken, which can lead to:
* No versioning of the data
* Incompatible schemes for the same data
* No metadata associated
* Joins between different datasets are not possible

## Data Lake Cloud Providers

* Google Cloud Platform > [Cloud Storage](https://cloud.google.com/storage)
* Amazon Web Services > [Amazon S3](https://aws.amazon.com/s3/)
* Microsoft Azure > [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/)

_[Back to the top](#table-of-contents)_
# Orchestration with Airflow

## Introduction to Workflow Orchestration

_[Video source](https://www.youtube.com/watch?v=0yK7LXwYeD0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=17)_

In the previous lesson we saw the definition of [data pipeline](1_intro.md#data-pipelines) and we created a [pipeline script](../1_intro/ingest_data.py) that downloaded a CSV and processed it so that we could ingest it to Postgres.

The script we created is an example of how **NOT** to create a pipeline, because it contains 2 steps which could otherwise be separated (downloading and processing). The reason is that if our internet connection is slow or if we're simply testing the script, it will have to download the CSV file every single time that we run the script, which is less than ideal.

Ideally, each of these steps would be contained as separate entities, like for example 2 separate scripts. For our pipeline, that would look like this:

```
(web) → DOWNLOAD → (csv) → INGEST → (Postgres)
```

We have now separated our pipeline into a `DOWNLOAD` script and a `INGEST` script.

In this lesson we will create a more complex pipeline:

```
(web)
  ↓
DOWNLOAD
  ↓
(csv)
  ↓
PARQUETIZE
  ↓
(parquet) ------→ UPLOAD TO S3
  ↓
UPLOAD TO GCS
  ↓
(parquet in GCS)
  ↓
UPLOAD TO BIGQUERY
  ↓
(table in BQ)
```
_Parquet_ is a [columnar storage datafile format](https://parquet.apache.org/) which is more efficient than CSV.

This ***Data Workflow*** has more steps and even branches. This type of workflow is often called a ***Directed Acyclic Graph*** (DAG) because it lacks any loops and the data flow is well defined.

The steps in capital letters are our ***jobs*** and the objects in between are the jobs' outputs, which behave as ***dependencies*** for other jobs. Each job may have its own set of ***parameters*** and there may also be global parameters which are the same for all of the jobs.

A ***Workflow Orchestration Tool*** allows us to define data workflows and parametrize them; it also provides additional tools such as history and logging.

The tool we will focus on in this course is **[Apache Airflow](https://airflow.apache.org/)**, but there are many others such as Luigi, Prefect, Argo, etc.

## Airflow architecture

_[Video source](https://www.youtube.com/watch?v=lqDMzReAtrw&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=18)_

A typical Airflow installation consists of the following components:

![airflow architecture](https://airflow.apache.org/docs/apache-airflow/stable/_images/arch-diag-basic.png)

* The **scheduler** handles both triggering scheduled workflows as well as submitting _tasks_ to the executor to run. The scheduler is the main "core" of Airflow.
* The **executor** handles running tasks. In a default installation, the executor runs everything inside the scheduler but most production-suitable executors push task execution out to _workers_.
* A **worker** simply executes tasks given by the scheduler.
* A **webserver** which seves as the GUI.
* A **DAG directory**; a folder with _DAG files_ which is read by the scheduler and the executor (an by extension by any worker the executor might have)
* A **metadata database** (Postgres) used by the scheduler, the executor and the web server to store state. The backend of Airflow.
* Additional components (not shown in the diagram):
  * `redis`: a _message broker_ that forwards messages from the scheduler to workers.
  * `flower`: app for monitoring the environment, available at port `5555` by default.
  * `airflow-init`: initialization service which we will customize for our needs.

  Airflow will create a folder structure when running:
* `./dags` - `DAG_FOLDER` for DAG files
* `./logs` - contains logs from task execution and scheduler.
* `./plugins` - for custom plugins

Additional definitions:
* ***DAG***: Directed acyclic graph, specifies the dependencies between a set of tasks with explicit execution order, and has a beginning as well as an end. (Hence, “acyclic”). A _DAG's Structure_ is as follows:
  * DAG Definition
  * Tasks (eg. Operators)
  * Task Dependencies (control flow: `>>` or `<<` )  
* ***Task***: a defined unit of work. The Tasks themselves describe what to do, be it fetching data, running analysis, triggering other systems, or more. Common Types of tasks are:
  * ***Operators*** (used in this workshop) are predefined tasks. They're the most common.
  * ***Sensors*** are a subclass of operator which wait for external events to happen.
  * ***TaskFlow decorators*** (subclasses of Airflow's BaseOperator) are custom Python functions packaged as tasks.
* ***DAG Run***: individual execution/run of a DAG. A run may be scheduled or triggered.
* ***Task Instance***: an individual run of a single task. Task instances also have an indicative state, which could be `running`, `success`, `failed`, `skipped`, `up for retry`, etc.
    * Ideally, a task should flow from `none`, to `scheduled`, to `queued`, to `running`, and finally to `success`.

## Setting up Airflow with Docker

_[Video source](https://www.youtube.com/watch?v=lqDMzReAtrw&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=18)_

### Pre-requisites

1. For the sake of standardization across this workshop's config,This tutorial assumes that the [service account credentials JSON file](1_intro.md#gcp-initial-setup) is named `google_credentials.json` and stored in `$HOME/.google/credentials/`. Copy and rename your credentials file to the required path.
```bash
    cd ~ && mkdir -p ~/.google/credentials/
    mv <path/to/your/service-account-authkeys>.json ~/.google/credentials/google_credentials.json
```
2. `docker-compose` should be at least version v2.x+ and Docker Engine should have at least 5GB of RAM available, ideally 8GB. On Docker Desktop this can be changed in _Preferences_ > _Resources_.  If enough memory is not allocated, it might lead to airflow-webserver continuously restarting.
3. Python version: 3.7+

### Airflow Setup (full version)
Please follow these instructions for deploying the "full" Airflow with Docker. Instructions for a "lite" version are provided in the next section but you must follow these steps first.

1. Create a new `airflow` subdirectory in your work directory.
1. Download the official Docker-compose YAML file for the latest Airflow version.
    ```bash
    curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.2.3/docker-compose.yaml'
    ```
    * The official `docker-compose.yaml` file is quite complex and contains [several service definitions](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#docker-compose-yaml).
    * For a refresher on how `docker-compose` works, you can [check out this lesson from the ML Zoomcamp](https://github.com/ziritrion/ml-zoomcamp/blob/main/notes/10_kubernetes.md#connecting-docker-containers-with-docker-compose).
1. We now need to [set up the Airflow user](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#setting-the-right-airflow-user). For MacOS, create a new `.env` in the same folder as the `docker-compose.yaml` file with the content below:
    ```bash
    AIRFLOW_UID=50000
    ```
    * In all other operating systems, you may need to generate a `.env` file with the appropiate UID with the following command:
        ```bash
        echo -e "AIRFLOW_UID=$(id -u)" > .env
        ```
1. The base Airflow Docker image won't work with GCP, so we need to [customize it](https://airflow.apache.org/docs/docker-stack/build.html) to suit our needs. You may download a GCP-ready Airflow Dockerfile [from this link](../2_data_ingestion/airflow/Dockerfile). A few things of note:
    * We use the base Apache Airflow image as the base.
    * We install the GCP SDK CLI tool so that Airflow can communicate with our GCP project.
    * We also need to provide a [`requirements.txt` file](../2_data_ingestion/airflow/requirements.txt) to install Python dependencies. The dependencies are:
      * `apache-airflow-providers-google` so that Airflow can use the GCP SDK.
      * `pyarrow` , a library to work with parquet files.
    * Change the `AIRFLOW__CORE__LOAD_EXAMPLES` value to `'false'`. This will prevent Airflow from populating its interface with DAG examples.
    * Change user: "0:0" to user:"${AIRFLOW_UID:-50000}:0"
     Add a volume and point it to the folder where you stored the credentials json file. Assuming you complied with the pre-requisites and moved and renamed your credentials, add the following line after all the other volumes:
      ```yaml
      - ~/.google/credentials/:/.google/credentials:ro
      ```
    * Add 2 new environment variables right after the others: `GOOGLE_APPLICATION_CREDENTIALS` and `AIRFLOW_CONN_GOOGLE_CLOUD_DEFAULT`:
      ```yaml
      GOOGLE_APPLICATION_CREDENTIALS: /.google/credentials/google_credentials.json
      AIRFLOW_CONN_GOOGLE_CLOUD_DEFAULT: 'google-cloud-platform://?extra__google_cloud_platform__key_path=/.google/credentials/google_credentials.json'
      ```
    * Add 2 new additional environment variables for your GCP project ID and the GCP bucket that Terraform should have created [in the previous lesson](1_intro.md#creating-gcp-infrastructure-with-terraform). You can find this info in your GCP project's dashboard.
      ```yaml
      GCP_PROJECT_ID: '<your_gcp_project_id>'
      GCP_GCS_BUCKET: '<your_bucket_id>'
You may find a modified `docker-compose.yaml` file [in this link](data_ingestion/airflow/docker-compose.yaml).
1. Additional notes:
    * The YAML file uses [`CeleryExecutor`](https://airflow.apache.org/docs/apache-airflow/stable/executor/celery.html) as its executor type, which means that tasks will be pushed to workers (external Docker containers) rather than running them locally (as regular processes). You can change this setting by modifying the `AIRFLOW__CORE__EXECUTOR` environment variable under the `x-airflow-common` environment definition.

You may now skip to the [Execution section](#execution) to deploy Airflow, or continue reading to modify your `docker-compose.yaml` file further for a less resource-intensive Airflow deployment.

### Setup (lite version)

_[Video source](https://www.youtube.com/watch?v=A1p5LQ0zzaQ&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=23)_

The current `docker-compose.yaml` file we've generated will deploy multiple containers which will require lots of resources. This is the correct approach for running multiple DAGs accross multiple nodes in a Kubernetes deployment but it's very taxing on a regular local computer such as a laptop.

If you want a less overwhelming YAML that only runs the webserver and the scheduler and runs the DAGs in the scheduler rather than running them in external workers, please modify the `docker-compose.yaml` file following these steps:

1. Remove the `redis`, `airflow-worker`, `airflow-triggerer` and `flower` services.
1. Change the `AIRFLOW__CORE__EXECUTOR` environment variable from `CeleryExecutor` to `LocalExecutor` .
1. At the end of the `x-airflow-common` definition, within the `depends-on` block, remove these 2 lines:
    ```yaml
    redis:
      condition: service_healthy
    ```
1. Comment out the `AIRFLOW__CELERY__RESULT_BACKEND` and `AIRFLOW__CELERY__BROKER_URL` environment variables.

You should now have a simplified Airflow "lite" YAML file ready for deployment and may continue to the next section.

For convenience, a simplified YAML version is available [in this link](../2_data_ingestion/airflow/extras/docker-compose-nofrills.yml).

### Execution
1. Build the image. It may take several minutes You only need to do this the first time you run Airflow or if you modified the Dockerfile or the `requirements.txt` file.
    ```bash
    docker-compose build
    ```
2. Initialize configs:
    ```bash
    docker-compose up airflow-init
    ```
3. Run Airflow
    ```bash
    docker-compose up -d
    ```
1. You may now access the Airflow GUI by browsing to `localhost:8080`. Username and password are both `airflow` .
>***IMPORTANT***: this is ***NOT*** a production-ready setup! The username and password for Airflow have not been modified in any way; you can find them by searching for `_AIRFLOW_WWW_USER_USERNAME` and `_AIRFLOW_WWW_USER_PASSWORD` inside the `docker-compose.yaml` file.

Huh, Ok Hon! It ran finally! and below are troubleshooting steps for some hicups: uwuhuwuhu :(
    * In airflow-init on docker-compose, it should be user: "0:0", absolutely not this: (user: "${AIRFLOW_UID:-50000}:0").
    * Next, the default user and pwd of aiflow didn't work, yeah, so created that with
    ```bash
    airflow users  create --role Admin --username admin --email admin --firstname admin --lastname admin --password admin
    ```
    * Now once decided to login, it again thrown some error like 
    ```bash
    Failed to log action with (psycopg2.errors.UndefinedTable) relation "log" does not exist
LINE 1: INSERT INTO log (dttm, dag_id, task_id, event, execution_dat...
```
    This specifically is telling that there are some db table missing, turns out we need to run couple migrations manually, for whatsoever fuckall reasons! Run this in aiflow webserver terminal
    ```bash
    airflow db upgrade
    ```
    That's finally able to access airflow gui on localhost:8080 and able to login! yay!