# ci-project

## Components 
The goal of this project is build on a previous project to set up a continuous integration pipeline for an application that provides secure login functionality. This will be done using kubernetes, jenkins and docker-compose(previous project: https://github.com/wrusselly/ci-project/tree/jenkins). The app consists of 14 components: 
*	1 gateway: this uses nginx to route traffic to the other components.
*	2 Clients: these are the webpages written with VueJS.
*	10 Services: these are the APIs for the app written in nodeJS. 
*	1 Database: this app uses mongo as its database.

## Architecture
![Architecture map](https://github.com/wrusselly/ci-project/blob/master/CI-project.png)

Above is the architecture map for this project. Each component depends on any downstream components(e.g. email-service depends on mongo-service), so when building it must be done from the bottom up. Currently the role and group service are not connected to the rest of the application. All requests from the user pass through the gateway which then routes them to the appropriate place. 


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
**1.** Deploy jenkins (check resources)
**2.** Create a job to a) rebuild and push images b) update cluster deployments
