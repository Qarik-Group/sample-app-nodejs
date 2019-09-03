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
curl <LB>
```

### Rollout update

In one terminal:

```plain
watch "curl -sS <LB>"
```

In another, watch pod changes:

```plain
kubectl get pods -w
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

## kpack demo

Kpack is an alternate approach to Cloud Native Buildpack lifecycle management. Rather than run `pack` on local machine, `kpack` is a system running atop your Kubernetes cluster that continuously monitors Git repositories and generates updated OCI images for any new commits.

`kpack` does not wrap `pack`, rather they both share the CNB lifecycle project to perform the same sequence for discovery and applying buildpacks to source code.

The `kpack-image.yaml` assumes you have a service account `service-account` setup, and a CNB builder resource named `cflinuxfs3-builder`. Any builder can be used, as long as it can support this NodeJS application.

```plain
$ kubectl get serviceaccounts,builder.build.pivotal.io
NAME                             SECRETS   AGE
serviceaccount/default           1         5m10s
serviceaccount/service-account   3         4m47s

NAME                                          AGE
builder.build.pivotal.io/cflinuxfs3-builder   4m
```

To initiate the first kpack build of this Git repository into an OCI/docker image:

```plain
kubectl apply -f kpack-image.yaml
```

This `image` resource initiates an initial `build`:

```plain
$ kubectl get images,builds
NAME                                       LATESTIMAGE   READY
image.build.pivotal.io/sample-app-nodejs                 Unknown

NAME                                                     IMAGE   SUCCEEDED
build.build.pivotal.io/sample-app-nodejs-build-1-zd2ht           Unknown
```

You can stream the in-progress CNB lifecycle with `kpack`'s `logs` CLI:

```plain
logs -image sample-app-nodejs
```

The initial build may take a minute whilst it initially clones the Git repo, etc.

The `image` and `build` are now updated with the resulting OCI ID for the target registry:

```plain
$ kubectl get images,builds
NAME                                       LATESTIMAGE                                                                                                               READY
image.build.pivotal.io/sample-app-nodejs   index.docker.io/starkandwayne/sample-app-nodejs@sha256:bf7ccdbb7c4790748c8613e9f7aac11de529bcb9cbb84ceea56f52045dbfe485   True

NAME                                                     IMAGE                                                                                                                     SUCCEEDED
build.build.pivotal.io/sample-app-nodejs-build-1-zd2ht   index.docker.io/starkandwayne/sample-app-nodejs@sha256:bf7ccdbb7c4790748c8613e9f7aac11de529bcb9cbb84ceea56f52045dbfe485   True
```

We can now deploy our application to Kubernetes with this new OCI by editing `deployment.yaml` and replacing the `image: index.docker.io...` line with the full image ID `index.docker.io/starkandwayne/sample-app-nodejs@sha256:bf7ccdbb7c4790748c8613e9f7aac11de529bcb9cbb84ceea56f52045dbfe485` from the example above.

```plain
$ kubectl apply -f deployment.yaml
deployment.apps/sample-app-nodejs created
service/sample-app-nodejs created
```

We can watch for the deployment's service to be allocated its load balancer:

```plain
$ kubectl get services -w
NAME                TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes          ClusterIP      10.99.0.1     <none>        443/TCP        12m
sample-app-nodejs   LoadBalancer   10.99.12.88   <pending>     80:30653/TCP   16s
sample-app-nodejs   LoadBalancer   10.99.12.88   35.189.19.51   80:30653/TCP   61s
```

The application can now be reached via the LoadBalancer's IP:

```plain
$ curl 35.189.19.51
Hello World!
```

In another terminal, poll this endpoint to observe the future change:

```plain
$ watch curl -sS 35.189.19.51
```

If in `server.js`, I change `Hello World!` to `Hello kpack`, save, commit, and push, then kpack will discover the new commit and start a new `build`:

```plain
$ git commit -m "test hello kpack"
$ git push
$ kubectl get builds -w
NAME                              AGE
sample-app-nodejs-build-1-zd2ht   11m
sample-app-nodejs-build-2-gqljg   0s
```

Once the new `build-2` has been created, we can switch to `logs` and stream its output:

```plain
logs -image sample-app-nodejs
```

The build sequence is much faster this time.

We now have one `image`, but two `builds`:

```plain
$ kubectl get images,builds
NAME                                       LATESTIMAGE                                                                                                               READY
image.build.pivotal.io/sample-app-nodejs   index.docker.io/starkandwayne/sample-app-nodejs@sha256:a2fd6417143a37cd27467f9bf38c42b8d9faaf031b2769e63ea5d8fa10180a4c   True

NAME                                                     IMAGE                                                                                                                     SUCCEEDED
build.build.pivotal.io/sample-app-nodejs-build-1-zd2ht   index.docker.io/starkandwayne/sample-app-nodejs@sha256:bf7ccdbb7c4790748c8613e9f7aac11de529bcb9cbb84ceea56f52045dbfe485   True
build.build.pivotal.io/sample-app-nodejs-build-2-gqljg   index.docker.io/starkandwayne/sample-app-nodejs@sha256:a2fd6417143a37cd27467f9bf38c42b8d9faaf031b2769e63ea5d8fa10180a4c   True
```

As before, update `deployment.yaml` `image: index.docker.io...` with the new OCI ID:

```yaml
    spec:
      containers:
      - name: sample-app-nodejs
        image: index.docker.io/starkandwayne/sample-app-nodejs@sha256:a2fd6417143a37cd27467f9bf38c42b8d9faaf031b2769e63ea5d8fa10180a4c
```

And apply the updated deployment:

```plain
$ kubectl apply -f deployment.yaml
deployment.apps/sample-app-nodejs configured
service/sample-app-nodejs unchanged
```

The `curl` terminal will intermittently show both `Hello World!` and `Hello kpack!` until all containers are replaced with the new image:

```plain
$ kubectl get pods -w
NAME                                        READY   STATUS              RESTARTS   AGE
sample-app-nodejs-64f94f848d-4r94q          1/1     Terminating         0          8m4s
sample-app-nodejs-64f94f848d-5sp5q          1/1     Running             0          8m4s
sample-app-nodejs-9c58f6897-5wnss           1/1     Running             0          23s
sample-app-nodejs-9c58f6897-qkh4b           0/1     ContainerCreating   0          1s
sample-app-nodejs-9c58f6897-xqbm5           1/1     Running             0          12s
sample-app-nodejs-build-1-zd2ht-build-pod   0/1     Completed           0          14m
sample-app-nodejs-build-2-gqljg-build-pod   0/1     Completed           0          3m12s
sample-app-nodejs-64f94f848d-4r94q   0/1   Terminating   0     8m4s
sample-app-nodejs-64f94f848d-4r94q   0/1   Terminating   0     8m5s
sample-app-nodejs-64f94f848d-4r94q   0/1   Terminating   0     8m5s

$ kubectl get pods
NAME                                        READY   STATUS      RESTARTS   AGE
sample-app-nodejs-9c58f6897-5wnss           1/1     Running     0          85s
sample-app-nodejs-9c58f6897-qkh4b           1/1     Running     0          63s
sample-app-nodejs-9c58f6897-xqbm5           1/1     Running     0          74s
sample-app-nodejs-build-1-zd2ht-build-pod   0/1     Completed   0          15m
sample-app-nodejs-build-2-gqljg-build-pod   0/1     Completed   0          4m14s
```

