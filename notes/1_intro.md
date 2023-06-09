### Table of contents

- [Introduction to Data Engineering](#introduction-to-data-engineering)

# Introduction to Data Engineering
***Data Engineering*** is the design and development of systems for collecting, storing and analyzing data at scale.

## Architecture

During the course we will replicate the following architecture:

![architecture diagram](https://github.com/DataTalksClub/data-engineering-zoomcamp/blob/main/images/architecture/arch_2.png)

# Docker and Postgres

## Docker basic concepts

_([Video source](https://www.youtube.com/watch?v=EYNwNlOrpr0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=3))_

Check notes on Notion for more Docker details.

Docker delivers software in packages in called containers. These containers are isolated from one another. If we run a data pipeline inside a container it is virtually isolated from the rest of the things running on the computer.

Docker provides the following advantages:
* Reproducibility
* Local experimentation
* Integration tests (CI/CD)
* Running pipelines on the cloud (AWS Batch, Kubernetes jobs)
* Spark (analytics engine for large-scale data processing)
* Serverless (AWS Lambda, Google functions)

Data pipeline: Gets in Data → Processes Data → Generates more Data!
[pipeline image](images/pipeline.png)

## Creating a custom pipeline with Docker

_([Video source](https://www.youtube.com/watch?v=EYNwNlOrpr0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=3))_

Let's create an example pipeline. We will create a dummy `pipeline.py` Python script that receives an argument and prints it.

```python
import sys
import pandas # we don't need this but it's useful for the example

# print arguments
print(sys.argv)

# argument 0 is the name os the file
# argumment 1 contains the actual first argument we care about
day = sys.argv[1]

# cool pandas stuff goes here

# print a sentence with the argument
print(f'job finished successfully for day = {day}')
```

We can run this script with `python pipeline.py <some_number>` and it should print 2 lines:
* `['pipeline.py', '<some_number>']`
* `job finished successfully for day = <some_number>`

Let's containerize it by creating a Docker image. Create the folllowing `Dockerfile` file:

```dockerfile
# base Docker image that we will build on
FROM python:3.9.1

# set up our image by installing prerequisites; pandas in this case
RUN pip install pandas

# set up the working directory inside the container
WORKDIR /app
# copy the script to the container. 1st name is source file, 2nd is destination
COPY pipeline.py pipeline.py

# define what to do first when the container runs
# in this example, we will just run the script
ENTRYPOINT ["python", "pipeline.py"]
```

Let's build the image:


```ssh
docker build -t test:pandas .
```
* The image name will be `test` and its tag will be `pandas`. If the tag isn't specified it will default to `latest`.

We can now run the container and pass an argument to it, so that our pipeline will receive it:

```ssh
docker run -it test:pandas some_number
```

You should get the same output you did when you ran the pipeline script by itself.

>Note: these instructions asume that `pipeline.py` and `Dockerfile` are in the same directory. The Docker commands should also be run from the same directory as these files.

## Running Postgres in a container
_([Video source](https://www.youtube.com/watch?v=2JM-ziJt0WI&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=4))_

In later parts of the course we will use Airflow, which uses PostgreSQL internally. For simpler tests we can use PostgreSQL (or just Postgres) directly.

You can run a containerized version of Postgres that doesn't require any installation steps. You only need to provide a few _environment variables_ to it as well as a _volume_ for storing data.

```bash
docker run -it \
    -e POSTGRES_USER="root" \
    -e POSTGRES_PASSWORD="root" \
    -e POSTGRES_DB="ny_taxi" \
    -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgresql/data \
    -p 5431:5432 \
    postgres:13
```
* The container needs 3 environment variables:
    * `POSTGRES_USER` is the username for logging into the database. We chose `root`.
    * `POSTGRES_PASSWORD` is the password for the database. We chose `root`
        * ***IMPORTANT: These values are only meant for testing. Please change them for any serious project.***
    * `POSTGRES_DB` is the name that we will give the database. We chose `ny_taxi`.
* `-v` points to the volume directory. The colon `:` separates the first part (path to the folder in the host computer) from the second part (path to the folder inside the container).
    * Path names must be absolute. If you're in a UNIX-like system, you can use `pwd` to print you local folder as a shortcut; this example should work with both `bash` and `zsh` shells, but `fish` will require you to remove the `$`.
    * This command will only work if you run it from a directory which contains the `ny_taxi_postgres_data` subdirectory you created above.
* The `-p` is for port mapping. We map the default Postgres port to the same port in the host.
* The last argument is the image name and tag. We run the official `postgres` image on its version `13`.
* Additionally, to run this in background, you can add -d(detached mode) after -it.

So, this will start a docker container using postgres image. Now, if you type
```bash
docker container ls
```
This will show the container running. 

Difference between docker container ls and docker ps?
docker ps is shorthand that stands for "docker process status", whilst docker container ls is shorthand for the more verbose docker container list . As the accepted answer explains, there is no difference in how they work, and docker container ls is the 'newer' command, so you should probably prefer it.

What effed up here? 
Yes, this command didn't work, and you know what's the root cause? The $pwd had a space in the folder name. Argh! damn why! wasted 1 hours on this stupid issue :(

-p 5431:5432 : This maps a postgres port on our host machine to one in our container. It seemed that 5432 was already in use on my machine, so I used 5431. 5432 is the port postgresql runs inside the container, so that cant be changed!

Next install pgcli on your folder "ny_taxi_postgres_data" to access pgcli to connect to db.

Once the container is running, we can log into our database with [pgcli](https://www.pgcli.com/) with the following command:

```bash
pgcli -h localhost -p 5431 -u root -d ny_taxi
```
* `-h` is the host. Since we're running locally we can use `localhost`.
* `-p` is the port.
* `-u` is the username.
* `-d` is the database name.
* The password is not provided; it will be requested after running the command.

Finally, it connected, it asks for db password, give it what you gave above while creating container, in this case, it's root. below would be the response.

```bash
Password for root:
Server: PostgreSQL 13.11 (Debian 13.11-1.pgdg110+1)
Version: 3.5.0
Home: http://pgcli.com
root@localhost:ny_taxi>
```
Also if you haven't installed psql on your local and still wanna access, you can connect to the container directly and the use psql there. for this, use below command
```bash
docker exec -it bdca2b8c09b7 psql -U root ny_taxi
```
Now we don't have anything yet there, so we can just run something like
```bash
\dt
```
to check list of tables.

## Ingesting data to Postgres with Python

_([Video source](https://www.youtube.com/watch?v=2JM-ziJt0WI&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=4))_

We will now create a Jupyter Notebook `upload-data.ipynb` file which we will use to read a CSV file and export it to Postgres.

We will use data from the [NYC TLC Trip Record Data website](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page). Specifically, we will use the [Yellow taxi trip records CSV file for January 2021](https://s3.amazonaws.com/nyc-tlc/trip+data/yellow_tripdata_2021-01.csv). A dictionary to understand each field is available [here](https://www1.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf).

Specifically, we will be ingesting data from the New York City Taxi & Limousine Commission (TLC) yellow taxi data for January 2021, which can be downloaded from this [link] (https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz) 🚕 , the actual data from that site is stored in PARQUET these days, so csv's are backedup here.

You can copy the csv to the same folder where your ipynb file is present. you can take a look for sample data using less command
```bash
cp /Users/mycomputer/Downloads/yellow_tripdata_2021-01.csv .
less yellow_tripdata_2021-01.csv
```
if you wanna transfer only first n rows from this big file to another, you can use 
```bash
head -n 100 yellow_tripdata_2021-01.csv > yellow_head.csv
```
Also to get the line count of the csv file, you can use: it means word count lines
```bash
wc -l yellow_tripdata_2021-01.csv
```
Check the completed `upload-data.ipynb` [in this link](upload-data.ipynb) for a detailed guide. in the same directory you will need to have the CSV file linked above and the `ny_taxi_postgres_data` subdirectory.

And that's it! We have successfully ingested our data into a PostgreSQL database running in a Docker container.
Ah! Thank god!

## Connecting pgAdmin and Postgres with Docker networking


