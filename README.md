# Governing Knative Eventing With Istio (A Demo)

As we take on cloud native architecture(s), it is imperative that our architectures consider cloud native characteristics that are distinct from our traditional architectural concerns. In this demo, we will take on those concerns: [Cloud Native Integration](https://github.com/rh-ei-stp/cloud-native-integration), and attempt to inject governance concepts. 

## What is *Cloud Native*? 

While laborious explanation is provided here: [What is Cloud Native?](https://github.com/mike-costello/cloud-native-integration/blob/master/definition-cloudnative.md) We can distill *cloud native* into the following set of characteristics: 
* **Elasticity**:	As this is a core characteristic of underlying cloud platforms, any workload that sits on top of a "cloud" must be capable of exploiting this capability as a core technology feature
* **On Demand Scaling**: As this is a core characteristic of underlying cloud platforms, this should be a core feature of any "cloud native" technology, architectural approach or deployment
* **Resilient**: High Availability should be a core concern of any technology or deployment. This implies both meainingful availability in calm and turbulent waters (i.e. outage of an availability zone, data center, etc.).
* **Manageable, Observable**: As the nature of a cloud platform provides some control plane (likely API based) to provision, manage and observe infrastructure deployments from the platform, these seems like core characteristics of any technology or deployment that is meant to be native to that platform
* **Location agnostic**: To be able to be effective in a cloud environment, it is critical that the underlying details of said cloud be abstracted away from technologies and deployments. This loose coupling of technology to platform implies an abstraction that allows the technology to be native to cloud as an architectural concept as opposed to a product. For instance, if something only works in a single compute deployment due to tight coupling to a proprietary platform it would only be useful to declare something native to that cloud
* **Event Driven**:	As our cloud platforms consist of an event driven set of API's through which we can provision, manage and observe our compute, effectively leveraging the platform implies event driven behaviour.
* **API Centric**: Beyond patterns of ways to serve API's (i.e. REST/SOAP, etc.), a core requirement of any deployment that hopes to be native to cloud deployments that have an API based event model, should itself depend on and define contractual communication

## What is *Governance*? 

Governance can be a rather amorphous word in software often associated with heavyweight and bureaucratic processes; however, as we take on cloud native architectures, we notice an exponentiation of our runtimes and deployments, communications protocols, payload types, and bespoke systems leading to an unwieldy explosion of deployment stuffs and things to manage. This presents a critical set of risks for organizations as they enter the cloud. 

"Governance" is an attempt to bring some order to our software chaos. Governance, in nearly all definitions of it, attempts to take on the following enterprise characteristics [What is Governance?](https://sloanreview.mit.edu/article/7-key-principles-to-govern-digital-initiatives/): 
* Centralize information about digital initiatives rather than the initiatives themselves
* Move from centralized to decentralized governance of digital initiatives as digital maturity grows.
* Decentralize ideation, but centralize idea evaluation and prioritization.
* Make sure that KPIs measure the real impact you want to achieve with each initiative.
* Avoid siloed solutions by ensuring data compatibility, technical consistency, and continuous integration of new initiatives with existing systems
* Implement a “fit-for-purpose” mapping system that recognizes value potential and degree of feasibility for each initiative.
* Evaluate different scenarios to proactively steward digital initiatives toward full-scale impact.

## Intro 

Now that we have introduced some concepts we're after, let's take a (*hexagonal*) look at what this means for our enterprise: 

![The View From Space](/images/CloudNativeIntegration.00.00.001.png "Cloud Native Integration: From Space")

### Composition of our *Cloud Native* Deployments: 

As we can see our *cloud native* deployments begin to take on aspects of *hexagonal architecture* ([What is Hexagonal Architecture?](https://alistair.cockburn.us/hexagonal-architecture/)), we notice a few distinct components: 

* **Event Mesh**
The *event mesh* intends to handle peer to peer event communication in a fashion that allows for several *Cloud Native* characteristics such as high availability, reliability, and location agnostic behaviour between peers. The event mesh acts as a rendezvous point between eventing peers (event emitters and event receivers), and provides event emitters a graph of event receivers that may span clusters or even clouds. 

* **Event Sink**
The *event sink* provides a port to the underlying event bus our integrations process events on. 

* **Event Bus**
The event bus provides a service bus so that event processors, that often need to integrate multiple different data sources to provide event aggregate level output, are able to communicate with each other in an asynchronous fashion. This decouples event emitters and receivers, as well as decoupling event aggregate behaviour that distills our series of events into meaningful business level data. As a result, integrations are bound to their domain, and are likely decomposed along their bounded context. It is along the *event bus* that features that constitute our enterprise perform the stream processing and integration work that satisfies enterprise features. Inevitably, these stream processors and integration components aggregate events into sources of truth to maintain consistency and state across the enterprise.    

### Governance of these components 

What is clearly missing from this architectural view of the world, is how we maintain governance of it.

More specifically, as we exercise our *ports and adapters* in a cloud native architecture, we need the following things: 

* A centralized way to ensure our ports and adapters are *allowed* to enter this architecture 
* A means to ensure our communications adhere to organization standards
* An centralized and api centric way to manage these things, with authentication and authorization    
* Centralized visibility into these components at runtime. *Remember* bespoke observability is better than nothing; however,  

## The Demo 

Now that we've defined some terms and we know what we're after let's lay down these components and this architectural view, and provide some means to ensure we meet some of our governance characteristics. 

### What you'll need 

In this demo we use the following technologies: 
* *An Openshift 4.8 Cluster* - while this demo uses Openshift, to use these components we only require a **K8s cluster version >= 1.19**
* *A User with Cluster Admin authority* - much of our setup will require cluster admin in our cluster
* *Maistra v2.0.8 aka Openshift ServiceMesh* - Maistra [Maistra](https://maistra.io/) is *Red Hat* distribution of Istio 2.x. While we do not need *Maistra*, we do need an Istio 2.x distribution. As we're using strictly Istio API's, this demo should fit neatly into most *servicemesh* distros that leverage Istio.  
* *Knative v0.23 aka Openshift Serverless* - Openshift Serverless is a distro of Knative [Knative](https://knative.dev/docs/). While the *Openshift Serverless* operators are handy, we can take in any distribution of Knative that is v0.23 or greater
* (Nice to Have) Strimzi* - in Knative Eventing, as we've discussed we create a pub-sub abstraction via the concept of *brokers, triggers, channels and subscriptions*. This can all simply be *in-memory*; however, for proper enterprise use we will want persistent messaging over our event bus  

* (Nice to Have) Camel-K* - ultimately, we need Knative services, and while we can easily do this without a thing like *Camel-K* it provides some ease of use, and abstracts our deployment needs; however, any deployment of *Knative Serving* will do for our demo runtime.    

### Installing our Istio Operator (aka *Openshift ServiceMesh* aka *Maistra*)  

Initially, we'll want to begin setting up operators in our cluster. There are a few distinct operators that we will need along with our Istio Operator: 
* **Jaeger Operator** - to provide granular observability in a centralized location as to our service invocations, we'll want to take traces of their interactions 
* **Elastic Search** - we will want essential monitoring capabilities and will leverage this via *Elastic Search*. *Elastic Search* provides a store for tracing and logging. 
* **Kiali** - Kiali provides a visual representation of our service mesh. Kiali also provides some administration capabilities  
* *(Optional)* **Prometheus** - *Prometheus* provides telemetry for deployments and runtimes. In our Openshift cluster, we have this enabled by default; however, it is worth leveraging *Prometheus* for its storage of runtime telemetry as well as alerting capabilities

#### Installing these operators 

While Openshift is not required for our demo, let's take a look at the operators we will want to provide: 

![Installed Operators](/images/installed-operators.png "Installed Operators")

**Note** *Community versions of these operators may be used via Operator Lifecycle Mananger [OLM](https://github.com/operator-framework/operator-lifecycle-manager)

#### Installing our *Service Mesh Control Plane*

Let's first create our *istio-system* namespace in Kubernetes via: 

``` 
kubectl create namespace istio-system 
```

Now that we have our operators installed and a namespace for our Istio system components, we will want to initiate our Istio based servicemesh usage by creating a *ServiceMeshControlPlane* object in Kubernetes: 

``` 
kubectl apply -f ./src/main/k8s/servicemesh/service-mesh-control-plane.yaml -n istio-system 
```

This will apply the following resource to our istio system namespace:

``` 
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
metadata:
  namespace: istio-system
  name: knative-governance-smcp
  labels:
    app: smcp
spec:
  security:
    controlPlane:
      mtls: true
  tracing:
    sampling: 10000
    type: Jaeger
  policy:
    type: Istiod
  addons:
    grafana:
      enabled: true
    jaeger:
      install:
        storage:
          type: Memory
    kiali:
      enabled: true
    prometheus:
      enabled: true
  version: v2.0
  telemetry:
    type: Istiod
```

**Note**: While the above *ServiceMeshControlPlane* object is a *Maistra* resource, the above may be accomplished via *istioctl* 

We'll notice a few things about our istio deployment from here: 
* We are using Istiod (i.e. Istio 2.x) 
* We have initiated *mTls* for control plane components. This is a critical aspect of our deployment as we will see later. 
* We setup a few *extras* that are not needed for functional usage (such as our use of *prometheus*, *grafana* etc.) but are critical components for enterprise use 

Upon installation, we should notice the following in our *istio-sytem* namespace: 

``` 
[mcostell@work-mcostell knative-eventing-examples]$ kubectl get pods -w -n istio-system 
NAME                                              READY     STATUS    RESTARTS   AGE
grafana-76c57d9895-jfjdn                          2/2       Running   0          1m
istio-egressgateway-fbd77fb49-59f9z               1/1       Running   0          1m
istio-ingressgateway-7bb8c7cbd6-8ddcm             1/1       Running   0          1m
istiod-knative-governance-smcp-767f9d56c5-fgn44   1/1       Running   0          2m
jaeger-dddc5465f-ncdrr                            2/2       Running   0          1m
kiali-548cb47b76-4tjvl                            1/1       Running   0          1m
prometheus-bfb5bd8b4-pqv82                        3/3       Running   0          2m

```
#### Establish our *ServiceMeshMembers* 

A critical part of our governance model is *"who"* gets to participate in the service mesh. This can be done a number of ways; however, we'll leverage our operator to initiate this and do some really cool stuffs. 

**Note** - the concept of a *ServiceMeshMember* is a *Maistra* concept; however, how the operator functionally handles this can be replicated via use of *istioctl* and [*Network Policy*](https://kubernetes.io/docs/concepts/services-networking/network-policies)

Let's first create our application namespace that we will be adding into the our mesh: 

```
kubectl create namespace amq-streams
kubectl create namespace servicemesh-con
```

And apply the *ServiceMeshMemberRoll* resource: 

```
kubectl apply -f ./src/main/k8s/servicemesh/service-mesh-member-roll-knative.yaml -n istio-system 

```
This applies the follow resource to our istio system namespace

```
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members: 
    - knative-serving
    - knative-eventing 
    - amq-streams
    - servicemesh-con
```

Upon issuing this resource, its worth noting the operator has completed a few tasks that we may need to replicate while using a different operator or deploying Istio via simply *istioctl*. 

Let's focus on one of the namespaces, we will have more work to do later; however, its worth taking a peak at what our operator is doing to constitute *members* of the mesh: 

* Initally, it creates our *ServiceMeshMemberRoll* resource:

```

apiVersion: v1
items:
- apiVersion: maistra.io/v1
  kind: ServiceMeshMemberRoll
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"maistra.io/v1","kind":"ServiceMeshMemberRoll","metadata":{"annotations":{},"name":"default","namespace":"istio-system"},"spec":{"members":["knative-serving","knative-eventing","amq-streams","servicemesh-con"]}}
    creationTimestamp: "2021-10-11T18:40:18Z"
    finalizers:
    - maistra.io/istio-operator
    generation: 1
    managedFields:
    - apiVersion: maistra.io/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:metadata:
          f:annotations:
            .: {}
            f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:spec:
          .: {}
          f:members: {}
      manager: kubectl
      operation: Update
      time: "2021-10-11T18:40:18Z"
    - apiVersion: maistra.io/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:metadata:
          f:finalizers:
            .: {}
            v:"maistra.io/istio-operator": {}
        f:status:
          .: {}
          f:annotations:
            .: {}
            f:configuredMemberCount: {}
          f:conditions: {}
          f:configuredMembers: {}
          f:meshGeneration: {}
          f:meshReconciledVersion: {}
          f:observedGeneration: {}
          f:pendingMembers: {}
      manager: istio-operator
      operation: Update
      time: "2021-10-11T18:40:19Z"
    name: default
    namespace: istio-system
    resourceVersion: "243424"
    uid: 55528609-77dc-4d42-8cfd-c1195ae92107
  spec:
    members:
    - knative-serving
    - knative-eventing
    - amq-streams
    - servicemesh-con
  status:
    annotations:
      configuredMemberCount: 4/4
    conditions:
    - lastTransitionTime: "2021-10-11T18:40:19Z"
      message: All namespaces have been configured successfully
      reason: Configured
      status: "True"
      type: Ready
    configuredMembers:
    - amq-streams
    - knative-eventing
    - knative-serving
    - servicemesh-con
    meshGeneration: 1
    meshReconciledVersion: 2.0.8-2.el8-1
    observedGeneration: 1
    pendingMembers: []
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""

```

This creates a namespace label that will pivotal for our dissemination of the mesh: 

```
kind: Namespace
metadata:
  annotations:
    openshift.io/sa.scc.mcs: s0:c26,c0
    openshift.io/sa.scc.supplemental-groups: 1000650000/10000
    openshift.io/sa.scc.uid-range: 1000650000/10000
  creationTimestamp: "2021-10-11T18:20:56Z"
  labels:
    kiali.io/member-of: istio-system
    kubernetes.io/metadata.name: knative-serving
    maistra.io/member-of: istio-system
```

This label is then used in a number of network policies that the operator has created: 

```

[mcostell@work-mcostell knative-eventing-examples]$ kubectl get networkpolicies -n knative-serving 
NAME                                         POD-SELECTOR                   AGE
istio-expose-route-knative-governance-smcp   maistra.io/expose-route=true   5m29s
istio-mesh-knative-governance-smcp           <none>                         5m30s

```

Looking further into these network policies, we'll notice that traffic can no longer flow from cluster resources that have not been labelled as belonging to the mesh: 

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.k8s.io/v1","kind":"NetworkPolicy","metadata":{"annotations":{"maistra.io/mesh-generation":"2.0.8-2.el8-1"},"labels":{"app":"istio","app.kubernetes.io/component":"mesh-config","app.kubernetes.io/instance":"istio-system","app.kubernetes.io/managed-by":"maistra-istio-operator","app.kubernetes.io/name":"mesh-config","app.kubernetes.io/part-of":"istio","app.kubernetes.io/version":"2.0.8-2.el8-1","maistra-version":"2.0.8","maistra.io/owner":"istio-system","release":"istio"},"name":"istio-mesh-knative-governance-smcp","namespace":"istio-system","ownerReferences":[{"apiVersion":"maistra.io/v2","blockOwnerDeletion":true,"controller":true,"kind":"ServiceMeshControlPlane","name":"knative-governance-smcp","uid":"c0b84427-f4d3-4273-8a81-fabf2055a50c"}]},"spec":{"ingress":[{"from":[{"namespaceSelector":{"matchLabels":{"maistra.io/member-of":"istio-system"}}}]}]}}
    maistra.io/mesh-generation: 2.0.8-2.el8-1
  creationTimestamp: "2021-10-11T18:40:18Z"
  generation: 1
  labels:
    app: istio
    app.kubernetes.io/component: mesh-config
    app.kubernetes.io/instance: istio-system
    app.kubernetes.io/managed-by: maistra-istio-operator
    app.kubernetes.io/name: mesh-config
    app.kubernetes.io/part-of: istio
    app.kubernetes.io/version: 2.0.8-2.el8-1
    maistra-version: 2.0.8
    maistra.io/member-of: istio-system
    maistra.io/owner: istio-system
    release: istio
  managedFields:
  - apiVersion: networking.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
          f:maistra.io/mesh-generation: {}
        f:labels:
          .: {}
          f:app: {}
          f:app.kubernetes.io/component: {}
          f:app.kubernetes.io/instance: {}
          f:app.kubernetes.io/managed-by: {}
          f:app.kubernetes.io/name: {}
          f:app.kubernetes.io/part-of: {}
          f:app.kubernetes.io/version: {}
          f:maistra-version: {}
          f:maistra.io/member-of: {}
          f:maistra.io/owner: {}
          f:release: {}
      f:spec:
        f:ingress: {}
        f:policyTypes: {}
    manager: istio-operator
    operation: Update
    time: "2021-10-11T18:40:18Z"
  name: istio-mesh-knative-governance-smcp
  namespace: knative-serving
  resourceVersion: "243396"
  uid: ce9c79cb-7151-4d57-8c2f-3dc9b0715575
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          maistra.io/member-of: istio-system
  podSelector: {}
  policyTypes:
  - Ingress

```

This network policy has been applied to all members of our mesh. As a result, despite our knative-serving and knative-eventing components not walling off the rest of the cluster from using them, our use of the ServiceMesh operator has done this for us.  

#### Installing *Knative-Serving* to Properly integrate into the mesh 

Knative Serving (which serves our *serverless* services) requires further configuration to be aware of the mesh. 

We'll apply our knative-serving resource, and take a peak at what it installs: 

```
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  ingress:
    istio:
      enabled: true 
  deployments: 
  - name: activator
    annotations:
      "sidecar.istio.io/inject": "true"
      "sidecar.istio.io/rewriteAppHTTPProbers": "true"
  - name: autoscaler
    annotations:
      "sidecar.istio.io/inject": "true"
      "sidecar.istio.io/rewriteAppHTTPProbers": "true"
```

And after applying the resource, we'll notice the following pods in our knative-serving namespace: 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl get pods -w -n knative-serving 
NAME                                     READY     STATUS    RESTARTS   AGE
activator-589fc5d55f-6spw2               2/2       Running   0          21m
activator-589fc5d55f-xv6lm               2/2       Running   0          21m
autoscaler-56b45ccff8-6zkfp              2/2       Running   0          21m
autoscaler-56b45ccff8-wnm77              2/2       Running   0          20m
autoscaler-hpa-57c88dc49-btbt6           1/1       Running   0          21m
autoscaler-hpa-57c88dc49-rmxtb           1/1       Running   0          21m
controller-6bc44cbc6d-227pg              1/1       Running   0          21m
controller-6bc44cbc6d-4kkh8              1/1       Running   0          21m
domain-mapping-79c68c64f6-276mb          1/1       Running   0          21m
domainmapping-webhook-7fcc4bddfc-j684l   1/1       Running   0          21m
istio-webhook-867cd474bd-wblvw           1/1       Running   0          21m
networking-istio-778494d8b4-6b8dt        1/1       Running   0          21m
networking-istio-778494d8b4-clcmn        1/1       Running   0          21m
webhook-647dc6b645-2cz8c                 1/1       Running   0          21m
webhook-647dc6b645-ffnbv                 1/1       Running   0          21m

```

It is worth noting, that we have only injected two of our *knative-serving* deployments with our envoy side car; however, we also have some new network policies: 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl get networkpolicies -n knative-serving 
NAME                                         POD-SELECTOR                   AGE
allow-from-openshift-monitoring-ns           <none>                         22m
domainmapping-webhook                        app=domainmapping-webhook      22m
istio-expose-route-knative-governance-smcp   maistra.io/expose-route=true   34m
istio-mesh-knative-governance-smcp           <none>                         34m
istio-webhook                                app=istio-webhook              22m
webhook                                      app=webhook                    22m

```

What is noticeable from the new network policies we see is that there are several aspects of *knative-serving* and (as we'll see) *knative-eventing* that require interaction from our Kube API. More specifically, as we apply *knative-serving* resources (which we will also see later), we will need the kubebrnetes api to be able to communication with: 
* Our Istio Ingress Gateway 
* Dynamic Admissions Webhooks that are required for taking in *knative-serving* resource(s) and determinging what to do with them 
  * Dynamic Admissions Webhooks do the following: 
   * They ensure that our resource is handled by the correct controller (they may be *duck typed*)
   * The ensure that the resource are valid   

At this point, we have two *knative-serving* resources that have been injected (our autoscaler and activator, both of which may be involved in a call into our servicemesh), and our *knative-serving* resources are wired up for *mTls* as well as for use of our *Istio Ingress Gateway*. This will be important as we will see later. 

#### Installing *Knative Eventing* (and Strimzi) to properly integrate into our ServiceMesh   

Now that we have joined *knative-serving* into our servicemesh and injected appropriate pods, let's setup the rest of our cloud native architecture to support the serverless pub-sub capabilities we are after. 

Initially, let's apply our *Strimzi* resource to launch a kubernetes native **Kafka** cluster. 

```
kubectl apply -f ./src/main/k8s/strimzi/small-event-cluster.yaml -n amq-streams
```

We'll notice this creates a **Kafka** cluster, instruments kubernetes services, *openshift* routes, and a bunch of other collateral stuff that is critical to administering Kafka. 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl get pods -w -n amq-streams 
NAME                                                   READY     STATUS    RESTARTS   AGE
small-event-cluster-entity-operator-7fb7bc6cb4-bxstr   3/3       Running   0          4m
small-event-cluster-kafka-0                            1/1       Running   0          4m
small-event-cluster-kafka-1                            1/1       Running   0          4m
small-event-cluster-kafka-2                            1/1       Running   0          4m
small-event-cluster-zookeeper-0                        1/1       Running   0          5m
small-event-cluster-zookeeper-1                        1/1       Running   0          5m
small-event-cluster-zookeeper-2                        1/1       Running   0          5m
small-event-cluster-zookeeper-3                        1/1       Running   0          5m
small-event-cluster-zookeeper-4                        1/1       Running   0          5m
```

Now that we have a *Kafka* broker cluster launched, let's wire up *knative-eventing* to do a few things: 
* Inject *knative-eventing* deployments to include an *istio* side car 
* Wire up application namespaces for default **broker** behaviour to ensure we have governance over how pub-sub activities are conducted in our application namespaces 

```
kubectl apply -f ./src/main/k8s/knative-eventing/knative-eventing.yaml -n knative-eventing
```

Upon succesfully applying this resource, we should notice the following pods in our knative-eventing namespace: 

```
mcostell@work-mcostell knative-eventing-examples]$ oc get pods -w -n knative-eventing 
NAME                                                       READY     STATUS      RESTARTS   AGE
eventing-controller-fcc6464bd-f9klf                        3/3       Running     1          1m
eventing-controller-fcc6464bd-wd4g5                        3/3       Running     1          1m
eventing-webhook-84c5bccd67-6hc8g                          3/3       Running     1          1m
eventing-webhook-84c5bccd67-tbrgc                          3/3       Running     1          1m
imc-controller-645b5ff5cd-nlt24                            3/3       Running     1          1m
imc-controller-645b5ff5cd-rk6vt                            3/3       Running     1          1m
imc-dispatcher-6ff4584bbc-5c447                            3/3       Running     1          1m
imc-dispatcher-6ff4584bbc-dm4h4                            3/3       Running     1          1m
mt-broker-controller-cbd567d6b-2jzp6                       2/2       Running     0          1m
mt-broker-controller-cbd567d6b-dpsdf                       2/2       Running     0          1m
mt-broker-filter-69b8ddc55-69zx5                           2/2       Running     0          1m
mt-broker-ingress-5c59dfbc88-bjtc4                         2/2       Running     0          1m
pingsource-cleanup-eventing-0.23.0-vt6cd                   0/1       Completed   0          1m
storage-version-migration-eventing-eventing-0.23.0-bl2cl   0/1       Completed   0          1m
sugar-controller-b97445649-4wk99                           2/2       Running     0          1m
sugar-controller-b97445649-fq6nf                           2/2       Running     0          1m
```

We'll notice that several of our deployments have been injected with sidecars, which will be neccesarry for *mTls* capabilities with our *knative-serving* services. 

Let's configure *knative-eventing* for jaeger with by applying the following config-map: 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/config-maps/config-tracing.yaml -n knative-eventing 
configmap/config-tracing configured
```
This will allow knative-eventing spans to be traced by Jaeger. Later on we'll see how this is a critical part of our governance capability. 

While we have properly configured *knative-eventing* as a servicemesh member, we need to expose our *knative-eventing* webhooks to our Kube APIServer to ensure we can take in *knative-eventing* **channel** resources to ensure we can take in *knative-eventing* resources as they are applied to our application namespaces. 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/networkpolicy-eventing-webhook.yaml -n knative-eventing 
networkpolicy.networking.k8s.io/eventing-webhook created
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/networkpolicy-inmemorychannel-webhook.yaml -n knative-eventing 
networkpolicy.networking.k8s.io/inmemorychannel-webhook created
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/networkpolicy-kafka-webhook.yaml -n knative-eventing 
networkpolicy.networking.k8s.io/kafka-webhook created
```

Once these network policies have been applied, we can now begin configuring *knative-eventing* for use with our **Kafka** cluster: 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/knative-kafka.yaml -n knative-eventing
knativekafka.operator.serverless.openshift.io/knative-kafka created
```

We'll notice this deploys two further pods that will act as a means to handle:
* **KafkaChannel** channels into our *knative-eventing* deployments via our admissions webhooks 
* Dispatching to our channel payloads to our Kafka Cluster and *Knative* services

```
mcostell@work-mcostell knative-eventing-examples]$ oc get pods -w -n knative-eventing 
NAME                                   READY     STATUS    RESTARTS   AGE
eventing-controller-fcc6464bd-f9klf    3/3       Running   1          15m
eventing-controller-fcc6464bd-wd4g5    3/3       Running   1          15m
eventing-webhook-84c5bccd67-6hc8g      3/3       Running   1          15m
eventing-webhook-84c5bccd67-tbrgc      3/3       Running   1          15m
imc-controller-645b5ff5cd-nlt24        3/3       Running   1          15m
imc-controller-645b5ff5cd-rk6vt        3/3       Running   1          15m
imc-dispatcher-6ff4584bbc-5c447        3/3       Running   1          15m
imc-dispatcher-6ff4584bbc-dm4h4        3/3       Running   1          15m
kafka-ch-controller-65c55b55c4-m99kd   2/2       Running   0          3m
kafka-webhook-7fdcf666d-gjcmx          2/2       Running   0          3m
mt-broker-controller-cbd567d6b-2jzp6   2/2       Running   0          15m
mt-broker-controller-cbd567d6b-dpsdf   2/2       Running   0          15m
mt-broker-filter-69b8ddc55-69zx5       2/2       Running   0          15m
mt-broker-ingress-5c59dfbc88-bjtc4     2/2       Running   0          15m
sugar-controller-b97445649-4wk99       2/2       Running   0          15m
sugar-controller-b97445649-fq6nf       2/2       Running   0          15m

```

While these pods are wired up, we still have some further configuration we need to add: 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/config-br-defaults.yaml -n knative-eventing 
configmap/config-br-defaults configured
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/config-kafka-channel.yaml -n knative-eventing 
configmap/config-kafka-channel created
[mcostell@work-mcostell knative-eventing-examples]$ kubectl apply -f ./src/main/k8s/knative-eventing/config-imc-channel.yaml -n knative-eventing 
configmap/config-imc-channel created
```

Applying the *config-br-defaults* takes advantage of a critical governance capability that *knative-eventing* offers us: 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-br-defaults
  namespace: knative-eventing
data:
  default-br-config: |
    clusterDefault:
      apiVersion: v1
      brokerClass: MTChannelBasedBroker
      kind: ConfigMap
      name: config-imc-channel
      namespace: knative-eventing
    namespaceDefaults:
      newco-tenant: 
        apiVersion: v1
        brokerClass: InMemory
        kind: ConfigMap
        name: config-imc-channel
        namespace: newco-tenant
      servicemesh-con:
        apiVersion: v1
        brokerClass: KafkaBroker
        kind: ConfigMap
        name: config-smcon-tenant-channel
        namespace: servicemesh-con
      important-tenant: 
        brokerClass: KafkaBroker
        kind: ConfigMap
        name: important-tenant-channel
        namespace: important-tenant
```

What is critical about this wiring up of our configmap is that we inject different eventing configurations by namespace. As we will label our application namespace later to be handled by *knative-eventing* this implies that we have a **central configuration for pub-sub** activities, and that this central configuration is managed by our servicemesh. 

We have one more activity we need to engage in before we are done configuring *knative-eventing*. As we laid down our *knative-kafka* resource, it is actually handled by a seperate operator [Knative Kafka Operator](https://github.com/openshift-knative/knative-kafka-operator). This operator has no *Istio* integration; however, we will need to inject these pods with proxies to handle *mTls* as well as the application of other istio policies. 

As a result, we'll patch these deployments to ensure injection of our proxy (as the pods are already members of the servicemesh): 

```
[mcostell@work-mcostell knative-eventing-examples]$ kubectl patch deployment kafka-webhook -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n knative-eventing 
deployment.apps/kafka-webhook patched
[mcostell@work-mcostell knative-eventing-examples]$ kubectl patch deployment kafka-ch-controller -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n knative-eventing 
deployment.apps/kafka-ch-controller patched
```

At this point we should notice all relevant pods in *knative-eventing* have been injected with istio proxy's. And we're ready to start configuring our application namespace to participate in the mesh while allowing our points of governance to abstract configuration for development teams.  

#### Configuring Application Namespaces for Pub-Sub 

Initally, while we *can* configure *knative-eventing* brokers and behaviour(s), we'd rather take that out of our developers hands so that they can get back to developing and not spend inordinate amounts of time with configuration. 

As a result, our *knative-eventing* operator watches namespaces for *eventing.knative.dev/injection* labels and will inject a *knative-eventing* Broker based on the configurations we have already provided if a namespace has the label. 

Because we have configured *knative-eventing* for multi-tenancy, each of our namespaces can inject its own knative-eventing configuration. This is critical as different application teams may have different state stores (in our case Kafka) and may need to wire up their backing channel implementation differently. 

To demonstrate this, let's configure our application namespace *servicemesh-con* with a configmap for its backing channel implementation: 

```
kubectl apply -f ./src/main/k8s/servicemeshcon-ns/config-smcon-tenant-channel.yaml -n servicemesh-con 
configmap/config-smcon-tenant-channel created

```

At this point, we can simply apply the knative-eventing label to the *servicemesh-con* namespace and notice that we have a *knative-eventing* broker deployed without our application team taking any further action (in fact, they will simply be able to interact with our pub-sub implementation without having to know the specifics of its internals).  

```
kubectl label namespace servicemesh-con eventing.knative.dev/injection=enabled
```
After applying this, we'll notice we have a "default" broker in our *servicemesh-con* namespace: 

```
kubectl get brokers -n servicemesh-con 
NAME      URL                                                                                AGE       READY     REASON
default   http://broker-ingress.knative-eventing.svc.cluster.local/servicemesh-con/default   2m9s      True      
```
And upon further inspection of this *knative-eventing* Broker we'll notice it is configured to use a *KafkaChannel*. 

```
kubectl get brokers default -n servicemesh-con -o yaml  
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  annotations:
    eventing.knative.dev/broker.class: MTChannelBasedBroker
    eventing.knative.dev/creator: system:serviceaccount:knative-eventing:eventing-controller
    eventing.knative.dev/lastModifier: system:serviceaccount:knative-eventing:eventing-controller
  creationTimestamp: "2021-10-11T21:33:41Z"
  generation: 1
  labels:
    eventing.knative.dev/injected: "true"
  managedFields:
  - apiVersion: eventing.knative.dev/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:eventing.knative.dev/broker.class: {}
        f:labels:
          .: {}
          f:eventing.knative.dev/injected: {}
      f:spec: {}
    manager: sugar-controller
    operation: Update
    time: "2021-10-11T21:33:41Z"
  - apiVersion: eventing.knative.dev/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        .: {}
        f:address:
          .: {}
          f:url: {}
        f:annotations:
          .: {}
          f:knative.dev/channelAPIVersion: {}
          f:knative.dev/channelAddress: {}
          f:knative.dev/channelKind: {}
          f:knative.dev/channelName: {}
        f:conditions: {}
        f:observedGeneration: {}
    manager: mtchannel-broker
    operation: Update
    time: "2021-10-11T21:34:35Z"
  name: default
  namespace: servicemesh-con
  resourceVersion: "517046"
  uid: 214cb83a-7616-492d-bfa7-ade517cfb950
spec:
  config:
    apiVersion: v1
    kind: ConfigMap
    name: config-smcon-tenant-channel
    namespace: servicemesh-con
status:
  address:
    url: http://broker-ingress.knative-eventing.svc.cluster.local/servicemesh-con/default
  annotations:
    knative.dev/channelAPIVersion: messaging.knative.dev/v1beta1
    knative.dev/channelAddress: http://default-kne-trigger-kn-channel.servicemesh-con.svc.cluster.local
    knative.dev/channelKind: KafkaChannel
    knative.dev/channelName: default-kne-trigger
  conditions:
  - lastTransitionTime: "2021-10-11T21:34:35Z"
    status: "True"
    type: Addressable
  - lastTransitionTime: "2021-10-11T21:34:35Z"
    status: "True"
    type: FilterReady
  - lastTransitionTime: "2021-10-11T21:34:35Z"
    status: "True"
    type: IngressReady
  - lastTransitionTime: "2021-10-11T21:34:35Z"
    status: "True"
    type: Ready
  - lastTransitionTime: "2021-10-11T21:34:35Z"
    status: "True"
    type: TriggerChannelReady
  observedGeneration: 1
```

At this point, we'll note we have no *knative-eventing* channels installed: 

```
kubectl get channels -n servicemesh-con 
No resources found.
```
This makes sense as we have not created any channels in the namespace; however, if we'll remember our *knative-eventing* primitives, we'll remember one of our core types is a *trigger* and upon interrogating the namespaces for *KafkaChannels* we'll notice this trigger has been established as a *KafkaChannel* and if we interrogate our *Kafka* brokers, we'll notice we now have a topic in kafka that represents this trigger: 

```
kubectl get kafkachannels -n servicemesh-con 
NAME                  READY     REASON    URL                                                                       AGE
default-kne-trigger   True                http://default-kne-trigger-kn-channel.servicemesh-con.svc.cluster.local   14m
```
And interrogating our *kafka* brokers, we find: 

```
kubectl exec small-event-cluster-kafka-0 -it bash
bash-4.4$ cd ./bin
bash-4.4$ ./kafka-topics.sh --list --bootstrap-server localhost:9092 
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
knative-messaging-kafka.servicemesh-con.default-kne-trigger
```

We are now completely configured for application deployment. So, let's add an application into our servicemesh. 

#### Deploying the Sample application into our *ServiceMesh*

**Note**: to complete this exercise requires the prior installation of the *Camel-K* operator as well as local installation of its cli client: [Camel-K CLI](https://camel.apache.org/camel-k/latest/cli/cli.html)

Intially, we will want to establish the channels our *knative-eventing* service uses. 

```
kubectl apply -f ./src/main/k8s/servicemeshcon-ns/channels/testing-dbevents-channel.yaml -n servicemesh-con 
kubectl apply -f ./src/main/k8s/servicemeshcon-ns/channels/testing-dbeventaggregate-channels.yaml -n servicemesh-con 
kubectl get channels -n servicemesh-con 
NAME                       URL                                                                            AGE       READY     REASON
testing-dbeventaggregate   http://testing-dbeventaggregate-kn-channel.servicemesh-con.svc.cluster.local   12s       True      
testing-dbevents           http://testing-dbevents-kn-channel.servicemesh-con.svc.cluster.local           34s       True      
```

We can now launch our integration that uses *knative-eventing*'s pub-sub abstractions. Let's take a quick look at our Camel DSL we'll ship as a *knative-service*: 

```
from("knative:channel/testing-dbevents")
			.id("from-channel-xform")
			.log("**Received message ${body}**")
			.setBody().simple("${body} processed after consumed by the event bus. Get on the bus!")
			.to("knative:channel/testing-dbeventaggregate")
			.log("**sent message to knative channel ${body} **");
```

We leverage the Camel *knative* component to interact with our knative channels. 

Now let's launch this and add it into the mesh with the *Camel-K* CLI: 

```
kamel run -n servicemesh-con --trait knative-service.enabled=true --trait knative.enabled=true --trait istio.enabled=true --trait istio.inject=true ./src/main/java/io/entropic/integration/examples/eventmesh/EventBusTransformationIntegration.java
```

Once *Camel-K* has done its thing we should notice it has deployed itself as a serverless service.

```
knative-eventing-examples]$ kubectl get service.serving.knative.dev -n servicemesh-con
NAME                                   URL                                                                                                         LATESTCREATED                                LATESTREADY                                  READY     REASON
event-bus-transformation-integration   http://event-bus-transformation-integration-servicemesh-con.apps.cluster-26ea.26ea.sandbox209.opentlc.com   event-bus-transformation-integration-00002   event-bus-transformation-integration-00002   Unknown   ReconcileIngressFailed
```
**Note** as we do not want to expose our servicebus to the outside world we have not bound this service to the *Istio's* Ingress Gateway. We restrict communication in the mesh to only the mesh. 

And that we have a Subscription: 

```
kubectl get subscriptions.messaging.knative.dev -n servicemesh-con -o yaml 
apiVersion: v1
items:
- apiVersion: messaging.knative.dev/v1
  kind: Subscription
  metadata:
    annotations:
      messaging.knative.dev/creator: system:serviceaccount:openshift-operators:camel-k-operator
      messaging.knative.dev/lastModifier: system:serviceaccount:openshift-operators:camel-k-operator
    creationTimestamp: "2021-10-11T22:36:06Z"
    finalizers:
    - subscriptions.messaging.knative.dev
    generation: 1
    labels:
      camel.apache.org/generation: "1"
      camel.apache.org/integration: event-bus-transformation-integration
    managedFields:
    - apiVersion: messaging.knative.dev/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:metadata:
          f:labels:
            f:camel.apache.org/generation: {}
            f:camel.apache.org/integration: {}
          f:ownerReferences:
            k:{"uid":"22bba735-dc70-4525-9688-d5f3a362cb00"}:
              .: {}
              f:apiVersion: {}
              f:blockOwnerDeletion: {}
              f:controller: {}
              f:kind: {}
              f:name: {}
              f:uid: {}
        f:spec:
          f:channel:
            f:apiVersion: {}
            f:kind: {}
            f:name: {}
          f:subscriber:
            f:ref:
              f:apiVersion: {}
              f:kind: {}
              f:name: {}
            f:uri: {}
      manager: camel-k-operator
      operation: Apply
      time: "2021-10-11T22:36:06Z"
    - apiVersion: messaging.knative.dev/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:metadata:
          f:finalizers:
            .: {}
            v:"subscriptions.messaging.knative.dev": {}
        f:status:
          .: {}
          f:conditions: {}
          f:observedGeneration: {}
          f:physicalSubscription:
            .: {}
            f:subscriberUri: {}
      manager: controller
      operation: Update
      time: "2021-10-11T22:36:37Z"
    name: testing-dbevents-event-bus-transformation-integration
    namespace: servicemesh-con
    ownerReferences:
    - apiVersion: camel.apache.org/v1
      blockOwnerDeletion: true
      controller: true
      kind: Integration
      name: event-bus-transformation-integration
      uid: 22bba735-dc70-4525-9688-d5f3a362cb00
    resourceVersion: "655283"
    uid: c5e7a70e-f004-42f0-a68f-2e9c836416ff
  spec:
    channel:
      apiVersion: messaging.knative.dev/v1
      kind: Channel
      name: testing-dbevents
    subscriber:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: event-bus-transformation-integration
      uri: /channels/testing-dbevents
  status:
    conditions:
    - lastTransitionTime: "2021-10-11T22:36:37Z"
      status: "True"
      type: AddedToChannel
    - lastTransitionTime: "2021-10-11T22:36:37Z"
      status: "True"
      type: ChannelReady
    - lastTransitionTime: "2021-10-11T22:36:37Z"
      status: "True"
      type: Ready
    - lastTransitionTime: "2021-10-11T22:36:37Z"
      status: "True"
      type: ReferencesResolved
    observedGeneration: 1
    physicalSubscription:
      subscriberUri: http://event-bus-transformation-integration.servicemesh-con.svc.cluster.local/channels/testing-dbevents
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

This is all wired up for us by *Camel-K*. 

*Camel-K* will report that our service is deployed; however, we have 0 replicas running (as we are not yet taking traffic). 

```
kubectl get integrations -n servicemesh-con 
NAME                                   PHASE     KIT                        REPLICAS
event-bus-transformation-integration   Running   kit-c5ibk3el6g8ks3j7st00   0
```

This particular service simply produces/consumes (pub-sub) so *Camel-K* simply indicates our "integtation" is "running"; however, because we are serverless, we do not have a pod running. 

So, let's deploy another *Camel-K* service that simply produces messages to the *Knative Eventing* channels that our above service communicates based on a timer: 

```
kamel run -n servicemesh-con --trait knative-service.enabled=true --trait knative.enabled=true --trait istio.enabled=true --trait istio.inject=true ./src/main/java/io/entropic/integration/examples/eventmesh/EventSinkIntegration.java
```
And as we'll notice, quickly we have 2 pods running both injected with istio-proxy's. 

```
kubectl get pods -w 
NAME                                                              READY     STATUS    RESTARTS   AGE
event-bus-transformation-integration-00002-deployment-6fbflph9k   3/3       Running   0          26s
event-sink-integration-00001-deployment-569854fbdb-zrfcd          3/3       Running   0          34s
```

And as we tail our logs, we'll see our message throughput: 

```
2021-10-11 23:11:37,266 INFO  [from-channel-xform] (vert.x-worker-thread-7) **Received message ServiceMeshCon welcome to Cloud Native Integration! processed by EventSinkIntegration**
2021-10-11 23:11:37,270 INFO  [from-channel-xform] (vert.x-eventloop-thread-1) **sent message to knative channel  **

```

Stay tuned for further discussion of Istio and Knative Eventing and how we can leverage Authentication Policies and Authorization Policies to provide further governance of our Cloud Native Event Bus!!!