# How to Model Your Gitops Environments

This is a counter-example on how to model Kustomize folders for a GitOps application and promote releases
between environments, using one branch per environment.  This has been forked from https://github.com/kostis-codefresh/gitops-environment-promotion and changed to use the branch-per-environment pattern.

THis fork has been made to address a discussion which can be found here: https://github.com/argoproj/argo-cd/discussions/5667#discussioncomment-2087895

## Folder structure

The base directory holds configuration which is common to all environments.  Because of the branching structure, there will be temporary differences between environments, but these are only temporal. (i.e. any change on one environment will eventually appear on all environments).

The variants folder holds characteristics that are common to some environments. They are not expected to change often, but are expected to remain different throughout the life of the application.

The envs folder holds the kustomization.yaml that defines which variants are used for the kustomization.  It is expected to change very infrequently.

In the example application, we have variants for all prod and non-prod environments, the regions and also gpu or non-gpu. Here is an example of the prod variant that applies to ALL production environments.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  template:
    spec:
      containers:
      - name: webserver-simple
        env:
        - name: ENV_TYPE
          value: "production"
        - name: PAYPAL_URL
          value: "production.paypal.com"   
        - name: DB_USER
          value: "prod_username"
        - name: DB_PASSWORD
          value: "prod_password"                     
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
```

In the example above we make sure that all production environments are using the production DB credentials, the production payment gateway and a liveness probe. These settings belong to the set of configuration that we don’t expect to promote between environments but we assume that they will be static across the application lifecycle.

With the base and variants ready we can now define every final environment with a combination of those properties.

Here is an example of the staging ASIA environment:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: staging
namePrefix: staging-asia-

resources:
- ../../base

components:
  - ../../variants/non-prod
  - ../../variants/asia
  - ../../variants/gpu

patchesStrategicMerge:
- environment.yml
- replicas.yml
```

First we define some common properties. We inherit all configuration from base, from non-prod environments, for all environments in Asia and for all environments with gpu.

The key point here is the patches that we apply.  The environment.yml only defines The replicas.yml is separate because we assume each environment will have different requirements for replica settings.  

Inside the base folder, the version.yml file (which is the most important thing to promote between environments) defines only the image of the application and nothing else.

The main reason for making this a separate file is so that the build pipeline can automatically generate this.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  template:
    spec:
      containers:
      - name: webserver-simple
        image: docker.io/kostiscodefresh/simple-env-app:2.0
```

Settings that are expected to be promoted between environments are inside the deployment.yml, service.yml et cetera.  They could be extracted into separate files (using patches), if desired.

## Branch structure

Each environment (as found in the envs folder) also gets its own branch.  These have the same name as the folder name.

The only branch that is open for commits should be the main (or development) branch.  (Yes, even changes to 'prod' avriants, or env-specific changes have to go through this process.)

Promoting between environments is therefore done exclusively by merging between branches.  Ideally, only fast-forward merges are performed, to completely eliminate any possibility of merge conflicts.

If this is maintained, the branches will not form a tree structure, but a simple timeline, with each environment pointing to a different point on that line (prod will obviously be behind staging, which in turn is behind qa).

