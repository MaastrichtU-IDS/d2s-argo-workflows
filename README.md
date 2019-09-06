# Run workflows with [Argo](https://github.com/argoproj/argo/)

* Access Maastricht University [DSRI OpenShift platform](https://app.dsri.unimaas.nl:8443/).

* Access its [Argo UI](http://argo-ui-argo.app.dsri.unimaas.nl/workflows).

## Install [oc client](https://www.okd.io/download.html)

```shell
wget https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz

tar xvf openshift-origin-client-tools*.tar.gz
cd openshift-origin-client*/
sudo mv oc kubectl /usr/local/bin/
```

---

## oc login

Login to the cluster using the OpenShift client
```shell
oc login https://openshift_cluster:8443 --token=MY_TOKEN
```

Get the command with the token from the `Copy Login Command` button in the user details at the top right of the OpenShift webpage.

---

## Install Argo

Install [Argo client](https://github.com/argoproj/argo/blob/master/demo.md#1-download-argo)

### Run workflows

`oc login` required

```shell
argo submit --watch d2s-sparql-workflow.yaml

# Define params in a separate YAML file (no _ in params name)
argo submit d2s-workflow-transform-xml.yaml -f workflow-params-drugbank.yml
argo submit d2s-workflow-transform-xml.yaml -f workflow-params-pubmed.yml
```

### Check running workflows

```shell
argo list
```

## Debug Argo

To get into the container. Create YAML with command `tail /dev/null` to keep it running

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    purpose: download-data-files
  name: d2s-download-pod
  namespace: argo
spec:
  volumes:
  - name: workdir
    persistentVolumeClaim:
      claimName: data2services-storage
  containers:
  - name: d2s-download
    image: vemonet/data2services-download:latest
    command: [ "tail", "-f", "/dev/null"]
    volumeMounts:
    - name: workdir
      mountPath: /data
```

Then start the pod

```shell
oc create -f archives/d2s-download-pod.yaml

# Connect with Shell
oc rsh d2s-download-pod
```



---

## oc commands

### List pods

```shell
oc get pod
```

### Create pod from JSON

```shell
oc create -f examples/hello-openshift/hello-pod.json
```

---

## Workflow administration

### Create persistent volume

https://app.dsri.unimaas.nl:8443/console/project/argo/create-pvc

* Storage class > `maprfs-ephemeral`
* Shared Acces (RWX)

### Mount filesystem

Deploy a [filebrowser](https://hub.docker.com/r/filebrowser/filebrowser) on MapR to access volumes

Go to https://app.dsri.unimaas.nl:8443/console/catalog > click `Deploy image`

* Add to Project: `argo`
* Image Name: `filebrowser/filebrowser` 
* Give a name to your image: `filebrowser`
* Click `Deploy`
* Go to `argo` project > Click on latest deployment of the `filebrowser`
* Delete the automatically mounted volume, and add the persistent volume (`data2services-storage`). Should be on `/srv`
* Add route

* Access on http://d2s-filebrowser-argo.app.dsri.unimaas.nl/files/

### Create temporary volume in the workflow

```yaml
volumeClaimTemplates:                 # define volume, same syntax as k8s Pod spec
  - metadata:
      name: workdir                     # name of volume claim
      annotations:
        volume.beta.kubernetes.io/storage-class: maprfs-ephemeral
        volume.beta.kubernetes.io/storage-provisioner: mapr.com/maprfs
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi 
```

### Create secret

* Access [OpenShift](https://app.dsri.unimaas.nl:8443/) > `Argo` project > `Resources` > `Secret`
* `Secret Type`: Generic Secret
* `Secret Name`: d2s-sparql-password
* `Key`: password
* Enter the password in the text box `Clean Value`

Now in the workflow definition you can use the secret as environment variable

```yaml
- name: d2s-sparql-operations
    inputs:
      parameters:
      - name: sparqlQueriesPath
      - name: sparqlInputGraph
      - name: sparqlOutputGraph
      - name: sparqlServiceUrl
    container:
      image: vemonet/data2services-sparql-operations:latest
      args: ["-ep", "{{workflow.parameters.sparqlEndpoint}}",
    "-un", "{{workflow.parameters.sparqlUsername}}", 
    "-pw", "$SPARQLPASSWORD",  # Secret here]
      env:
      - name: SPARQLPASSWORD
        valueFrom:
          secretKeyRef:
            name: d2s-sparql-password
            key: password
```

