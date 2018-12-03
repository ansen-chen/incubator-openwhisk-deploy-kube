<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

# IBM IKS for OpenWhisk

## Overview

IBM provides both a "Lite" and a "Standard" Kubernetes offering in its
public cloud Kubernetes service (IKS). These differ in capabilities,
so they are described separately below.

## Initial setup

### Creating the Kubernetes Cluster

Follow IBM's instructions to provision your cluster.

### Configuring OpenWhisk

####  IBM Cloud Standard cluster

An IBM Cloud Standard cluster has full support for TLS
and can be configured with additional annotations to
fine tune ingress performance.

First, determine the values for <domain> and <ibmtlssecret> for
your cluster by running the command:
```
bx cs cluster-get <mycluster>
```
The CLI output will look something like
```
bx cs cluster-get <mycluster>
Retrieving cluster <mycluster>...
OK
Name:    <mycluster>
ID:    b9c6b00dc0aa487f97123440b4895f2d
Created:  2017-04-26T19:47:08+0000
State:    normal
Master URL:  https://169.57.40.165:1931
Ingress subdomain:  <domain>
Ingress secret:  <ibmtlssecret>
Workers:  3
```

Now define `mycluster.yaml` as below (substituting the real values for
`<domain>` and `<ibmtlssecret>`).
```yaml
whisk:
  ingress:
    apiHostName: <domain>
    apiHostPort: 443
    apiHostProto: https
    type: standard
    domain: <domain>
    tls:
      enabled: true
      secretenabled: true
      createsecret: false
      secretname: <ibmtlssecret>
    annotations:
      # A blocking request is held open by the controller for slightly more than 60 seconds
      # before it is responded to with HTTP status code 202 (accepted) and closed.
      # Set to 75s to be on the safe side.
      # See https://console.bluemix.net/docs/containers/cs_annotations.html#proxy-connect-timeout
      # See http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout
      ingress.bluemix.net/proxy-read-timeout: "75s"

      # Allow up to 50 MiB body size to support creation of large actions and large
      # parameter sizes.
      # See https://console.bluemix.net/docs/containers/cs_annotations.html#client-max-body-size
      # See http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size
      ingress.bluemix.net/client-max-body-size: "size=50m"

      # Add the request_id, generated by nginx, to the request against the controllers. This id will be used as tid there.
      # https://console.bluemix.net/docs/containers/cs_annotations.html#proxy-add-headers
      ingress.bluemix.net/proxy-add-headers: |
        serviceName=controller {
          'X-Request-ID' $request_id;
        }

k8s:
  persistence:
    hasDefaultStorageClass: false
    explicitStorageClass: default
```

IKS does not provide a properly configured DefaultStorageClass,
instead you need to tell the Helm chart to use the `default`
StorageClassName as shown above. This StorageClass does have
a dynamic provisioner, so it is not necessary to manually create
the PersistentVolumes. Note that it is not unusual for it to take
several minutes for your PersistentVolumes to be created
(dependent resources will be in `Pending` state).

####  IBM Cloud Lite cluster

The only available ingress method for an IBM Cloud Lite cluster is to
use a NodePort. Obtain the Public IP address of the sole worker node
by using the command
```shell
bx cs workers <my-cluster>
```
Then define `mycluster.yaml` as
```yaml
whisk:
  ingress:
    type: NodePort
    apiHostName: YOUR_WORKERS_PUBLIC_IP_ADDR
    apiHostPort: 31001

nginx:
  httpsNodePort: 31001

k8s:
  persistence:
    hasDefaultStorageClass: false
    explicitStorageClass: default
```

IKS does not provide a properly configured DefaultStorageClass,
instead you need to tell the Helm chart to use the `default`
StorageClassName as shown above. This StorageClass does have
a dynamic provisioner, so it is not necessary to manually create
the PersistentVolumes. Note that it is not unusual for it to take
several minutes for your PersistentVolumes to be created
(dependent resources will be in `Pending` state).

## Hints and Tips

On IBM Standard clusters, you can configure OpenWhisk to integrate
with platform logging and monitoring services following the general
instructions for enabling these services for pods deployed on
Kubernetes.

## Limitations

Using an IBM Cloud Lite cluster is only appropriate for development
and testing purposes.  It is not recommended for production
deployments of OpenWhisk.

When using an IBM Cloud Lite cluster, TLS termination will be handled
by OpenWhisk's `nginx` service and will use self-signed certificates.
You will need to invoke `wsk` with the `-i` command line argument to
bypass certificate checking.

IBM's 1.11 and 1.12 Kubernetes clusters have switched to using
`containerd` as the underlying container runtime system. This is not
compatible with OpenWhisk's DockerContainerFactory. Therefore you
either need to provision a 1.10 cluster or use the
KubernetesContainerFactory by adding the following to your
mycluster.yaml:
```yaml
invoker:
  containerFactory:
    impl: kubernetes
    kubernetes:
      agent:
        enabled: false
```
