## Lab 04 - Docker

[TOC]

### Introduction

**DISCLAIMER**: In this lab, we will go through one possible approach
to manage a scalable infrastructure where we can add and remove nodes
without having to rebuild the HAProxy image. This is not the only way
to achieve this goal. If you do some research you will find a lot of
tools and services to achieve the same kind of behavior.


### Task 0: Identify issues and install the tools

#### Identify issues

> **[M1]** Do you think we can use the current solution for a production environment? What are the main problems when deploying it in a production environment?

The solution is not suitable for a real production environment likely to receive a high number of requests. If the load balancer container crashes, there is no backup solution : next requests will fail to reach one of the servers.



> **[M2]** Describe what you need to do to add new `webapp` container to the infrastructure. Give the exact steps of what you have to do without modifying the way the things are done. Hint: You probably have to modify some configuration and script files in a Docker image.

`webapp1` and `webapp2` are docker containers, coming from the same docker image and using the same `Dockerfile` (located in `/webapp`). They are both defined in the file `docker-compose.yml`.

Actually, these two web application servers are defined in HAProxy configuration file as nodes to be included in the balancing mechanism.

In order to add a new webapp container to the infrastructure, we took the following steps :

1. Add a new `webapp3` service in `docker-compose.yml` located in the project root directory.

```dockerfile
services:
	webapp3:
       container_name: ${WEBAPP_3_NAME}
       build:
         context: ./webapp
         dockerfile: Dockerfile
       networks:
         heig:
           ipv4_address: ${WEBAPP_3_IP}
       ports:
         - "4002:3000"
       environment:
            - TAG=${WEBAPP_3_NAME}
            - SERVER_IP=${WEBAPP_3_IP}
```

2. Since we used environment variables (`${}`), we need to define them in the **hidden** file `.env`, also located in the project root directory.

```bash
WEBAPP_3_NAME=s3
WEBAPP_3_IP=192.168.42.33
```

3. Now, we have to add the new webapp in `/ha/config/haproxy.cfg` as a node.

```bash
server s3 ${WEBAPP_3_IP}:3000 check
```



> **[M3]** Based on your previous answers, you have detected some issues in the current solution. Now propose a better approach at a high level.

Actually, it is impossible to add a new web application server dynamically.

The solution would be to allow HAProxy to include a new node when a new server is available. Practically, when a new web application container would be created, it could send the load balancer a message saying that it is available and ready to be used as a node.



> **[M4]** You probably noticed that the list of web application nodes is hardcoded in the load balancer configuration. How can we manage the web app nodes in a more dynamic fashion?

A solution would be to use a tool like `Serf`, offering cluster membership, failure detection and orchestration. The load balancer could benefit from an efficient lightweight `gossip protocol` to communicate with nodes and exchange messages periodically.



