# Docs
Please see the Github Pages Site for complete documentation: [quarkusdroneshop.github.io](https://quarkusdroneshop.github.io)

# quarkusdroneshop Tekton pipelines Guide

### Requirements 
* [Postgres Operator](https://github.com/quarkusdroneshop/quarkusdroneshop-helm/wiki#install-postgres-operator)
* AMQ Streams

**Once Postgres Operator Database is installed run the following below**
```
$ curl -OL https://raw.githubusercontent.com/quarkusdroneshop/quarkusdroneshop-ansible/master/files/deploy-quarkusdroneshop-ansible.sh
$ chmod +x deploy-quarkusdroneshop-ansible.sh
$ NAMESPACE=quarkusdroneshop-homeoffice
$  sed -i "s/quarkusdroneshop-demo/${NAMESPACE}/g" deploy-quarkusdroneshop-ansible.sh
$ ./deploy-quarkusdroneshop-ansible.sh 
 Options:
  -d      Add domain 
  -o      OpenShift Token
  -p      Postgres Password
  -s      Store ID
  -h      Display this help and exit
  -r      Destroy droneshop 
  To deploy qaurkusdroneshop-ansible playbooks
  ./deploy-quarkusdroneshop-ansible.sh  -d ocp4.example.com -o sha-123456789 -p 123456789 -s ATLANTA
$  ./deploy-quarkusdroneshop-ansible.sh  -d ocp4.example.com -o sha-123456789 -p 123456789 -s ATLANTA
```

**Install OpenShift Pipelines**
```
cat <<EOF | oc -n openshift-operators create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
spec:
  channel: stable
  installPlanApproval: Automatic
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

**Install tkn cli**  
`on linux AMD 64`
```
# Get the tar.xz
curl -LO https://github.com/tektoncd/cli/releases/download/v0.18.0/tkn_0.18.0_Linux_x86_64.tar.gz
# Extract tkn to your PATH (e.g. /usr/local/bin)
sudo tar xvzf tkn_0.18.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```

`on mac`
```
brew install tektoncd-cli
```

**Create Quarkus Drone Shop project**
```
oc new-project quarkusdroneshop-cicd
```

**Set up quay permissions**
```
oc  create -f sa/pipeline-sa.yaml 
```
**Set privileged containers for the pushImageroQuay task OCP 4.7.x**
```
oc adm policy add-scc-to-user privileged -z pipeline -n  quarkusdroneshop-cicd
```

**Set quay credentials**  
```
$ cat quay-secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: quay-auth-secret
data:
  .dockerconfigjson: changeinfo==
type: kubernetes.io/dockerconfigjson
```

**Create secret**
```
oc create -f quay-secret.yml --namespace=quarkusdroneshop-cicd
```

**configure slack webhook**  
* [tekton hub](https://hub-preview.tekton.dev/) 
* [Sending messages using Incoming Webhooks](https://api.slack.com/messaging/webhooks)
```
# oc create -f event-notification/send-to-webhook-slack.yaml -n quarkusdroneshop-cicd
# WEBHOOKURL=https://hooks.slack.com/services/xxxxx/Xxxxxx
# cat >webhook-secret.yaml<<YAML
kind: Secret
apiVersion: v1
metadata:
  name: webhook-secret
stringData:
  url: ${WEBHOOKURL}
YAML
# oc create -f webhook-secret.yaml -n quarkusdroneshop-cicd
```

## HOME Office (Backoffice)
**quarkusdroneshop-homeoffice-ui tekton pipeline**  
[quarkusdroneshop-homeoffice-ui](quarkusdroneshop-homeoffice-ui/README.md)

**homeoffice-backend tekton pipeline**  
[homeoffice-backend](homeoffice-backend/README.md)

**homeoffice-ingress tekton pipeline**  
[homeoffice-ingress](homeoffice-ingress/README.md)


## Store front microservices  

**quarkusdroneshop-qdca10 tekton pipeline**  
[quarkusdroneshop-qdca10](quarkusdroneshop-qdca10/README.md)

**quarkusdroneshop-counter tekton pipeline**  
[quarkusdroneshop-counter](quarkusdroneshop-counter/README.md)

**quarkusdroneshop-qdca10pro tekton pipeline**  
[quarkusdroneshop-qdca10pro](quarkusdroneshop-qdca10pro/README.md)

**quarkusdroneshop-web tekton pipeline**   
[quarkusdroneshop-web](quarkusdroneshop-web/README.md)

## RHEL Edge Pipelines
**quarkusdroneshop-majestic-monolith tekton pipeline**   
[quarkusdroneshop-majestic-monolith](quarkusdroneshop-majestic-monolith/README.md)

## Pipeline の構成

                         GitHub Repository
                               │
                               ▼
                    fetch-repository (git-clone)
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
      Maven Unit Test                  Checkstyle
     (clean test)                (checkstyle:check)
              │                                 │
              │                                 ▼
              │                             PMD Scan
              │                        (pmd:pmd)
              │                                 │
              │                                 ▼
              │                         Semgrep Scan
              │                    (SAST / Secrets)
              │                                 │
              │                                 ▼
              │                         Gitleaks Scan
              │                    (Secret Detection)
              │                                 │
              │                                 ▼
              │                          Trivy Scan
              │                (Vulnerability + Secret)
              │                                 │
              │                                 ▼
              └───────────────┬─────────────────┘
                              │
                              ▼
                       SpotBugs Scan
                    (spotbugs:spotbugs)
                              │
                              ▼
                     Maven Integration Test
                  (Failsafe Integration Test)
                              │
                              ▼
                    Push to OpenShift Apps
                 (BuildConfig / Deployment)
                              │
                              ▼
                        Wapiti Scan
                  (Dynamic Security Test)
                              │
                              ▼
                     Generate HTML Report
                     (exec:generate-report)
                              │
                              ▼
                     Print CSV Summary
                     (print-csv-table)
                              │
                              ▼
                    Publish Test Report
                              │
                              ▼
                     Pipeline Summary
                    (Tekton Results API)