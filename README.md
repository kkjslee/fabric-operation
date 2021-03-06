# fabric-operation

This project contains scripts that let you define, create, and test a Hyperledger Fabric network in Kubernetes locally or in a cloud.  Supported cloud services include AWS, Azure, and Google Cloud.  The fabric network parameters can be specified by a property file, such as the sample network, [netop1.env](./config/netop1.env).

The scripts support both `docker-compose` and `kubernetes`.  All steps are done in docker containers, and thus you can get a Fabric network running without pre-downloading any artifact of Hyperledger Fabric.

This utility is implemented using bash scripts, and thus it does not depend on any other scripting tool or framework, either.

## Prerequisites
* Your workstation must support `bash` shell scripts.
* If you want to create and test a Fabric network on local host, you need to install docker-compose and/or kubernetes locally, i.e.,
  * Install Docker and Docker Compose as described [here](https://docs.docker.com/compose/install/).
  * Mac user can enable kubernetes as described [here](https://docs.docker.com/docker-for-mac/#kubernetes).
  * I have not tested the scripts with [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/), although I would expect it to work without much change.
* If you want to create and test a Fabric network in a cloud, you would not need to download anything except a `CLI` required to access the corresponding cloud service.  We currently support Amazon EKS, Azure AKS, and Google GKE.  Other cloud services may be supported in the future. 
  * For AWS, refer the scripts and instructions in the [aws folder](./aws).
  * For Azure, refer the scripts and instructions in the [az folder](./az).
  * For Google cloud, refer the scripts and instructions in the [gcp folder](./gcp)

## Prepare Kubernetes namespace
This step is necessary only if you use Kubernetes.  So, skip it when `docker-compose` is used.
```
cd ./namespace
./k8s-namespace.sh create
```
This command creates a namespace for the default Fabric operator company, `netop1`. It also sets `netop1` as the default namespace, so you won't have to specify the namespace in the following `kubectl` commands.

To revoke to the default namespace for `docker-desktop`, you can use the following command:
```
kubectl config use-context docker-desktop
```
## Start CA server and generate crypto data
Following steps use `docker-desktop` Kubernetes on Mac to start `fabric-ca` PODs and generate crypto data required by the sample network, `netop1`.
```
cd ../ca
# cleanup old ca-server data
rm -R ../netop1.com/canet
./ca-server.sh start
# wait until the 3 PODs for ca server and client are in running state
./ca-crypto.sh bootstrap
```
You can edit the network specification [netop1.env](./config/netop1.env) if you want to use a different operating company name, or make it run more orderer or peer nodes.  The generated crypto data will be stored in the folder [netop1.com](./netop1.com) on localhost, or in a cloud file system, such as Amazon EFS, or Azure Files. 

These scripts take 2 additional parameters, e.g.,
```
./ca-server.sh start -p <config_file> -t <env_type>
./ca-crypto.sh bootstrap -p <config_file> -t <env_type>
```
where `config_file` is file in the [config](./config) folder with a suffix `.env` that contains the fabric network specification; `env_type` can be `k8s`, `docker`, `aws`, or `az`.  When no parameter is specified, it uses default `-p netop1 -t k8s`.  Refer [ca](./ca) folder for more detailed description of these scripts.
* `k8s` uses the local `docker-desktop` kubernetes on Mac.  Non-Mac users may use `docker` option below, or try Minikube (which has not been tested).
* `docker` uses `docker-compose`.
* `aws` uses AWS EKS when executed on a `bastion` host of an EC2 instance.  Refer the folder [aws](./aws) for more details on AWS.
* `az` uses Azure AKS when executed on a `bastion` VM instance in Azure.  Refer the folder [az](./az) for more details on Azure.
More cloud support will be added in the future.
* `gcp` uses Google GKE when executed on a `bastion` host in Google Cloud.  Refer the folder [gcp](./gcp) for more details on Google Cloud.

## Sample crypto data
When the above steps are executed on localhost, the crypto data will be stored in [netop1.com](./netop1.com/), which is specified by `FABRIC_ORG` in the network definition file [netop1.env](./config/netop1.env).  The resulting crypto data is similar to that generated by the fabric `cryptogen` tool as demonstrated by [fabric-samples](https://github.com/hyperledger/fabric-samples). However, by using a fabric CA server in the above step, the generated certificates will include a few extra attributes that would make them usable for cloud deployment using kubernetes, as well as attribute-based-access-control (ABAC).  Besides, CA server is also more flexible for generating certificates for more nodes and users in production environment as the network grows.  Although the CA servers use a self-signed root CA for simplicity, you may supply your real root CA for production deployment.

You may verify the generated crypto data by using a preconfigured sample network as described in [docker-netop1](./docker-netop1).  However, if you do not have a local hyperledger fabric environment, you can skip the test and read on.  The following steps will show you how to start a fabric network by using a few simple scripts even if you do not have a fabric development environment.

## Generate MSP definition and genesis block
The following script generates a genesis block for the sample network in Kubernetes using 2 peers and 3 orderers with `etcd raft` consensus.
```
cd ../msp
./msp-util.sh start
# wait until the too POD is running
./msp-util.sh bootstrap
```
It also generates transactions for creating a test channel, `mychannel`, for smoke testing.  Similar to other scripts, this command also accepts 2 parameters, e.g.,
```
./msp-util.sh start -p <config_file> -t <env_type>
./msp-util.sh bootstrap -p <config_file> -t <env_type>
```
so you can specify a different network definition file, or generate artifacts for other deployment environment, e.g., `docker`, `aws`, `az`, or `gcp`. Refer [msp](./msp) folder for more detailed description of these scripts.

## Start and smoke test the Fabric network
The following script will start and test the sample fabric network by using the `docker-desktop` Kubernetes on a Mac:
```
cd ./network
./network.sh start
# wait until 3 orderer and 2 peer nodes are running, Raft leader is elected in orderers
./network.sh test
./network.sh shutdown
```
After the network startup, use `kubectl logs orderer-2` to check RAFT leader election result.  When RAFT leader is elected, the log should show
```
INFO 101 Raft leader changed: 0 -> 2 channel=netop1-channel node=2
```
Before you shutdown the network, you can verify the running fabric containers by using `kubectl`, e.g.,
```
kubectl get pod,svc -n netop1
```
Note that the scripts use the operating company name `netop1`, as a Kubernetes namespace, and so they can support multiple member organizations.

After the smoke test succeeds, you should see a test result of `90` printed on the screen. If you used `docker-compose` for this excersize (as described below), you can look at the blockchain state via the `CouchDB` futon UI at `http://localhost:7056/_utils`, which is exposed for `docker-compose` only because it is not recommended to expose `CouchDB` in production configuration using Kubernetes.

## Start gateway service and use REST APIs to test chaincode
Refer [gateway](./service/README.md) for more details on how to build and start a REST API service for applications to interact with one or more Fabric networks. The following commands will start a gateway service that exposes a Swagger-UI at `http://localhost:30081/swagger`.
```
cd ../service
./gateway.sh start
```
## Operations for managing the Fabric network
The above bootstrap network is for a single operating company to start a Fabric network with its own orderer and peer nodes of pre-configured size.  A network in production will need to scale up and let more organizations to join and co-operate.  Organizations may create their own Kubernetes networks using the same or different cloud service providers. We provide scripts to support such network activities.

The currently supported operations include
* Create and join new channel;
* Install and instantiate new chaincode;
* Add new peer nodes of the same bootstrap org;
* Add new orderer nodes of the same bootstrap org;
* Add new peer org to the same Kubernetes cluster;

Refer [operations](./operations.md) for description of these activities. More operations (as described in `TODO` bellow) will be supported in the future.

## Non-Mac users
If you are not using a Mac, you can run these scripts using `docker-compose`, `Amazon EKS`, `Azure AKS`, or `Google GKE`. Simply add a corresponding `env_type` in all the commands, e.g.,
* `./ca-server.sh start -t docker` to use `docker-composer`, or
* `./ca-server.sh start -t aws` to use AWS as described in the folder [aws](./aws), or
* `./ca-server.sh start -t az` to use Azure as described in the folder [az](./az), or
* `./ca-server.sh start -t gcp` to use Google Cloud as described in the folder [gcp](./gcp), or
* try to verify if the scripts would work on [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

When `docker-compose` is used, start and test the Fabric network using the following commands:
```
cd ./network
./network.sh start -t docker
./network.sh test -t docker
./network.sh shutdown -t docker
```
## TODO
Stay tuned for more updates on the following items:
* Add new orderer org to the same bootstrap Kubernetes cluster for etcd raft consensus;
* Add new orderer org to a new Kubernetes cluster;
* Add new peer org to a new Kubernetes cluster;
* Test multiple org multiple Kubernetes clusters across multiple cloud providers.