> **[M5]** In the physical or virtual machines of a typical infrastructure we tend to have not only one main process (like the web server or the load balancer) running, but a few additional processes on the side to perform management tasks.
>
> For example to monitor the distributed system as a whole it is common to collect in one centralized place all the logs produced by the different machines. Therefore we need a process running on each machine that will forward the logs to the central place. (We could also imagine a central tool that reaches out to each machine to gather the logs. That's a push vs. pull problem.) It is quite common to see a push mechanism used for this kind of task. 
>
> Do you think our current solution is able to run additional management processes beside the main web server / load balancer process in a container? If no, what is missing / required to reach the goal? If yes, how to proceed to run for example a log forwarding process?

We think that it is not actually possible to run an additional management process beside the container main process. Our Docker containers are not capable of running multiple processes, making it hard to centralize the logs in one place.

Like said on this [docker docs page](https://docs.docker.com/config/containers/multi-service_container/), there are a few different ways to accomplish this :

* Put all the commands in a wrapper script and run it as `CMD` in the Dockerfile
* **Use a process manager** like `supervisord`



> **[M6]** In our current solution, although the load balancer configuration is changing dynamically, it doesn't follow dynamically the configuration of our distributed system when web servers are added or removed. If we take a closer look at the `run.sh` script, we see two calls to `sed` which will replace two lines in the `haproxy.cfg` configuration file just before we start `haproxy`. You clearly see that the configuration file has two lines and the script will replace these two lines.
> 
>What happens if we add more web server nodes? Do you think it is really dynamic? It's far away from being a dynamic configuration. Can you propose a solution to solve this?

The load balancer doesn't know when a configuration changes or when a node goes online/offline. A new node could replace a crashed node and the load balancer would not know it.

Like said before, there are solutions like `Serf` to maintain cluster membership lists and execute custom handler scripts when that membership changes. For example, `Serf` can maintain the list of web servers for a load balancer and notify that load balancer whenever a node comes online or goes offline.



#### Install the tools

**Deliverables**:

1. Take a screenshot of the stats page of HAProxy at
   <http://192.168.42.42:1936>. You should see your backend nodes.

   ![managementInterface](../img/0_1.png)
   
2. Give the URL of your repository URL in the lab report.

[https://github.com/Tsimwi/Teaching-HEIGVD-AIT-2019-Labo-Docker](


### Task 1: Add a process supervisor to run several processes

**Deliverables**:

1. Take a screenshot of the stats page of HAProxy at
   <http://192.168.42.42:1936>. You should see your backend nodes. It
   should be really similar to the screenshot of the previous task.

![ManagementInterface](../img/1_1.png)

1. Describe your difficulties for this task and your understanding of
   what is happening during this task. Explain in your own words why
   are we installing a process supervisor. Do not hesitate to do more
   research and to find more articles on that topic to illustrate the
   problem.

> We install a supervisor because we want to run multiple process on the same container, but the mantra of Docker is "one process per container" so we need to find a way to bypass this behaviors. To do that we use a supervisor, it will be the process with the pid 1. He will be responsible to launch other process.
>
> 
>
> ![ManagementInterface](../img/1_2.png)
>
> On this image we can see that systemd start right after the kernel and manage other services.



### Task 2: Add a tool to manage membership in the web server cluster

**Deliverables**:

> 1. Provide the docker log output for each of the containers: `ha`, `s1` and `s2`. You need to create a folder `logs` in your repository to store the files separately from the lab report. For each lab task create a folder and name it using the task number. No need to create a folder when there are no logs.
>
>    Example :
>
>    ```bash
>    |-- root folder
>      |-- logs
>        |-- task 1
>        |-- task 3
>        |-- ...
>    ```

See `logs/task2`.

> 2. Give the answer to the question about the existing problem with the current solution.

We think that the problem is 

> 3. Give an explanation on how `Serf` is working. Read the official website to get more details about the `GOSSIP` protocol used in `Serf`. Try to find other solutions that can be used to solve similar situations where we need some auto-discovery mechanism.

Once started, a `serf` node either join an existing cluster or start a new one. To join an existing cluster, a node only needs to be given the name/address of an existing member of this cluster. For example, if `ha` and `s1` are in the same cluster, `s2` can be given either `s1` or `ha` address to join them.

Then, the new member synchronizes its state with the existing member over TCP. Once done, it can begin gossiping its existence to the cluster over UDP, using `gossip` protocol. 

Gossip is done at regular intervals, ensuring constant network usage. Failure detection is also done by periodic random probing. `Serf` has a dedicated gossip layer, separated from the failure detection protocol, allowing to have a higher gossip rate and a slower failure detection rate.

If a failing node is discovered, random other nodes are asked to probe the failing node, in case there where network issues with the first node. A failing node first become *suspicious* before being considered *dead*, and all these state changes are gossiped to the cluster.

For example, in the [NeighbourCast](http://www.actapress.com/PDFViewer.aspx?paperId=31994) algorithm, instead of talking to random nodes, information is spread by talking only to neighbouring nodes.

There is also Consul that works like Serf. [Consul](https://www.consul.io) is a tool for service discovery and configuration. It provides high level features such as service discovery, health checking and key/value storage. It makes use of a group of strongly consistent servers to manage the datacenter.

### Task 3: React to membership changes

**Deliverables**:

> 1. Provide the docker log output for each of the containers:  `ha`, `s1` and `s2`.
>    Put your logs in the `logs` directory you created in the previous task.

[ha startup](../logs/task3/ha_startup)

[s1 when it join the cluster](../logs/task3/s1_after_join)

[ha after that s1 join](../logs/task3/ha_after_s1_join)

[s2 when it join the cluster](../logs/task3/s2_after_join)

[ha after that s2 join the cluster](../logs/task3/ha_after_s1_and_s2_join)

[s1 after that s2 join the cluster](../logs/task3/s1_after_s2_join)

> 2. Provide the logs from the `ha` container gathered directly from the `/var/log/serf.log`
>    file present in the container. Put the logs in the `logs` directory in your repo.

[Serf cluster logs](../logs/task3/ha_serf_logs)


### Task 4: Use a template engine to easily generate configuration files

**Deliverables**:

> 1. You probably noticed when we added `xz-utils`, we have to rebuild
>    the whole image which took some time. What can we do to mitigate
>    that? Take a look at the Docker documentation on
>    [image layers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/#images-and-layers).
>    Tell us about the pros and cons to merge as much as possible of the
>    command. In other words, compare:
>
> ```
> RUN command 1
> RUN command 2
> RUN command 3
> ```
>
> vs.
>
> ```
> RUN command 1 && command 2 && command 3
> ```
>
> There are also some articles about techniques to reduce the image
>   size. Try to find them. They are talking about `squashing` or
>   `flattening` images.

A Docker image is built up from a series of layers. Each layer represents an instruction in the image’s Dockerfile. Each layer except the very last one is read-only. Each layer is only a set of differences from the layer before it. The layers are stacked on top of each other. When you create a new container, you add a new writable layer on top of the underlying layers. This layer is often called the “container layer”. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this thin writable container layer. Layers can be shared by container if they need the same usage.

So if you chain commands like in the first example it will create 3 different layer, all of them could be reused by other container but it will represent 3 different layers and will took more place. In the second example it will be only one layer with all the information. So if other container want to take this layer they need to have the exact same line in the Dockerfile.

`Squashing` image mean that you take all layer of a running container and with them you make only on custom layer. [Here there is an explanation and a tool](http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/)

`Flattening` a container mean that we are going to "hide" all the layer in one. For example if in the Dockerfile have secret as environment variable it will be easy to get that specific layer (with it ID) and to get secrets. So we are going to export and import the container. In fact when we export a container it will lost his history so it won't be possible to brows his layer history. [More info here](https://medium.com/@l10nn/flattening-docker-images-bafb849912ff) [and here](https://tuhrig.de/flatten-a-docker-container-or-image/)

> 2. Propose a different approach to architecture our images to be able
>    to reuse as much as possible what we have done. Your proposition
>    should also try to avoid as much as possible repetitions between
>    your images.

The idea would be to make as more as possible the same layer in all containers. For that we should make same commands in the Dockerfiles. For example if all containers need to install common package we will make one command that install these packages. We could also make a custom images with Serf and s6 installed by default and then make custom commands in Dockerfiles (squashing).

> 3. Provide the `/tmp/haproxy.cfg` file generated in the `ha` container
>    after each step.  Place the output into the `logs` folder like you
>    already did for the Docker logs in the previous tasks. Three files
>    are expected.
>
>    In addition, provide a log file containing the output of the 
>    `docker ps` console and another file (per container) with
>    `docker inspect <container>`. Four files are expected.

[haproxy.cfg after ha join](../logs/task4/ha_1)

[haproxy.cfg after s1 join](../logs/task4/ha_2)

[haproxy.cfg after hs2 join](../logs/task4/ha_3)

[docker ps](../logs/task4/docker_ps)

[docker inspect ha](../logs/task4/inspect_ha)

[docker inspect s1](../logs/task4/inspect_s1)

[docker inspect s2](../logs/task4/inspect_s2)

We can see that we have the "history" of who join the cluster. Note it would be better in the command that write in this file to happen result instead of overwriting it. 

> 4. Based on the three output files you have collected, what can you
>    say about the way we generate it? What is the problem if any?

One problem is that, as we said before, we overwrite the file instead of happen to it. An other problem is that we didn't get the information about a node leaving the cluster, we should do the same in the script for a leaving node. 


### Task 5: Generate a new load balancer configuration when membership changes

**Deliverables**:

> 1. Provide the file `/usr/local/etc/haproxy/haproxy.cfg` generated in
>    the `ha` container after each step. Three files are expected.
>
>    In addition, provide a log file containing the output of the 
>    `docker ps` console and another file (per container) with
>    `docker inspect <container>`. Four files are expected.

[haproxy.cfg after ha join](../logs/task5/ha_1)

[haproxy.cfg after s1 join](../logs/task5/ha_2)

[haproxy.cfg after s2 join](../logs/task5/ha_3)

[docker ps](../logs/task5/docker_ps)

[docker inspect ha](../logs/task5/inspect_ha)

[docker inspect s1](../logs/task5/inspect_s1)

[docker inspect s2](../logs/task5/inspect_s2)

> 2. Provide the list of files from the `/nodes` folder inside the `ha` container.
>    One file expected with the command output.

[liste /nodes](../logs/task5/ls_node)

> 3. Provide the configuration file after you stopped one container and
>    the list of nodes present in the `/nodes` folder. One file expected
>    with the command output. Two files are expected.
>
>    In addition, provide a log file containing the output of the 
>    `docker ps` console. One file expected.

[haproxy.cfg after removing on backend](../logs/task5/config_after_removing_one_container)

[liste /nodes](../logs/task5/ls_node_after_removing_one)

[docker ps after removing one backend](../logs/task5/docker_ps_after_removing_one_container)

> 4. (Optional:) Propose a different approach to manage the list of backend
>    nodes. You do not need to implement it. You can also propose your
>    own tools or the ones you discovered online. In that case, do not
>    forget to cite your references.

We could use [Traefic](https://docs.traefik.io/) for the reverse proxy. It will replace the Serf cluser and match the docker idea of one container for one service. [Portainer](https://www.portainer.io/) could be use to manager containers

### Task 6: Make the load balancer automatically reload the new configuration

**Deliverables**:

> 1. Take a screenshots of the HAProxy stat page showing more than 2 web
>    applications running. Additional screenshots are welcome to see a
>    sequence of experimentations like shutting down a node and starting
>    more nodes.
>
>    Also provide the output of `docker ps` in a log file. At least 
>    one file is expected. You can provide one output per step of your
>    experimentation according to your screenshots.



> 2. Give your own feelings about the final solution. Propose
>    improvements or ways to do the things differently. If any, provide
>    references to your readings for the improvements.

This final solution is fine but not very good. Indeed we have the problem that if all backend nodes fail there is no mecanisme to redeploy them automaticaly. To do that a solution would be to set a number of wanted backend component and on each "leaving cluster" event we look if we vace less backend component than ou setted value. If we have less container than expecter we run a script that will redeploy N container to match the setted value.

An other problem is that the reverse proxy is an "one point of failure", that mean that if the reverse proxy container is down everything will fail. To avoid this, a solution would be to have multiple reverse proxy container that share a Virual IP. The virtual IP would be the entry point and there is a active reverse proxy container and others are passivs. If the active reverse proxy fail an passive one become the active.

![](ha-diagram-animated.gif)

[More information about VIP config](https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-haproxy-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04)

> 3. (Optional:) Present a live demo where you add and remove a backend container.

To add and remove backend containers we simply have to run the command `docker-compose -f docker-compose-scale.yml up --scale webapp=N` where N is the **total** number of backend containers that we want. There are 2 videos to demonstrate.

#### Add containers

![](add_webapp.gif)

#### Remove containers

![](remove.gif)



### Difficulties

### Conclusion

...