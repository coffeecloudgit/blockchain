# 以太坊 Prysm 和 Geth 服务 Docker 部署 README

## 一、概述
本项目旨在通过 Docker 容器技术，使用 Prysm 信标链客户端和 Geth 执行客户端来搭建以太坊服务。但在部署和运行过程中可能会遇到一些诸如连接认证方面的问题，以下文档将详细介绍整个部署流程以及对应问题的解决方法。
## 二、准备工作
安装 Docker：确保你的系统已经安装好了 Docker。不同操作系统安装方式各异，例如在 Ubuntu 系统中，可通过以下命令安装：

```
sudo apt update
sudo apt install docker.io
```

安装完成后，使用 docker --version 命令确认安装成功并查看版本信息。
拉取镜像：
需要拉取 Geth 和 Prysm 的 Docker 镜像，执行以下命令拉取最新版本（你也可以按需指定具体版本号）：

```
docker pull ethereum/client-go:latest
docker pull gcr.io/prysmaticlabs/prysm/beacon-chain:latest
```

## 三、创建桥接网络

为了让 Geth 和 Prysm 容器能够相互通信，我们需要创建一个 Docker 桥接网络，使用以下命令创建名为 ethereum-network 的桥接网络：

```
docker network create ethereum-network
```
## 四、运行 Geth 容器

运行 Geth 容器并挂载相关数据目录、配置端口映射以及开启必要的服务，同时连接到创建的 ethereum-network 桥接网络，示例命令如下：

```
## 六、生成 JWT 令牌 在配置认证之前，需要生成一个 JWT 令牌，可通过以下命令实现：

openssl rand -hex 32 > /your/local/jwtsecret

docker run -it --name my-geth-node \
  --network ethereum-network \
  -v /your/local/geth/data:/data \
  -v /your/local/jwtsecret:/jwtsecret \
  -p 8545:8545 \
  -p 8551:8551 \
  -p 30303:30303 \
  ethereum/client-go:latest \
  --http --http.addr "0.0.0.0" --http.api "eth,net,web3,personal" --http.port 8545 \
  --authrpc.addr "0.0.0.0" --authrpc.port 8551 --authrpc.vhosts "*" \
  --datadir /data \
  --authrpc.jwtsecret /jwtsecret
```

参数说明：
-v /your/local/geth/data:/data：将宿主机上的 /your/local/geth/data 目录挂载到容器内的 /data 目录，用于存储 Geth 的数据。
-v /your/local/jwtsecret:/jwtsecret：挂载包含 JWT 令牌的文件到容器内的 /jwtsecret 目录，用于后续认证。
-p 8545:8545：将容器内的 8545 端口（HTTP API 服务端口）映射到宿主机的 8545 端口，方便外部应用访问。
-p 8551:8551：将容器内的 8551 端口（Authrpc 服务端口）映射到宿主机的 8551 端口，用于与信标链客户端通信。
-p 30303:30303：将容器内的 30303 端口（P2P 通信端口）映射到宿主机的 30303 端口，方便 Geth 与其他以太坊节点通信。
--http --http.addr "0.0.0.0" --http.api "eth,net,web3,personal" --http.port 8545：开启 HTTP 服务，允许来自任何地址的连接，开放 eth、net、web3 和 personal API，并设置 HTTP 服务端口为 8545。
--authrpc.addr "0.0.0.0" --authrpc.port 8551 --authrpc.vhosts "*"：开启 Authrpc 服务，允许来自任何地址的连接，设置端口为 8551。
--datadir /data：指定 Geth 的数据存储在 /data 目录。
--authrpc.jwtsecret /jwtsecret：告诉 Geth 使用 /jwtsecret 目录下的 JWT 令牌进行认证。

## 五、运行 Prysm 容器

运行 Prysm 信标链容器，同样挂载相关目录、配置端口映射，并连接到 ethereum-network 桥接网络，同时配置好与 Geth 容器通信的执行端点以及 JWT 认证相关设置，示例命令如下：

```
docker run -it --name my-prysm-beacon \
  --network ethereum-network \
  -v /your/local/prysm/data:/data \
  -v /your/local/prysm/config:/config \
  -v /your/local/jwtsecret:/jwtsecret \
  -p 4000:4000 \
  -p 13000:13000 \
  gcr.io/prysmaticlabs/prysm/beacon-chain:latest \
  --datadir /data \
  --config-file=/config/prysm_config.yaml \
  --execution-endpoint=http://my-geth-node:8551 \
  --jwt-secret=/jwtsecret
```

参数说明：
-v /your/local/prysm/data:/data：将宿主机上的 /your/local/prysm/data 目录挂载到容器内的 /data 目录，用于存储 Prysm 的数据。
-v /your/local/prysm/config:/config：将宿主机上的 /your/local/prysm/config 目录挂载到容器内的 /config 目录，用于存储配置文件。
-v /your/local/jwtsecret:/jwtsecret：挂载包含 JWT 令牌的文件到容器内的 /jwtsecret 目录，用于认证。
-p 4000:4000 和 -p 13000:13000：将容器内的 4000 和 13000 端口（Prysm 常用服务端口）映射到宿主机的相应端口，方便通信。
--datadir /data：指定 Prysm 的数据存储在 /data 目录。
--config-file=/config/prysm_config.yaml：使用 /config 目录下的 prysm_config.yaml 作为配置文件，确保该文件存在且配置正确。
--execution-endpoint=http://my-geth-node:8551：将 Prysm 的执行端点指向 Geth 的 Authrpc 服务（通过容器名称 my-geth-node 访问，端口为 8551）。
--jwt-secret=/jwtsecret：告诉 Prysm 使用 /jwtsecret 目录下的 JWT 令牌进行认证。

此命令会生成一个 32 字节的随机十六进制字符串，并将其存储在 jwtsecret 文件中。
注意事项：
确保 /your/local/geth/data、/your/local/prysm/data、/your/local/prysm/config 和 /your/local/jwtsecret 等目录在宿主机上存在，并且具有适当的权限，以便 Docker 容器可以进行读写操作。
确保生成的 JWT 令牌文件的安全性，避免泄露，因为它是认证的关键。

七、检查容器状态和日志
在容器启动后，可以通过以下命令检查容器是否正常运行以及查看日志信息，便于排查可能出现的问题：
bash
docker ps
docker logs -f my-geth-node
docker logs -f my-prysm-beacon
