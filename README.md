# AKS for Game Developers

This Workshop is designed for Game Developers that want to learn more about Kubernetes, Azure Kubernetes Service (AKS) and how to utilize this Technology for their game development needs.

# Table of Contents
- [AKS for Game Developers](#aks-for-game-developers)
- [Table of Contents](#table-of-contents)
- [Intended Audience and Expectations](#intended-audience-and-expectations)
- [Architecture](#architecture)
  - [Dedicated Servers](#dedicated-servers)
  - [Gameserver Orchestrator](#gameserver-orchestrator)
  - [Simple Server Browser](#simple-server-browser)
  - [Simple Matchmaking](#simple-matchmaking)
- [Workshop Tasks](#workshop-tasks)
  - [Task 1 - Setting up AKS](#task-1---setting-up-aks)
    - [Enabling Public IP addresses on Nodes](#enabling-public-ip-addresses-on-nodes)
    - [Create Resource Group](#create-resource-group)
    - [Create AKS Cluster](#create-aks-cluster)
  - [Task 2 - Deploy UDP test container](#task-2---deploy-udp-test-container)
    - [Define Game Server Pod and expose Service](#define-game-server-pod-and-expose-service)
    - [Connect to test Pod](#connect-to-test-pod)
  - [Task 3 - Set up CosmosDB](#task-3---set-up-cosmosdb)
  - [Task 4 - Implement player test service](#task-4---implement-player-test-service)
  - [Task 5 - Implement server browser](#task-5---implement-server-browser)
  - [Task 6 - Implement matchmaking](#task-6---implement-matchmaking)



# Intended Audience and Expectations
> TODO: define required skillset and prerequisites and standard quotas

This workshop requieres you to use Docker, Kubernetes (AKS) and write code. Failing will be part of this workshop and we want to guid you towards the right answer without writing it out for you. Please come prepared and have access to the following:

- Your own Laptop
- Linux Docker Environment 
- Ready to go coding setup. We will be using C# and .NET Core for references, if you want to a different language / environment please make sure that dependencies are available.
  - CosmosDB Packages
  - JSON libary
- Azure subscription

Due to time constrains in a workshop environment, we need to simplify some topics in the complexity, but we want to make sure to communicate best practices to enable you to use the outcome of this workshop for your next project. Service design as well as the database design is designed with scale ability and production use in mind. Even if the algorithms and strategies used are simplified to reduce complexity. 

# Architecture
In this workshop we will be building a solution that covers some of the common backend services needed in game development. Our focus will be on running Dedicated Game Servers and how to orchestrate them. 

Our solution will have the following components:
- Game server orchestrator and state management 
- Simple Matchmaking 
- Simple Server Browser

> TODO: Insert fancy architecture diagramm here
[Please update me](https://github.com/dgkanatsios/azuregameserversscalingkubernetes/raw/master/docs/diagram.png)


## Dedicated Servers
For this workshop we will be focusing on running only one dedicated game server on one virtual machiene. This will enable us to simplifie our gameserver orchestration logic so we do not need to take defragmentation into account. The current limit on Nodes within AKS is (TODO ASK FOR LIMIT CONFIRMATION). 

We as game developers have very specific requirements when it comes to Infrastructure and Services. For dedicated game servers we need the highest frequency available to run our complex Realtime simulations, low network latency and high bandwidth as well as the highest single node availability possible.  

We are aware that many game servers today are build on windows and might have dependencies to DX / D3D, in the context of this workshop we will be focusing on running docker on linux VMs. For one we think this is the better choice to reduce costs and provides the most stable environment for docker as of today.

## Gameserver Orchestrator
> TODO: might want to change this into one monolithic service in order to simplifie due to time constrains
The game server orcestrator is a service that contains the business logic nessesary to decide if new servers need to be provisoned or deallocated. The algorithm we will use here is pretty straight forward. We want to define the number of game servers that we expect to stand by in the "ready" state. If we have less availeable servers we will provision new one, if we have to many we will dealocate.

We will be utilizing CosmosDB as a state store fo the server list as this list is requiered by additional services, and therefore need to be scale able to satify scaling requierments for a public facing api.

## Simple Server Browser
> TODO: might want to change this into one monolithic service in order to simplifie due to time constrains
This service will be a public faceing api which will provide a list of all game servers and their current state. We can expect a high volume of read only requets against this endpoint. This will be utilizing CosmosDB to enable to desired scale abilty for this service.

## Simple Matchmaking
> TODO: might want to change this into one monolithic service in order to simplifie due to time constrains
In this workshop this API will be public, for production we can expect that this API Endpoint requieres authentication in order to work. This service is used for players to request a new match. A new match means that we need to find X players, group them together and assigne them to a server that we want them to connect to. In the context of this Workshop the alogrithm for finding players will be random.


# Workshop Tasks
TODO: Insert expectations on order of completion and over all completion

## Task 1 - Setting up AKS
This Task is intended to get familiar with the deployment and configuration of AKS. We expect you to complete the tasks listed here by utilizing the Azure CLI, because this will teach you the basics for automated deployments in the future. If you prefer another method, please feel free to use whatever you are most comfortable with.

> Note: If you have access to multiple azure subscriptions, please make sure that you have select the correct one for deployment.

```bash
az account set --subscription <subscription id>
```

### Enabling Public IP addresses on Nodes

---

As stated in our requirements, we are looking for a dedicated public IP address on each AKS node. This feature is currently in preview and we do not recommend production use quite yet. For this workshop we will be utilizing this feature because this will make our life much easier.

In order to use this preview feature, you need to register for the use of this extension with the following command.

```bash
az feature register --name NodePublicIPPreview --namespace Microsoft.ContainerService
```

This operation might take some time (we are talking ~10 minutes here) but it needs to be completed before you can provision your AKS cluster. To check the status of your registration you can use the following command.

```bash
az feature show --name NodePublicIPPreview --namespace Microsoft.ContainerService
```

[Link to official Azure Documentation for this feature](https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#assign-a-public-ip-per-node-in-a-node-pool)

### Create Resource Group

---

>Note: You don’t need to create a resource group if you’re using the lab environment. You can use the resource group created for you as part of the lab. To retrieve the resource group name in the managed lab environment, run az group list.

```bash
az group create --name <resource-group> --location <region>
```

### Create AKS Cluster

---

When we are creating the cluster, we don't want to enable autoscaling. The reason for this is that it will be very hard to control the scaling of the infrastructure while have stateful, Realtime in memory workloads in the form of dedicated game servers that require very careful handling and can’t be migrated at runtime to other nodes in case of a scaling operation.

>TODO: insert deployment ARM template here, as preview feature is not available for provisioning through portal

## Task 2 - Deploy UDP test container

---

For this task we will use a simple "ECHO UDP" server that will respond to messages send by us. We chose this option because we want to focus on the orchestration and mangement aspects and don't want you to write a gameserver from scratch.

>TODO: Decide on udp test image to be used here, depends on simplification efforts.

```bash
docker pull dgkanatsios/simplenodejsudp
```

or


```bash
docker pull annonator/aciudptest
```

### Define Game Server Pod and expose Service

---

For this part to work we want to make sure the container we are starting is running on the same network namespace as the host. As all our hosts do have a public IP associated to them we can be sure that our container will be available through the host IP adress.

```yaml
apiVersion: v1
kind: Pod
metadata:
  generateName: "game-"
  labels:
    name: myGameServer
spec:
  hostNetwork: true
  restartPolicy: never
---
apiVersion: v1
kind: Service
metadata:
  name: openarena-service
  labels:
    name: openarena-service
spec:
  ports:
    # the port that this service should serve on
    - port: 27960
      targetPort: port27960
      protocol: UDP 
      name: port27960
  # label keys and values that must match in order to receive traffic for this service
  selector:
    name: openarena-pod
  type: LoadBalancer
```

### Connect to test Pod
>TODO: based on server used adjust here

Looking up Public IP adress and start test client.

## Task 3 - Set up CosmosDB

---

CosmosDB will be our data store of choice because it can quite easily handle our scaling requierments for our services.

- Set up CosmosDB account
- Set up CosmosDB database


## Task 4 - Implement player test service

---

In order for us to match players we do need a very simple Endpoint that is able to fill a User database. We would define a user object as the following

```json
{
    "user_id": UNIQUE_GUID,
}
```


## Task 5 - Implement server browser

---

```json
{
    "server_id": UNIQUE_GUID,
    "ip": string,
    "port": INT,
    "region": string,
    "state": string
}
```

Every game server is either idle or running. 

## Task 6 - Implement matchmaking

---

