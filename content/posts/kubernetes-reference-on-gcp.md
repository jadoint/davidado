+++
title = 'Kubernetes Reference on Gcp'
date = 2023-09-03T00:44:59-07:00
draft = false
+++

This is just a quick reference sheet for me to get something up and running on Google Cloud quickly so I'm writing this as if no one else is reading it...

Starts you off with 2 nodes (e2-medium by default) in a multi-zonal configuration located in *us-central1* with auto-scaling enabled with a minimum of 1 node and a maximum of 4. See: [https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler)

`gcloud container clusters create example-cluster --zone us-central1-a --node-locations us-central1-a,us-central1-b,us-central1-f --num-nodes 2 --enable-autoscaling --min-nodes 1 --max-nodes 4`

Scale each deployment based on the cpu requests (spec.resources.requests.cpu) set for each pod.

See: [https://cloud.google.com/kubernetes-engine/docs/how-to/scaling-apps](https://cloud.google.com/kubernetes-engine/docs/how-to/scaling-apps)

See: [https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

`kubectl autoscale deployment my-app --min 1 --max 10 --cpu-percent 50`

If you have an existing docker-compose file and are just getting started with using kubernetes yaml files, try using kompose to convert your docker-compose file into a kubernetes format. See: https://kompose.io/

*Linux curl -L*

`https://github.com/kubernetes/kompose/releases/download/v1.22.0/kompose-linux-amd64 -o kompose`

*macOS curl -L*

`https://github.com/kubernetes/kompose/releases/download/v1.22.0/kompose-darwin-amd64 -o kompose`

```
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
kompose convert -f docker-compose.yml
```

Use `kubectl apply -f .` to launch the kubernetes deployments for the resulting yaml files from the step above. See: [https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/](https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/)

Run `kubectl get services` to get the external IP of your LoadBalancer service (Nginx for me) and use that for your domain's DNS. See: [https://cloud.google.com/kubernetes-engine/docs/quickstarts/deploying-a-language-specific-app](https://cloud.google.com/kubernetes-engine/docs/quickstarts/deploying-a-language-specific-app)

I use Google Cloud Build to auto-build my images whenever I push to a specific branch on my GitHub repo and use `$SHORT_SHA` as the tag for the image name. To update my kubernetes cluster, I would just change the image tag in the Kubernetes deployment yaml file then run `kubectl apply -f [name of deployment]-deployment.yaml` again to re-deploy the pods.

## Secrets

Use Kubernetes secrets to store sensitive credentials and expose them as environment variables.

See: [https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/)

See: [https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)

Store secrets from files: [https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)

Using secrets and volume mounts for Google application credentials: [https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform)

*db-creds-secret.yaml*

```
apiVersion: v1

kind: Secret

metadata:

   name: db-creds

stringData:

   username: dbusername

   password: dbpassword
```

Run `kubectl apply -f db-creds-secret.yaml` to store credentials in Kubernetes. To assign secret values to pod environment variables, update the pod deployment file:

```
apiVersion: apps/v1

kind: Deployment

metadata:

   name: blog

spec:

   selector:

   matchLabels:

      app: blog

      tier: backend

      track: stable

   replicas: 1

   template:

      metadata:

         labels:

            app: blog

            tier: backend

            track: stable

      spec:

         containers:

            - name: blog

              image: gcr.io/[project-id]/github.com/jadoint/blog:blog

             ports:

               - name: https

                 containerPort: 9000

            env:

               - name: DB_USER

                 valueFrom:

                    secretKeyRef:

                       name: db-creds

                      key: username

            - name: DB_PASSWORD

              valueFrom:

                 secretKeyRef:

                    name: db-creds

                    key: password
```

## Cronjobs

See: [https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

See: [https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

Use `.spec.failedJobsHistoryLimit` and `.spec.successfulJobsHistoryLimit` to clear out old failed or successful pods after running the job. Use `.spec.concurrencyPolicy` to let cronjob know whether or not to run a new job if the previous job is still running. Run `kubectl create -f cronjob.yaml` to create the cronjob.

```
apiVersion: batch/v1beta1

kind: CronJob

metadata:

 name: hello

spec:

 schedule: "*/1 * * * *"

concurrencyPolicy: Forbid

failedJobsHistoryLimit: 1

successfulJobsHistoryLimit: 3

 jobTemplate:

   spec:

     template:

       spec:

         containers:

         - name: hello

           image: busybox

           imagePullPolicy: IfNotPresent

           command:

           - /bin/sh

           - -c

           - date; echo Hello from the Kubernetes cluster

         restartPolicy: OnFailure
```

## Even Pod Distribution

See: [https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/)

See: [https://stackoverflow.com/questions/59434797/how-to-deploy-pods-across-all-nodes-evenly-in-kubernetes](https://stackoverflow.com/questions/59434797/how-to-deploy-pods-across-all-nodes-evenly-in-kubernetes)

```
apiVersion: apps/v1

kind: Deployment

metadata:

 name: nginx

spec:

 selector:

   matchLabels:

     app: nginx

     tier: frontend

     track: stable

 replicas: 4

 template:

   metadata:

     labels:

       app: nginx

       tier: frontend

       track: stable

  spec:

    topologySpreadConstraints:

      - maxSkew: 1

        topologyKey: node

        whenUnsatisfiable: ScheduleAnyway

        labelSelector:

          matchLabels:

          app: nginx

    affinity:

      podAntiAffinity:

        preferredDuringSchedulingIgnoredDuringExecution:

          - weight: 100

            podAffinityTerm:

              labelSelector:

                matchExpressions:

                  - key: app

                   operator: In

                   values:

                     - nginx

              topologyKey: kubernetes.io/hostname

# Volume and container config...#
```

## Get Pod Requests and Limits

`kubectl top pod <podname>`
