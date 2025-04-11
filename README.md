# bootc tekton pipeline

Installer l'opérateur OpenShift Pipelines, puis créer un namespace dans lequel nous allons créer le Pipeline.
`oc new-project bootc-pipeline`

Une fois l'opérateur installé, donner des droits priviliégiés au compte de service `pipeline` afin de pouvoir exécuter la Task Buildah qui fabriquera notre image OS:
`oc adm policy add-scc-to-user privileged -z pipeline -n bootc-pipeline`

Créer la ConfigMap qui contient notre Containerfile, on pourrait tout à faire le récupérer depuis un repository Git dans le Pipeline:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: containerfile-cm
data:
  Containerfile: |
    FROM registry.redhat.io/rhel9/rhel-bootc
    CMD ["mkdir", "/opt/custom"]
```

Créer le pull secret permettant de récupérer les images depuis notre registry privée, et l'attacher au compte de service `pipeline`:

```
apiVersion: v1
data:
  .dockerconfigjson: BASE64_SECRET
kind: Secret
metadata:
  name: registry-redhat-pull-secret
  namespace: bootc-pipeline
type: kubernetes.io/dockerconfigjson

```

 ajouter le secret au ServiceAccount:
 ```
apiVersion: v1
imagePullSecrets: #<---- ICI
- name: registry-redhat-pull-secret #AJOUTER LE SECRET
- name: pipeline-dockercfg-4tmsw
kind: ServiceAccount
metadata:
  annotations:
    openshift.io/internal-registry-pull-secret-ref: pipeline-dockercfg-4tmsw
  creationTimestamp: "2025-04-11T10:31:28Z"
  name: pipeline
  namespace: bootc-pipeline
secrets: #<---- ICI
- name: pipeline-dockercfg-4tmsw
- name: registry-redhat-pull-secret #AJOUTER LE SECRET
```

Créer le Pipeline qui permet de build l'image et de la pousser vers une Registry:
```
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: bootc-builder
  namespace: bootc-pipeline
spec:
  tasks:
  - name: buildah
    params:
    - name: IMAGE
      value: quay.io/vbroniko/bootc-base-rhel:v1 #Le nom de l'image souhaitée à la fabrication
    - name: BUILDER_IMAGE
      value: quay.io/buildah/stable:v1
    - name: STORAGE_DRIVER
      value: overlay
    - name: DOCKERFILE
      value: ./Containerfile
    - name: CONTEXT
      value: .
    - name: TLSVERIFY
      value: "true"
    - name: FORMAT
      value: oci
    - name: BUILD_EXTRA_ARGS
      value: ""
    - name: PUSH_EXTRA_ARGS
      value: ""
    - name: SKIP_PUSH
      value: "false"
    - name: BUILD_ARGS
      value:
      - ""
    taskRef:
      kind: Task
      name: buildah
    workspaces:
    - name: source
      workspace: bootc-workspace
  workspaces:
  - name: bootc-workspace
```

Le workspace prit en paramètre est la ConfigMap contenant le  Containerfile.
