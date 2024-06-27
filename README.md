# DS-load-balancer

# Implementing a Customizable Load Balancer

# Table of Content

- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
  - [Building Docker Images](#building-docker-images)
  - [Running Docker Containers](#running-docker-containers)
  - [Interact With System](#interact-with-system)
  - [Remove Existing Container](#interact-with-system)
  - [Clear Existing Images](#clear-existing-images)
- [Design Choices](#design-choices)
- [Troubleshooting](#troubleshooting)
  - [Docker Exit with Code 137](#docker-exit-with-code-137)
  - [Removing Docker Containers](#removing-docker-containers)
- [Evaluation](#evaluation)
  - [Server Load Analysis](#server-load-analysis)
  - [Server Avg Load Analysis](#server-avg-load-analysis)
  - [Server Failure Testing](#server-failure-testing)
  - [Hashing Function Variation Analysis](#hashing-function-variation-analysis)
- [Group Details](#group-details)


# Prerequisite

## 1. Docker: latest

    sudo apt-get update

    sudo apt-get install \
        ca-certificates \
        curl \
        gnupg \
        lsb-release

    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update

    sudo apt-get install docker-ce docker-ce-cli containerd.io

## 2. Docker-compose standalone 
    sudo curl -SL https://github.com/docker/compose/releases/download/v2.15.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
    
    sudo chmod +x /usr/local/bin/docker-compose
    
    sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose


# Getting Started


## Building Docker Images
To create the necessary Docker images for the load balancer and servers, execute the following command:

```bash
make install
```

## Running Docker Containers
To initiate the deployment of load balancer containers, execute the following command:

```bash
make deploy
```
This command will launch the load balancer container, which, in turn, will spawn the initial N server Docker containers along with their heartbeat probing threads. Ensure that the necessary configurations are in place for a seamless deployment. The command also clears any existing containers using server or load balancer image (i.e. execute make clean).

<span style="color:red">**Note:** The deployment command launches Docker in the background mode. Therefore, users are advised to check the docker-compose logs to view load-balancer logs.</span>

## Interact with System
To interact with the load balancer and send GET/POST requests, launch the interactive terminal using the following command:

```bash
bash client.sh
```
## Remove Existing Container
To stop and remove all containers using the server image and load balancer, run the following command:

```bash
make clean
```

Executing this command is recommended before running the main code to ensure there are no conflicting container instances already running. This helps prevent potential errors and ensures a clean environment for the code execution.

## Clear Existing Images
To remove previously created server and load balancer images, execute the following command:

```bash
make deepclean
```

It is advisable to run this command before executing the main code to eliminate any pre-existing images with the same name. This ensures a clean slate and avoids potential conflicts during the code execution.

# Design Choices
<ol>
<li> Users may provide current server hostnames in their request when using the /add endpoint. In certain situations, the load balancer ensures  the specific num_add parameter is added. The load balancer will produce new hostnames for additional servers to meet the precise amount given by num_add, even if the user gives hostnames that currently exist in the system.
<li> The /rm endpoint allows users to provide hostnames to be removed. In order to guarantee that the number of servers to be eliminated is always the same, the load balancer uses a technique in which, in the event that the hostname supplied by the user is not already in the system, it chooses at random a server hostname to remove from the list.
<li> A heartbeat thread that transmits a heartbeat message every 0.2 seconds is installed on every server. A new server spawns when a server is pronounced dead, which happens after two attempts without detecting a heartbeat. This method ensures system stability by preventing premature server death declarations caused by network oscillations.</li>
</ol>


# Evaluation

## Server Load Analysis
To analyze the distribution load of 10,000 asynchronous requests on N=3 servers, follow these commands:

#### Change to the 'analysis' directory:

```bash
cd analysis
```
#### Run the Python script for analysis:
```bash
python analysis.py
```

The analysis.py script sends 10,000 asynchronous requests (/home requests) to the load balancer. Based on the responses received, it plots a frequency map illustrating how many responses come from each server.

Ensure that the necessary dependencies are installed and the system is properly configured before executing the analysis script.

<img src="analysis/old_hash/n_3_load.png" >



## Server Avg Load Analysis

Modify the NUM_INITIAL_SERVERS = 3 line number 9 of `load_balancer/client_handler.py` with 2 to 6 and run analysis.py to get average load and standard distribution of load distribution.
Follow folder `analysis/old_hash/` for exact distribution bar plots. 

<img src = "analysis/part2.png">

## Server Failure Testing

To analyze the server failure scenario execute the following:

#### Change to the 'analysis' directory:

```bash
cd analysis
```
#### Run the Python script for analysis:
```bash
python kill_server.py
```

The kill_server.py script is designed to simulate the process of killing a server, validating the functionality of all endpoints in the event of server failure, and assessing the prompt respawn of the server. The effectiveness of server respawn is verified through the analysis of load distribution and the count of dropped requests(failed) during the time it takes for the heartbeat mechanism to detect and respawn the server.

## Hashing Function Variation Analysis
After testing all the 9 combinations of **SHA-256, SHA-1, and MD5** hashing functions for request and server hashings, we found that the most effective combination is:

- Server Hashing: SHA-1
- Request Hashing: MD5

To implement this new hash function configuration, follow these steps:

1. Open the file `load_balancer/consistent_hashing.py` and locate the `server_hash_func`  and `request_hash_func` definition.

2. Comment the alternative hash function provided. 

3. Save the changes.

4. Re-run the analysis script after deploying the load balancer again.

5. Ensure that you have cleared previous containers and images, and rebuild the project before executing the analysis script. This ensures that the new hash function is applied to the load balancer and the analysis is based on the updated configuration.

The evaluation results of new hash function on variable number of servers are present in `analysis/md5_hash`

<img src="analysis/part2_new.png">

</font>
