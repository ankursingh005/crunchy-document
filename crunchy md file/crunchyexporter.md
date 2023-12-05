<h1 align="center"><b> <u> Crunchy-Postgres-Exporter  </u></b></h1>

## Overview :
This document provides a step-by-step overview of set-up Crunchy-Postgres-Exporter using Podman, Crunchy Exporter, PostgreSQL, Prometheus and Grafana on an Ubuntu 20.04 system.

## Environment details :
 - Ubuntu version 20.04
 - RAM 4 GB

## List of tools and technologies :
 - Podman
 - Crunchy_Exporter
 - PostgreSQL
 - Prometheus
 - Grafana

## Definition of tools :
**Podman** 
 - Podman is an open-source containerization tool that works on Linux operating systems. It allows you to create, manage, and run containers, and it offers many features that are compatible with docker commands.

**Crunchy_Exporter**
- Crunchy Exporter is a containerized tool that collects real-time metrics from your postgreSQL database and exposes them for monitoring with tools like prometheus and grafana.

**PostgreSQL**  
 - PostgreSQL is an open-source relational database management system used for data storage and retrieval. It offers advanced features and scalability, making it a popular open-source RDBMS.

**Prometheus** 
 - Prometheus is an open-source monitoring and alerting system used for system and application performance monitoring. Its primary purpose is data collection, storage, querying, and alerting, allowing the detection of real-time insights and performance issues.

**Grafana** 
 - Grafana is an open-source tool used for data visualization and monitoring. It allows data to be visually represented in the form of graphs, charts, making data analysis and performance monitoring easier.

## Command for the setup or configuration :

**Step 1.  Install podman.**
```
sudo sh -c "echo 'deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_$(lsb_release -rs)/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
```
![Alt text](image1.png)

```
wget https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_$(lsb_release -rs)/Release.key
```
![Alt text](image2.png)

```
sudo apt-key add - < Release.key
```
![Alt text](image3.png)

```
sudo apt update
```
![Alt text](image4.png)

```
sudo apt install -y podman
```
![Alt text](image5.png)

```
podman --version
```
![Alt text](image6.png)

## Step 2. Create pod name crunchy-postgres for all the 4 containers.

```
 podman pod create --name crunchy-postgres --publish 9090:9090 --publish 9187:9187 --publish 5432:5432 --publish 3000:3000
 ```

![Alt text](image7.png)


## Step 3. Check the created  pod status.

```
podman pod ps
```
![Alt text](image8.png)

## Step 4. Create a directory for mounting the persistent storage for the postgres data.

```
mkdir -p shiksha_portal/crunchy/postgres/data
```
![Alt text](image9.png)

## Step 5. Create a postgres container in this pod which was created with the name crunchy-postgres.

```
podman run -d --pod crunchy-postgres --name postgres_crunchy -e "POSTGRES_DB=postgres" -e "POSTGRES_USER=postgres" -e "POSTGRES_PASSWORD=redhat" -v /home/ankur/shiksha_portal/crunchy/postgres/data:/var/lib/postgresql/data docker.io/postgres:12
```
![Alt text](image10.png)


## Step 6. Check the container status.

```
podman ps
```
![Alt text](image11.png)

## Step 7. Do changes in postgresql.conf file.

 We will need to modify your postgresql.conf configuration file to tell PostgreSQL to load shared libraries.
**Change directory to postgres then run sudo su for root privilege and change directory to data.**

```
cd shiksha_portal/cruchy/postgres/
```
```
sudo su
```
```
cd data/
```
![Alt text](image12.png)

![Alt text](image13.png)

```
home/ankur/shiksha_portal/crunchy/postgres/data# echo "shared_preload_libraries = 'pg_stat_statements,auto_explain'" >> postgresql.conf
```
![Alt text](image14.png)

## Step 8. Install vim.

```
sudo apt install vim
```
![Alt text](image15.png)

"We can also make these configuration changes in the postgresql.conf file. Search for '**shared_preload_libraries**' and remove the '#' symbol to uncomment it and add following line “**pg_stat_statements,auto_explain**”

```
vim postgresql.conf
```
![Alt text](image16.png)

## Step 9. Crunchy-postgres-exporter image.

Download tar file from the given  link. and run the given command on the same path.
Link:- https://drive.google.com/file/d/1pTXscjFxxsv1g0NFMCDofeRD-vf_I06M/view

```
podman load -i crunchy.tar
```
![Alt text](image17.png)

```
podman images
```
![Alt text](image18.png)

Then run the container with the image ID and use your container's image ID.

```
podman run -itd --pod crunchy-postgres --name crunchy -e EXPORTER_PG_PASSWORD=redhat c9110d4443a6
```


Again check container status.
```
podman ps
```
![Alt text](image19.png)

## Step 10. Copy setup.sql from Container to Host.

Copy the **setup.sql** file from the **/opt/cpm/conf/pg12/** directory within the Crunchy container to your current working directory on the host machine.

```
podman cp crunchy:/opt/cpm/conf/pg12/setup.sql .
```


**NOTE:- Don't miss the ( . ) which is taking place after setup.sql  because it represents a copy in your current directory.**

## Step 11. Remove the test container.

Remove the crunchy container.

```
podman rm -f crunchy
```
![Alt text](image20.png)

Here, we will copy the **setup.sql** file from the host to inside the PostgreSQL container in the **/var/lib/postgresql** directory.

