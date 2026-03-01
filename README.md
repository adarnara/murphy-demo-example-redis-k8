# Kubernetes Scripts for deploying Murphy-Movies Project

This repo contains kubernetes scripts to be executed in your Ubuntu instance after logging in as the kOps user.
## Pre-requisites
1. Make sure you have an AWS Kubernetes cluster set up (follow Task 2.1 and 2.2 from Project 5 in your AWS Ubuntu instance).
2. Create a `regcred` secret which contains your Dockerhub credentials. Replace variables with your info.
   ```
    kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<YOUR_DOCKER_USERNAME> --docker-password=<YOUR DOCKER PASSWORD> --docker-email=<YOUR DOCKER EMAIL>
    ```
3. Set up MySQL pods through the helm chart.
```
helm install mysql --set auth.rootPassword=root,auth.database=murphymovies,auth.username=murphyuser,auth.password='My7$Password',secondary.persistence.enabled=true,secondary.persistence.size=2Gi,primary.persistence.enabled=true,primary.persistence.size=2Gi,architecture=replication,auth.replicationPassword=texera,secondary.replicaCount=1 oci://registry-1.docker.io/bitnamicharts/mysql
```
Test by running `kubectl get pods`, both mysql pods should be in `RUNNING` state and `READY (1/1)`.
4. Populate MySQL database => 
   1. Run `kubectl exec -it pod/mysql-primary-0 -- /bin/bash`.
   2. Run `mysql -u root -p` & enter password as `root`.
   3. Run the SQL scripts from [Murphy Movies repo](https://github.com/UCI-Chenli-teaching/cs122b-project5-murphy-movies?tab=readme-ov-file#prepare-the-database-murphymovies).
   4. Grant `murphyuser` privileges: `GRANT ALL PRIVILEGES ON * . * TO 'murphyuser'@'%';`
5. Enable ingress in your AWS cluster.
   1. Run the following to install ingress-nginx.
   ```
   helm upgrade --install ingress-nginx ingress-nginx \
      --repo https://kubernetes.github.io/ingress-nginx \
      --namespace ingress-nginx --create-namespace 
   ```
6. Deploy Redis service/pod for application session and state storage (defined in `murphy-movies.yaml`).
## Steps to deploy scripts
1. Clone the repo into your Ubuntu instance
   - Before applying `murphy-movies.yaml`, replace `<YOUR_DOCKERHUB_USERNAME>` in image names with your DockerHub username.
2. Run `kubectl apply -f murphy-movies.yaml`. 
   1. Test by running `kubectl get pods`, redis/login/star pods should be in `RUNNING` state and `READY (1/1)`. 
   2. If your pod is showing as `PENDING`, run `kubectl describe <POD_NAME>` and inspect the lifecycle of the pod to debug further. 
   3. Run `kubectl logs <MURPHY_MOVIES_POD_NAME>`. If you see JDBC connection exceptions, you haven't configured MySQL properly. Check the username, password, user permissions for your MySQL user, database name.
   4. If login/star shows Redis connection errors, verify `redis-service` is running (`kubectl get svc redis-service` and `kubectl get pods -l app=murphy-redis`).
   5. After fixing any errors, run `kubectl delete -f murphy-movies.yaml` and repeat Step 2.
3. Run `kubectl apply -f ingress.yaml`.
4. Run `kubectl get ingress` to see your list of ingresses.
5. You should see an `ADDRESS` after a couple of minutes. You can then access the application on http://<AWS_ELB_URL>/cs122b-project5-murphy-movies.
![img.png](img.png)
6. You can test shared Redis state by verifying Redis keys:
   1. `kubectl exec -it <redis-pod-name> -- redis-cli`
   2. `GET accessCount:anteater`
   3. `KEYS session:*`
## Viewing
In three different terminal command lines, ssh into your k8-admin instance. After doing so, run the following commands, each in different terminal sessions:

Viewing Redis Updates:
`kubectl exec -it "$(kubectl get pod -l app=murphy-redis -o jsonpath='{.items[0].metadata.name}')" -- redis-cli MONITOR | rg '"(GET|SET|INCR)"'`

Viewing Star 1:
`kubectl logs -f <murphy-star-pod-1-name> | rg "LoginTimeStamp|accessCount"`

Viewing Star 2:
`kubectl logs -f <murphy-star-pod-2-name> | rg "LoginTimeStamp|accessCount"`

You will see value accessCount increase once you log in and it increment each time you reload the page.
