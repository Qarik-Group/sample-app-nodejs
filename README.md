# Node.js Sample App

Decide on a builder that contains buildpacks that support this application.

```plain
pack set-default-builder cloudfoundry/cnb:cflinuxfs3
```

Auto-detect this application against the builder's buildpacks:

```plain
pack build starkandwayne/sample-app-nodejs --path .
```

The output will show that the NodeJS and Yarn buildpacks are used to create the runnable Docker image:

```plain
===> DETECTING
[detector] Trying group 1 out of 12 with 14 buildpacks...
[detector] ======== Results ========
[detector] skip: Cloud Foundry Archive Expanding Buildpack
[detector] pass: Cloud Foundry OpenJDK Buildpack
[detector] skip: Cloud Foundry Build System Buildpack
[detector] fail: Cloud Foundry JVM Application Buildpack
[detector] skip: Cloud Foundry Spring Boot Buildpack
[detector] skip: Cloud Foundry Apache Tomcat Buildpack
[detector] skip: Cloud Foundry DistZip Buildpack
[detector] skip: Cloud Foundry Procfile Buildpack
[detector] skip: Cloud Foundry Azure Application Insights Buildpack
[detector] skip: Cloud Foundry Debug Buildpack
[detector] skip: Cloud Foundry Google Stackdriver Buildpack
[detector] skip: Cloud Foundry JDBC Buildpack
[detector] skip: Cloud Foundry JMX Buildpack
[detector] pass: Cloud Foundry Spring Auto-reconfiguration Buildpack
[detector] Trying group 2 out of 12 with 2 buildpacks...
[detector] ======== Results ========
[detector] pass: Node Engine Buildpack
[detector] pass: Yarn Buildpack
```

These two buildpacks are then applied in sequence against the application:

```plain
===> BUILDING
[builder] -----> Node Engine Buildpack 0.0.26
[builder]   Node Engine 10.16.2: Contributing to layer
[builder]     Downloading from https://buildpacks.cloudfoundry.org/dependencies/node/node-10.16.2-linux-x64-cflinuxfs3-9d21a165.tgz
[builder]     Verifying checksum
[builder]        Expanding to /layers/org.cloudfoundry.node-engine/node
[builder]     Writing NODE_HOME to shared
[builder]     Writing NODE_ENV to shared
[builder]     Writing NODE_MODULES_CACHE to shared
[builder]     Writing NODE_VERBOSE to shared
[builder]     Writing NPM_CONFIG_PRODUCTION to shared
[builder]     Writing NPM_CONFIG_LOGLEVEL to shared
[builder]     Writing WEB_MEMORY to shared
[builder]     Writing WEB_CONCURRENCY to shared
[builder]     Writing .profile.d/0_memory_available.sh
[builder] -----> Yarn Buildpack 0.0.18
[builder]   Yarn 1.17.3: Contributing to layer
[builder]     Downloading from https://buildpacks.cloudfoundry.org/dependencies/yarn/yarn-1.17.3-any-stack-e3835194.tar.gz
[builder]     Verifying checksum
[builder]        NODE_HOME Value /layers/org.cloudfoundry.node-engine/node
[builder]        Expanding to /layers/org.cloudfoundry.yarn/yarn
[builder]   Node Dependencies 0560d3380dc3613e4e8359b4a62121acae182faa262ec51a55b6540b9ad4468e: Contributing to layer
[builder] NODE HOME VALUE /layers/org.cloudfoundry.node-engine/node
[builder]
[builder] Running yarn in online mode
[builder] yarn install v1.17.3
[builder] [1/4] Resolving packages...
[builder] [2/4] Fetching packages...
[builder] [3/4] Linking dependencies...
[builder] [4/4] Building fresh packages...
[builder] Done in 2.23s.
[builder] yarn.lock and package.json match
[builder]     Writing NODE_PATH to shared
[builder]     Writing PATH to shared
[builder]     Writing npm_config_nodedir to shared
[builder]   Process types:
[builder]     web: yarn start
```

Running `pack build` a second time will be faster as it reuses cached layers:

```plain
===> BUILDING
[builder] -----> Node Engine Buildpack 0.0.26
[builder]   Node Engine 10.16.2: Reusing cached layer
[builder] -----> Yarn Buildpack 0.0.18
[builder]   Yarn 1.17.3: Reusing cached layer
[builder]   Node Dependencies 0560d3380dc3613e4e8359b4a62121acae182faa262ec51a55b6540b9ad4468e: Reusing cached layer
[builder]   Process types:
[builder]     web: yarn start
[builder]   Removing unused layers
[builder]     modules_cache
```

## Kubernetes Pod

```plain
kubectl apply -f pod.yaml
kubectl get pods
kubectl delete -f pod.yaml
```

## Kubernetes Deployment

```plain
kubectl apply -f deployment.yaml
kubectl get all
curl <LB>:8080
```

### Rollout update

In one terminal:

```plain
watch "kubectl get pod; echo; curl -sS <LB>:8080"
```

In another, create `:0.0.2` image and deploy:

1. Edit `server.js`
1. `pack build starkandwayne/sample-app-nodejs:0.0.2 --no-pull --publish`
1. Edit `deployment.yaml` to use `:0.0.2`
1. `kubectl apply -f deployment.yaml`

### Clean up

```plain
kubectl delete -f deployment.yaml
```
