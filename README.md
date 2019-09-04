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

