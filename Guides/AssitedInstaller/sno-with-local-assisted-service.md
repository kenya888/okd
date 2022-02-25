# Install the OKD single node cluster(SNO) with the local assisted-service 

This is a guide for installation of the OKD single node cluster(SNO) with the [assisted-service](https://github.com/openshift/assisted-service)

## Contents


## Prerequisites

### Server specification

The minimum requirement for the OKD single node cluster (baremetal machine)

1. 8 cores CPU
2. 16GB Mem
3. 120GB install disk (just 120GB SSD may not be suitable because it is a little less than 120GB)
4. 1 NIC

Thi mininum requirement for the VM to serve the assisted-service

1. 1 or 2 cores CPU
2. more than 4GB Mem
3. more than 10GB disk space to store the container images of assisted-service

## 

### DHCP

The OKD cluster machine should access to any DHCP service in your homelab.

### DNS

The DNS services and domain are needed to which you can set DNS A records for OKD cluster.
You should reserve some DNS A record to setup OKD cluster as flollows.
In this instruction IPv6 is not covered.

| DNS Name | Description |
|:---------|:------------|
| api.CLUSTER_NAME.DOMAIN.NAME | like "api.okdsno.example.com |
| api-int.CLUSTERNAME.DOMAIN.NAME | like "api-int.okdsno.example.com (see [PR#3361](https://github.com/openshift/assisted-service/pull/3361)) |
| \*.apps.CLUSTERNAME.DOMAIN.NAME | (like "\*.api.okdsno.example.com) |

These record should be set after the OKD cluster have IP address from the DHCP by booting from the discovery ISO.

## STEP1: Setup the Assisted Service 

Install podman and git

```shell
dnf update -y
dnf install podman git-core
```

Clone the [assisted-service repository](https://github.com/openshift/assisted-service)

```shell
git clone https://github.com/openshift/assisted-service
cd assisted-service/deploy/podman
```

Edit `okd-configmap.yml` to adjust IP address settings of services to your machine.

```
IMAGE_SERVICE_BASE_URL: http://127.0.0.1:8888 <--- replace IP address with the one of your assisted-service host(like 192.168.2.199)
SERVICE_BASE_URL: http://127.0.0.1:8090 <--- the same as above
```

On the assisted-service machine, set firewalld to allow access to assisted-service from the client machine.

```shell
firewall-cmd --add-port=8888/tcp --add-port=8080/tcp --add-port=8090/tcp --permanent 
firewall-cmd --add-port=8888/tcp --add-port=8080/tcp --add-port=8090/tcp
```

Edit `pod.yml`(or `pod-persistent.yml` as you like) to avoid error of running initdb cmd in the postgresql container
(see [this issue](https://github.com/openshift/assisted-service/issues/3383) for deitals)

```
spec:
  containers:
  - args:
    - run-postgresql
    image: quay.io/centos7/postgresql-12-centos7:latest
    name: db
    securityContext: <--- add this line
      runAsUser: 26  <--- add this line. The "26" is the uid of postgres user in this container
    envFrom:
    - configMapRef:
        name: config
```

Launch assisted-service by podman play kube

```shell
podman play kube --configmap=okd-configmap.yml pod.yml
```

Check assisted-service pod is running successfully.

```shell
podman pod list
```


## STEP2: OKD cluster Installation

## 2. Prepare OKD SNO cluster installation with assisted-service UI
### 2.1 Access the assisted-service web UI and 

Access the assisted service web UI from any browser. The URL is `http://YOUR_IP:8080`

[Image]

Push `Create cluster` button

[Image]

abc


Write the discovery iso image to a usb stick.

### 2.2 Boot the SNO node with the discovery ISO

Boot the machine from the USB storage. Currently the tweak is needed for UEFI boot. See [This issue](https://github.com/openshift/assisted-service/issues/3384)

[Image]

[Image]


Download kubeconfig file

[Image]

## STEP3: Access to the OKD cluster

### Access the OKD cluster with oc

Obtain the oc command binary from the [release page](https://github.com/openshift/okd/releases/tag/4.9.0-0.okd-2022-01-29-035536) of OKD 

```shelkl
export KUBECONFIG=/path/to/your/kubeconfig/of/okd/cluster
oc get co
```

### Fix the state of machine-config-operator

In the almost all installation attempts, machine-config operator reports that failure.
You should fix this by hand. Please see this [Red Hat Knowledgebase article](https://access.redhat.com/solutions/4970731)


### Update the OKD cluster

After success of installtion, you can upgrade the cluster to the latest.
