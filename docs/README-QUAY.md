# Prepare for quay deployment


## Create config-bundle and QuayRegistry yaml file

Create a quay-config:

```bash
cat << EOF > quay-config.yml
ALLOW_PULLS_WITHOUT_STRICT_LOGGING: false
AUTHENTICATION_TYPE: Database
DEFAULT_TAG_EXPIRATION: 2w
ENTERPRISE_LOGO_URL: /static/img/RH_Logo_Quay_Black_UX-horizontal.svg
FEATURE_BUILD_SUPPORT: false
FEATURE_DIRECT_LOGIN: true
FEATURE_MAILING: false
REGISTRY_TITLE: Clustership - Red Hat Quay
REGISTRY_TITLE_SHORT: Red Hat Quay
TAG_EXPIRATION_OPTIONS:
- 2w
TEAM_RESYNC_STALE_TIME: 60m
TESTING: false
SERVER_HOSTNAME: hubcluster-registry.apps.clustership.com
EXTERNAL_TLS_TERMINATION: true
FEATURE_USER_INITIALIZE: true
BROWSER_API_CALLS_XHR_ONLY: false
SUPER_USERS:
- quayadmin
FEATURE_USER_CREATION: false
EOF
```

Adjust quay-config.yaml to fit specific requirements.

Then create the secret file.

```bash
oc --dry-run=client create secret generic --from-file config.yaml=./quay-config.yml init-config-bundle-secret -o yaml > 01-init-config-bundle-secret.yaml
```



## Deploy the new QuayRegistry

Create your Quay registry from CRD:

```bash
cat << EOF > 02-quay-registry.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: quay-enterprise
---
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: hubcluster-registry
  namespace: quay-enterprise
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  configBundleSecret: init-config-bundle-secret
EOF

cat << EOF > kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  # topology.kubernetes.io/region: demo
  # Each SITE_ID is a different zone
  #topology.kubernetes.io/zone: sample-zone
  telco-gitops/cluster-name: hubcluster-espoo
  telco-gitops/cluster-type: hubcluster

# commonAnnotations:
#   # Annotations to work around automatically generated resoruces
#   # to avoid ArgoCD reporing out-of-sync
#   argcd.argoproj.io/compare-options: IgnoreExtraneous
#   argocd.argoproj.io/sync-options: Prune=false


namespace: quay-enterprise
# namespace: hubcluster-registry
# Namespace is already created by the operator deployment

resources:
- 01-init-config-bundle-secret.yaml
- 02-quay-registry.yaml
EOF
```

```bash
oc apply -k .
```


## Create the admin user

```bash
QUAY_API=$(awk '/SERVER_HOSTNAME: / { print "https://" $2 }' quay-config.yml)
# or 
# QUAY_API=$(oc get quayregistry  hubcluster-registry -o jsonpath='{.status.registryEndpoint}')

curl -X POST -k  ${QUAY_API}/api/v1/user/initialize --header 'Content-Type: application/json' --data '{ "username": "quayadmin", "password":"quaypass123", "email": "quayadmin@example.com", "access_token": true}'
```

## Create an Organization

```bash
OAUTH_TOKEN=<oauth-token>
QUAY_API=$(oc get quayregistry  hubcluster-registry -o jsonpath='{.status.registryEndpoint}')
curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer ${OAUTH_TOKEN}" ${QUAY_API}/api/v1/organization/ --data '{"name": "openshift", "email": "openshift@example.com"}'
```


## Create repository

```bash
curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer ${OAUTH_TOKEN}" ${QUAY_API}/api/v1/repository --data '{ "repository": "ocp-release", "visibility": "private", "namespace": "openshift", "repo_kind": "image", "description": "OpenShift installation mirrored repo" }'
```

## Create robot

```bash
curl -X PUT -k --header 'Content-Type: application/json' -H "Authorization: Bearer ${OAUTH_TOKEN}" ${QUAY_API}/api/v1/organization/openshift/robots/ocp-mirror --data '{ "description": "robot to mirror OpenShift mirrored repo" }'
```

```bash
curl -X PUT -k --header 'Content-Type: application/json' -H "Authorization: Bearer ${OAUTH_TOKEN}" ${QUAY_API}/api/v1/user/robots/ocp-mirror --data '{ "description": "robot to mirror OpenShift mirrored repo" }'
```

#
# Bad idea, it is a super_user robots
# but I can't find other way to get everything working for ZTP

# Add repository permission to robot

```bash
ORG=openshift
REPOSITORY=ocp-release
ROBOT=ocp-mirror
curl -X PUT -k --header 'Content-Type: application/json' -H "Authorization: Bearer ${OAUTH_TOKEN}" ${QUAY_API}/api/v1/repository/${ORG}/${REPOSITORY}/permissions/user/${ORG}+${ROBOT} --data '{ "role": "write" }'
```


## Create mirror pull secret

```bash
ROBOT_OAUTH_TOKEN=<robot-token>
cat << EOF > mirror-pull-secret.json
{
  "auths": {
    "${QUAY_API}": {
      "auth": "$(echo -n ${ORG}+${ROBOT}:${ROBOT_OAUTH_TOKEN} | base64 -w0)",
      "email": ""
    }
  }
}
EOF
```


Merge your pull secrets

```bash
jq -s '.[0] * .[1]' pullsecret.json  /repos/RH_POC/mgmt/hubcluster/quay-registry/mirror-pull-secret.json > mirror-pull-secret.json
```
