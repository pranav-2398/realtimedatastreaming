# Data Streaming | Data Engineering Project

## List of Contents
- [Introduction](#introduction)
- [Architecture](#architecture)
- [Technologies](#technologies)
- [How does it work](#how-does-it-work)
- [Explanation](#explanation)

## Introduction

This project has been created for learning purposes and has been replicated from the project initially done by Airscholar whose repo can be found here (https://github.com/airscholar/e2e-data-engineering).
The project tries to cover a typical end to end data engineering use case where data is ingested from source like API and pushed to a database in real time.

## Architecture

![Architecture](https://github.com/pranav-2398/realtimedatastreaming/blob/main/Data%20engineering%20architecture.png)

## Technologies

The project utilises the following components:

- **Data Source**: We use `Randomuser` API to generate random user data for our pipeline.
- **Apache Airflow**: Used as a orchestator backed by PostgreSQL database.
- **Apache Kafka and Zookeeper**: Used for streaming data from API to the processing engine.
- **Control Center and Schema Registry**: Use for visualising and monitoring the schema of our Kafka streams.
- **Apache Spark**: For data processing with its master and worker nodes.
- **Cassandra**: Final Destination to store the processed Data.
- **Docker**: Everything has been containerised for minimal installation and scalability

## How does it work

Assuming you have docker installed on your system. 
1. Run the following command to spin up all the services:
     ```bash
     docker compose up -d
     ```

2. Go to Airflow (`localhost:8080`) and run the user_automation dag to trigger the pipeline

3. Submit the spark job to the spark master (`localhost:9090`) to start the spark stream job.

4. Now the job will run and by the time airflow is sending data to Kafka, the job will read from Kafka & insert the data into the cassandra database.

## Explanation 

### Dags

1. The kafka-stream.py file is where the DAG resides and it contain just one `PythonOperator` task which calls the function `stream_data`
2. The stream_data function runs for `1 min` where it calls the random user api and after formatting the data to get user details, pushes into the topic `users_created`.

### Docker Compose File

1. It is a docker compose which creates all the services i.e.
    - Apache Zookeeper, One Apache Kafka Broker, Schema Registry & Control Center
    - One Spark Master and One Spark Worker Node
    - PostGres SQL
    - Apache Airflow Web Server & Scheduler

2. Also Bind Mounts are created for dags & requirements text to copied into the Airflow folder (created after airflow db init)

3. An `EntryPoint.sh` is also created under scripts folder which does the following:
   - This Bash script initializes and starts an Airflow instance, assuming Airflow is already installed and configured in the /opt/airflow directory.
   - It performs the following tasks:
      - Shebang Line: `#!/bin/bash` specifies that this script should be executed using the Bash interpreter.
      - `set -e`: enables the "exit on error" option, causing the script to exit immediately if any command returns a non-zero exit code. This helps in early detection of issues.
      - First if Block:
        - `[ -e "/opt/airflow/requirements.txt" ]`: Checks if the requirements.txt file exists.
        - `$(command python) pip install --upgrade pip`: If it exists, it uses the full path to the Python interpreter found by command python to run pip install --upgrade pip, upgrading pip to the latest version.
        - `$(command -v pip) install --user -r requirements.txt`: Uses the full path to the pip executable found by command -v pip to install or upgrade packages from requirements.txt into the user's profile.
      - Second if Block:
        - `[ ! -f "/opt/airflow/airflow.db" ]`: Checks if the airflow.db file exists (i.e., if the database is new).
        - `airflow db init`: If it doesn't exist, initializes a new Airflow database.
        - `airflow users create ...`: Creates the admin user.
        - `$(command -v airflow) db upgrade`: Upgrades the Airflow database schema if necessary.
        - `exec airflow webserver`: Starts the Airflow webserver process in the background, making the script run indefinitely to serve web requests.

### Spark Job

1. It 1st tries to create a spark connection
2. It then initiates a connection with Kafka
3. It then tries to read from the topic `users_created` from the earliest message available and formats it into a dataframe.
4. Establishes a connection with Apache Cassandra
5. Tries to create a Keyspace `spark_streams` and table `created_users` if they doesn't exist.
6. Finally, inserts the received data into the table.

**Note**: Make sure that jar files related to cassandra and kafka connector are added if it's not able to fetch from Maven repository.   