```
podman cp /home/ankur/setup.sql postgres_crunchy:/var/lib/postgresql
```


![Alt text](image21.png)

## Step 12. Go inside the postgres container &  Push setup.sql in postgres database.

```
podman exec -it postgres_crunchy bash
```
```
cd var/lib/postgresql/
```
```
/var/lib/postgresql# psql -h 127.0.0.1 -U postgres -d template1 < setup.sql

```
![Alt text](image22.png)

This command is used to execute SQL commands from the **'setup.sql'** file within a PostgreSQL database. It facilitates making changes to the database schema, data, or settings as specified in the SQL file.

## Step 13. Create an extension.

This command is used to enable the 'pg_stat_statements' extension in the PostgreSQL database. It enables the collection of statistics about executed SQL statements for performance analysis.

```
/var/lib/postgresql# psql -h 127.0.0.1 -U postgres -d template1 -c "CREATE EXTENSION pg_stat_statements;"

```
![Alt text](image23.png)




## Step 14. Here, we will log in to the PostgreSQL database, create a password for the user 'ccp_monitoring,' and also create a database named 'ankur.'

```
root@crunchy-postgres:/var/lib/postgresql# psql -h 127.0.0.1 -U postgres -d postgres


postgres=# \password ccp_monitoring
Enter new password for user "ccp_monitoring":      (I am giving here redhat)
Enter it again:
postgres=# create database ankur;
```
In the PostgreSQL database, to exit, use the command **'\q'** and then press '**Enter**.'

![Alt text](image24.png)



## Step 15. Now create a crunchy-postgres-exporter container.

```
podman run -itd --pod crunchy-postgres --name crunchy -e EXPORTER_PG_PASSWORD=redhat -e EXPORTER_PG_HOST=127.0.0.1 -e EXPORTER_PG_USER=ccp_monitoring -e DATA_SOURCE_NAME=postgresql://ccp_monitoring:redhat@127.0.0.1:5432/ankur?sslmode=disable c9110d4443a6
```

![Alt text](image25.png)



## Step 16. Install curl.

curl is a command-line tool to transfer data to or from a server.
```
sudo apt install curl
```
![Alt text](image26.png)

**Check metrics.**
```
curl localhost:9187/metrics | grep query
```
![Alt text](image27.png)

- **grep query:** This part of the command uses the grep command to search for lines in the input that contain the word "query". This is used to filter the metrics output to only show lines related to queries


Now, for the Prometheus container, we need to create the 'prometheus_crunchy' directory and the 'prometheus.yml' file inside the directory. Change directory to shiksha portal and create file.

```
:~$ cd shiksha_portal/
:~/shiksha_portal$ mkdir prometheus_crunchy
:~/shiksha_portal$ ls
:~/shiksha_portal$ cd prometheus_crunchy/
:~/shiksha_portal/prometheus_crunchy$ touch prometheus.yml
```

![Alt text](image28.png)

## Step 17. Create a prometheus container.

Please check the path name **/home/ankur/shiksha_portal/prometheus_crunchy.**

```
podman run -itd --pod crunchy-postgres --name prometheus_crunchy -v /home/ankur/shiksha_portal/prometheus_crunchy/prometheus.yml:/etc/prometheus/prometheus.yml docker.io/prom/prometheus
```
![Alt text](image29.png)

## Step 18. Configure the prometheus.yml file.

Check IP

```
hostname -I
```
![Alt text](image31.png)

```
vim Prometheus.yml
```
To insert the given content into the file, use 'i', add the content, then press 'Esc' followed by ':wq!' to save and exit.

```
/shiksha_portal/prometheus_crunchy$ cat prometheus.yml
```
```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
	static_configs:
  	- targets: ['192.168.122.23:9090']
  	- targets: ['192.168.122.23:9187']
```
![Alt text](image32.png)

Set the target in the prometheus.yml file to get metrics in prometheus and hit the browser (after restarting the prometheus container).

```
:~/shiksha_portal/prometheus_crunchy$ podman restart prometheus_crunchy
```
![Alt text](image33.png)

http://192.168.122.23:9090/  
Or
 http://localhost:9090/
Open Prometheus in the browser by using the IP or 'localhost' and then click 'Status' > 'Targets' > 'Status' to confirm that all statuses are up.

![Alt text](image34.png)

![Alt text](image35.png)

![Alt text](image36.png)


## Step 19. Create a grafana container for Visualisation of the metrics data.

```
podman run -itd --pod crunchy-postgres --name grafana_crunchy docker.io/grafana/grafana
```
![Alt text](image37.png)



**Now check all container status.**

```
podman ps
```
![Alt text](image38.png)

Visit http://localhost:3000/
Login to Grafana using the default ID and password (admin), and then change the password.
Click on "**Data source" > Select "Prometheus" > Paste the Prometheus URL > Save and Test**.
Next, click on "**Create dashboard" > "Import dashboard" > Enter ID 9628 > Choose "DS Prometheus**" as the data source we created.
Select "**Prometheus**" as the data source and import the dashboard with **ID 9628**.

![Alt text](image39.png)