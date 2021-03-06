⚠️⚠️⚠️ **This is currently a development feature and has not been released** ⚠️⚠️⚠️

# Quickstart Create a Game Server Fleet

This guide covers how you can quickly get started using Agones to create a Fleet 
of warm GameServers ready for you to allocate out of and play on!

## Prerequisites

The following prerequisites are required to create a GameServer:

1. A Kubernetes cluster with the UDP port range 7000-8000 open on each node.
2. Agones controller installed in the targeted cluster
3. kubectl properly configured
4. Netcat which is already installed on most Linux/macOS distributions, for windows you can use [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

>NOTE: Agones required Kubernetes versions 1.9+ to run. See the [cluster requirements](../README.md#requirements) for more details.

If you don't have a Kubernetes cluster you can follow [these instructions](installing_agones.md) to create a cluster on Google Kubernetes Engine (GKE) or Minikube, and install Agones.

For the purpose of this guide we're going to use the [simple-udp](../examples/simple-udp/) example as the GameServer container. This example is very simple UDP server written in Go. Don't hesitate to look at the code of this example for more information.

While not required, you may wish to go through the [Create a Game Server](create_gameserver.md) quickstart before this one.

## Objectives

- Create a Fleet in Kubernetes using Agones custom resource.
- Scale the Fleet up from it's initial configuration
- Request a GameServer allocation from the Fleet to play on
- Connect to the allocated GameServer.


### 1. Create a Fleet

Let's create a Fleet using the following command :

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/agones/master/examples/simple-udp/server/fleet.yaml
```

You should see a successful ouput similar to this :

```
fleet "simple-udp" created
```

This has created a Fleet record inside Kubernetes, which in turn creates two warm [GameServers](gameserver_spec.md) to
be available to being allocated for usage for a game session.
 
```
kubectl get fleet
```
It should look something like this:

```
NAME         AGE
simple-udp   5m
```

You can also see the GameServers that have been created by the Fleet by running `kubectl get gameservers`, 
the GameServer will be prefixed by `simple-udp`.

```
NAME                     AGE
simple-udp-xvp4n-jvhbm   36s
simple-udp-xvp4n-x6z5m   36s
```

For the full details of the YAML file head to the [Fleet Specification Guide](./fleet_spec.md#fleet-specification)

### 2. Fetch the Fleet status

Let's wait for the two `GameServers` to become ready.

```
watch kubectl describe fleet simple-udp
```

```
Name:         simple-udp
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"stable.agones.dev/v1alpha1","kind":"Fleet","metadata":{"annotations":{},"name":"simple-udp","namespace":"default"},"spec":{"replicas":2,...
API Version:  stable.agones.dev/v1alpha1
Kind:         Fleet
Metadata:
  Cluster Name:
  Creation Timestamp:  2018-04-27T20:06:23Z
  Generation:          0
  Resource Version:    72035
  Self Link:           /apis/stable.agones.dev/v1alpha1/namespaces/default/fleets/simple-udp
  UID:                 75832daa-4a56-11e8-925f-080027bb1955
Spec:
  Replicas:  2
  Template:
    Metadata:
      Creation Timestamp:  <nil>
    Spec:
      Container Port:  7654
      Health:
      Template:
        Metadata:
          Creation Timestamp:  <nil>
        Spec:
          Containers:
            Image:  gcr.io/agones-images/udp-server:0.1
            Name:   simple-udp
            Resources:
Status:
  Ready Replicas:  2
  Replicas:        2
Events:
  Type    Reason                 Age   From              Message
  ----    ------                 ----  ----              -------
  Normal  CreatingGameServerSet  8s    fleet-controller  Created GameServerSet simple-udp-jp8xw
```

If you look towards the bottom, you can see there is a section of `Status > Ready Replicas` which will tell you
how many `GameServers` are currently in a Ready state. After a short period, there should be 2 `Ready Replicas`.

### 3. Scale up the Fleet

Let's scale up the `Fleet` from 2 `replicates` to 5.

Run `kubectl edit fleet simple-udp`, which will open an editor for you to edit the Fleet configuration.

Scroll down to the `spec > replicas` section, and change the values of `replicas: 2` to `replicas: 5`.

Save the file and exit - this will apply the changes.

If we now run `kubectl get gameservers` we should see 5 `GameServers` prefixed by `simple-udp`.

```
NAME                     AGE
simple-udp-xvp4n-jvhbm   11m
simple-udp-xvp4n-x6z5m   11m
simple-udp-xvp4n-z8znu   36s
simple-udp-xvp4n-a6z0e   36s
simple-udp-xvp4n-i6bnm   36s
```

### 4. Allocate a Game Server from the Fleet

Since we have a fleet of warm gameservers, we need a way to request one of them for usage!

We can do this through a `FleetAllocation`, which will both return to us a `GameServer` (assuming one is available)
and also move it to the `Allocated` state.

In production, you would likely do this through a [Kubernetes API call](./access_api.md), but we can also
do this through `kubectl` as well, and ask it to return the response in yaml so that we can see what has happened.

```
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/agones/master/examples/simple-udp/server/fleetallocation.yaml -o yaml
```

For the full details of the YAML file head to the [Fleet Specification Guide](./fleet_spec.md#fleet-allocation-specification)

You should get back a response that looks like the following:

```
apiVersion: stable.agones.dev/v1alpha1
kind: FleetAllocation
metadata:
  clusterName: ""
  creationTimestamp: 2018-04-30T04:02:00Z
  generateName: simple-udp-
  name: simple-udp-5g82v
  namespace: default
  ownerReferences:
  - apiVersion: stable.agones.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: GameServer
    name: simple-udp-gwrsj-gzd9k
    uid: fd2e889e-4c2a-11e8-a2db-0800273eb123
  resourceVersion: "761"
  selfLink: /apis/stable.agones.dev/v1alpha1/namespaces/default/fleetallocations/simple-udp-5g82v
  uid: 3bb689f3-4c2b-11e8-a2db-0800273eb123
spec:
  fleetName: simple-udp
status:
  GameServer:
    metadata:
      creationTimestamp: 2018-04-30T04:00:15Z
      finalizers:
      - stable.agones.dev
      generateName: simple-udp-gwrsj-
      labels:
        stable.agones.dev/gameserverset: simple-udp-gwrsj
      name: simple-udp-gwrsj-gzd9k
      namespace: default
      ownerReferences:
      - apiVersion: stable.agones.dev/v1alpha1
        blockOwnerDeletion: true
        controller: true
        kind: GameServerSet
        name: simple-udp-gwrsj
        uid: c4181444-4c2a-11e8-a2db-0800273eb123
      resourceVersion: "759"
      selfLink: /apis/stable.agones.dev/v1alpha1/namespaces/default/gameservers/simple-udp-gwrsj-gzd9k
      uid: fd2e889e-4c2a-11e8-a2db-0800273eb123
    spec:
      PortPolicy: dynamic
      container: simple-udp
      containerPort: 7654
      health:
        failureThreshold: 3
        initialDelaySeconds: 5
        periodSeconds: 5
      hostPort: 7635
      protocol: UDP
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - image: gcr.io/agones-images/udp-server:0.1
            name: simple-udp
            resources: {}
    status:
      address: 192.168.99.100
      nodeName: agones
      port: 7635
      state: Allocated
```

If you see the `status` section, you should see that there is a `GameServer`, and if you look at its
`status > state` value, you can also see that it has been moved to `Allocated`. This means you have been successfully
allocated a `GameServer` out of the fleet, and you can now connect your players to it!

A handy trick for checking to see how many `GameServers` you have `Allocated` vs `Ready`, run the following:

```
kubectl describe gs | grep State
```

This will get you a list of all the current `Status > State` of all the `GameSevers`.

```
  State:      Allocated
  State:      Ready
  State:      Ready
  State:      Ready
  State:      Ready
```

### 5. Deploy a new version of the GameServer on the Fleet
_Coming Soon._

### 6. Scale down the Fleet

Not only can we scale our fleet up, but we can scale it down as well.

The nice thing about Agones, is that it is smart enough to know when `GameServers` have been moved to `Allocated`
and will automatically leave them running on scale down -- as we assume that players are playing on this game server,
and we shouldn't disconnect them!

Let's scale down our Fleet to 0 (yep! you can do that!), and watch what happens.
Run `kubectl edit fleet simple-udp` and replace `replicas: 5` with `replicas 0`, save the file and exit your editor.

It may take a moment for all the `GameServers` to shut down, so let's watch them all and see what happens:
```
watch kubectl describe gs | grep State
```

Eventually, one by one they will be removed from the list, and you should simply see:

```
  State:      Allocated
```

That lone `Allocated` `GameServer` is left all alone, but still running!

If you would like, try editing the `Fleet` configuration `replicas` field and watch the list of `GameServers`
grow and shrink.

### 7. Connect to the GameServer

Since we've only got one allocation, we'll just grab the details of the IP and port of the
only allocated `GameServer`: 

```
kubectl get $(kubectl get fleetallocation -o name) -o jsonpath='{.status.GameServer.status.address}':'{.status.GameServer.status.port}'
```

This should output your Game Server IP address and port. (eg `10.130.65.208:7936`)

You can now communicate with the `GameServer`:

```
nc -u {IP} {PORT}
Hello World !
ACK: Hello World !
EXIT
```

You can finally type `EXIT` which tells the SDK to run the [Shutdown command](../sdks/README.md#shutdown), and therefore shuts down the `GameServer`.  

If you run `kubectl describe gs | grep State` again - either the GameServer will be replaced with a new, `Ready` `GameServer`
, or it will be in `Shutdown` state, on the way to being deleted.

Since we are running a `Fleet`, Agones will always do it's best to ensure there are always the configured number
of `GameServers` in the pool in either a `Ready` or `Allocated` state. 

## Next Steps

If you want to use your own GameServer container make sure you have properly integrated the [Agones SDK](../sdks/).
