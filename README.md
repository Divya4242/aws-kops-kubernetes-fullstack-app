# Full-stack application deployed on Kubernetes using kops on AWS

This guide provides a step-by-step process to create a Kubernetes cluster on AWS using Kops. 
This project demonstrates a Kubernetes setup with a full-stack application consisting of a ReactJS frontend, NodeJS backend, and MongoDB database. The project also includes a robust monitoring system using Prometheus, Grafana, kube-state-metrics, and Alert Manager.

![kops](https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/abb76432-0b97-4f1a-8270-3428bcd4d7ed)

## Tech Stack

- **Frontend**: ReactJS (Deployment and Service)
- **Backend**: NodeJS (Deployment and Service)
- **Database**: MongoDB (StatefulSet for data persistence and Service)
- **Volumes**: 
  - PVC for MongoDB (AWS EBS gp2)
- **Monitoring**: 
  - Prometheus
  - Grafana
  - kube-state-metrics
  - Alert Manager
- **Security**: Ingress network policy for MongoDB to only allow traffic from backend pods


## Prerequisites

- An AWS account
- Basic knowledge of AWS and Kubernetes
- AWS CLI configured with appropriate permissions
- Domain managed in Route 53
- Prometheus and Grafana

## Getting Started

## Infrastructure Creation
**1. Launch an EC2 Instance**

  Launch an EC2 instance with the `t2.micro` instance type. You can either create a new instance or use an existing one. Ensure you have SSH access to the instance.

