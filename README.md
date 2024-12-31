# Create AWS EKS Cluster with a Node Group

## Project Overview
This project demonstrates the creation of an AWS EKS (Elastic Kubernetes Service) cluster and the deployment of a sample application. It focuses on:

1. Configuring necessary IAM roles.
2. Creating a VPC for worker nodes using a CloudFormation template.
3. Setting up an EKS cluster (Control Plane nodes).
4. Creating and attaching a Node Group to the EKS cluster for worker nodes.
5. Configuring auto-scaling for the worker nodes.
6. Deploying a sample NGINX application to the EKS cluster.

## Technologies Used
- **Kubernetes**
- **AWS EKS**

## Project Workflow

### 1. Create EKS IAM Role
AWS Identity and Access Management (IAM) roles are essential to manage access to the AWS resources required by the EKS cluster. Follow these steps to create the necessary role:

1. Log in to the AWS Management Console.
2. Navigate to the IAM service and select **Roles** from the sidebar.
3. Click **Create Role** to start creating a new IAM role.
4. In the **Select Trusted Entity** section, choose **EKS** as the service and select the **EKS - Cluster** use case.
5. Click **Next**. The system automatically selects the `AmazonEKSClusterPolicy`. Expand this policy to review the list of permissions it grants.
6. Click **Next** to proceed.
7. Provide a name for the role, such as `eks-cluster-role`.
8. Click **Create Role**. The `eks-cluster-role` is now created and ready to be used during EKS cluster creation.

Refer to [Setting up AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html) for additional information.

### 2. Create VPC for Worker Nodes
EKS clusters require specific networking configurations that include both Kubernetes-specific and AWS-specific networking rules. The default VPC is not optimized for this purpose. A new VPC is created for the worker nodes to ensure compatibility and security. Worker nodes require a firewall configuration to allow control plane nodes to connect and manage them effectively. Note that the control plane runs on an AWS-managed VPC, while worker nodes operate in the user-created VPC. 

**Best Practice:** Create a mixture of public and private subnets within the VPC.

Follow these steps to create the VPC:

1. Go to the AWS Management Console and navigate to the **CloudFormation** service.
2. Click **Create Stack**.
3. Copy the template link from the AWS EKS User Guide's page on [Creating a VPC for your Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html). Use the following S3 link for the VPC creation template with public and private subnets: [amazon-eks-vpc-private-subnets.yaml](https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml).
4. In the CloudFormation **Create Stack** page, select **Template is ready** and **Amazon S3 URL** as the source. Paste the copied link into the URL field.
5. Click **Next** and provide a stack name, e.g., `eks-worker-node-vpc-stack`.
6. Review the parameters for IP address ranges of subnets in the worker node network configuration. Adjust if needed.
7. Click **Next**, keep the default settings, and click **Next** again.
8. Review the summary page of the stack to be created, then click **Submit**.
9. Once the stack creation is complete, navigate to the **Outputs** tab of the stack to retrieve the details such as the security group, subnet IDs, and VPC ID.

The VPC for worker nodes is now ready, with all necessary configurations in place for seamless integration with the EKS cluster.

### 3. Create EKS Cluster (Control Plane Nodes)
The Control Plane for the EKS cluster can be set up through the AWS Management Console. Follow these steps:

1. Log in to the AWS Management Console and navigate to the **Amazon EKS** page.
2. Click **Add cluster** and select **Create cluster**.
3. Enter a cluster name, such as `eks-cluster-test`.
4. Verify the Kubernetes version and leave it at the default setting.
5. Ensure that the `eks-cluster-role` created earlier is selected in the **Cluster service role** field.
6. Click **Next** to proceed to networking configuration.
7. Select the VPC created in Step 2 (e.g., `eks-worker-node-vpc-stack-VPC`). Subnets will be automatically selected.
8. Select the security group related to the EKS worker node VPC.
9. For **Cluster endpoint access**, choose **Public and private**, and click **Next**.
10. Leave the default settings unselected for control plane logging and click **Next**.
11. In the **Select add-ons** page, ensure that the default add-ons (CoreDNS, kube-proxy, and Amazon VPC CNI) are selected, and click **Next**.
12. Keep the default versions for the add-ons and click **Next** again.
13. On the review page, confirm all configurations and click **Create**.

