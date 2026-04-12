# Getting Started

This shows how to install Mium on local to experience it quickly.

## Prerequisites

Because Mium is written in Java, Java 17 needs to be installed on local.

## Install Mium on Local

Mium distribution can be downloaded like this.

```agsl
curl -L -O https://github.com/cloudcheflabs/mium-pack/releases/download/mium-archive/mium-1.0.0.tar.gz
```

And untar the downloaded package.
```agsl
tar zxvf mium-1.0.0.tar.gz

cd mium-1.0.0;
```

Run the example servers which are 1 Master and 1 Worker with Zookeeper on local.

```agsl
export MIUM_MASTER_KEY="MiumMasterKey12003030030000abcde"
bin/start-example-servers.sh;
```

> Environment variable `MIUM_MASTER_KEY` that must be at least 32 characters needs to be exported when running Mium servers.


After a few seconds, visit admin page of Mium.

```agsl
http://localhost:8090/admin
```

First initial admin user and password is `admin` / `admin`, after that you need to change the initial password.

<img width="1200" src="../../images/getting-started/dashboard.png"/>


## Stop Example Servers

In order to stop the running example servers, run this.

```agsl
bin/stop-example-servers.sh
```
