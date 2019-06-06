# CI-Project

## Components 
This project builds on a previous project (https://github.com/wrusselly/ci-project/tree/jenkins) to set up a continuous integration pipeline for an application that provides secure login functionality. This will be done using kubernetes, jenkins and docker-compose. The app consists of 14 components: 
*	1 gateway: this uses nginx to route traffic to the other components.
*	2 Clients: these are the webpages written with VueJS.
*	10 Services: these are the APIs for the app written in nodeJS. 
*	1 Database: this app uses mongo as its database.

## Architecture
![Architecture map](https://github.com/wrusselly/ci-project/blob/master/CI-project.png)

Above is the architecture map for this project. Each component depends on any downstream components(e.g. email-service depends on mongo-service), so when building it must be done from the bottom up. Currently the role and group service are not connected to the rest of the application. All requests from the user pass through the gateway which then routes them to the appropriate place. 

## Pipeline
![Pipeline image](https://www.draw.io/?lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=ci-pipeline.drawio#R5ZjbctowEIafxpftYIw5XCaQJmmblmkmk%2BRSWIutIHtdWQ7Qp69kZLAtMqRpIEzCBUgraSX9nw4rHG8YL84FSaMrpMCddosuHG%2FktNuuN%2BioH21Zriy9QXdlCAWjptLGcM3%2BgDG2jDVnFLJaRYnIJUvrxgCTBAJZsxEhcF6vNkVe7zUlIViG64Bw23rLqIyM1e0ONgUXwMLIdN1v91YFMSkrm5lkEaE4r5i8M8cbCkS5SsWLIXAtXqnLqt2XJ0rXAxOQyOc0GIf5j4uf%2FuXt8ubs9xUk9P7O%2F2S8PBKemwmfMxnlEzNkuSx1EJgnFLSrluOdziMm4TolgS6dK%2FLKFsmYq5yrksYpCAmLJ0frrjVQiwcwBimWqkrZoG9kM%2BvGN9l5BUKpbFTRv2tsxHAP15430qiEUecflGpbSo3gETimICyxFOpUJ%2FOYnwQShZJEi8HUsvpOJsDHmDHJMFFVJiglxqoC1wWnJJiFhdRD5Lqd8uZNi0%2FFxwlnoW4rsaE75pKzBIbrvdB6HRidnSy8Q6LoWCi%2BQjJjSfbmq7bjHtmq9e39PRwrw7d8AiIBqY5WdXbyPJNblvGh1ev3jky9rr3nMZiBOIYDsrt7Ux5WrJ4l1pPn4pTD4kRfzkoK1dokRwEnWcaCulRKDrG8M7IWmXud%2BeyX2dGiWjha1o49oNb13hBYxRNEhCB3XZI2iIrS%2FhahS5sATiR7rA9jm%2FqmhzEyNcA158ae6DfwZZiLAEyb6u3fcDOou3H9hp%2BVCJafYiWs5%2FzyxdHfz%2BJ4LuQ3g9d9HXrNk7HpZ8%2F0Bh%2BTnteqq94bvIxeMypo%2BtkzvbL3j4bP914HX%2FOmPTQ%2B%2B402zrNIx20RScIigpOocRbz0Xb1CnfaXRLrqCOZZOkaVoW4ijxknWcmBc6gfHgkmKiap1PGecNEzPMjUJih%2BrYp3yUxo1R3szUKqsdJzv8HQs2bzbUDof6WRdbeVxzk2i%2FFW5hEiDMNSrAwBKGZPZRvltYD2uHku8Hjubv5DA7Kx7P4%2FIJJzjjVLEhC9fTUBis2Fi3ife0yJsVWe6%2BY%2FM6xYbJf%2BZeGQQun%2BuIhgT4EKaQclzEUJ7aarvrOU0qkEufdwur1DgZLZTd%2FUa6utM0fvd7ZXw%3D%3D)

Above is the CI pipeline used for this project.

## Set-up 
### Prerequisites: 
* docker-compose.yaml from previous project (link above)
* images for each service pushed to docker hub
* GCP account 

### Kubernetes 
**1.** Set up a cluster in the GCP cloud shell. 
    
    gcloud container clusters create <cluster-name> --region <region>
    glcoud container clusters get-credentials <cluster-name> --region <region>
  
**2.** Clone the yaml files for the services (https://github.com/wrusselly/ci-project-k8s.git). These files have been split into a deployments and services folder with a a file in each for every component. The gateway has been kept seperate to be deployed last as it depends on multiple other components. 

**3.** Run the services folder using: `kubectl apply -f services/`, and the gateway service `kubectl apply -f gateway/gateway-service.yaml`.

**4.** Once the gateway service is up use the external ip of the load balancer and put it into the ACTIVATION_LINK enviroment variable in the auth-c-deployment.yaml

**5.** Run the services folder using: `kubectl apply -f deployments/`, and the gateway service `kubectl apply -f gateway/gateway-deployment.yaml`. You can now connect to the page by going to the load balancer ip followed by `/authentication/login`

### Jenkins 
**1.** Deploy jenkins by running the jenkins folder `kubectl apply -f jenkins/`

**2.** Create a job to:
    
   a. rebuild and push images using the previous project as the source code.
      
      docker-compose up –d 
      docker-compose push
    
   b. update cluster deployments using the following command for each component: 
      
      `kubectl --record deployment.apps/<deployment-name> set image deployment.v1.apps/<deployment-name> <image-name-name>=docker.io/wrusselly/<image-name>:${BUILD_NUMBER}`
  
  `BUILD_NUMBER` is a jenkins environment variable used to tag each new image. 
  
## Common Problems
### Jenkins pod won't start
may have insufficient CPUs

Solution:
* Add extra nodes with this command:
  `gcloud container clusters resize <cluster-name> --node-pool <default-pool> --num-nodes <number-of-nodes> --region <cluster-region>`
  
### Jenkins can't connect to docker daemon. 
  
Solution:
* Find the container ID and the Node its on `Kubectl describe pod <jenkins-pod>`
* SSH onto the node and connect to the jenkins container as the root user `docker exec -it -u root <container-id> bash`
* Find the group id for docker `ls -al /var/run/docker.sock`
* Delete the docker group `groupdel docker`
* Remake the group with the id `groupadd -g <id> docker`
* Add jenkins user to docker group `usermod -aG docker jenkins`
* exit and restart the container `docker restart <container-id>`
  
### Jenkins 403 crumb error 
  
Solution: 
* On the jenkins home page click manage jenkins
* click configure global security
* Under CSRF Protection tick "enable proxy compatability"
