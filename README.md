# docker-swarm-demo

『Docker実践ガイド第2版』の Docker swarm overlay network の説明を Vagrant で再現する。  
書籍は[こちら](https://amzn.to/2CK1X0D)(Amazonアソシエイト)

## 手順

### Vagrant で VM 作成

Vagrant, Virtual Box は導入済みとする。以下に3つの `Vagrantfile` があるため、それぞれのディレクトリで `vagrant up` して VM を作成する。`Vagrantfile` 中では IP アドレスの設定、Docker, Docker-compose のインストール、設定などを行っている。

```sh
$ cd manager_node
$ vagrant up
$ vagrant ssh
  (前略)
  System load:  0.44              Users logged in:        0
  Usage of /:   3.6% of 61.80GB   IP address for eth0:    10.0.2.15
  Memory usage: 35%               IP address for eth1:    172.16.1.171
  Swap usage:   0%                IP address for docker0: 172.17.0.1
  Processes:    94
  (後略)
```

```sh
$ cd worker_node1
$ vagrant up
$ vagrant ssh
  (前略)
  System load:  0.53              Users logged in:        0
  Usage of /:   3.6% of 61.80GB   IP address for eth0:    10.0.2.15
  Memory usage: 35%               IP address for eth1:    172.16.1.172
  Swap usage:   0%                IP address for docker0: 172.17.0.1
  Processes:    94
  (後略)
```

```sh
$ cd worker_node2
$ vagrant up
$ vagrant ssh
  (前略)
  System load:  0.19              Users logged in:        0
  Usage of /:   3.6% of 61.80GB   IP address for eth0:    10.0.2.15
  Memory usage: 35%               IP address for eth1:    172.16.1.173
  Swap usage:   0%                IP address for docker0: 172.17.0.1
  Processes:    94
  (後略)
```

### manager の起動

manager node 内で、

```sh
$ docker swarm init --advertise-addr=172.16.1.171
Swarm initialized: current node (87keqmiomwh27r3m84y0rpj79) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-05q61jzfvl7qxwaq46r9gvp4kuwayuq9fs0jcq9h9bn5fvgfuf-9s704xy00gzo8vvz9z81k198n 172.16.1.171:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
87keqmiomwh27r3m84y0rpj79 *   vagrant             Ready               Active              Leader              19.03.12

$ docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-05q61jzfvl7qxwaq46r9gvp4kuwayuq9fs0jcq9h9bn5fvgfuf-9s704xy00gzo8vvz9z81k198n 172.16.1.171:2377
```

### worker node の起動

worker node 1 および 2 内で、


```sh
$ docker swarm join --token SWMTKN-1-05q61jzfvl7qxwaq46r9gvp4kuwayuq9fs0jcq9h9bn5fvgfuf-9s704xy00gzo8vvz9z81k198n 172.16.1.171:2377
This node joined a swarm as a worker.
```

した後、manager node 内で、

```sh
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
02d8mkhadv9zxvm33hn5tol04     vagrant             Ready               Active                                  19.03.12
2qg91dx6swvqft7p2hojkak55     vagrant             Ready               Active                                  19.03.12
87keqmiomwh27r3m84y0rpj79 *   vagrant             Ready               Active              Leader              19.03.12
```

```sh
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
35745daaa485        bridge              bridge              local
1d085944cf9e        docker_gwbridge     bridge              local
487579679c0e        host                host                local
64d7vh2ktjav        ingress             overlay             swarm
06f7f66cd3e3        none                null                local
```

### creating overlay network

```sh
$ docker network create -d overlay --subnet 11.0.0.0/24 --attachable mynet01
li5pgomgss5vxei0t10bzg0i0

# 書籍では 10.0.0.0/24 となっているが、VirtualBox が使っている(?) ため 11.0.0.0/24 に変更。

$ docker network inspect mynet01
[
    {
        "Name": "mynet01",
        "Id": "li5pgomgss5vxei0t10bzg0i0",
        "Created": "2020-06-26T09:01:56.253826461Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "11.0.0.0/24",
                    "Gateway": "11.0.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": null,
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4101"
        },
        "Labels": null
    }
]
```

### running containers

worker node1 で web server を立ち上げる。

```sh
$ docker run -itd --name websvr01 -h websvr01 --network mynet01 --ip 11.0.0.2 larsks/thttpd
2a136991fe9c36941d0fe006eeac416ab0343fceb2d19872f5aa03e60e105984
```

worker node2 内で、client を起動して server にアクセス。

```
$ docker run -it --name client01 -h client01 --network mynet01 --ip 11.0.0.9 centos bash

# コンテナ内で、

# curl http://11.0.0.2
<!DOCTYPE html>
<html>
        <head>
                <title>Your web server is working</title>
    <style type="text/css">
    body {
      text-align: center;
      font-family: Arial,"Helvetica Neue",Helvetica,sans-serif;
    }
    pre {
      border: thin solid black;
      padding: 1em;
      background-color: #c0c0c0;
    }

    #summary {
      max-width: 40em;
      margin: auto;
      text-align: left;
    }
    </style>
        </head>
        <body>
  <div id="header">
  <pre>
  ____                            _         _       _   _
 / ___|___  _ __   __ _ _ __ __ _| |_ _   _| | __ _| |_(_) ___  _ __  ___
| |   / _ \| '_ \ / _` | '__/ _` | __| | | | |/ _` | __| |/ _ \| '_ \/ __|
| |__| (_) | | | | (_| | | | (_| | |_| |_| | | (_| | |_| | (_) | | | \__ \
 \____\___/|_| |_|\__, |_|  \__,_|\__|\__,_|_|\__,_|\__|_|\___/|_| |_|___/
                  |___/
  </pre>

  <p><strong>You have a web server.</strong></p>
</div>

  <div id="summary">
    <p>This is a statically compiled version of <a href="http://acme.com/software/thttpd/">thttpd</a>
    put together to build a demonstration container for my
    <a href="https://github.com/larsks/heat-kubernetes">Heat templates for Kubernetes</a>.  But maybe
    you'll find it useful for other things.</p>
  </div>
        </body>
</html>
```

これで疎通確認ができた。なお、 master node で、以下のようにできたため、master node でコンテナを動かしても問題ない。

```sh
$ docker run -it --name client01 -h client01 --network mynet01 --ip 11.0.0.8 centos bash

# コンテナ内で、

#  curl http://11.0.0.2
<!DOCTYPE html>
<html>
        <head>
                <title>Your web server is working</title>
    <style type="text/css">
    body {
      text-align: center;
      font-family: Arial,"Helvetica Neue",Helvetica,sans-serif;
    }
    pre {
      border: thin solid black;
      padding: 1em;
      background-color: #c0c0c0;
    }

    #summary {
      max-width: 40em;
      margin: auto;
      text-align: left;
    }
    </style>
        </head>
        <body>
  <div id="header">
  <pre>
  ____                            _         _       _   _
 / ___|___  _ __   __ _ _ __ __ _| |_ _   _| | __ _| |_(_) ___  _ __  ___
| |   / _ \| '_ \ / _` | '__/ _` | __| | | | |/ _` | __| |/ _ \| '_ \/ __|
| |__| (_) | | | | (_| | | | (_| | |_| |_| | | (_| | |_| | (_) | | | \__ \
 \____\___/|_| |_|\__, |_|  \__,_|\__|\__,_|_|\__,_|\__|_|\___/|_| |_|___/
                  |___/
  </pre>

  <p><strong>You have a web server.</strong></p>
</div>

  <div id="summary">
    <p>This is a statically compiled version of <a href="http://acme.com/software/thttpd/">thttpd</a>
    put together to build a demonstration container for my
    <a href="https://github.com/larsks/heat-kubernetes">Heat templates for Kubernetes</a>.  But maybe
    you'll find it useful for other things.</p>
  </div>
        </body>
</html>
```