**2. Install Kops**

  SSH into your EC2 and install Kops. Follow the installation instructions from the official Kops documentation [here](https://kops.sigs.k8s.io/getting_started/install/).

**3. Update the System**

Update the system packages.
```sh
sudo apt update -y
```

**4. Create IAM Role and Attach it to EC2**

Create an IAM role with necessary permissions and programmatic access. Attaching **AdministratorAccess** is recommended for simplicity. Attach the IAM role to your EC2 instance.

**5. Create an S3 Bucket**

Create an S3 bucket which will be used by Kops to store cluster state information.

**6. Create a Hosted Zone in Route53**

Create a hosted zone in Route53. If you already have a hosted zone, you can skip this step.

**7. Install kubectl**

Install kubectl by following the instructions [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

**8. Create the Cluster**

Run the following command to create your cluster. Update the placeholders with your specific values. Also change the clsuter name that end with your hosted zone name. Example. kubernetes.myhosteszone.com, kubernetes.mydomain.com, etc... 
```sh
kops create cluster \
    --name=k8s-cluster.example.com \
    --state=s3://<your-s3-bucket> \
    --zones=<your-availability-zone> \
    --node-count=1 \
    --node-size=t2.medium \
    --control-plane-count=1 \
    --control-plane-size=t2.medium \
    --dns-zone=<your-domain>
```

**9. Update and Apply the Cluster Configuration**

The previous command generates a preview of all the resources Kops will create. If the preview looks good, run the following command to create the resources. (Same like terraform plan(8th step) and terraform apply(9th step))

```sh
kops update cluster --name=k8s-cluster.example.com--yes --admin --state=s3://<your-s3-bucket>
```
This process may take 5-10 minutes.

**10. Validate the Cluster**

Validate that the cluster is ready.

```sh
kops validate cluster --name=k8s-cluster.example.com --state=s3://<your-s3-bucket>
```

Here is the example of expected above command output:
![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/adc2d923-f7f0-44f2-a916-7f44a41bedad)

## Application Deployment

Once the cluster is ready, follow these steps to deploy the application:

* Clone the repository using the following command:
  ```sh
  git clone https://github.com/Divya4242/k8s-aws-kops-ingress.git
  ```

* Apply the application deployment, service and statefulset configuration:
  ```sh
  kubectl apply -f .
  ```

* Apply the ingress controller and resource configuration:
  ```sh
  cd ingress && kubectl apply -f .
  ```

below is the snapshot of expected above commands output:
![image](https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/c6f35918-1535-4b2d-aad0-f69febe937dd)


To check deployment status of the application, open browser and navigate to the following URLs:

    • <ec2-public-ip>:30006 # for the frontend
    • <ec2-public-ip>:30007 # for the backend
###

Until this, we've successfully configured a Kops cluster, deployed Kubernetes resources, and established Kubernetes services. 

**Add NGINX Ingress Controller:**

To expose your services using NGINX Ingress Controller, follow these steps:

1. Download the NGINX Ingress Controller manifest: (Optional)
    ```sh
    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
    ```

2. Go to ingress directory and edit the `nginx-ingress-controller.yaml` file and update the following configurations:
   - Replace `XXX.XXX.XXX/XX` with your VPC CIDR in use for the Kubernetes cluster under `proxy-real-ip-cidr`.
   - Replace `arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX` with your ACM certificate ID.

3. Deploy the NGINX Ingress Controller and NGINX INGRESS Resource:
    ```sh
    cd ingress && kubectl apply -f .
    ```

4. Verify that the Ingress Controller is deployed successfully by checking the Kubernetes namespace:
    ```sh
    kubectl get ns
    ```
You should see a namespace named `ingress-nginx` as output.

> [!TIP]
> The **NGINX Ingress Controller** is the runtime component that actively manages traffic, while the **NGINX Ingress Resource File** is a configuration artifact that defines the desired behavior for that traffic within the Kubernetes ecosystem. (same like we deploy nginx web server(nginx ingress controller) for serving content to user and reverse proxy and deployment.conf(nginx ingress resource file) in sites availiable that define the servername, listning port, etc.)
   
**Obtain Load Balancer DNS**
To find the DNS name of the Load Balancer:
* Run the following command and look for the EXTERNAL-IP of the ingress-nginx-controller service
  ```sh
  kubectl get svc -n ingress-nginx
  ```
![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/466978b1-5e99-488d-bf6f-35ee0468eafb)

* Alternatively, you can also find the Load Balancer DNS name in the AWS Management Console under EC2 -> Load Balancers.

**Configure DNS in Route 53:**
Add a CNAME record in Route 53 to point to the Load Balancer DNS name:

1. Go to Route 53 and select your hosted zone.
2. Add a new record:
   
    • Name: The record name specified in your ingress-resource.yaml file.

    • Type: CNAME.

    • Value: The DNS name of the AWS Load Balancer.

![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/23899d31-b70e-4634-9b5b-3a0efb14932b)

Cheers! 🎉 You can now access your deployed website.

Open your browser and navigate to your domain or the appropriate service URL. Your application should be live and accessible.

## Monitoring
- **Prometheus**: Collects metrics from the Kubernetes cluster and applications.
- **Grafana**: Visualizes the metrics collected by Prometheus.
- **kube-state-metrics**: Exposes cluster-level metrics.
- **Alert Manager**: Manages alerts sent by Prometheus. (For this demo, two alerts are configured)

      1. Node not ready for more than 5 minutes

      2. Node has less than 1 GB memory
      
### 1. Apply the monitoring deployment and service configuration:
```sh
cd monitoring && kubectl apply -f .
```

To check Montoring of the application, open Chrome and navigate to the following URLs:

    • <ec2-public-ip>:30010 # for the prometheus
    • <ec2-public-ip>:30013 # for the grafana
    • <ec2-public-ip>:30011 # for the kube-state-metrics

### 2. Setup Prometheus & Grafana

<img align="right" width="100" height="100" src="https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/9cbd9a78-d244-4dac-9d8a-e483bd78d54a">

* 1. Ensure all targets in Prometheus are up, as shown in the image. In the targets section, you can see there are two endpoints: one for `kube-state-metrics` which collects Kubernetes data, and another for Prometheus metrics. It is crucial to ensure that both targets are operational to maintain accurate and comprehensive monitoring of your Kubernetes cluster. Ensure all targets in prometheus are up as shown in the image.
   
<img align="right" width="100" height="100" src="https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/eae03bdd-990a-4168-8e19-767797168586">

* 2. Navigate to Grafana at `http://<ec2-public-ip>:30013` and add the Prometheus server URL in the connection settings as depicted in the image. Make sure to include the Prometheus URL with port number `30010`. After entering the URL, click the "Save & Test" button at the bottom of the Grafana page. Once the test is successful, you will see a message indicating that the connection is successful. This ensures that Grafana is properly linked to Prometheus for monitoring and data visualization.
 
<img align="right" width="100" height="100" src="https://github.com/user-attachments/assets/23422a22-6eed-4e9e-b779-9bd28dfdc33a">

* 3. Once you've successfully added the data source, proceed to Grafana's Dashboards section. Navigate to Import, where you'll enter Dashboard ID `13332` into the designated field. Click 'Load' to import the kube-state-metrics dashboard. This dashboard provides comprehensive visualizations of Kubernetes nodes, pods, deployments, and more, offering valuable insights into your cluster's performance and resource utilization.
     
## Dashboard Photos

|Kubernetes Cluster|Kubernetes Nodes|
|:-:|:-:|
|![Kubernetes Cluster](https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/e399a45d-746f-4d59-98fc-c8529c1f0b6d)|![Kubernetes Nodes](https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/163203a6-6b38-4f85-9ba0-8b25f3d07962)|

![image](https://github.com/Divya4242/k8s-AWS-Kops-Ingress/assets/113757574/ac3575b8-58ca-496e-a57e-0c5432890a43)


## Cleaning Up
To completely remove a Kops-managed Kubernetes cluster, execute the following command:
```sh
kops delete cluster k8s-cluster.example.com --state=s3://<your-s3-bucket> --yes
```

## Conclusion

By following these steps, you will have set up a Kubernetes cluster on AWS with Kops, deployed your application, and configured NGINX Ingress for external access via Route 53.

## Important Kubernetes and Kops Commands
1. **Get CRI details used in nodes**: `kubectl get nodes -o wide`
2. **List pods (controller, scheduler, api-server) running on control-plane**: `kubectl get pods -n kube-system`
3. **List the nodes present in the cluster**: `kubectl get nodes`
4. **Grab the name of the control-plane node and describe it**: `kubectl describe node master-node`
5. **Get the instance group list from the cluster**: `kops get ig --state=s3://<your-s3-bucket>`
6. **Edit the node-machine to add the number of instances**: `kops edit ig nodes-ap-south-1a --state=s3://<your-s3-bucket>`
7. **In the default namespace**: `kubectl get events`
8. **In all namespaces**: `kubectl get events --all-namespaces`
9. **Get logs of a pod**: `kubectl logs pod-name`
10. **Get the instance group list of Kubernetes Cluster**: `kops get ig --state=s3://<your-s3-bucket>`
11. **Add/remove an instance in Kubernetes Cluster**: `kops edit ig <instance-name> --state=s3://<your-s3-bucket>`

## Troubleshooting
If you encounter an error such as: 
> [!NOTE]
> error: You must be logged in to the server (the server has asked for the client to provide credentials)

Run the following command to refresh your Kubernetes configuration:
```sh
kops export kubecfg --admin --state=s3://<your-s3-bucket>
```