The EKS cluster will begin provisioning. Once completed, it will be ready for further configuration and Node Group attachment.

### 4. Connect kubectl with EKS Cluster
To interact with the cluster, configure `kubectl`:

1. Open a terminal and create a kubeconfig file for the newly created EKS cluster.
2. Review the default region using the following command to ensure it matches the region where the cluster was created:
    ```bash
    aws configure list
    ```
3. Run the following command to create the kubeconfig file locally:
    ```bash
    aws eks update-kubeconfig --name eks-cluster-test
    ```
4. The kubeconfig file will be located in the `.kube/config` folder. View the context entry for the cluster in this file and confirm that the current context points to the newly created cluster.

Run the following `kubectl` commands to verify the connection:

1. Check nodes in the cluster (output will indicate no resources as nodes aren't yet created):
    ```bash
    kubectl get nodes
    ```
    **Output:**
    ```
    No resources found
    ```
2. List namespaces in the cluster:
    ```bash
    kubectl get ns
    ```
    **Output:**
    ```
    default
    kube-node-lease
    kube-public
    kube-system
    ```
3. Get cluster information, including the API server endpoint:
    ```bash
    kubectl cluster-info
    ```

At this point, `kubectl` is successfully connected to the EKS cluster and is communicating with the API server in the control plane.

### 5. Create EC2 IAM Role for Node Group
Create an IAM role for the worker nodes with necessary policies by following these steps:

1. Go to the **IAM** page in the AWS Management Console.
2. Click **Roles** in the sidebar and select **Create Role**.
3. In the **Trusted Entity** section, ensure **AWS service** is selected.
4. Under **Use Case**, select **EC2** and click **Next**.
5. In the **Add Permissions** step, add the following policies:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEC2ContainerRegistryReadOnly`
   - `AmazonEKS_CNI_Policy`
6. Click **Next** to proceed.
7. Enter a name for the role, such as `eks-cluster-node-group-role`, and review the selected policies.
8. Click **Create Role**. The `eks-cluster-node-group-role` is now ready to be attached to the Node Group.

### 6. Create Node Group and Attach to EKS Cluster
Node Groups provide the computational power for workloads. Follow these steps to create and attach a Node Group to the EKS cluster:

1. Go to the EKS cluster `eks-cluster-test` in the AWS Management Console.
2. Click on **Compute** and then click **Add Node Group**.
3. Provide a name for the node group, e.g., `eks-node-group`.
4. Select the IAM role `eks-node-group-role` created in Step 5 for the **Node IAM role**.
5. Click **Next**.
6. On the **Node group compute configuration** page:
   - Leave the AMI type as the default `Amazon Linux 2`.
   - Set the capacity type to `On-Demand`.
   - Choose the instance type, e.g., `t3.small`.
   - Leave the disk size as the default `20 GiB`.
7. For the **Node group scaling configuration**, set the desired, minimum, and maximum nodes to `2`.
8. Leave the **Node group update configuration** at the default value of `1 node` and click **Next**.
9. Review the selected subnets and toggle on **Configure remote access to nodes**.
10. In **EC2 Key Pair**, select an existing key pair (e.g., `docker-server`) to enable SSH access to the instances.
11. Under **Allow remote access from**, select **All** and click **Next**.
12. Review the configuration and click **Create**.
13. Navigate to the EC2 console to verify that instances are being launched.
14. Return to the EKS cluster page to confirm the status of `eks-node-group` as **Active**.
15. Open a terminal and run the following command to verify the nodes:
    ```bash
    kubectl get nodes
    ```
    This will display the created nodes.

If additional compute capacity is required later, you can edit the node group scaling configuration from the EKS page. However, to save infrastructure costs, it is recommended to configure a cluster autoscaler, which will be addressed in the next step.

### 7. Configure Auto-Scaling
Auto-scaling ensures that the cluster scales up or down based on workloads. Follow these steps:

1. Go to the **IAM** page in the AWS Management Console.
2. Create a new policy with the following permissions:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "autoscaling:DescribeAutoScalingGroups",
                    "autoscaling:DescribeAutoScalingInstances",
                    "autoscaling:DescribeLaunchConfigurations",
                    "autoscaling:DescribeTags",
                    "autoscaling:SetDesiredCapacity",
                    "autoscaling:TerminateInstanceInAutoScalingGroup",
                    "ec2:DescribeLaunchTemplateVersions"
                ],
                "Resource": "*",
                "Effect": "Allow"
            }
        ]
    }
    ```
3. Click **Next**, specify the policy name as `node-group-autoscale-policy`, review the permissions, and click **Create policy**.
4. Attach the policy to the existing Node Group IAM role:
   - Go to **Roles** in IAM and select `eks-node-group-role`.
   - Click **Attach policies** under **Add permissions**.
   - Select `node-group-autoscale-policy` and click **Add permissions**.

5. Deploy the Cluster Autoscaler:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
    ```
6. Edit the Cluster Autoscaler deployment to configure it for the EKS cluster:
    ```bash
    kubectl edit deployment cluster-autoscaler -n kube-system
    ```
   - Add the following annotation:
     ```yaml
     cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
     ```
   - In the **specs**, configure the cluster name as `eks-cluster-test` and add these options:
     ```yaml
     - --balance-similar-node-groups
     - --skip-nodes-with-system-pods=false
     ```
   - Update the image version to match the Kubernetes version running in the cluster. Refer to the [Cluster Autoscaler Tags](https://github.com/kubernetes/autoscaler/tags) page for the relevant version.
7. Save the changes. You should see a message indicating the deployment has been edited:
    ```
    deployment.apps/cluster-autoscaler edited
    ```
8. Verify the Autoscaler pod is running:
    ```bash
    kubectl get pod -n kube-system
    ```
9. Check the pod logs to observe the scaling process:
    ```bash
    kubectl logs -f <cluster-autoscaler-pod-name> -n kube-system
    ```
10. To test Autoscaler functionality, reduce the desired and minimum number of nodes to `1` in the Node Group scaling configuration from the EKS page. Check the logs of the Autoscaler pod for events related to scaling down.

By configuring Autoscaler, you enable the EKS cluster to automatically adjust its compute capacity, optimizing cost and performance.

### 8. Deploy Application to EKS Cluster
Deploy a sample NGINX application to test the cluster setup. Steps include:

1. Create a configuration file (`nginx-config.yaml`) with the following content:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 80
      selector:
        app: nginx
    ```
2. Apply the configuration:
    ```bash
    kubectl apply -f nginx-config.yaml
    ```
3. Verify the pods and service:
    ```bash
    kubectl get pods
    kubectl get svc
    ```
4. Note the LoadBalancer endpoint in the output of `kubectl get svc`.
5. Open the AWS Management Console and navigate to the EC2 page.
6. Under **Load Balancers**, locate the LoadBalancer created for the NGINX application and verify the DNS endpoint matches the one displayed by `kubectl get svc nginx`.
7. Copy the LoadBalancer URL and paste it into a web browser to access the running NGINX application.

**Note:** The LoadBalancer is created in the public subnet.

To test autoscaling:
1. Increase the number of replicas for the NGINX deployment to 20:
    ```bash
    kubectl scale deployment nginx-deployment --replicas=20
    ```
2. Observe some pods in a pending state while the Cluster Autoscaler scales up the nodes.
3. Check the Autoscaler logs to monitor the scale-up process:
    ```bash
    kubectl logs -f <cluster-autoscaler-pod-name> -n kube-system
    ```
4. Verify the increased number of nodes and ensure all pods are in the running state:
    ```bash
    kubectl get nodes
    kubectl get pods
    ```


## Additional References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Kubernetes Autoscaler GitHub](https://github.com/kubernetes/autoscaler/tags)


