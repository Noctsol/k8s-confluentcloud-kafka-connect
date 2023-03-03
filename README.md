# k8s-confluentcloud-kafka-connect

## What is this for?

This guide will show you an **actual functional** example of how to set up **self-managed connectors** in kubernetes to a Kafka instance hosted by confluent cloud. This will be an example of something that is actually based off something that is actually running in production at an actual company. The two connectors I will show case are:

- The Mongo DB Atlas Cloud Source Connector
- The Elastic Connector

## Why would you want to self-managed Connectors?

Quick aside. Why would you want to set this up? Well, if you set up a Kafka cluster on confluent cloud and you did it for a company. There's good chance you did a private set up using private-link. The reason you did this is probably because Kafka cluster deployments require /16 address spaces for bring-your-own-vnet. This probably poses some networking issues you would rather not deal with. Now, you face another problem with a private-link cluster: fully managed connectors can only connect over public internet. Security is probably not gonna let you get away with that. So now, you need a worker sitting in between your private Kafka Cluster and your private data stores(SQL, mongo, blob/s3, w/e).

Or you just are one of those guys that hates using any managed service ever. Or just want to be cheap and have power of doing it on k8s. I don't know.

## Why did I make this?

I made this because I was frustrated with this whole process. A lot of the resources are not well documented in my opinion despite there being tons of documentation. The most useful thing I did in this whole process was just end up searching through all of GitHub for keywords. It was quite the lesson in writing unhelpful documentation. What most engineers really need at the beginning mostly is just a bunch of **<u>working</u>** examples shoved in their face.

I spent a week bashing my head against the wall when this could have been set up in about an hour. Maybe this will be useful for you.

## What do you need?

- A k8s cluster (duh). I use minikube.
- An Elastic Cluster. I connected to both a cloud and self-host intance.
- A Mongo DB Atlas instance. The cloud hosted one.
- helm
- (optional) argocd. It's really neat and pretty easy to install for something on k8s.

## Overview of Architecture

There's really 3 components you need to know about. I think I'm mostly accurate in my explanation.

1. [Confluent Operator](https://docs.confluent.io/5.5.1/installation/operator/index.html) - You need to install this thing via helm. All items belonging to the confluent platform will be managed by this little sucker. That being said, don't worry about this too much. When you install it, it won't do anything besides deploy the operator and install the CRD's(custom resource definitions). This includes

   - Zookeeper

   - Kafka cluster

   - Kakfa-Connect

   - Schema Registry

   - Replicator

   - KSQL

   - Control Center

2. **Connect Cluster** - This will come from the Connect CRD. This will generate a Kafka-Connect cluster. All of your connectors/plugins you run will be deployed onto here. You can control what gets deployed and the resources and scaling out here. This is also where you make the connection to your confluent cloud Kafka Cluster or 
3. **Connector Definition** - This will come from the Connector CRD. These will define what the heck your connectors will be doing. For example:
   1. What topic is writing to or reading from?
   2. The poll time in between checking from sources
   3. Batch size of data moved
   4. etc.

## Steps

1. Create a namespace on your confluent cluster named **confluent** and switch your current context to it

```bash
kubectl create namespace confluent
kubectl config set-context --current --namespace confluent
```

2. Install via helm. Here is what you need to run:

```bash
helm repo add confluentinc https://packages.confluent.io/helm
helm repo update
helm upgrade --install confluent-operator \
  confluentinc/confluent-for-kubernetes \
  --namespace confluent \
  --set installCRDs=true
```

2. (Optional) Instead you could just install via argocd. I just left the values as is.Here's the yaml:

```yaml
project: default
source:
  repoURL: 'https://packages.confluent.io/helm'
  targetRevision: 0.581.34
  chart: confluent-for-kubernetes
destination:
  server: 'https://kubernetes.default.svc'
  namespace: confluent
syncPolicy:
  automated: {}
```

3. Wait for the install to finish. Once it's done, if you run **kubectl get pods** you should see the confluent operator pod running.

4. Here comes the annoying part, you need to replace all the secrets here and then encode them in base 64:

   1. (Optional)For the connect-cluster-secrets(if you decide you want auth to your connect cluster), it needs to be in this format:

      ```bash
      <username>:<password>,<role1>,<role2>....
      
      # Example (Unfortunately, I have no idea what roles exist)
      myconnectadmin: reallygoodpassword,somerole
      ```

   2. For anything authenticating to confluent cloud in the connect yml, this is the format:

      ```bash
      username={username}
      password={password}
      
      # Example 
      username=47MYAPIID8
      password=76MYSECRET9728345BG335J35HJ
      ```

   3. For any secrets you mount and want to reference in your connectyors, they need to be the below format:

      ```bash
      {key1}={value1}
      {key2}={value2}
      {key3}={value3}
      
      # Example:
      
      user=myusername
      apikey=T87RGBIUB8IH8
      password=securepasswordtotally
      somesecret=234982th92h
      
      # When you refer to them in connector configs, it looks like this:
        configs:
      
      
          ### Mongo props
          connection.uri: "mongodb+srv://mongoadmin:${file:/mnt/secrets/mongo-secrets/credentials.txt:password}"
      ```

      ```yaml
      # let's say we named the k8s secrets test-secrets and name the key stuff.txt
      # This is what it would look like when referencing your value
       configs:
          ### Mongo props
          connection.uri: "myconnstring${file:/mnt/secrets/test-secrets/stuff.txt:password}"
          apikey: "${file:/mnt/secrets/test-secrets/stuff.txt:apikey}"
          supersecret: "${file:/mnt/secrets/test-secrets/stuff.txt:user}:${file:/mnt/secrets/test-secrets/stuff.txt:somesecret}"
      ```

5.  Modify the configs to point to correct URLS and have your preferential settings

   1. Yes, you need to replace things with the actual URL to things

6. Run the command

   ```bash
   kubectl apply -k ./kafka-connect/overlay/
   ```

7. Trouble shoot anything by using any of the relevant commands

   ```bash
   kubectl describe connect connect
   kubectl describe source-connector-mongo
   kubectl describe sink-connector-elastic
   kubectl logs -f connect-0 -c config-init-container
   kubectl logs -f connect-0
   ```

8. There you go

Notes:

- You can use the base version of the **confluentinc/cp-server-connect** and build it with plugins you need. However, I don't see any reason to do that when you can just do it on demand. It remains way more flexible like this.
- Regarding secrets, on the actual deployment, this is using [External Secrets operator](https://external-secrets.io/v0.7.2/) and pulling data from various vaults. For this example, I just used basic secrets
- I didn't change any of the secrets used on my test resources. If you want to try to login to my free public deployments and break them. Go for it.
- If you decide you need to enable auth for your connect cluster, you need to generate a secret(s) and mount them to the connect cluster. There is a commented section in the connectexample. The format is "{username}: {password},{role1},{role2}". Unfortunately, I have no clue what the roles are. I'm not sure confluent knows either.

