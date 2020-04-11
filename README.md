# Rocket.Chat.Monitoring
Easy ad-hoc monitoring of a single or multiple Rocket.Chat servers

## Monitoring your existing server(s)

So you have a few Rocket.Chat servers running for your family and friends, perhaps for your office or special interest group.
You've always wanted some way to "look inside" these servers and monitor their health during operations but fell short of doing something about it.

This project will help you to realize that goal.   It will show you step by step how to setup monitoring for your existing Rocket.Chat server.  

It is assumed that you may have deployed your running Rocket.Chat servers using manual or docker or snap methods.   The "monitoring tools" and "stack" will be deployed via docker-compose.

This is a manual, ad-hoc, setup for Rocket.Chat server(s) running on one single machine/VM/VPS/swarm.  If you are clustered and/or are already using orchestration platforms such as Kubernetes, this project will not be useful for you. 

The monitoring stack used in this project is adaptive to all platforms and can easily run even on a Raspberry Pi 4 - 4GB.

###  Pre-requisites

*  the latest supported docker and docker-compose installed and working for your platform (on Ubuntu 18.04 LTS and later, `snap install docker` )

### Step by Step

#### 1.  Prepare your Rocket.Chat servers to expose metrics

For security reasons, Rocket.Chat servers do not expose any operational metrics by default.  You must turn it on. 

For each server you want to monitor, sign in as administrator and then go to Administration -> Logs -> Prometheus and make sure you **enable** it and remember the default port used.  Save your changes.

![Enable Prometheus metrics exposure](images/turnonprom.png)

#### 2.   Create monitoring network and add server containers (for docker deployed servers)

If your server(s) are deployed using docker on your multi-cores machine/vps, then create a monitoring network:

```
docker network create monitoring
```

And then connect the server(s) that you want to be monitored to this network, issuing this command for each running server container:

```
docker network connect monitoring <<container name>>
```

Where ``<<container name>>`` is the name or id of the container currently running Rocket.Chat.

Once you've added all your server containers to the monitoring network, confirm it using:

```
docker network inspect monitoring
```

You should see all your containers listed as connected to monitoring network.  

Confirm that everyone of your Rocket.Chat server is now exposing metrics:


```
curl http://<your server's (container's) ip address>: port
```

Use either the server's IPv4 or the containers IPv4 address, and the port that you noted in step 1.   You should see a page of metrics in text format.   

#### 3.   Setup Prometheus to scrape metrics from servers and store data

We are using industry de-facto standard monitoring solution, open source [prometheus](https://prometheus.io/), to collect and store the metrics.  

First, clone this repository:

```
git clone https://github.com/Sing-Li/Rocket.Chat.Monitoring
```

Next, you need to tell prometheus how to find your Rocket.Chat servers.  prometheus will periodically reach out to each server and "scrape" the metrics.   This configuration is done in the `prometheus.yml`  YAML configuration file.


```
cd   Rocket.Chat.Monitoring/prometheus/config
vi prometheus.yml
```

Every one of the server that you want to monitor has a `job` section under `scrape_configs` within the file.

For every server container you added in step 2, you should add a dns_sd (Naming service discovery) job:

```
     - job_name: <short server name>
       dns_sd_configs:
       - names: ["<actual container name or id>"]
         type: A
         port: <port exposing metrics>
```

The `<short server name>` should be a unique string to identify the server to be scraped (for example `familyserver`).   The `<actual container name or id>`  must be the name or id of a container that you have confirmed to be connected to the monitoring network in step 3.    The `<port exposing metrics>` is the port that you've noted in step 1 and verified at step 2.


For every server that you want to monitor, but is not running in docker, or not connected to the monitoring network, use a `static_configs` job description:

```
     - job_name: <short server name>
       static_configs:
         - targets: ['<IP address of server host>:<port exposing mettrics']
```

The `<short server name>` should be a unique string to identify the server to be scraped (for example `familyserver`).   The `<IP address of server host>`  must be the IP address that you can reach the server to be monitored by.  The `<port exposing metrics>` is the port that you've noted in step 1.

After you've add `scrape_configs` for every one of the servers to be monitored, save the YAML file changes. 

Next, you can now create a persistent storage volume for the prometheus container to use for data storage.

```
   docker volume create prom-storage
```


You can then start the prometheus container.   Make sure you're in the `prometheus` subdirectory where you will find the `docker-compose.yml` file.   Then start prometheus by:

```
docker-compose up -d
```

This should start prometheus scraping your servers immediately.   Check the logs to make sure it did not crash or fail.  Typical reason for failure is problem parsing your `prometheus.yml` file.


```
docker logs <prometheus container name or id>
```

 If everything is working well, the last line in the log should have a message similar to:
 
 ```
"Server is ready to receive web requests."

```

At this point, the prometheus web UI is available on your host at port 9091 (configurable in the `docker-compose.yml` file).  Point a browser to the web UI at:

```
http://<host IP of prometheus container>:9091/targets
```

You should see your list of server scraping`jobs` and their status.  If any of them do not report **UP** status, double check the `promethus.yml` and restart the container.

Here is the web UI of a working prometheus scraper, handling 4 servers.


![Prometheus targets all up](images/promtargets.png)

At this point, you may want to experiment with the Prometheus web UI and become familiar with PromQL.   This will become more important when you need to design and customize your own monitoring system and dashboard.  See [Prometheus documentation](https://prometheus.io/docs/introduction/overview/) for more information.

Before you move onto the next step of setting up graphical dashboards, you may want to close up the web UI port currently exposed on the host machine (this can become a security problem if your machine is exposed to the Internet).   You can do this by removing the port mapping in the `docker-compose.yml` file and recreate the prometheus container.

#### 4.   Setup grafana to display graphical dashboards based on collected data

You will use [grafana](https://grafana.com/), the open source de-facto visualization platform that works extremely well with prometheus, to display our dashboards (as well as creation of new dashboards and customization of existing ones).   

Grafana will be running in a docker container that is connected to the monitoring network that you setup previously  (prometheus should be now running and connected to this monitoring network from step 3).

First thing we need to do is to create a persistent storage volume for the grafana container to use for data storage.

```
   docker volume create grafana-storage
```

