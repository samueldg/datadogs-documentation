---
title: Usage basic de l'Agent v5 avec l'Agent Kubernetes
kind: faq
---

{{< img src="integrations/kubernetes/k8sdashboard.png" alt="Kubernetes Dashboard" responsive="true" popup="true">}}

## Aperçu

Obtenir les métriques de kubernetes en temps réel pour:

* Visualiser et monitorer les états de kubernetes
* Être informé des failovers et des événements kubernetes.

For Kubernetes, it’s recommended to run the [Agent in a DaemonSet][1]. We have created a [Docker image][2] with both the Docker and the Kubernetes integrations enabled.

You can also just [run the Datadog Agent on your host][3] and configure it to gather your Kubernetes metrics.

## Implémenter Kubernetes
### Installation
#### Installation containeur

Thanks to Kubernetes, you can take advantage of DaemonSets to automatically deploy the Datadog Agent on all your nodes (or on specific nodes by using nodeSelectors).

*If DaemonSets are not an option for your Kubernetes cluster, [install the Datadog Agent][4] as a sidecar container on each Kubernetes node.*

If your Kubernetes has RBAC enabled, see the [documentation on how to configure RBAC permissions with your Datadog-Kubernetes integration][5].

* Create the following `dd-agent.yaml` manifest:

```yaml

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: dd-agent
spec:
  template:
    metadata:
      labels:
        app: dd-agent
      name: dd-agent
    spec:
      containers:
      - image: datadog/docker-dd-agent:latest
        imagePullPolicy: Always
        name: dd-agent
        ports:
          - containerPort: 8125
            name: dogstatsdport
            protocol: UDP
        env:
          - name: API_KEY
            value: "YOUR_API_KEY"
          - name: KUBERNETES
            value: "yes"
        volumeMounts:
          - name: dockersocket
            mountPath: /var/run/docker.sock
          - name: procdir
            mountPath: /host/proc
            readOnly: true
          - name: cgroups
            mountPath: /host/sys/fs/cgroup
            readOnly: true
      volumes:
        - hostPath:
            path: /var/run/docker.sock
          name: dockersocket
        - hostPath:
            path: /proc
          name: procdir
        - hostPath:
            path: /sys/fs/cgroup
          name: cgroups
```

Replace `YOUR_API_KEY` with [your api key][6] or use [Kubernetes secrets][7] to set your API key [as an environment variable][8].

* Deploy the DaemonSet with the command:
  ```
  kubectl create -f dd-agent.yaml
  ```

**Note**:  This manifest enables autodiscovery's auto configuration feature. To disable it, remove the `SD_BACKEND` environment variable definition. To learn how to configure autodiscovery, please refer to [its documentation][9].

#### Host Installation

Install the `dd-check-kubernetes` package manually or with your favorite configuration manager.

### Configuration

Edit the `kubernetes.yaml` file to point to your server and port, set the masters to monitor:

```yaml

instances:
    host: localhost
    port: 4194
    method: http
```

See the [example kubernetes.yaml][10] for all available configuration options.

### Validation
#### Container Running

To verify the Datadog Agent is running in your environment as a daemonset, execute:

    kubectl get daemonset

If the Agent is deployed you will see output similar to the text below, where desired and current are equal to the number of nodes running in your cluster.

    NAME       DESIRED   CURRENT   NODE-SELECTOR   AGE
    dd-agent   3         3         <none>          11h

#### Agent check running

[Lancez la commande `status`de l'Agent][11]  et cherchez `kubernetes` dans la section Checks:

    Checks
    ======

        kubernetes
        -----------
          - instance #0 [OK]
          - Collected 39 metrics, 0 events & 7 service checks

## Implémenter Kubernetes State
### Installation
#### Installation containeur

If you are running Kubernetes >= 1.2.0, you can use the [kube-state-metrics][12] project to provide additional metrics (identified by the `kubernetes_state` prefix in the metrics list below) to Datadog.

To run kube-state-metrics, create a `kube-state-metrics.yaml` file using the following manifest to deploy the kube-state-metrics service:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-state-metrics
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      containers:
      - name: kube-state-metrics
        image: gcr.io/google_containers/kube-state-metrics:v1.2.0
        ports:
        - name: metrics
          containerPort: 8080
        resources:
          requests:
            memory: 30Mi
            cpu: 100m
          limits:
            memory: 50Mi
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
spec:
  ports:
  - name: metrics
    port: 8080
    targetPort: metrics
    protocol: TCP
  selector:
    app: kube-state-metrics
```

Then deploy it by running:
```
kubectl create -f kube-state-metrics.yaml
```

The manifest above uses Google’s publicly available `kube-state-metrics` container, which is also available on [Quay][13]. If you want to build it manually, refer [to the official project documentation][12].

If you configure your Kubernetes State Metrics service to run on a different URL or port, you can configure the Datadog Agent by setting the `kube_state_url` parameter in `conf.d/kubernetes_state.yaml`, then restarting the Agent.
For more information, see the [kubernetes_state.yaml.example file][14]. If you have enabled [Autodiscovery][9], the kube state URL will be configured and managed automatically.

#### Host Installation

Install the `dd-check-kubernetes_state` package manually or with your favorite configuration manager (On CentOS/AWS, [Find your rpm package here](https://yum.datadoghq.com/rpm/x86_64/), and information on installation on [this page][15].
Then edit the `kubernetes_state.yaml` file to point to your server and port and set the masters to monitor. See the [example kubernetes_state.yaml][14] for all available configuration options.

### Validation
#### Container validation
To verify the Datadog Agent is running in your environment as a daemonset, execute:

    kubectl get daemonset

If the Agent is deployed you will see similar output to the text below, where desired and current are equal to the number of running nodes in your cluster.

    NAME       DESIRED   CURRENT   NODE-SELECTOR   AGE
    dd-agent   3         3         <none>          11h

#### Validation de check de l'Agent

[Run the Agent's `info` subcommand][16] and look for `kubernetes_state` under the Checks section:

    Checks
    ======

        kubernetes_state
        -----------
          - instance #0 [OK]
          - Collected 39 metrics, 0 events & 7 service checks

## Implémenter Kubernetes DNS
### Installation

Install the `dd-check-kube_dns` package manually or with your favorite configuration manager.

### Configuration

Edit the `kube_dns.yaml` file to point to your server and port, set the masters to monitor. See the [sample kube_dns.yaml][17] for all available configuration options.

#### Using with service discovery

If you are using one `dd-agent` pod per kubernetes worker node, you could use the following annotations on your kube-dns pod to retrieve the data automatically.

```yaml

apiVersion: v1
kind: Pod
metadata:
  annotations:
    service-discovery.datadoghq.com/kubedns.check_names: '["kube_dns"]'
    service-discovery.datadoghq.com/kubedns.init_configs: '[{}]'
    service-discovery.datadoghq.com/kubedns.instances: '[[{"prometheus_endpoint":"http://%%host%%:10055/metrics", "tags":["dns-pod:%%host%%"]}]]'
```

**Remarks:**

 - Notice the "dns-pod" tag will keep track of the target DNS pod IP. The other tags will be related to the dd-agent that is polling the informations using the service discovery.
 - The service discovery annotations need to be applied to the pod. In case of a deployment, add the annotations to the metadata of the template's spec.

### Validation

[Lancez la commande `status`de l'Agent][16]  et cherchez `kube_dns` dans la section Checks:

    Checks
    ======

        kube_dns
        -----------
          - instance #0 [OK]
          - Collected 39 metrics, 0 events & 7 service checks

[1]: https://github.com/DataDog/docker-dd-agent
[2]: https://hub.docker.com/r/datadog/docker-dd-agent/
[3]: /#host-setup
[4]: https://docs.datadoghq.com/integrations/docker_daemon/
[5]: /integrations/faq/using-rbac-permission-with-your-kubernetes-integration
[6]: https://app.datadoghq.com/account/settings#api
[7]: https://kubernetes.io/docs/concepts/configuration/secret/
[8]: https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables
[9]: https://docs.datadoghq.com/agent/autodiscovery
[10]: https://github.com/DataDog/integrations-core/blob/master/kubernetes/conf.yaml.example
[11]: /agent/faq/agent-status-and-information
[12]: https://github.com/kubernetes/kube-state-metrics
[13]: https://quay.io/coreos/kube-state-metrics
[14]: https://github.com/DataDog/integrations-core/blob/master/kubernetes_state/conf.yaml.example
[15]: /agent/faq/how-do-i-install-the-agent-on-a-server-with-limited-internet-connectivity
[16]: https://docs.datadoghq.com/agent/faq/agent-commands/#agent-status-and-information
[17]: https://github.com/DataDog/integrations-core/blob/master/kube_dns/conf.yaml.example