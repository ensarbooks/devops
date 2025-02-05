# **AWS CI/CD for Microservices – An Advanced Project-Based Guide**

## **Introduction**  
In modern cloud architecture, **continuous integration and continuous delivery (CI/CD)** are essential for deploying microservices rapidly and reliably. This guide provides an in-depth, **step-by-step journey** through building a CI/CD pipeline on AWS for a microservices-based application. We will use a hands-on project to demonstrate how to automate code builds, deployments, and infrastructure provisioning using AWS services. The target audience is advanced users, so we’ll dive deep into each service’s role, best practices, and troubleshooting techniques. By the end, you will have built a fully operational CI/CD pipeline deploying containerized microservices to AWS, with monitoring, scaling, and high availability across multiple Availability Zones.

**What We Will Build:** A microservices application (for example, an e-commerce web service with separate microservices for user management and order processing) deployed on AWS. We’ll use **Amazon CodeCommit** for source control, **AWS CodePipeline** to orchestrate the CI/CD workflow, **AWS CodeBuild** to compile and containerize the application, and **Amazon Elastic Container Registry (ECR)** to store Docker images. The microservices will run on an **Amazon ECS Cluster (EC2 launch type)** behind a **Network Load Balancer (NLB)** for high availability. We will use **AWS CodeDeploy** to manage blue/green deployments on ECS, ensuring zero-downtime updates. An **Amazon Aurora (MySQL)** database will provide a scalable, multi-AZ data store. Throughout, we incorporate **AWS CloudWatch** for logging and monitoring, **Auto Scaling** for both ECS tasks and EC2 instances, and **IAM** for fine-grained security control. We’ll also address AWS billing considerations and cost optimization strategies for each component.

**Guide Structure:** We’ll start by setting up the foundational pieces (source repo, Docker images, and AWS environment). Then we’ll progressively build out the pipeline and the AWS infrastructure (ECS cluster, load balancer, database). Each section provides detailed steps and explanations, including best practices and common pitfalls to avoid. Along the way, we include troubleshooting tips for typical issues (for instance, what to do if a deployment fails or a container won’t start). Finally, we discuss how to optimize costs and ensure security throughout your AWS CI/CD workflow.

Let’s begin the journey by preparing our AWS environment and source code repository.

## **Architecture Overview and AWS Services Used**  
Before diving into implementation, it’s crucial to understand the **architecture** we’ll be creating and how each AWS service fits into the CI/CD pipeline for microservices:

- **Source Control – AWS CodeCommit:** We will store our application’s code in CodeCommit, a fully-managed Git repository service. CodeCommit securely hosts private Git repositories without needing to manage your own version control servers ([AWS CodeCommit Now Available](https://aws.amazon.com/about-aws/whats-new/2015/07/aws-codecommit-now-available/#:~:text=AWS%20CodeCommit%20is%20now%20available,with%20your%20existing%20Git%20tools))  This is where developers commit code for each microservice.

- **Build and Containerization – Docker, CodeBuild, and Amazon ECR:** Each microservice is packaged as a Docker container. AWS CodeBuild will fetch the code from CodeCommit, build the application (run tests if any), containerize it using Docker, and push the Docker image to Amazon ECR (Elastic Container Registry). ECR is a managed Docker registry for storing container images. We’ll create separate ECR repositories for each microservice. This ensures a consistent, repeatable build process as part of CI.

- **Continuous Delivery Pipeline – AWS CodePipeline:** CodePipeline automates the workflow from code commit to deployment. It’s a fully managed service that orchestrates various stages like source, build, and deploy for fast, reliable releases ([AWS CodePipeline is now available in AWS GovCloud (US-East) ](https://aws.amazon.com/about-aws/whats-new/2023/05/aws-codepipeline-govcloud-us-east/#:~:text=AWS%20CodePipeline%20is%20a%20fully,the%20release%20model%20you%20define))  Our pipeline will be triggered by code pushes to CodeCommit. It will then invoke CodeBuild to build/test, and finally deploy the new container image to the ECS cluster via CodeDeploy.

- **Deployment Orchestration – AWS CodeDeploy:** We use CodeDeploy (ECS blue/green mode) to handle deployments of the new version of our microservices on ECS. AWS CodeDeploy automates code deployments and can coordinate safe, zero-downtime releases ([Introducing AWS CodeDeploy](https://aws.amazon.com/about-aws/whats-new/2014/11/12/introducing-aws-codedeploy/#:~:text=AWS%20CodeDeploy%20is%20a%20service,one%20EC2%20instance%20or%20thousands))  In our case, CodeDeploy will manage creating a new task set on ECS, re-routing the load balancer to the new tasks, and terminating old tasks if health checks pass – a blue/green deployment strategy. This minimizes downtime and allows quick rollback if something goes wrong.

- **Compute and Container Orchestration – Amazon ECS (Elastic Container Service) with EC2 Launch Type:** The microservices will run on an ECS cluster backed by EC2 instances (we’ll use the EC2 launch type for ECS tasks, meaning containers run on a fleet of EC2 virtual machines). An **ECS Cluster** is essentially a logical grouping of EC2 instances (or Fargate capacity) where tasks run ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Amazon%20ECS%20cluster))  We will create an ECS cluster spanning two Availability Zones for high availability. Within the cluster, we’ll define **ECS Task Definitions** for each microservice (which specify container settings like the Docker image, CPU/memory, environment variables, and port mappings) and run them as an **ECS Service**. An ECS service ensures the specified number of task instances is always running and allows linking with a load balancer ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=An%20Amazon%20ECS%20service%20maintains,Elastic%20Container%20Service%20User%20Guide))  We will use the EC2 launch type (rather than Fargate) to meet the requirement of running Docker within EC2 instances and to have more control over the underlying servers.

- **Networking and Load Balancing – Amazon EC2 + Network Load Balancer:** The ECS cluster will be created in a VPC spanning two **Availability Zones (AZ 1 & AZ 2)** for redundancy. We will launch EC2 container instances in both AZs. For distributing traffic to microservices, we’ll set up an **Elastic Load Balancing** service. Specifically, we’ll use a **Network Load Balancer (NLB)**. The NLB operates at Layer 4 (connection level) and is capable of handling millions of requests per second with ultra-low latency ([Network Load Balancer | Elastic Load Balancing | Amazon Web Services](https://aws.amazon.com/elasticloadbalancing/network-load-balancer/#:~:text=Network%20Load%20Balancer%20operates%20at,ACM))  It provides a single front-end IP per Availability Zone and is ideal for TCP/UDP traffic or when extreme performance is needed. (Note: An **Application Load Balancer (ALB)** could also be used for HTTP-based microservices with advanced routing, and indeed AWS CodeDeploy for ECS supports both ALB and NLB ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Application%20Load%20Balancer%20or%20Network,Load%20Balancer))  We’ll proceed with NLB to meet the specified component list, but will note differences as needed.) The NLB will have listeners to route client requests to our ECS service’s tasks (via target groups). By spanning the NLB across two AZs with healthy targets in each, we achieve fault tolerance – if one AZ or instance goes down, traffic can still flow to the other AZ.

- **Database – Amazon Aurora (MySQL):** Our microservices might need a persistent data store. We will use Amazon Aurora MySQL, a managed relational database that is MySQL-compatible and designed for high performance and availability. Aurora automatically handles replication across multiple AZs and failover. In a multi-AZ Aurora deployment, data is replicated across at least three AZs, and the system can fail over to a replica in case the primary instance fails ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=On%20instance%20failure%2C%20Aurora%20uses,RDS%20Proxy%20to%20reduce%20failover))  We’ll set up an Aurora cluster with a primary instance (writer) and at least one replica (reader) in another AZ for high availability. This database can be used by one or more of the microservices (for example, an orders service might use it to store orders). We will also consider best practices like keeping the DB in private subnets and using IAM or Secrets Manager for credentials, which we’ll discuss later.

- **Monitoring and Logging – Amazon CloudWatch:** CloudWatch will be used to collect logs and metrics from our pipeline and running services. We will configure ECS to send container logs to CloudWatch Logs using the **awslogs** driver, so we can view application logs centrally ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=show%20the%20command%20output%20that,service%20in%20the%20Docker%20documentation))  CloudWatch also automatically collects key metrics from ECS (such as CPU and memory utilization of services) at 1-minute intervals ([Monitor Amazon ECS using CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-metrics.html#:~:text=You%20can%20monitor%20your%20Amazon,the%20Amazon%20CloudWatch%20User%20Guide))  and from other AWS services. We will set up CloudWatch Alarms (e.g., on high CPU or error rates) to demonstrate automated monitoring. CloudWatch Events (now Amazon EventBridge) can be used to trigger notifications or actions on pipeline or deployment events for troubleshooting and alerting.

- **Auto Scaling – ECS Services and EC2 Instances:** To ensure the system can handle varying load while optimizing cost, we’ll implement auto-scaling at two levels. **ECS Service Auto Scaling** will adjust the number of running task instances based on demand (for example, scale out when CPU usage is high, scale in when low). **EC2 Auto Scaling** will adjust the number of EC2 container instances in the cluster to ensure there’s enough capacity for tasks. We will use an Auto Scaling Group (ASG) for the ECS cluster’s EC2 instances. ECS can integrate with ASGs via **Capacity Providers** to scale the cluster dynamically—when tasks need more room, ECS can signal the ASG to launch instances, and scale in when excess capacity is idle ([Amazon ECS capacity providers for the EC2 launch type - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/asg-capacity-providers.html#:~:text=When%20you%20use%20Amazon%20EC2,to%20handle%20the%20application%20load))  This combination provides a self-adjusting system: the ECS Service scales tasks based on load, and the ASG scales EC2 infrastructure based on task requirements, keeping our microservices highly available and efficient. We’ll cover how to configure scaling policies and discuss **Availability Zone** spreading (ECS will try to place tasks evenly across AZs by default to improve resilience ([Amazon ECS availability best practices | Containers](https://aws.amazon.com/blogs/containers/amazon-ecs-availability-best-practices/#:~:text=ECS%20groups%20available%20capacity%20used,instances%20into%20ECS%20Clusters%20here)) .

- **Security – AWS IAM and Networking Best Practices:** Security is woven throughout each component. We will create **IAM Roles** to grant least-privilege permissions to the services. For example, CodePipeline’s role will allow it to invoke CodeBuild and CodeDeploy; CodeBuild’s role will allow pushing to ECR; ECS tasks will use an **ECS task execution role** to pull images from ECR and publish logs to CloudWatch ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=The%20task%20execution%20role%20grants,services%20associated%20with%20your%20account))  We’ll ensure that our IAM policies follow the principle of least privilege (grant only the minimum necessary permissions) ([AWS Identity and Access Management (IAM) Best Practices](https://aws.amazon.com/iam/resources/best-practices/#:~:text=the%20goal%20of%20achieving%20least,IAM%20provides%20last%20accessed))  On the networking side, we use **Security Groups** and VPC settings to control traffic: only the load balancer will be exposed to the internet, while ECS instances and Aurora DB will reside in private subnets. This layered approach (often called a **defense-in-depth** strategy) will be discussed in the best practices section. 

- **Cost Considerations:** As we proceed, we’ll note the cost implications of each service. For instance, CodeCommit is free for small teams (up to 5 active users per month are free, then $1 per additional user) ([Use Hosted Git Repositories from AWS CodeCommit with AWS Elastic Beanstalk](https://aws.amazon.com/about-aws/whats-new/2016/10/use-hosted-git-repositories-from-aws-codecommit-with-aws-elastic-beanstalk/#:~:text=AWS%20CodeCommit%20%20is%20a,to%20learn%20more%20about%20pricing))  and CodePipeline costs $1 per active pipeline per month (first pipeline is free in the Free Tier) ([Now Available – AWS CodePipeline | AWS News Blog](https://aws.amazon.com/blogs/aws/now-available-aws-codepipeline/#:~:text=You%E2%80%99ll%20pay%20%241%20per%20active,the%20course%20of%20a%20month))  We will cover pricing details in a dedicated section and suggest ways to minimize expenses (like using AWS Free Tier, shutting down resources in dev environments, auto-scaling to avoid over-provisioning, etc.).

With this high-level overview in mind, let’s start setting up the environment, beginning with source control and our codebase.

## **1. Setting Up Source Control with AWS CodeCommit**  
**Goal:** Create a CodeCommit repository to hold your microservices’ source code, and prepare the repository with initial code and configuration files (like a build spec and deployment instructions).

**Why CodeCommit?** As noted, AWS CodeCommit is a fully managed Git-based source control service. It eliminates the need to manage your own Git server and scales effortlessly. It securely stores anything from source code to binaries and works with standard Git tools ([AWS CodeCommit Now Available](https://aws.amazon.com/about-aws/whats-new/2015/07/aws-codecommit-now-available/#:~:text=AWS%20CodeCommit%20is%20now%20available,with%20your%20existing%20Git%20tools))  Using CodeCommit keeps our entire CI/CD workflow on AWS. (If you prefer, GitHub or Bitbucket could be used as well; CodePipeline integrates with them too. But here we’ll use CodeCommit for seamless AWS integration and to incorporate AWS developer tools best practices.)

**Steps:**

1. **Create a CodeCommit Repository:**  
   - Navigate to the **AWS CodeCommit console**, and click “Create repository.” Give it a name (e.g., `microservices-demo`). Optionally add a description and tags (tags are useful for cost allocation).  
   - Once created, note the **clone URL** of the repo (choose either HTTPS or SSH based on your preference and credentials). For HTTPS, you can use your IAM user credentials or set up Git credentials in IAM. For SSH, upload your SSH public key in IAM.

2. **Prepare Local Repository and Code:**  
   - On your development machine, initialize a Git repository for your project (or use an existing repo if you have one).  
   - Create a basic project structure. For a microservices project, you might have separate directories for each service. For example:  
     ```
     microservices-demo/
     ├── service-a/
     │   ├── src/...
     │   ├── Dockerfile
     │   └── buildspec.yml
     └── service-b/
         ├── src/...
         ├── Dockerfile
         └── buildspec.yml
     ```  
     Here we have two microservices (`service-a` and `service-b`). Each has its own source code and a Dockerfile to build the container image. We also include a **buildspec.yml** in each service directory – this file is used by CodeBuild to know how to build that service (we will configure CodeBuild to use these). Alternatively, you could have a single buildspec at root that builds all services sequentially. For simplicity, consider one service at first; you can extend to multiple services by replicating pipeline components.

   - Write a simple application for each microservice. For example, service A could be a Python Flask or Node.js Express app that connects to the Aurora database and provides some API, and service B could be another API or perhaps a worker service. The specifics of the application are flexible – the focus is on the pipeline and AWS infrastructure. Ensure each service can run in a container (e.g., listening on a configurable port, reading DB connection info from environment variables).

   - Create a Dockerfile for each service. A basic Dockerfile might use an official language runtime image, copy the source, install dependencies, and run the app. For example, a Node.js service Dockerfile:
     ```Dockerfile
     FROM node:18-alpine
     WORKDIR /app
     COPY package.json package-lock.json ./
     RUN npm install --only=production
     COPY . .
     EXPOSE 3000   # the port your app listens on
     CMD ["node", "server.js"]
     ```
     Modify for your application and confirm it runs correctly with `docker build` and `docker run` locally.

   - **Add AWS Config Files:** In addition to code, our repo needs a few configuration files for AWS deployment:
     - **Build Specification (buildspec.yml):** This tells CodeBuild how to build/test. It can be at the root or one per service. For a multi-service build, we might have separate CodeBuild projects or a single project that iterates through services. As an example, a buildspec for building the Docker image and pushing to ECR:
       ```yaml
       version: 0.2
       phases:
         pre_build:
           commands:
             - echo Logging in to Amazon ECR...
             - aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-west-2.amazonaws.com
             - REPO_URI=<aws_account_id>.dkr.ecr.us-west-2.amazonaws.com/microservices-demo/service-a
             - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7 || echo "latest")
         build:
           commands:
             - echo Building the Docker image...
             - docker build -t $REPO_URI:$IMAGE_TAG .
         post_build:
           commands:
             - echo Pushing the Docker image to ECR...
             - docker push $REPO_URI:$IMAGE_TAG
             - echo Writing image definitions file...
             - printf '[{"name":"service-a","imageUri":"%s"}]' "$REPO_URI:$IMAGE_TAG" > imagedefinitions.json
       artifacts:
         files: 
           - imagedefinitions.json
       ```  
       This example logs into ECR, builds the Docker image with a tag (using the Git commit ID as tag), pushes it, and writes an `imagedefinitions.json` artifact. That artifact is used by CodePipeline’s ECS deploy action or CodeDeploy to know which image to deploy. We’ll adjust names/IDs for your environment. (If you have multiple services, you might produce multiple image definition files or a combined one listing all images.)

     - **Task Definition JSON** (for CodeDeploy blue/green): If using CodeDeploy for ECS, we’ll need the ECS task definition and an AppSpec file in the repo (we will create these later in the CodeDeploy section). The CodeDeploy AppSpec file (`appspec.yaml`) will reference the task definition and container names. Keep in mind for standard ECS deploy (without CodeDeploy), CodePipeline can update the service with a new image automatically using the imagedefinitions file. In our guide, we’ll use CodeDeploy, so we’ll prepare those files soon.

   - With all necessary code and config in place, initialize git, add files (`git add .`), and commit (`git commit -m "Initial commit of microservices code and configs"`).

3. **Push to CodeCommit:**  
   - Add the CodeCommit remote:
     ```bash
     git remote add origin <codecommit_repo_URL>
     ```  
     Use the clone URL from earlier. For HTTPS, it will look like `https://git-codecommit.<region>.amazonaws.com/v1/repos/microservices-demo`. For SSH, it’s an SSH URL.
   - Push the main branch:
     ```bash
     git push -u origin main
     ```  
     (Ensure your branch name is ‘main’ or ‘master’ as appropriate and push accordingly.)

   - Verify in the AWS Console that CodeCommit now shows your repository files. You should see your application code, Dockerfiles, buildspec(s), etc.

**Branching Strategy:** For simplicity, we assume a single main branch triggering deployments. In a real-world scenario, you might have development, staging, and production branches with separate pipelines or pipeline stages for each environment. Advanced users can integrate CodePipeline with Git branching and AWS CodePipeline’s ability to detect changes in specific branches.

**Best Practices & Tips:**  
- *Use Git Ignore:* Ensure you have a `.gitignore` to avoid committing sensitive or unnecessary files (e.g., node_modules, secrets, etc.).  
- *Small Commits:* Commit changes in small chunks with clear messages. This will help in pinpointing issues if a certain commit breaks the pipeline.  
- *Code Reviews:* Use CodeCommit’s pull request and code review features for team collaboration. It supports commenting on diffs and approvals. While not a focus of this guide, maintaining code quality through peer reviews is important even in CI/CD.  
- *Security:* Avoid hardcoding secrets (DB passwords, API keys) in the code repo. We will later discuss using AWS Secrets Manager or Parameter Store to provide such secrets to the application via environment variables or ECS task definitions, instead of storing them in Git.

With our code repository ready, the next step is to set up the container registry and build pipeline so we can automate the building of Docker images and their deployment.

## **2. Creating an Amazon ECR Repository for Docker Images**  
**Goal:** Set up Amazon Elastic Container Registry (ECR) to store Docker images for our microservices. We will create ECR repositories and push the initial version of our microservice images.

**Why ECR?** ECR is a fully-managed Docker container registry. It’s integrated with AWS IAM, making it easy to control access, and with services like ECS, simplifying deployments. By using ECR, our CodeBuild process can push images to a repository that ECS and CodeDeploy can pull from securely (using IAM roles rather than public Docker Hub). It also helps avoid Docker Hub rate limits and provides vulnerability scanning for images. ECR has a minimal cost (a small charge for storage after a generous free tier and data transfer), and keeps your images close to your AWS infrastructure for fast deployment.

**Steps:**

1. **Create ECR Repository:**  
   - Go to the **Amazon ECR console** > “Repositories” > “Create repository”.  
   - Create a repository for each microservice. For example, make one called `microservices-demo/service-a` and another `microservices-demo/service-b` (you can use a namespace in ECR by using names with slashes). If you have just one service in this project, a single repo like `microservices-demo` is fine.  
   - For each repository, note the repository URI (it will be in the form `<aws_account_id>.dkr.ecr.<region>.amazonaws.com/your-repo-name`). We used this in the buildspec.  
   - Set the repository settings as needed. You can enable image scan on push (to detect vulnerabilities) and tag immutability (to prevent overwriting a tag, e.g., ensure version tags are unique). For CI/CD, an approach is to use unique image tags per build (like commit hash or build number) so you don’t usually need to force push tags. We recommend enabling scan on push for security. Keep “private” as the repository type (only your account can access by default, which is what we want).

2. **Push an Initial Image (Optional):**  
   - If you want to test ECR setup or pre-populate an image, you can manually build and push an image from your development machine:
     - Authenticate to ECR:
       ```bash
       aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
       ```
       This uses the AWS CLI to get a login token (ensure you have AWS CLI configured with credentials that have ECR access, e.g., your IAM user or role). After this, your Docker CLI can push to ECR.
     - Build and tag the image:
       ```bash
       docker build -t <aws_account_id>.dkr.ecr.<region>.amazonaws.com/microservices-demo/service-a:latest .
       ```  
       (Run this in the directory of service A’s Dockerfile; adjust repository URI accordingly. Tag “latest” is fine for initial push.)
     - Push the image:
       ```bash
       docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/microservices-demo/service-a:latest
       ```
     - Repeat for other service(s) if needed.

   - This step ensures everything is configured correctly. If the push succeeds, ECR now has your image. You should see it in the ECR console with the tag.

3. **Set Up Permissions for ECR:**  
   - ECR permissions are managed via IAM roles. Later, when we configure CodeBuild, we must ensure its role can push to ECR (we’ll attach an ECR write policy or use a managed policy for AmazonEC2ContainerRegistryPowerUser). Also, the ECS task execution role will need permission to pull from ECR. By default, if ECS tasks run in the same account, the **AmazonECSTaskExecutionRolePolicy** (managed policy) attached to the task execution role includes the permission to retrieve ECR images. This policy allows actions like `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, and `ecr:BatchGetImage` ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=,an%20Amazon%20ECR%20private%20repository))  which are necessary for ECS to pull images. We’ll cover IAM roles in detail soon.

   - If you stick to same-account deployments, the default ECR setup requires no custom repository policy. If you needed cross-account access, ECR repository policies or resource sharing would come into play (advanced scenario).

**Best Practices & Notes:**  
- *Tagging Strategy:* Use informative tags for images. In CI, using the Git commit SHA as the tag is popular (immutable and traceable). Also consider tagging “latest” for the most recent main build, or semantic versions if you do releases. Our pipeline will output an `imagedefinitions.json` with the exact image URI (including tag) for deployment.  
- *Retention Policy:* ECR allows setting lifecycle policies to clean up old images (e.g., keep only last N images or those used in running tasks). This helps save storage costs over time. Configure a lifecycle policy to avoid unbounded growth of images, especially if you push on every commit.  
- *Scanning:* Review the scan results in ECR after pushes. If critical vulnerabilities are found in base images or dependencies, address them (update base image, patch packages). Keeping images secure is an ongoing task.  
- *Repository per Microservice:* This is generally recommended (one repo per microservice image) to manage them independently. We’ve done that. This also allows separate scanning and permissions if needed.  
- *Consider AWS Copilot or CDK:* As an aside for advanced users, AWS offers tools like Copilot (CLI) or Cloud Development Kit (CDK) that can set up ECS services, ECR, and pipeline in a more automated fashion. In this guide, we focus on manual setup to explain each component, but know that these tools exist to simplify repetitive tasks once you understand the underlying processes.

Now that code and image repositories are ready, let’s move on to configuring the deployment environment: the ECS cluster, including the EC2 instances and networking.

## **3. Setting Up an ECS Cluster with EC2 Instances (EC2 Launch Type)**  
**Goal:** Create an Amazon ECS cluster that will host our microservice tasks using EC2 instances. We will provision EC2 instances (Container Instances) across two Availability Zones, set up an Auto Scaling Group for these instances, and ensure they are ready to run Docker containers.

**Understanding ECS Cluster (EC2 Launch Type):** An ECS cluster is essentially a pool of resources where you run your containerized tasks. For EC2 launch type, this means a pool of EC2 instances registered to the cluster, running the **ECS agent** and Docker. The cluster is a logical grouping – think of it as a namespace for your resources. ECS will place tasks on these instances. As AWS documentation states, *“An Amazon ECS cluster is a logical grouping of tasks or services”* ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Amazon%20ECS%20cluster))  and when using EC2, it’s also a group of container host instances. We want a cluster that spans multiple AZs for high availability. By having instances in AZ1 and AZ2, if one AZ faces an outage, the other can continue serving.

**Steps:**

1. **Create an ECS Cluster:**  
   - Go to the **Amazon ECS console**.  
   - Click “Clusters” > “Create Cluster”. Choose the **“EC2 Linux + Networking”** cluster template (since we want EC2 launch type).  
   - Configure the cluster:
     - **Cluster Name:** e.g., `microservices-cluster`.  
     - **Provisioning Model:** Choose On-Demand or Spot. On-Demand is safer (no interruptions), Spot is cheaper but instances can be terminated if capacity is needed elsewhere. For a production scenario, you might use On-Demand or a mix. We’ll assume On-Demand for now for reliability.
     - **EC2 Instance Type:** e.g., `t3.small` or `t3.medium` for a small cluster. Choose instance type based on your app’s needs (CPU/Memory). We can start small and allow auto scaling to add more if needed.
     - **Number of instances:** You could let it create some instances now (e.g., 2, one in each AZ). The wizard might ask for “Availability Zones” or subnet selection. Select two subnets in different AZs (the cluster template typically lets you pick a VPC and subnets; choose the default VPC or one you’ve set up, and ensure at least two subnets, each in a different AZ, are selected).
     - **Security Group for instances:** The wizard can create one or you specify an existing one. We can allow the default which typically opens port 22 (for SSH) to your IP. These instances themselves don’t need to be directly accessible publicly for our architecture (only the NLB will need to reach them on certain ports). However, for initial setup and debugging, having SSH or SSM access can be useful. We can refine security group rules later. For now, allow SSH from your IP and allow all outbound (the default for security groups).
     - **Key Pair:** If you want SSH access, choose an existing EC2 key pair or create one and select it. This will be needed if you plan to SSH into these container instances for troubleshooting.
     - **Container Instance IAM Role:** The wizard should list the **ecsInstanceRole** (Amazon ECS container instance IAM role). If it doesn’t exist, the wizard can create one. This IAM role (typically named `ecsInstanceRole` with AWS managed policy `AmazonEC2ContainerServiceforEC2Role`) allows the EC2 instances to register to ECS and, importantly, to send logs to CloudWatch and retrieve ECR credentials, etc. ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=Your%20Amazon%20ECS%20container%20instances,For%20information%20about))  Ensure this role is attached; it includes permissions like `ecs:RegisterContainerInstance`, `ecs:DiscoverPollEndpoint`, `logs:CreateLogStream`, `logs:PutLogEvents`, etc.
     - **EC2 AMI:** By default, the cluster creation will use the latest Amazon ECS-optimized AMI (Amazon Linux 2) for the instances. This AMI comes with the ECS agent and Docker pre-installed and configured. Use that for convenience. (Alternatively, AWS now also offers *Bottlerocket* for ECS or you can use Ubuntu with the ECS agent, but those are advanced scenarios. The optimized AMI simplifies things.)
     - **User Data:** The wizard will automatically include user-data script to join the cluster (it will pass the cluster name to the ECS agent on startup). You typically don’t need to modify it in the simple setup, but be aware the user data is how the instance knows which cluster to join.
     - **Instance Networking:** You might choose to place instances in **private subnets** (with no direct public IPs) if you want them isolated (then they’d need NAT Gateway for internet access to pull images/updates). Or place them in public subnets with public IPs if you need direct access. For better security, use private subnets if possible. In our setup, if the NLB is internal or the clients are within AWS, instances can be private. If NLB is internet-facing, it will still route to instances in private subnets via their private IPs, as long as the NLB is in the same VPC. We’ll assume private subnets for instances and that you have a NAT Gateway for them to reach the internet (for OS updates, pulling images, etc.). If using default VPC, by default its subnets are public, so you may end up with public instances – which is okay for learning, just ensure security groups are tight. In production, crafting a custom VPC with private subnets for ECS and public for NLB is recommended.

   - Create the cluster. AWS will launch the specified EC2 instances, and the ECS agent on those will register them to the cluster. After a few minutes, in the ECS console, the cluster should show e.g., 2 registered container instances. You can verify each instance’s status is “ACTIVE” in the cluster’s “Instances” tab.

2. **Verify ECS Agent and Docker on Instances:** (Optional, for insight)  
   - Connect to one of the EC2 instances (via SSH or Session Manager if you set it up).  
   - Run `sudo docker info` to ensure Docker is running. Also, `sudo systemctl status ecs` (or `ecs --version`). The ECS agent runs as a service and should be active and connected to your cluster.  
   - Check `/var/log/ecs/ecs-agent.log` for any errors. This log will show the agent registration.  
   - By default, the ECS-optimized AMI configures the Docker daemon and ECS agent with sane defaults, including the log driver to awslogs (the agent will default allow awslogs since Amazon Linux AMI includes that configured ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=)) .  
   - This manual step isn’t required but helps confirm that the infrastructure is correctly set up before deploying tasks. If the agent isn’t connected, the tasks won’t run.

3. **Auto Scaling Group (ASG) Setup:**  
   - If you used the ECS wizard, an Auto Scaling Group may have been created behind the scenes to manage these instances (especially if you specified a number). Check the **EC2 Auto Scaling** console – you might see an ASG named `EC2ContainerService-microservices-cluster-EcsInstanceAsg-...` or similar.  
   - The ASG ensures that if an instance goes down or is terminated, a new one launches to replace it, keeping the cluster at desired capacity. Initially, the desired, min, max might all be set to the number you launched (e.g., 2). You can adjust these values. Set the min to 2, desired 2, and perhaps max 4 or 6 to allow scaling out. We will later configure scaling policies, but for now, ensure the ASG spans both AZs (it should have the subnets for AZ1 and AZ2, and “Balance Across AZs” enabled by default).  
   - **Capacity Providers (optional advanced):** ECS offers capacity providers which link an ASG to the cluster for **Cluster Auto Scaling**. If you want the cluster to automatically add instances when tasks can’t be placed (and scale in when there’s excess capacity), you would create an ECS Capacity Provider for this ASG and enable managed scaling. This is somewhat advanced, but essentially, the capacity provider uses a target capacity metric to keep your cluster at, say, 75% utilization. If launching via the console, you might not have configured this yet. We can do without it and manually manage scaling policies: for example, add a CloudWatch alarm on high cluster CPU to increase desired count. But be aware that capacity providers exist and are recommended for production – they allow ECS to orchestrate the ASG scaling. We will mention how to set one up in the Auto Scaling section later.

4. **Multi-AZ Confirmation:**  
   - Ensure the two instances are in different Availability Zones. In the ECS console’s Instances list, it shows the AZ for each container instance (or check in EC2 console). If the wizard placed both in the same AZ (which can happen if subnets selection was not careful), it’s better to have at least one in a second AZ. You can add another instance manually via the ASG (increase desired capacity by 1 and specify the subnet/AZ). High availability best practice is to run compute in multiple AZs. In fact, AWS recommends using at least 3 AZs for critical services to handle an AZ outage with minimal over-provisioning ([Amazon ECS availability best practices | Containers](https://aws.amazon.com/blogs/containers/amazon-ecs-availability-best-practices/#:~:text=So%20what%20does%20all%20of,be%20in%20a%20position%20to))  ([Amazon ECS availability best practices | Containers](https://aws.amazon.com/blogs/containers/amazon-ecs-availability-best-practices/#:~:text=Having%20provisioned%20your%20ECS%20Cluster,to%20achieve%20your%20availability%20goals))  but 2 AZs is a common baseline for smaller setups. *“You should be ensuring that you have EC2 instances… in multiple Availability Zones”* ([Amazon ECS availability best practices | Containers](https://aws.amazon.com/blogs/containers/amazon-ecs-availability-best-practices/#:~:text=single%20Availability%20Zone,instances%20into%20ECS%20Clusters%20here))  so ECS can spread tasks across AZs.

5. **Container Instance Security Group & Ports:**  
   - By default, ECS tasks on EC2 use the EC2 instance’s network (unless you use awsvpc mode – we will use the default “bridge” networking for simplicity). This means the containers share the EC2 host’s security group. We will be running a web service on each container (say on port 3000 inside container). We will map it to some port on the host (ECS can assign a random high port by default with bridge networking, or you can use host networking or awsvpc for direct IP mapping). A simpler path: use an Application Load Balancer with dynamic port mapping – in that scenario ECS can assign a random port on the host and ALB finds it. However, with NLB, dynamic port mapping is not supported in the same way; NLB target groups with instance targets can use any port (you have to register the correct port). If we want to use NLB effectively, we might consider using the host port same as container port for simplicity, and a static service port. Alternatively, use awsvpc networking so each task gets its own elastic network interface and IP, and then NLB can target that IP+port (which *does* support dynamic port in a sense). For advanced realism, let’s do this:
     - Use **awsvpc network mode** in ECS task definitions (each task gets its own ENI and security group). This is now a common best practice as it isolates tasks at network level. It requires that your ECS instances have available ENI capacity (t3.small can handle a few ENIs, should be fine for a few tasks).
     - If using awsvpc, the **ECS service** will attach an ENI for each task in a **subnet** you specify (commonly the same subnets as instances). The NLB can have target type either instance or IP. To use awsvpc mode, we’d set NLB target type to “IP”, meaning it will register the task ENI IPs in the target group. That allows dynamic port mapping seamlessly because each task can use the same container port and NLB will target the IP:port of each task. (If this feels complex: an easier albeit less optimal approach is to use an ALB which directly supports ECS service integration with dynamic ports. But since NLB is mandated, we can handle it with IP targets.)
     - We will reflect this when defining the ECS Task and Service. Just note: if awsvpc mode, then the **security group to control container traffic** is not the EC2’s SG but the one attached to the ENI of the task. We’ll have to set that security group to allow traffic from the load balancer to the container port.
     - For now, ensure you have a security group ready for the tasks. We can create one e.g., `ecs-service-sg` that allows inbound from the NLB. We might not know the NLB SG or IP yet, so we’ll come back to this after creating the NLB. Keep this in mind as a TODO: security group rules for allowing NLB -> tasks traffic on the service port.

   - **If not using awsvpc** (bridge mode): then containers use the EC2 instance’s security group. In that case, you would open the service’s port on that security group for the NLB. We’ll assume awsvpc (more isolation). The rest of the guide will proceed with that assumption, adjusting as needed.

**At this stage**, we have an ECS cluster ready: EC2 instances are running and part of the cluster, Docker and the ECS agent are operational, and we have the capacity to deploy tasks. Next, we will set up IAM roles needed for the pipeline and services, then create our ECS Task Definitions and Services for the microservices.

## **4. Configuring IAM Roles and Policies for CI/CD and ECS**  
Proper IAM roles are crucial for our CI/CD pipeline and ECS tasks to function securely. We will set up the necessary roles and highlight what each does, adhering to the principle of least privilege (granting only the permissions required for each component to do its job) ([AWS Identity and Access Management (IAM) Best Practices](https://aws.amazon.com/iam/resources/best-practices/#:~:text=the%20goal%20of%20achieving%20least,IAM%20provides%20last%20accessed)) 

**Key IAM Roles and Entities in this project:**

- **CodePipeline Service Role:** Allows CodePipeline to orchestrate actions on your behalf.
- **CodeBuild Project Role:** Allows CodeBuild to access AWS resources (like pulling from CodeCommit, pushing to ECR, etc.) during build.
- **CodeDeploy Role:** Allows CodeDeploy to perform deployments to ECS (like launching new task sets, updating target groups).
- **ECS Task Execution Role:** Attached to each ECS Task Definition; allows the ECS agent (on the container instance) to pull container images from ECR and publish logs to CloudWatch for that task ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=The%20task%20execution%20role%20grants,services%20associated%20with%20your%20account))  ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=,an%20Amazon%20ECR%20private%20repository))  (Sometimes called the **ECS task execution IAM role**.)
- **ECS Task Role (Application role):** (Optional, if the app needs AWS API access) This role, if specified in task definition, is assumed by the application inside the container. For example, if your microservice needs to read from S3 or access Secrets Manager, you’d grant those permissions to this role. If not needed, you can omit a task role.
- **ECS Container Instance Role (ecsInstanceRole):** Attached to the EC2 instances in the cluster; already configured in previous step. It allows the ECS agent on EC2 to communicate with ECS and CloudWatch Logs, etc. ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=Your%20Amazon%20ECS%20container%20instances,For%20information%20about))  We won’t modify it beyond ensuring it exists.
- **CodeCommit Access for Developers:** If you have actual user accounts pushing to CodeCommit, ensure they have permissions (either via IAM user credentials or assumed roles). This is outside the pipeline scope but worth noting for completeness.

Now, let’s configure or verify each role:

1. **CodePipeline Service Role:**  
   - When creating a pipeline (which we’ll do in a later step), the console can create a service role for CodePipeline (usually named `AWSCodePipelineServiceRole-<pipeline-name>`). This role needs a policy that allows it to invoke the actions in the pipeline. For example, it must be able to trigger CodeBuild, CodeDeploy, pull artifacts from S3, etc. AWS provides a managed policy `AWSCodePipelineFullAccess`, but it’s broader than least privilege. There’s also a more restrictive AWS-managed policy `AWSCodePipelineRole` intended for service roles, which allows actions like `codebuild:StartBuild`, `codedeploy:CreateDeployment`, and various others needed for typical pipelines.  
   - We will let the console create the role with proper trust relationship (trusted by the CodePipeline service). After pipeline creation, verify the role’s policy. It should allow at minimum:
     - CodeCommit access to read the source (e.g., `codecommit:GetBranch`, `codecommit:GetRepository`, etc., on the repo).
     - CodeBuild to start builds (`codebuild:BatchGetBuilds`, `codebuild:StartBuild` on the build project).
     - CodeDeploy to create a deployment (`codedeploy:CreateDeployment`, etc., on the deployment group).
     - S3 access to the artifact bucket (read/write the pipeline artifacts).
     - CloudWatch Events to put events (for pipeline state changes).
     - IAM pass role permissions if needed (for passing roles to CodeBuild or CodeDeploy actions – the pipeline will need to pass the CodeBuild IAM role to CodeBuild). Often a policy like `iam:PassRole` with resource constraint to the CodeBuild role is necessary so that pipeline can trigger CodeBuild with its role.

   - We’ll refine this if needed when setting up the pipeline. The main idea is to ensure the pipeline’s role can do all the actions defined in the pipeline stages.

2. **CodeBuild Project Role (Build IAM Role):**  
   - Create an IAM role for CodeBuild (if not already). This role will be assumed by CodeBuild container during the build. The console usually calls it something like `codebuild-microservices-build-project-service-role`. Attach policies:
     - **AWSCodeBuildDeveloperAccess** managed policy (allows CodeBuild to interact with other services commonly needed). However, we need some additional rights:
     - ECR push/pull: You can attach `AmazonEC2ContainerRegistryPowerUser` or craft a policy for just needed actions: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, etc. The PowerUser policy is broad (pull/push to all ECR in the account) – that’s usually acceptable in a single-account CI context.
     - CodeCommit read: CodeBuild will get the source via CodePipeline’s artifact, so direct CodeCommit access might not be needed (CodePipeline fetches the source and passes to CodeBuild). If CodeBuild needed to pull additional source or submodules, ensure it can access those (e.g., if you have Git submodules in CodeCommit, you’d need CodeBuild role to auth to CodeCommit; there’s a specific policy for CodeCommit access via Git).
     - S3: If CodeBuild needs to upload/download artifacts to S3 (in our buildspec, we output to artifact which CodePipeline will put to S3), CodePipeline’s role covers moving that artifact. CodeBuild’s role may need to write to `$CODEBUILD_SRC_DIR` etc., but usually CodePipeline handles S3.
     - CloudWatch Logs: CodeBuild manages its own logs to CloudWatch (the CodeBuild service does it if enabled). The role might need `logs:CreateLogGroup`/`logs:CreateLogStream`/`logs:PutLogEvents` if not granted implicitly. The managed policy likely covers it.
     - Systems Manager: If you want to use SSM Parameter Store in buildspec to retrieve secrets, give `ssm:GetParameter` as needed (just an example of additional perms).
     - Keep this role minimal but functional. AWS’s CodeBuild documentation lists minimal required permissions for certain actions – since our build involves Docker, ensure the role allows Docker within CodeBuild (the CodeBuild inside container has necessary Docker privileges by selecting a privileged build image in project settings, not an IAM aspect).

3. **CodeDeploy Role (for ECS deployments):**  
   - AWS CodeDeploy for ECS requires a service role that it uses to actually perform the deployment actions. This role must be trusted by CodeDeploy. Create an IAM role named e.g. `CodeDeployECSServiceRole`.  
   - Attach the AWS-managed policy **AWSCodeDeployRoleForECS** (or if not available, use **AWSCodeDeployRole** which covers EC2/on-prem and Lambda too). Specifically for ECS, the role needs to manipulate: ECS services/tasks, Elastic Load Balancing, CloudWatch alarms (if used), etc. The managed policy should include things like:
     - `ecs:DescribeServices`, `ecs:UpdateService`, `ecs:CreateTaskSet`, `ecs:DeleteTaskSet`, `ecs:DescribeTaskSets` – CodeDeploy uses these to create new task sets for blue/green.
     - `elasticloadbalancing:DescribeTargetGroups`, `elasticloadbalancing:ModifyTargetGroup`, `elasticloadbalancing:DescribeListeners`, `elasticloadbalancing:ModifyListener` – to shift traffic between target groups.
     - Possibly permissions to deregister/register targets in target group.
     - `codedeploy:*` for its own stuff (maybe not needed if this is the role itself).
     - If using CloudWatch alarms for deployment stop, then access to those.
     - The **AWSCodeDeployRoleForECS** managed policy (if your account has it) is ideal because it’s maintained by AWS for exactly this purpose. Attach it to the role.  
   - When we set up CodeDeploy (in a later section), we will specify this role as the Service Role for the CodeDeploy application.

4. **ECS Task Execution Role:**  
   - AWS suggests a role named `ecsTaskExecutionRole` for this. If you used the ECS console to create a Task Definition, it often can create this role automatically. Check IAM for a role called `ecsTaskExecutionRole`.  
   - If not present, create a new role with trust entity **ecs-tasks.amazonaws.com** (trusting ECS to assume it for tasks). Attach the managed policy **AmazonECSTaskExecutionRolePolicy** ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=Amazon%20ECS%20provides%20the%20managed,role%20for%20special%20use%20cases))  This policy covers:
     - ECR read access (pull images) ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=,an%20Amazon%20ECR%20private%20repository)) 
     - CloudWatch Logs write access (create log stream, put log events) ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=Your%20Amazon%20ECS%20container%20instances,For%20information%20about)) 
     - If using Secrets Manager or Parameter Store in task definitions, it also covers those (the policy has conditional access for decrypting secrets that are attached to tasks).
     - This role does **not** grant the app in the container any access to other AWS services; it’s purely for the ECS agent actions. The containers themselves cannot assume this execution role for AWS API calls ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=))  That’s where the next role comes in if needed.

5. **ECS Task Role (Application Role):**  
   - If your microservice needs AWS API access (say to read an S3 file, or fetch a secret from AWS Secrets Manager at runtime), you assign a Task Role to the task definition. This role is assumed by the container’s process via the metadata service. In our case, maybe the app doesn’t call AWS services directly except Aurora MySQL (which is not an AWS API but a DB connection, so IAM not needed unless using IAM DB authentication – out of scope here). If not needed, you can leave the task role as None in the task definition. We mention it for completeness. If using it, you’d create a role with trust `ecs-tasks.amazonaws.com` and attach policies needed for the app (e.g., read specific S3 bucket, etc.). Best practice: scope it narrowly (only specific resources). 
   - We won’t emphasize further since our example app’s main external interaction is the database (which uses username/password auth).

6. **Verify CodeCommit Access for Developers:**  
   - Ensure your IAM user or whatever you use to push code has permissions for CodeCommit. For example, attach policy AWSCodeCommitFullAccess or a repo-specific policy. Also set up HTTPS Git credentials or SSH key for authentication. This is just to avoid a situation where you cannot push or your pipeline cannot fetch code. In a straightforward setup, the pipeline’s source stage will handle pulling code using its role and an AWS internal mechanism, so as long as pipeline role can access CodeCommit repo, we’re fine on CI side.

**Best Practices & Notes:**  
- *Least Privilege:* Try to avoid attaching overly broad policies. For instance, instead of AWSCodePipelineFullAccess (which allows management of pipelines too), use the specific service role policies. Use IAM **PassRole** carefully – your CodePipeline role might need to pass the CodeDeploy or CodeBuild roles to those services, so include an `iam:PassRole` for each with resource limited to those role ARNs.  
- *Secrets:* If storing database password in Secrets Manager, ensure the task role has decrypt permission and the execution role allows retrieving it. But a simpler approach is to inject the DB credentials via ECS environment variables (not plaintext in template, but from SSM or Secrets) – that’s beyond this guide’s scope, but be mindful of secret management.
- *Isolation:* Use separate roles for separate purposes. Do not reuse the same role for multiple services if not necessary. This way, if one component is compromised, it only has limited access. For example, CodeBuild’s role doesn’t need deployment permissions, and CodeDeploy’s role doesn’t need CodeCommit access, etc.
- *Access Analyzer:* Advanced tip: AWS IAM Access Analyzer can review your policies and point out if something is too broad or not used. After setting up, consider reviewing the IAM policies for opportunities to tighten them.
- *AWS Managed Policies:* We leveraged managed policies (like `AmazonECSTaskExecutionRolePolicy`, `AWSCodeDeployRoleForECS`). These are vetted by AWS and updated if services add new API calls, reducing maintenance. However, always check what each managed policy allows to ensure it’s appropriate.

With IAM roles configured, we can now proceed to define our ECS Task Definition and Service for our microservice, which will tie together the container image, cluster, load balancer, and IAM roles into a running service.

## **5. Defining ECS Task Definitions and Services for Microservices**  
In this section, we will create an **ECS Task Definition** for our microservice(s) and then create an **ECS Service** to run tasks from that definition on our cluster. We’ll integrate the service with the Network Load Balancer for traffic routing and prepare it for CodeDeploy deployments.

**Task Definition:** This is a blueprint for our containerized task. It includes which Docker image to run, how much CPU/memory to allocate, which port the container uses, and what IAM role and logging config to apply. Think of it as the "recipe" for a container task. We will create a task definition for each microservice (though if they’re identical in structure, one could templatize it – but likely each has a different container image and maybe different env vars, so separate definitions).

**Service:** An ECS Service ensures that a specified number of tasks (instances of the task definition) run continuously. It also allows deploying new versions (task definition updates) in a controlled way (e.g., rolling update or blue/green if used with CodeDeploy). The service will be linked to our load balancer’s target group so that as tasks come and go, they register for traffic.

Let’s proceed with **Service A** as our example (repeat for Service B if you have multiple):

1. **Create a Task Definition (for EC2 launch type):**  
   - In the **ECS console**, go to Task Definitions > “Create new Task Definition”. Choose **EC2** launch type (since we use EC2 instances).  
   - **Task Definition Name:** e.g., `service-a-taskdef`.  
   - **Task Role:** leave blank or None (unless your app needs to call AWS APIs, as discussed).  
   - **Network Mode:** select **awsvpc**. This is important – it means each task will get its own network interface and IP. (If you prefer to keep it simple and use the EC2 host network, you could choose bridge; but awsvpc is recommended for production and works well with load balancers. We proceed with awsvpc.)  
   - **Task Execution Role:** choose `ecsTaskExecutionRole` (the role we set up that allows pulling from ECR and logging).  
   - **Container Definitions:** Add a container for your microservice:  
     - Container name: e.g., `service-a`.  
     - Image: put the ECR image URI for the current version. Since we will deploy via CodeDeploy, initially you could put the `:latest` tag we pushed. For example, `<account>.dkr.ecr.us-west-2.amazonaws.com/microservices-demo/service-a:latest`. (CodeDeploy will later override this with the new image as specified in AppSpec during deployments.)  
     - Memory and CPU: allocate appropriate resources. For instance, 256 MB memory (hard limit) and perhaps 128 CPU units (which is 0.125 vCPU) for a lightweight service. These should align with your cluster EC2 instance capacity and how many tasks per instance you expect. If unsure, start small — you can adjust later. Just ensure the total CPU/Memory of all tasks can fit on your EC2 instances.  
     - Port Mappings: since we use awsvpc, container port mapping is straightforward. Enter **Container port** = `3000` (or whatever port your app listens on inside the container). For awsvpc mode, you **do not** need to specify a host port; it will be the same as container port on the task’s ENI by default. (In the console, they might still show a host port field but usually it’s disabled in awsvpc mode). This container port (3000 in example) is what the NLB target group will target.  
     - **Health Check (container level):** You can define a health check command for the container (e.g., for a web service, you might use `CMD-SHELL curl -f http://localhost:3000/health || exit 1`). If your application exposes a health endpoint, set this up. ECS will use it to decide if the container is healthy. This is separate from the NLB health check, but complementary. Container health checks help ECS stop/restart unhealthy containers. Set an appropriate interval and timeout. This is optional but recommended if your app can self-report health.
     - **Environment Variables:** Add any needed env vars, such as database connection string or credentials. For example, `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`. Since this is sensitive (password), you might use ECS’s integration with Secrets Manager or SSM Parameter Store: instead of putting plaintext, you can refer to a secure value. The task definition allows specifying `valueFrom` with a secret ARN if you have one. If not, and for simplicity, you might just put the credentials here (not best practice for production). At minimum, point to the Aurora endpoint and DB name. E.g., `DB_HOST = mydbcluster.cluster-abcdef.us-west-2.rds.amazonaws.com`, `DB_USER = appuser`, `DB_PASSWORD = (your password)`. We’ll assume you have created the DB and user which we do in the next section, so you can come back to fill this or update task def later when DB is ready. Alternatively, skip DB connectivity for now and just have a placeholder or a service that doesn’t need env vars.
     - **Logging:** Enable CloudWatch Logs. In the container definition’s “Log configuration”, choose log driver **awslogs**. Then set: 
       - Log Group = e.g., `/ecs/microservices-demo` (you can create a new log group for your app).  
       - Log Region = your region.  
       - Log Stream Prefix = `service-a`.  
       - (The ECS agent (via task execution role) will create log streams for each task with this prefix.)  
       This ensures all stdout/stderr from the application container goes to CloudWatch Logs. As AWS notes, the awslogs driver will send whatever your app prints to stdout/err into CloudWatch ([Send Amazon ECS logs to CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html#:~:text=show%20the%20command%20output%20that,service%20in%20the%20Docker%20documentation))  This is essential for troubleshooting later.
     - Leave other fields default (Storage and other advanced settings usually can remain default unless you need specialized config).

   - You can add definitions for additional containers if a task has multiple containers (sidecars), but in a microservice usually one container per task is typical. (An example of multiple could be a log router sidecar, but not needed here.)
   - Click **Create**. The task definition (a specific revision) is now saved. ECS will show e.g., `service-a-taskdef:1` as the revision.

   - **Repeat** for Service B if applicable (or any other microservice). Or if you prefer a single task def that runs both A and B containers together (less common in microservices, since you usually separate them), you could combine. We assume separate services, each with its own ECS service.

2. **Create a Target Group for the Service (NLB):**  
   Before creating the ECS service, set up the load balancer resources it will use:
   - Go to the **EC2 Console** > Load Balancing > Target Groups.  
   - Click “Create target group”. Choose **Instances** or **IP** as target type depending on our approach:
     - If using **awsvpc**, each task gets its own IP. We can use target type “IP” which means the target will be the task ENI’s IP. However, using “Instances” can also work if we register the instance on a static port. But since tasks might use ephemeral host ports if not awsvpc, it's tricky. We’ll proceed with **IP** target type (which is ideal for awsvpc). Note: IP target type requires that the tasks use subnets in the same VPC (they will) and that the NLB is also in that VPC (we will create NLB accordingly).
   - **Target Group settings:**  
     - Name: e.g., `tg-service-a`.  
     - Protocol: choose whatever your service uses, e.g., HTTP or TCP. If it’s a web API on HTTP, you could use HTTP. But since we are using NLB (Layer 4), if we choose HTTP here, NLB will still forward at TCP level without inspecting. Actually, for NLB you often choose TCP as the protocol for target group if you want raw TCP pass-through. NLB can do basic health checks at TCP level or HTTP if you specify. To keep it simple, we can choose **TCP** as protocol on, say, port 3000 (or whatever container port). However, if we want the NLB health check to call an HTTP endpoint on the container, we can choose HTTP protocol for the target group with the port and provide a health check path. NLB supports HTTP health checks even if it’s just doing TCP proxy. Let’s do: **Port** = 3000, **Protocol** = HTTP, and in health check settings give a path (e.g., `/health` if your app has one). If you don’t have a health endpoint, use TCP health checks (which just checks if port is open). But HTTP gives deeper insight if app responds 200 OK.
     - VPC: select your VPC (same as ECS cluster). Then for “IP address target type”, it asks for target group’s associated subnet or none? Actually, for IP type, you will later register IPs. We don’t need to register targets now (that will be done by CodeDeploy or ECS service).
     - Health check: set protocol HTTP and path `/health` (or `/` if that returns 200 when app is healthy). If using TCP health check, it’s just a basic connection check. Health check interval and thresholds – use defaults or adjust (maybe 10s interval, 3 unhealthy threshold, 3 healthy threshold for reasonably quick detection).
     - Create the target group.

   - **Repeat** for each service if multiple (each service should ideally have its own target group so we can route traffic separately).

3. **Create the Network Load Balancer:**  
   - Now create the NLB that will use these target groups. Go to Load Balancers > Create Load Balancer > **Network Load Balancer**.  
   - Name: e.g., `microservices-nlb`.  
   - Scheme: choose **Internet-facing** if this is to serve public clients (most likely, yes, for a public-facing microservice API). If internal (service-to-service calls within VPC), you could choose internal, but we assume at least one service is client-facing.  
   - IP address type: IPv4 is fine (unless you need IPv6).  
   - Network Mapping: select the VPC and **two subnets** (AZ1 and AZ2) where it will deploy load balancer nodes. Choose public subnets if internet-facing (these subnets should have an Internet Gateway). By having two AZs, NLB will have one static IP per AZ by default ([Network Load Balancer | Elastic Load Balancing | Amazon Web Services](https://aws.amazon.com/elasticloadbalancing/network-load-balancer/#:~:text=within%20Amazon%20VPC%2C%20based%20on,ACM))  and will route to targets in those AZs if available.  
   - Listener configuration: by default it might add one listener on TCP port 80. You can modify it: perhaps we want to use port 80 (HTTP) or 443 (HTTPS) for clients:
     - If it’s a web API, you might use TCP 80 for now (and later add TCP 443 with TLS if needed). NLB can do TLS termination if you attach a certificate, but often ALB is used for that. NLB can pass TLS through or terminate if configured with an TLS listener. For this guide, we can keep it simple: add a listener on **TCP port 80**. 
     - Select the target group for the service to attach. If we have only one service behind this LB, use that target group for port 80. If you plan to expose multiple services on different ports, you can add more listeners or use one listener with different path (but NLB doesn’t do path-based routing, that’s ALB’s job). With NLB, one approach for multiple services is to use different listener ports (e.g., port 80 -> service A TG, port 81 -> service B TG), or use one target group and handle routing in app. But typically, for microservices, an ALB with host/path routing is easier. Since NLB is required, maybe we treat each service as a separate NLB or separate port.
     - For demonstration, we’ll assume one service for simplicity or that all traffic goes to one entrypoint service. If multiple needed: you can add another listener now or later (e.g., TCP 81 -> target group B).
   - Security Groups: NLB (being layer 4) doesn’t use security groups (only ALB does). So nothing to configure here; it will allow all traffic on the listener ports to the targets.
   - Tags: add any tags if needed.  
   - Create the NLB. Note the DNS name it gives (something like `microservices-nlb-xxxx.elb.amazonaws.com`). We’ll use that to test the service later. Also note the **Static IPs** allocated per AZ – if needed, you can associate Elastic IPs to them for fixed addresses (not required unless you want known IPs).

   - After creation, the NLB will start doing health checks on the target group. But currently, we have no targets (tasks) running yet, so health will fail. That’s okay; once we run the service tasks, they will register and NLB will see them.

4. **Adjust Security for NLB and ECS Tasks:**  
   - If using awsvpc mode (task ENIs): We need to set a security group on the tasks that allows NLB traffic. When we create the ECS service, we will specify a security group for the tasks’ ENIs. Create a security group now for the service tasks, say `sg-service-a`. In that SG:
     - Allow inbound traffic **from the NLB** on the container port (3000). NLB’s traffic comes from the NLB nodes which have static IPs in subnets. The simplest way: allow inbound from the security group that the NLB attaches. But NLB doesn't have an SG (only ALB does). Instead, for NLB, the source will be the NLB nodes’ IP addresses. You could restrict by source to the VPC CIDR (assuming only LB will call tasks) or the subnets of the NLB. A more open but easy way: allow inbound from anywhere on port 3000 (since ideally only NLB is publicly reachable, tasks are in private subnets). However, to be safe, we can do: source = the two subnet CIDR blocks of the NLB (the LB node subnets). Or if the NLB is in the same VPC, source could be the VPC’s CIDR (which covers anything in VPC including LB).
     - Outbound: tasks’ SG can allow all outbound (default SG behavior), so the containers can reach the DB or external services.
   - If using EC2 bridge networking: Then the EC2 instance SG must allow the NLB’s traffic. That SG we likely configured at cluster setup (e.g., allowing port 80 or 3000 from anywhere or LB). But we’ll stick to awsvpc approach as it’s cleaner.
   - Ensure the Aurora DB’s SG allows connections from the ECS tasks (we’ll detail in DB section, but keep in mind).

5. **Create the ECS Service:**  
   - In **ECS console**, go to your cluster > Services > Create.  
   - Launch Type: EC2.  
   - Task Definition: select the task def for service A and the latest revision.  
   - Service Name: e.g., `service-a`.  
   - Number of tasks: start with 2 (so one in each AZ ideally, assuming cluster capacity). The service scheduler will try to place them distinct (it by default uses AZ spread placement ([Amazon ECS availability best practices | Containers](https://aws.amazon.com/blogs/containers/amazon-ecs-availability-best-practices/#:~:text=ECS%20supports%20specifying%20a%20placement,use%20an%20Availability%20Zone%20spread)) .  
   - Deployment type: choose **Blue/Green (powered by AWS CodeDeploy)**. This is important if we plan to use CodeDeploy for zero-downtime deploys. If you choose blue/green, ECS will integrate with CodeDeploy. Alternatively, you could choose “Rolling update” (ECS default) which does in-place update with some control of min/max tasks. But since we want to include CodeDeploy in our pipeline, select Blue/Green. This will prompt you for additional details:
     - It will ask for the CodeDeploy application and deployment group, etc. If not created yet, it can create placeholders. Actually, let's do this: we will create a CodeDeploy application after the service, but we can still fill some values.
     - You will need to specify the **load balancer details** here. Choose **Load balancer type: Application/Network Load Balancer**. Then select the NLB we created. For the listener, choose the listener port (80). Then for production listener target group, select the target group we created (`tg-service-a`). For test listener, if doing blue/green, you should have a second target group (and maybe second listener) to host the “green” tasks before cutover. The CodePipeline ECS-to-CodeDeploy tutorial uses two listeners (one for prod, one for test) on ALB ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Application%20Load%20Balancer%20or%20Network,Load%20Balancer))  With NLB, we might simulate similarly by having, say, port 80 as prod listener and port 81 as test listener (with separate target groups).
     - If we did not set up a second listener/group, an alternative CodeDeploy approach is traffic mirroring using the same listener but toggling target group registrations. However, CodeDeploy Blue/Green for ECS expects two target groups. So, let’s set it up: Add another **listener** on NLB on a different port for test (e.g., port 8080 or 81). Could be TCP 81 -> target group for test. Or reuse port 80 but CodeDeploy will swap target groups behind the same listener (that’s how it works with ALB by shifting the listener to point to new TG). Actually, to mimic ALB style, we should have one listener and two target groups, but NLB doesn’t let you switch target group on one listener via API easily (it’s more static). ALB has a production listener that shifts target group. For NLB, CodeDeploy might require two listeners: one for prod TG, one for test TG, and then maybe it swaps traffic by some DNS or so. This is a bit complex. We might consider using ALB for CodeDeploy ease, but the requirement specifically said NLB. CodeDeploy doc does say you can use NLB ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Application%20Load%20Balancer%20or%20Network,Load%20Balancer))  but it recommends ALB. Given the complexity, an easier path: use ECS rolling deployment or use CodeDeploy but with the knowledge that maybe only one TG (which is not how CodeDeploy does blue/green). However, since they listed CodeDeploy explicitly, likely expecting use of CodeDeploy’s blue/green.
     - Let’s attempt to set up CodeDeploy integration properly: We will use **two target groups**: 
       - `tg-service-a-blue` (production) and `tg-service-a-green` (test). We already created `tg-service-a` – call that production. Create another TG similarly on same port for test (just copy settings, name it `tg-service-a-green`). 
       - For NLB listeners: NLB doesn’t automatically switch one listener’s target group (because with instance target type you could register/deregister but with IP type you just register/deregister IPs which CodeDeploy could do instead). Actually, CodeDeploy might not manipulate NLB listeners for NLB. Possibly it registers the new tasks in the "green" target group, runs health checks, then switches by shifting traffic. With ALB, switching traffic means moving listener to new TG. With NLB, maybe CodeDeploy registers/deregisters targets in the prod TG to shift traffic. 
       - To avoid guesswork, we might consult AWS documentation on ECS Blue/Green with NLB, but given time, we assume CodeDeploy can handle NLB if given two TGs and one listener. Or we use two listeners: one for each TG, and CodeDeploy flips which one is front? Actually, NLB can’t forward between listeners; the client would have to switch ports – not ideal. 
       - I suspect CodeDeploy for ECS with NLB uses one listener: the production TG initially has it, CodeDeploy registers new tasks in the second TG and then changes the targets of the production TG or something. But since CodeDeploy interface requires two TG ARNs (one as live, one as test), we’ll provide them.
     - So, in ECS service create, specify: Production Listener Port = 80, Production Target Group = `tg-service-a` (blue), Test Target Group = `tg-service-a-green` (the new one). It might also ask for the listener port for test, but if using same listener for both, maybe not. If it does, perhaps specify none or different port. (In ALB, you’d specify listener port for test, e.g., 8080).
     - Because of uncertainty in NLB, we might do an easier path: Use an ALB for CodeDeploy demonstration (since CodeDeploy specifically documented it), and mention that NLB usage is similar with some adjustments. However, since NLB is required by user, let’s stick with it and assume CodeDeploy can manage switching target groups behind the scenes (via registration).
     - **Service IAM Role for CodeDeploy:** When enabling Blue/Green, ECS may require an IAM role that allows ECS to interact with CodeDeploy. Actually, CodeDeploy will do the heavy lifting using its role we created. There is a concept of CodeDeploy agent on EC2, but for ECS it's not needed. So probably just ensure we have the CodeDeploy role set.

   - **Service Networking:**  
     - Under Networking, choose the cluster VPC and **subnets** for the service tasks. These should be private subnets where tasks will get ENIs. Choose subnets in both AZs.  
     - Security Groups: choose the `sg-service-a` we created for the tasks to allow NLB traffic.  
     - Load Balancer: since we already configured above, the fields should be filled due to Blue/Green selection. If not using Blue/Green, we’d attach target group directly here as well.  
     - Auto-assign public IP: choose **DISABLED** if tasks are in private subnets (they don’t need public IPs – NLB can reach them internally).  
   - **Auto Scaling (for the service):** You can enable ECS service auto-scaling here or later. For now, we might skip it or set a simple scaling policy (like keep CPU around 50%). We will cover auto scaling in a separate section, so it’s fine to leave the service at a fixed desired count for now.

   - Click Create. ECS will create the service. Initially, it may launch tasks (as the current “blue” fleet). Because we chose Blue/Green deployment, it might not immediately register them in target group until CodeDeploy manages a deployment. Actually, at creation, it likely registers them to the prod TG. The CodeDeploy deployment might be in a “Succeeded” state with just the original tasks as baseline (depending on how the console handles initial deploy – possibly it already did an initial CodeDeploy with the tasks as baseline).

   - Verify in ECS: you should see the service with 2 running tasks. Check ECS “Tasks” tab – tasks should be RUNNING on different container instances (if capacity allowed). Check the target group `tg-service-a` in EC2 console > Target Groups: the tasks’ IPs (or instance:port) should be registered and healthy if the app is running. The `tg-service-a-green` should have no targets yet (since no deployment in progress).

   - Now our service is up. If you point a browser or curl to the NLB DNS on port 80, you should get a response from the service (if the app is properly running). For example: `curl http://microservices-nlb-xxxx.elb.amazonaws.com/` -> should hit one of the tasks. If you have a health endpoint, test that as well. NLB has a very simple round-robin or flow hash algorithm; you can send multiple requests to see it hitting both tasks (you might see differing responses or check the container logs to confirm both get traffic).

**Recap:** We have an ECS service (service A) running 2 tasks (containers) of our microservice, behind a network load balancer, accessible across two AZs. We have integrated it with CodeDeploy (blue/green) by specifying two target groups for shifting traffic. We’ll set up the CodeDeploy side (Application and Deployment Group) next to complete that configuration.

**For a second microservice (service B):** repeat similar steps:
- Create task definition for B.
- Create target groups for B (blue and green).
- Possibly use the same NLB or another NLB. If using same NLB but different port, add a listener on that port mapping to B’s prod TG. However, NLB cannot host two services on the same port with different paths as ALB does, so either use different ports or separate NLB. For simplicity, one NLB per service might be acceptable, but cost-wise not optimal. If it’s an internal microservice, it could reuse the same NLB if you don’t mind adding a port or making clients specify port.
- Considering complexity, you might just demonstrate with one service in detail and mention adding another is similar.  
- We’ll proceed with one primary service for the example pipeline to keep it focused.

So far, we haven’t actually created a CodeDeploy application explicitly. The ECS console Blue/Green flow might have implicitly created one. We should verify in **CodeDeploy console**:
- There should be a new **CodeDeploy application** of type ECS, named something like `CodeDeployApp-[cluster]-[service]-...`. And a **deployment group** within it that references our cluster, service, and target groups.
- Check CodeDeploy console > Applications. If it’s there, we can use that for pipeline. If not, we’ll create it.

Now that our service is running, let’s prepare the Aurora database that our microservice will use, and then we’ll circle back to the CI/CD pipeline setup (CodePipeline, CodeBuild, and linking CodeDeploy deployments).

## **6. Setting Up an Aurora MySQL Database (Amazon Aurora) for the Microservices**  
In a microservices architecture, a database is often a separate component that services can use. We will set up an Amazon Aurora MySQL database cluster to simulate a production-ready database for our application. We’ll configure it for high availability across two AZs and discuss how the microservice connects to it.

**Why Aurora MySQL?** Aurora is a managed relational database that’s compatible with MySQL (and PostgreSQL, but we choose MySQL here as specified). It offers performance improvements and automates tasks like replication, backups, and failover. With Aurora, we can easily create a cluster with a primary (writer) and one or more replicas (readers) across AZs. In case of an instance or AZ failure, Aurora will fail over to a replica, typically in under 30 seconds. The storage is distributed across 3 AZs and is self-healing ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=Fault))  This makes it ideal for a highly-available setup where the database shouldn’t be a single point of failure. Aurora will also **auto-scale storage** as needed and can handle very large data sets without manual provisioning of storage ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=Aurora%20automatically%20scales%20I%2FO%20to,visit%20Aurora%20storage%20and%20reliability)) 

**Steps to create Aurora MySQL:**

1. **Create Aurora DB Cluster:**  
   - In the AWS Management Console, go to **Amazon RDS** (Aurora is under RDS).  
   - Click “Create database”.  
   - Engine options: Choose **Amazon Aurora** as engine type, and **Aurora MySQL** for engine. Select a recent MySQL-compatible version (e.g., Aurora MySQL 3 which is compatible with MySQL 8.0, or Aurora MySQL 2 for MySQL 5.7 depending on requirements).  
   - Templates: Choose Production (so Multi-AZ is selected by default; you can also custom choose).  
   - Settings:  
     - **DB cluster identifier:** e.g., `microservices-aurora`.  
     - **Master username:** choose a master user (e.g., `admin` or `dbuser`).  
     - **Master password:** set a password (store it securely, we’ll need to provide it to ECS tasks).  
   - Instance configuration:  
     - Instance class: for a small test, `db.t3.small` or `db.t3.medium` is economical. Aurora separates cluster storage from compute instances. You’ll have one primary instance and optionally one or more readers.  
     - Number of instances: Since we want multi-AZ, set at least 2 (one writer, one reader in another AZ). Some wizards explicitly ask: “Writer in AZ1, reader in AZ2?” – do that. If using Aurora Serverless v2 you can auto-scale the instance itself, but to keep it straightforward we use provisioned capacity.  
   - Availability & durability:  
     - Multi-AZ deployment: Yes, create reader in a different AZ (should be default in production template). This ensures high availability. Aurora will handle failover to the reader if needed ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=On%20instance%20failure%2C%20Aurora%20uses,RDS%20Proxy%20to%20reduce%20failover))   
     - You can also enable “Auto minor version upgrade” so Aurora auto-patches itself in maintenance windows.  
   - Connectivity:  
     - Choose the same VPC as ECS.  
     - For Subnet group, if none exists specifically, the default will have subnets in multiple AZs. Make sure it covers at least two AZs.  
     - **Public access:** Typically **No** for a production DB (you don’t want the DB to have a public IP). The microservices in ECS will access it privately.  
     - VPC Security Group: create or choose one that allows the ECS tasks to connect. For example, create a security group `aurora-sg` with inbound rule allowing MySQL/Aurora port (3306) from the ECS tasks. If using ECS tasks in a SG (like `sg-service-a`), you can specifically allow that SG as source. e.g., inbound rule: protocol TCP, port 3306, source = `sg-service-a` (i.e., allow any resource in service-a SG to connect). This is a secure way to allow only the app tasks to reach DB. Alternatively, you could allow the entire VPC CIDR, but SG reference is tighter.  
   - Additional config:  
     - Initial database name: You can specify a default database name to create (e.g., `microservices_db`). Otherwise, you’ll have to create a DB via SQL later. It’s convenient to put one now.  
     - Backup retention, maintenance window, etc. – leave defaults or adjust. (Defaults: 7 days backup retention, etc.)  
     - Deletion protection: for learning, you might disable it so you can clean up later; in production, usually enabled to prevent accidental deletion.  
   - Create the database cluster. It will take a few minutes to create the instances. After creation, you will have an **endpoint** for the cluster. Aurora gives a cluster endpoint (writer endpoint) and a reader endpoint (that load balances reads across replicas). You can find these in the RDS console on the cluster details page. For example:
     - Writer endpoint: `microservices-aurora.cluster-xxxxxxxx.us-west-2.rds.amazonaws.com` (this always points to the current writer).
     - Reader endpoint: `microservices-aurora.cluster-ro-xxxxxxxx.us-west-2.rds.amazonaws.com` (this will round-robin among replicas, or point to single reader if one). If your app has heavy read load, you could configure it to use the reader endpoint for read queries to scale reads. For simplicity, we’ll just use the writer endpoint in the app config so all operations go to primary (or whichever is primary after a failover).
   - Note the endpoints, database name, username, and password. These will be needed by the microservice.

2. **Configure Microservice to use Aurora:**  
   - In your microservice code, the database connection string or parameters should be configurable via environment variables or config. We already placed env vars in ECS task definition possibly: `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`. Now fill them with actual values via ECS console (or update the task def):
     - `DB_HOST` = the Aurora cluster writer endpoint.
     - `DB_NAME` = the initial database name (if you set one, or any schema name your app expects, you might need to create tables).
     - `DB_USER` = master username (or create a specific DB user with limited privileges for the app, which is best practice: you can connect Aurora and create a user just for the app with needed perms).
     - `DB_PASSWORD` = the password.
     - If you updated the task definition with these, you’ll need to redeploy tasks. Since we have CodeDeploy set up, the proper way to update an environment variable change is to register a new task def revision and do a deployment (blue/green).
     - However, for an initial setup, you might just manually edit the task def and update service (since maybe pipeline not fully running yet). Alternatively, store these in AWS Systems Manager Parameter Store (plaintext or SecureString) and have the app fetch them – advanced approach, but storing in ECS plaintext is not ideal for real secrets. For now, proceed to inject them for functionality demonstration.
   - **Test Connectivity:** If possible, exec into a container or use a MySQL client from the ECS instance to test the DB. Because the DB is in a private network, you might need to use an EC2 jump box or use AWS Data API for Aurora (if serverless) – but likely easier: from the ECS container instance, install mysql client and attempt connecting to Aurora endpoint on port 3306. Or from the app logs, see if it successfully connects on startup. If security groups are correct (app SG allowed in DB SG) and env vars right, the connection should succeed. If it fails, check:
     - SG rules (maybe forgot to allow correct SG or port).
     - Network ACLs (should be default open for ephemeral ports).
     - Aurora took longer to be available or credentials wrong.
   - **Database Initialization:** For a new DB, it’s empty. You might want to run an initialization script to create tables or seed data. You can do this manually via a MySQL client. E.g., connect using master credentials and run some CREATE TABLE statements for your app. This is app-specific, so we won’t detail it. But ensure the app isn’t failing due to missing tables. Alternatively, if the app creates schema on startup or is tolerant to it, fine. Many real setups use migrations run via the CI/CD pipeline or on deploy.

**Aurora Cost and Scaling Considerations:**  
- Aurora on a db.t3.small with one reader is fairly low-cost, but note that running multiple instances will incur hourly charges. Aurora’s storage will automatically grow (starting at 10GB, and you pay per GB-month). Keep an eye on costs; for experimentation, you can shut down the DB when not in use (Aurora doesn’t have a stop feature unless Aurora Serverless, but you can delete cluster to save cost and restore later from snapshot).  
- Multi-AZ gives you automated failover. If the primary in AZ1 fails, Aurora will promote the replica in AZ2 to primary usually within ~30 seconds to a minute, and your writer endpoint will now point to AZ2 instance ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=On%20instance%20failure%2C%20Aurora%20uses,RDS%20Proxy%20to%20reduce%20failover))  The microservice’s connection logic should ideally retry on failure to reconnect (most DB libraries do).  
- If needed, Aurora can scale read capacity by adding more replicas (up to 15). Also, you can use Aurora Global Database for multi-region DR, but that’s beyond our scope.  
- We won’t set up Aurora Auto Scaling for replicas here, but note it exists to automatically add/remove read replicas based on load ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=Aurora%20provides%20a%20reader%20endpoint,Auto%20Scaling%20with%20Aurora%20Replicas)) 

Now our database is ready and our microservice is configured to use it (with correct network access). We have all the infrastructure in place: CodeCommit repo, ECR, ECS cluster, service running (with CodeDeploy hooks), and Aurora DB. It’s time to set up the actual CI/CD pipeline to tie it all together. The pipeline will detect code changes, build the Docker image, push to ECR, and trigger a deployment of the new version to ECS via CodeDeploy.

## **7. Building the CI/CD Pipeline with AWS CodePipeline**  
In this section, we create the CI/CD pipeline using **AWS CodePipeline** that automates our build, test, and deployment process for the microservice. The pipeline will have the following stages:

- **Source Stage:** Monitors the CodeCommit repository for changes (e.g., a push to the main branch) and triggers the pipeline with the latest code.
- **Build Stage:** Runs CodeBuild to compile the code (if needed), run tests, build the Docker image, and push it to ECR. It will output an artifact containing the new image details.
- **Deploy Stage:** Initiates a deployment of the new container image to ECS. We will use CodeDeploy (Blue/Green) for the deployment action to achieve zero downtime. CodeDeploy will create a new task set with the new image, test it, then shift traffic on the NLB to the new tasks and terminate the old ones.

We will also include CloudWatch notifications or manual approvals as needed (for advanced pipelines, but we might skip those here for brevity).

Let’s assemble each part:

1. **Source Stage – CodeCommit:**  
   - **Action Type:** AWS CodeCommit.  
   - In CodePipeline’s console, when creating a pipeline, you’ll specify the source provider as CodeCommit. Select the repository (`microservices-demo`) and branch (e.g., `main`).  
   - CodePipeline will set up a CloudWatch Events rule (EventBridge) under the hood to trigger on commits to that branch.  
   - Output: The source output artifact will be the contents of the repo zipped up. CodePipeline will store it in an S3 bucket (which it creates for pipeline artifacts). The artifact store bucket is usually named like `codepipeline-<region>-<acct>`... and created automatically (or you specify one). Ensure the bucket has versioning enabled (usually required by CodePipeline).  
   - CodePipeline’s service role must have access to this bucket and CodeCommit (the console will handle adding permissions if you use the wizard, but we already gave the role necessary rights in IAM section).

2. **Build Stage – CodeBuild (Docker image build and push):**  
   - **Action Type:** AWS CodeBuild.  
   - We need a CodeBuild project set up. Let’s create one manually first (or via the pipeline wizard):
     - **Project name:** `microservices-build`.  
     - **Environment:** Use a managed image, e.g., Amazon Linux 2, and choose an image with Docker support. The easiest is to check "Enable this project to use Docker" (privileged mode), since we’ll run `docker build` inside. Alternatively, use the standard CodeBuild image `aws/codebuild/standard:5.0` which includes Docker and just check Privileged.  
     - **Service Role:** attach the role we created (CodeBuild service role with ECR push permissions). The wizard can create one if not existing, but use the one with proper access.  
     - **Buildspec:** We already included a `buildspec.yml` in the repo. Configure the project to use that (default behavior is to look for buildspec.yml in source). Or explicitly point to `service-a/buildspec.yml` if needed. But if each service has its own buildspec, you might need separate CodeBuild projects or orchestrate via a single buildspec. For now, assume one service and one buildspec at root or in repository.
     - **Artifacts:** We will produce an `imagedefinitions.json` file which CodePipeline will use in the next stage. So in the CodeBuild project config, the artifact is set to type “CodePipeline” (meaning it passes to pipeline). The buildspec we wrote already moves the imagedefinitions.json to the output artifact.
     - No need for secondary artifacts or additional configs beyond enabling Privileged mode (for Docker).
   - In CodePipeline, add CodeBuild action:
     - Input artifacts: the output of Source stage (source artifact from CodeCommit).
     - Output artifacts: name it something like `BuildOutput` (this will contain imagedefinitions.json).
     - Project: select the CodeBuild project.
     - When pipeline runs, CodePipeline will zip source and give to CodeBuild, CodeBuild will execute buildspec:
       - It logs in to ECR, builds the Docker image, tags it, pushes it, and writes imagedefinitions.json. This JSON typically contains mappings of container name to image URI for ECS. For example:
         ```json
         [ 
           {
             "name": "service-a",
             "imageUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/microservices-demo/service-a:abcdef0"
           }
         ]
         ``` 
         (where `abcdef0` is the short commit ID or build tag).
       - We made sure the container name in JSON matches the ECS task definition’s container name (“service-a”). CodePipeline will use this to know what image to update.
   - **Verify ECR push:** After a run, you should see a new image tag in ECR for service-a repository. CodeBuild’s role had permission to push, so it should succeed. If not, check IAM role.

3. **Deploy Stage – CodeDeploy to ECS (Blue/Green deployment):**  
   - **Action Type:** AWS CodeDeploy (specifically, CodeDeploy ECS). CodePipeline has a built-in integration for ECS Blue/Green either directly or via CodeDeploy. We’ll use CodeDeploy since we set that up.  
   - Before configuring the pipeline action, we need to ensure CodeDeploy application and deployment group exist and are properly set. Let’s do that in CodeDeploy console (if not already from ECS Blue/Green setup):
     - **CodeDeploy Application:** If ECS Blue/Green was configured via ECS, you might find an app like `AppECS-microservices-cluster-service-a`. If it’s there, use it. If not, create new:
       - Application Name: `microservicesApp` (for example).
       - Compute Platform: select **ECS**.
     - **Deployment Group:** Within that application, create a deployment group:
       - Name: `service-a-deployment-group`.
       - ECS Cluster: choose our cluster.
       - ECS Service: choose `service-a`.
       - Load Balancer: choose **Blue/Green** and specify:
         - Production Listener: (it might show a dropdown if ALB, but for NLB possibly you provide target groups). Possibly CodeDeploy console for ECS asks directly for the prod and test target group ARNs for the ECS service (since NLB doesn’t have rules to change, CodeDeploy will manage target groups). Provide the ARNs of `tg-service-a` and `tg-service-a-green`.
         - If it asks for listener ARNs (only relevant for ALB), since we used NLB, maybe it doesn’t. In CLI/SDK, you do give both TG ARNs and maybe the listener port (for ALB).
       - Deployment settings: you can specify how traffic shift happens (e.g., linear, all-at-once). For simplicity, all-at-once or 100% immediately after test is fine. You can also set a wait time for manual approval or automated bake time. We can just do immediate shift after new tasks are healthy.
       - Choose the CodeDeploy service role (the IAM role we created for CodeDeploy, e.g., `CodeDeployECSServiceRole`). This allows CodeDeploy to perform the deployment.
       - (Optional) Trigger or alarm settings: skip for now. You can integrate CloudWatch alarms to automatically rollback if e.g., high error rate is detected in new version.
       - Save the deployment group.
     - Now we have CodeDeploy app & group. It is essentially linked with our ECS service and NLB target groups.
     - CodeDeploy will look for an **AppSpec file** in the deployment artifact to know how to deploy. For ECS, AppSpec is a YAML that references:
       - The task definition file name and container name to deploy.
       - The names of the target groups for blue and green (the ones we provided in the deployment group, but it double checks names in AppSpec).
       - Optionally hooks (like Lambda functions) during lifecycle events.
     - We need to include the AppSpec and taskdef in our pipeline artifacts. How? Possibly our CodeBuild’s artifact (imagedefinitions.json) is not enough for CodeDeploy; CodeDeploy ECS Blue/Green expects an AppSpec and taskdef in the Source input. In AWS’s official tutorial, they handle this by storing the taskdef JSON and appspec in CodeCommit as part of source too ([Tutorial: Create a pipeline with an Amazon ECR source and ECS-to-CodeDeploy deployment - AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ecs-ecr-codedeploy.html#:~:text=repository%20Step%202%3A%20Create%20task,your%20pipeline%20%20%2010))  ([Tutorial: Create a pipeline with an Amazon ECR source and ECS-to-CodeDeploy deployment - AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ecs-ecr-codedeploy.html#:~:text=,your%20Amazon%20ECR%20image%20repository)) 
       - They commit a file `taskdef.json` (a full ECS task definition JSON with placeholder for image URI) and an `appspec.yaml`.
       - CodePipeline then has two source inputs: one from CodeCommit (with these files) and one from ECR image (with imagedefinitions or from CodeBuild).
       - A simpler approach: since CodePipeline’s ECS integration can replace image via imagedefinitions.json automatically, we might not need a separate taskdef file. But CodeDeploy in blue/green specifically might want an appspec to know what to do.
       - Let’s assume we followed their approach and included in our repo a `taskdef.json` and `appspec.yaml`. If not, we should add them:
         - **taskdef.json:** Can be the same content as the one we used to create the task earlier, but with the image field’s value replaced by a placeholder. For example:
           ```json
           {
             "family": "service-a-taskdef",
             "networkMode": "awsvpc",
             "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
             "containerDefinitions": [ 
                {
                  "name": "service-a",
                  "image": "<IMAGE1_NAME>",
                  ...
                  "portMappings": [ { "containerPort": 3000, "protocol": "tcp" } ],
                  "logConfiguration": { ... }
                }
             ],
             "requiresCompatibilities": ["EC2"],
             "cpu": "128",
             "memory": "256"
           }
           ```
           Where `<IMAGE1_NAME>` is a placeholder CodeDeploy recognizes.
         - **appspec.yaml:** For ECS CodeDeploy, looks like:
           ```yaml
           version: 0.0
           Resources:
             - TargetService:
                 Type: AWS::ECS::Service
                 Properties:
                   TaskDefinition: <TASK_DEFINITION>
                   LoadBalancerInfo:
                     ContainerName: "service-a"
                     ContainerPort: 3000
           ```
           This tells CodeDeploy to deploy a new task definition (which will be the above JSON with actual image filled in) to the ECS service. CodeDeploy also already knows the LB target groups from the deployment group settings.
         - Ensure these files are in the CodeCommit repo (maybe in a `codedeploy/` folder or root).
         - In CodePipeline, we might need to adjust source actions or add a separate source stage for these files. Alternatively, treat them as part of the source artifact with code. If they are committed along with code, the source artifact from CodeCommit already has them.
         - CodeBuild could bundle them into its output artifact, but since imagedefinitions is separate approach, better to have CodeDeploy use its own input.
       - For simplicity, we might skip the explicit AppSpec and rely on CodePipeline’s built-in “ECS deploy” action which can take imagedefinitions.json. However, the built-in ECS deploy (without CodeDeploy) would do a rolling update, not blue/green. To use CodeDeploy, including AppSpec is the way.
       - Given advanced users, we demonstrate the proper CodeDeploy path:
   - **Configure CodePipeline Deploy Action:**
     - Provider: CodeDeploy.
     - Application Name: (select the CodeDeploy app we made, e.g., `microservicesApp`).
     - Deployment Group: `service-a-deployment-group`.
     - Input Artifacts: Now, which artifact contains the appspec and taskdef? Possibly the source artifact if our repo has them. We can have CodePipeline pass both the Build artifact (with imagedefinitions) and the Source artifact to CodeDeploy action. The CodeDeploy action can take multiple input artifacts: one containing AppSpec, one for images? Actually, CodeDeploy ECS expects the AppSpec and taskdef in one of the inputs, and image info either in AppSpec or as image definitions file.
     - According to AWS docs, if you use AppSpec for ECS, you reference the taskdef JSON in appspec (with placeholder), and provide an "ImageURI" through an AppSpec hook or use CodePipeline’s artifact replacement feature. There is a bit of complexity here. Alternatively, CodePipeline has a specific ECS Blue/Green action type (distinct from CodeDeploy action type) that just needs imagedefinitions. But they specifically said CodeDeploy in requirements.
     - If using CodeDeploy action, to automate image injection, CodePipeline can perform placeholder replacement in the taskdef file if we tell it which file and placeholder. In the pipeline YAML, there's something like:
       ```yaml
       - Name: Deploy
         ActionTypeId:
           Provider: CodeDeploy
           Category: Deploy
           Version: 1
         Configuration:
           ApplicationName: microservicesApp
           DeploymentGroupName: service-a-deployment-group
           TaskDefinitionTemplateArtifact: SourceArtifactName
           TaskDefinitionTemplatePath: codedeploy/taskdef.json
           AppSpecTemplateArtifact: SourceArtifactName
           AppSpecTemplatePath: codedeploy/appspec.yaml
           Image1ArtifactName: BuildOutput
           Image1ContainerName: service-a
           Image1Tag: <some tag variable>
       ```
       Actually, CodePipeline can auto-fill `<IMAGE1_NAME>` in the taskdef if we specify which artifact has the image and the tag (via something called ImageDefinitions or if using ECR source trigger).
       - Possibly easier: use CodePipeline's built-in "Amazon ECS" action for deploy (not CodeDeploy one). There's an action type "Amazon ECS" which does a rolling update. But since CodeDeploy was listed, likely want that.
     - Considering the complexity, let’s assume we set it up correctly: The CodeDeploy action will take:
       - AppSpec and TaskDef from the **Source artifact** (or a specific artifact that we might create by zipping those files in CodeBuild output as well).
       - It will take the image details from the **Build artifact** (imagedefinitions.json).
       - CodePipeline will substitute the `<IMAGE1_NAME>` placeholder in taskdef with the image URI from imagedefinitions.json (this is an automated step if configured).
     - When pipeline runs Deploy stage, CodePipeline will create a bundle that CodeDeploy uses: the AppSpec with now a fully formed taskdef (with image URI inserted). CodeDeploy then creates a new revision of the task definition in ECS, and instructs ECS to start new tasks (the green fleet) with that taskdef. Then CodeDeploy monitors the new tasks’ health using the target group (green TG). Once healthy, it shifts traffic by directing the NLB to those tasks (in our case, presumably by deregistering old tasks from prod TG and adding new, or swapping target groups associated with service – CodeDeploy manages that as per deployment group config). Then it will terminate the old tasks (blue). All of this happens under the hood, and CodePipeline just waits for CodeDeploy to report success or failure.

4. **Pipeline Creation and Execution:**  
   - Use the CodePipeline console or AWS CLI to create the pipeline with above stages. If using console wizard:
     - Pipeline name: e.g., `microservices-CI-CD`.
     - Service role: choose existing if we created, or let it create one (if creates, ensure to adjust its policy for CodeDeploy pass role).
     - Artifact store: choose default S3 bucket or specify one.
     - Add Source (CodeCommit, branch main).
     - Add Build (CodeBuild project).
     - Add Deploy: choose Deploy provider = CodeDeploy, then pick application and deployment group.
     - The console should detect this is an ECS CodeDeploy deployment and might ask for AppSpec artifact mappings. If not, after creation, you can edit the pipeline JSON in CodePipeline to ensure placeholders are set. Alternatively, if using the "Create pipeline" wizard, if you selected CodeDeploy ECS, it might ask for the AppSpec file and image mappings explicitly.
   - Once created, release a change (or push a commit to CodeCommit if not already done after pipeline creation to trigger it). 
   - Monitor the pipeline:
     - Source stage should quickly show "Succeeded" if it got the revision.
     - Build stage will take a few minutes. You can watch logs in CodeBuild (CodePipeline provides a link to logs).
     - If Build passes, check ECR for a new image tag and CodeBuild’s artifact in S3 (it will contain imagedefinitions.json).
     - Deploy stage: this will invoke CodeDeploy. In CodeDeploy console, you’ll see a new deployment under the application. It will go through phases: e.g., **DownloadBundle**, **CreateDeployment** (it registers new task def and starts new tasks (green)), **Routing** (shift traffic), etc. If all goes well, it ends with status Succeeded.
     - Verify after deployment: The ECS service should now be running the new task definition revision. The old tasks (blue) should be stopped. The NLB target group `tg-service-a` now has the new tasks registered (it may actually be the same target group all along, just updated membership). If using two target groups, CodeDeploy might have swapped them, but with NLB likely it managed targets differently.
     - The pipeline then finishes successfully. Congrats, you have a working CI/CD pipeline!

5. **Testing the End-to-End Pipeline:**  
   - To truly test, make a change in the code (e.g., modify a message or version number the service returns), commit and push to CodeCommit. This should trigger CodePipeline. 
   - The pipeline will build a new image with the change and deploy it. You can observe no downtime on the service: continuously call the API endpoint via NLB while deployment happens. Ideally, some calls go to old version until switch, then new calls go to new version. If done right, there’s no error or downtime.
   - If something fails (e.g., the new tasks don’t become healthy), CodeDeploy will automatically rollback by stopping the new tasks and keeping the old ones serving. It will mark deployment failed, and pipeline will fail. You can check CodeDeploy logs/events for the cause. This mechanism is very powerful for safe deploys.

**Summary of CI/CD Flow:** When a developer pushes code to CodeCommit, CodePipeline retrieves the latest code and triggers CodeBuild. CodeBuild runs our tests (if any) and builds the Docker container, pushing it to ECR. Then CodePipeline (via CodeDeploy) takes the new image and deploys it to ECS using a blue/green strategy: launching new containers alongside the old ones, waiting for health checks, then switching the load balancer to the new containers. Throughout, we’ve leveraged AWS-managed services to handle the heavy lifting of provisioning and orchestrating these steps, making the process reliable and repeatable. This aligns with DevOps best practices of automation and continuous delivery.

Now that the pipeline is established, we will address how to monitor this system and scale it, as well as discuss troubleshooting common issues and optimizing costs.

## **8. Monitoring, Logging, and Alerts with Amazon CloudWatch**  
With our application and pipeline running, it’s important to monitor their health and performance. **Amazon CloudWatch** provides multiple capabilities for this:

- **CloudWatch Logs:** Captures logs from various services (ECS tasks, CodeBuild, CodePipeline, etc.).
- **CloudWatch Metrics:** Provides numeric metrics like CPU utilization, memory usage, network I/O for ECS, as well as pipeline execution metrics.
- **CloudWatch Alarms:** Allows setting thresholds on metrics to trigger notifications or automated actions (like scaling or alerts).
- **CloudWatch Events (EventBridge):** Captures events from AWS services (like CodePipeline state changes or CodeDeploy deployment events), which can trigger targets like notifications or Lambda functions.
- **Container Insights:** An optional feature for ECS that provides advanced metrics (like per-container CPU/memory) and logs aggregation in a convenient package.

Let’s break down monitoring for each part of our system:

1. **Monitoring the CI/CD Pipeline:**  
   - CodePipeline emits metrics such as **Success/Failure count** and **Duration** of pipeline executions. You can see these in CloudWatch Metrics under the namespace `AWS/CodePipeline`. For example, a metric `PipelineExecutionSuccess` can be used to alarm if the pipeline fails.  
   - CodePipeline also sends events for state changes. For instance, an event when a pipeline execution starts, succeeds, or fails. We could set up an EventBridge rule to catch the failure event and notify the team (via Amazon SNS or Slack integration through AWS Chatbot).  
   - Similarly, CodeBuild has metrics like build duration and whether builds succeed. CloudWatch Logs for CodeBuild (accessible from the CodeBuild console or CloudWatch Logs under `/aws/codebuild/project-name`) contain the detailed build output. If a build fails, those logs show where (e.g., test failing or Docker build error).  
   - You might create an alarm on the CodeBuild project for **Failed Builds** > 0 in the last N minutes to alert on build failures that might otherwise go unnoticed if not checking the pipeline.  
   - CodeDeploy has events for deployment start, success, or failure. In CodeDeploy console you can see these, but you can also catch them via EventBridge. If a deployment fails and is rolled back, you’d want to know. AWS can send an SNS notification on CodeDeploy failures configured via the deployment group (there’s an option for notifications).

2. **ECS Service and Cluster Monitoring:**  
   - ECS provides many metrics automatically to CloudWatch:
     - At the **Cluster level**: CPU and Memory utilization (as a percentage of total capacity). These are visible as `ClusterName, metric CPUReservation` etc. This tells how packed your cluster is. For example, if CPUReservation is 80%, it means tasks are using 80% of the CPU capacity of the cluster. These metrics are published at 1-minute intervals ([Monitor Amazon ECS using CloudWatch - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-metrics.html#:~:text=You%20can%20monitor%20your%20Amazon,the%20Amazon%20CloudWatch%20User%20Guide)) 
     - At the **Service level**: similar metrics but for the tasks in that service – CPUUtilization and MemoryUtilization (the percentage of the task's requested CPU/Memory that's being used). If these are consistently high (like > 80%), it might mean the service needs more resources or more task instances.
     - **CloudWatch Container Insights** (if enabled) can give even more granular metrics like per-container CPU, memory, and also network, storage, and custom metrics. It works by running a CloudWatch agent or using ECS integrated metrics. Enabling it might incur extra cost, but it’s useful for deep monitoring.
   - **Logs from ECS Tasks:** We configured the tasks to use awslogs driver with a log group (e.g., `/ecs/microservices-demo`). Each task (and each container in task) will create log streams like `service-a/ecs/service-a/<task-id>`. These logs contain the console output from our application (e.g., any log statements, errors, stack traces). You can view them in CloudWatch Logs console, and should set a retention period (maybe 7 or 14 days) to avoid unlimited growth (default is never expire, which could become costly).  
   - If the application experiences an error (like an exception, or cannot connect to DB), those should appear in these logs. Always check here first when the app isn’t behaving as expected.
   - **Custom Metrics:** If needed, you could have the application push custom CloudWatch metrics (e.g., number of processed orders). This can be done via CloudWatch API using the AWS SDK from within the app, but requires AWS creds (the task role could allow CloudWatch PutMetricData for a specific namespace). This is advanced and not always necessary, but can be useful for business-level monitoring.
   - **CloudWatch Alarms for ECS:** You might set alarms such as:
     - If Service CPU > 70% for 5 minutes, then trigger an auto-scaling action (scale out tasks) – we’ll configure this in auto scaling section.
     - If Service CPU < 20% for a long time, scale in tasks.
     - If Memory utilization > 80%, perhaps alert or scale (depending on app).
     - If cluster CPU reservation > 90%, maybe we are running out of capacity – could trigger adding more EC2 instances.
     - If number of running tasks drops below desired (this could mean tasks are crashing repeatedly).
   - ECS also sends events (visible in ECS console or through EventBridge). For example, when a task is stopped due to an error, an event is generated with reason (e.g., OOMKilled, or Essential container died). You can capture ECS events via EventBridge for significant issues like task failures.

3. **Aurora MySQL Monitoring:**  
   - RDS/Aurora provides a wealth of metrics: CPU, memory, connections, read/write IOPS, replication lag, etc. In the RDS console or CloudWatch (namespace `AWS/RDS` with `DBClusterIdentifier`), you can view these. Important ones:
     - CPUUtilization of the DB instances – if this is high consistently, DB might be bottleneck.
     - DatabaseConnections – number of active connections (our microservice likely uses a connection pool; ensure it’s not exhausting allowed connections).
     - FreeableMemory and FreeStorageSpace – to ensure the instance isn’t running out of memory or storage.
     - Aurora has specific metrics like `ReplicationLag` for replicas.
   - By default, Aurora is quite automated, but you should set alerts for:
     - CPU too high on DB.
     - Too many connections (approaching the max, which for MySQL might be around  max_connections config).
     - Failover events – CloudWatch Events can catch an RDS failover event if it happens.
     - Also ensure backups are succeeding (Aurora does continuous backup by default).
   - **Logs**: Aurora MySQL can publish slow query logs or error logs to CloudWatch if enabled. For deep troubleshooting (like a query is slow), enabling slow query log and analyzing it is useful. Given advanced users, they might set that up.

4. **End-to-End Tracing (optional):**  
   - For advanced monitoring of microservices, AWS X-Ray could be used to trace requests through the system (especially if there are multiple services calling each other, you’d see a service map and latencies). Setting up X-Ray would require running the X-Ray daemon or integrating SDK in the app, plus granting X-Ray permissions to the task role. We mention this as an advanced idea but not implement here.

5. **Dashboards and Visualization:**  
   - CloudWatch allows creating **Dashboards** where you can pin various metrics (graphs or numbers) together. For quick overview, one could create a dashboard “MicroservicesCI-CD” showing:
     - Pipeline status (maybe using a widget that shows last pipeline execution status, or a metric graph of successes/failures).
     - ECS service CPU/Memory graphs.
     - Number of tasks running.
     - Aurora CPU and connections.
     - Perhaps a log query widget summarizing recent log errors from the app logs.
   - This single pane can be useful during deployments to watch system health.

6. **Notifications:**  
   - Use Amazon SNS for sending notifications (email or otherwise) when alarms fire or events happen. For example, an SNS topic “OpsAlerts” subscribed by the dev ops team’s email can be triggered by CloudWatch alarms (like pipeline failure, ECS task crash, high DB CPU, etc.).  
   - Also consider AWS Chatbot integration with SNS or CloudWatch – it can send alerts to Slack or Amazon Chime channels directly. This allows teams to get alerts in their chat and even run some commands if configured.

**Logging Best Practices:**  
- Ensure logs contain useful context (timestamps, request IDs, etc.). Since multiple tasks run the same app, adding something like instance or task ID in logs can help differentiate which container logged what. AWS already tags logs by task ID in separate streams, but within the app log, maybe include a short container ID prefix for clarity.
- Set log retention to a reasonable period. You can do this in CloudWatch Logs console for each group (e.g., 1 month). This prevents large costs due to stored logs over time, and you likely don’t need very old logs once code has changed.
- If logs are very high volume, consider using **Centralized Logging solutions** or log filtering. CloudWatch Logs Insights allows you to query logs with SQL-like queries, which is useful to troubleshoot issues after the fact (like “count of ERROR occurrences in last 5 hours”).
- If compliance requires, you can export logs to S3 for long-term archival using CloudWatch Logs subscription or AWS Lambda.

**Metric Monitoring Best Practices:**  
- Use **targets** for auto scaling (like CPU target, see next section) rather than static thresholds, to automate responsiveness.
- Set some **baselines** and adjust thresholds as you learn normal behavior. For example, if normal CPU is 50%, maybe alarm at 80%. If memory usage naturally goes up and down, alarm only on sustained high usage.
- Leverage **anomaly detection** in CloudWatch Alarms for metrics that have predictable patterns – this is an advanced feature where CloudWatch learns the metric’s normal range and can alert on deviations without you setting a specific static threshold.
- Tag resources (like EC2 instances, or tasks via ECS metadata) with identifiers. CloudWatch can group metrics by tags sometimes or you can filter.

By putting a robust monitoring system in place, you ensure that when something goes wrong (and at some point, something will), you have the data to quickly diagnose it. CloudWatch, combined with the other AWS services, essentially acts as the eyes and ears of our deployment.

Next, we will configure auto scaling for both our ECS service and the EC2 instances to ensure the system can handle load variations and stay cost-efficient.

## **9. Implementing Auto Scaling for ECS Services and EC2 Instances**  
Auto Scaling is a critical aspect of managing a production system. It allows your application to **scale out** (add more resources) under high load and **scale in** (remove resources) when load decreases, saving cost. In our architecture, we have two layers to consider for scaling:

- **ECS Service Auto Scaling (Task Scaling):** Adjusts the number of running task instances for our microservice.
- **EC2 Auto Scaling (Cluster Capacity Scaling):** Adjusts the number of EC2 container instances in the ECS cluster.

These need to work in tandem: there’s no point in increasing tasks if the cluster doesn’t have capacity to place them. Likewise, running a huge cluster of EC2 instances is wasteful if tasks are few. AWS provides mechanisms to coordinate these via ECS Capacity Providers, but we can also configure them with CloudWatch alarms independently.

**9.1 ECS Service Auto Scaling (Tasks):**  
AWS uses the **Application Auto Scaling** service to handle scaling of ECS tasks (and other scalable targets like DynamoDB throughput, etc.). We will define a scaling policy for our ECS service (service-a).

Steps to enable service auto scaling:
1. **Register the service as scalable target:**  
   Using either the ECS console or Application Auto Scaling API, you specify that the ECS service’s desired count can be dynamically scaled between a minimum and maximum. For example, min=2, max=10 tasks.

   In ECS console: Go to Cluster > service-a > “Auto Scaling” (or “Update” service and find scaling options). There is typically an option “Configure Service Auto Scaling”. Enable it, set min, max, and initial (current desired).

2. **Choose a scaling policy type:**  
   There are a few types:
   - **Target Tracking Scaling:** Easiest; you set a target for a metric and AWS will adjust the tasks to maintain that target. For example, target CPU utilization = 50%. This is analogous to a thermostat: if CPU usage per task goes above 50% on average, add tasks; if below, remove tasks. This is often recommended because it auto-adjusts and is simple.
   - **Step Scaling:** You define specific CloudWatch alarms for metrics and how to change task count when those alarms fire (increase by N tasks if CPU > 70% for 5 min, etc.).
   - **Scheduled Scaling:** Scale at specific times (not needed here, unless you expect known daily cycles and want to pre-scale).

   We’ll use Target Tracking on CPU as an example, since CPU is a good proxy for load if the app is CPU-bound. If it’s more I/O or memory bound, you could target memory or a custom metric (like requests per second via ALB RequestCount, but with NLB we don’t have ALB metrics).

3. **Configure Target Tracking Policy (CPU):**  
   - In the scaling policy, set target CPU utilization = say **50%**. This means the average CPU across all tasks in the service should be 50%. If each task starts getting more work and CPU climbs, auto scaling will add tasks to bring the average down. Conversely, if CPU is way below 50% (meaning we have over-provisioned tasks relative to load), it will remove tasks until average goes up to 50%. This provides a balance of performance and efficiency.
   - The auto scaling system needs some time to adjust and it prevents rapid flapping by having cooldown periods. With target tracking, those are usually automatically tuned (e.g., scale-out cooldown 60 sec, scale-in 60 sec by default).
   - After setting this, behind the scenes AWS creates a CloudWatch alarm on `ECSServiceAverageCPUUtilization` metric for that service and manages it.

4. **Test Task Scaling:**  
   - To see it in action, you’d need to simulate load. For example, run a CPU stress test or send many requests to the service so CPU jumps. CloudWatch will pick up high CPU, the target tracking might add tasks (provided we have capacity on cluster).
   - Similarly, when load subsides, tasks will scale down (but typically leaving at least the minimum count).
   - This can take a few minutes to react. CloudWatch metrics are every 1 min, and target tracking might need 2-3 consecutive data points to decide.
   - Ensure your service code can handle multiple instances (which it should if stateless or if using DB for state). Also ensure new tasks can come up quickly enough to handle bursts (maybe keep container image slim for fast start, etc.)

**9.2 EC2 Auto Scaling (Cluster Scaling):**  
We have an Auto Scaling Group for our ECS cluster instances, currently maybe fixed at 2. We want it to scale out if more tasks are needed and scale in to save cost when idle.

There are two approaches:
- **Manual CloudWatch Alarms & Scaling Policies:** Create alarms on cluster metrics and tie to ASG scaling. For example, if cluster CPUReservation > 75% (meaning cluster’s used capacity high), add an instance. If CPUReservation < 30% for 10 minutes (lots of free capacity), remove an instance. Similar for memory.
- **ECS Cluster Auto Scaling (Capacity Provider):** A newer method where ECS and Application Auto Scaling manage the ASG for you. You create an ECS Capacity Provider with the ASG and set a target for cluster utilization. For example, keep cluster at 75% usage. ECS will then launch/terminate instances as tasks are added/removed to achieve that target. This method is more direct and accounts for tasks waiting for capacity.

Given an advanced setup, let’s outline the **Capacity Provider** method, as it’s best practice now:
1. **Create Capacity Provider for the ASG:**  
   Using ECS Console: go to cluster, in “Capacity Providers” section, create new.
   - Select the Auto Scaling Group (the one our instances are in).
   - Enable Managed Scaling, set target capacity % (like 80%). This means: ECS will try to ensure that at least 20% of the cluster is always free (or rather, only 80% is used), launching new instances if usage > 80, terminating if usage much lower, so it always hovers around 80.
   - Min/Max scaling parameters: these can limit how far to scale.
   - Managed Termination Protection: can enable to ensure ECS tasks drain on an instance before the ASG kills it on scale-in (prevents interrupting tasks).

   This capacity provider will then be associated with the cluster. We then need to update the ECS service to use this capacity provider (instead of the default implicit one).
   
2. **Update ECS Service to use Capacity Provider Strategy:**  
   - Edit the ECS service (service-a). There’s an option to set Capacity Provider Strategy. Choose the new capacity provider with a weight (if only one, weight 1 is fine). This tells ECS to place tasks using that provider (the ASG capacity). Now ECS knows it can launch instances if needed for this service’s tasks.
   - Now, when our service scales out tasks and if the cluster doesn’t have enough room, ECS will record “unable to place task” events. With capacity provider managed scaling on, ECS detects this and triggers the ASG to scale out (it does so by setting a higher desired capacity in ASG) ([Capacity Providers Primer - Amazon ECS Workshop](https://ecsworkshop.com/capacity_providers/capacityprovider_primer/#:~:text=Capacity%20Providers%20Primer%20,from%20managing%20autoscaling%20the))  It will add as many instances as needed to fit the tasks (respecting the target utilization).
   - Conversely, if cluster has too much free capacity (like tasks scaled in and now usage is below target), ECS will scale in the ASG (remove instances) down to the minimum or until usage is back to target.

3. **ASG Policy (if not using capacity provider):** If we weren't using capacity providers, we'd do:
   - Create a CloudWatch alarm on `ECS Cluster CPUReservation` > 75% for 5 minutes. Set this alarm as trigger to ASG scaling policy that adds e.g., +1 instance (scale out).
   - Another alarm on CPUReservation < 30% for 10 min triggers scale in (remove 1 instance).
   - Similar for memory if needed. (Though one of CPU or Memory often is the bottleneck; you'd choose the one that more often hits limits. Or do multiple policies and the ASG will use the one that triggers first.)
   - Ensure scale-in is conservative to avoid flapping (like longer sustained low usage needed to remove).
   - Attach these to ASG. The ASG will then handle actual launching/terminating. ECS, via the container instance role and agent, will automatically register/deregister instances. We should ensure ECS container instance draining is enabled for ASG scale-in (so tasks move off gracefully). If using capacity provider with managed termination protection, that does it. If doing manually, you can enable "drain on terminate" in ECS cluster (using a Lambda triggered by ASG lifecycle hook, or simpler in recent ECS features there's an option for ASG to put instance in DRAINING on terminate, which was introduced and is on by default with capacity providers) ([Amazon ECS capacity providers for the EC2 launch type - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/asg-capacity-providers.html#:~:text=We%20recommend%20you%20use%20managed,workloads%20running%20on%20EC2%20instances)) 

4. **Test ASG Scaling:**  
   - If you simulate load and tasks scale out beyond current instance capacity, new instances should launch automatically. You can see in EC2 Auto Scaling console. Instances take a couple minutes to boot and join cluster, then ECS places the pending tasks on them. 
   - When load drops, tasks scale in (after some cooldown). After tasks are removed and cluster is underutilized for a while, ASG should terminate an instance. ECS will detect instance termination and move any tasks on it (but if tasks already scaled in, it might have been empty).
   - Always keep at least 2 instances (multi-AZ) for availability, unless cost is absolutely critical.

**Important Auto Scaling Best Practices:**  
- **Scale Out Events** should ideally happen quickly on sudden load spikes to maintain performance. Use a lower threshold and shorter alarm period for scale-out (like 1-2 data points of high usage to trigger). Scale-in can be more cautious to avoid removing capacity too soon (maybe require 5-10 min of low usage).
- **Cooldowns:** If using step scaling, define a reasonable cooldown (the default might be 300s). This prevents another scale action from triggering immediately. Target tracking handles this internally – it won’t keep adjusting too frequently.
- **Over-Provision Slightly for Spikes:** The target usage 50% for tasks and 80% for cluster leaves headroom. If you run at 90-100% on tasks, any small spike could degrade performance until scale-out kicks in. So choosing a lower target ensures there's buffer.
- **Instance Warmup:** On scale-out, new instances need to pull Docker images. Our images are in ECR and hopefully not huge, but for big images it takes time. A trick is to use **Instance user data** or a lifecycle hook to pre-pull frequently used images when instance launches, so that tasks start faster. Or keep images small.
- **Spot Instances:** For cost optimization, you could configure ASG to use Spot Instances (maybe a mix of On-Demand base and Spot for extra capacity). ECS integrates well with spot if you handle interruption (ECS will get a 2-minute warning event). This can cut EC2 costs 70% but you risk task interruption when spot reclaimed. That’s an advanced cost-saving strategy if your app can tolerate occasional restarts or have redundancy.
- **Scale other components:** If load increases, database might become a bottleneck eventually. Aurora can scale read by adding replicas (Aurora Auto Scaling of replicas can add one if CPU or connections high). Write scaling is harder (vertical scaling or Aurora Serverless v2 which can scale compute in seconds). Keep an eye if DB becomes the limiting factor when more app tasks generate more DB traffic.

Our system is now quite dynamic: as usage grows, more app containers and instances come online; as usage falls, it contracts. This ensures high availability and performance (through redundancy and avoiding overload) and cost efficiency (not running more than needed).

In the next sections, we will look at some common issues that might arise (troubleshooting) and how to handle them, and finally wrap up with cost breakdown and best practices summary.

## **10. Troubleshooting Common Issues in AWS CI/CD Deployments**  
Even with the best setups, things can go wrong. Let’s cover some common problems one might encounter in this CI/CD pipeline and microservices deployment, along with strategies to troubleshoot and resolve them. We’ll break it down by stages/components:

**10.1 CodePipeline & CodeBuild Issues:**  
- *Pipeline Not Triggering:* After pushing code, the pipeline didn’t start. Check that the CodePipeline source trigger is correctly configured. The CodePipeline console’s “Triggers” section for the pipeline should show an EventBridge rule. If not, you might need to re-create the source stage or manually release changes. Also ensure your CodeCommit branch name matches exactly what pipeline expects.  
- *Source Stage Failing:* Possibly CodePipeline cannot fetch from CodeCommit. This could be an IAM issue – the pipeline role might not have permission to read the repo. Verify the pipeline role’s policy includes access to CodeCommit (e.g., `codecommit:GetBranch`, `GetCommit`, `UploadArchive` on that repo). Or check if the repo name/branch in pipeline config is correct.  
- *Build Stage Failing:* If CodeBuild fails, review the logs in CloudWatch (or console). Common issues:
  - **Buildspec errors:** Perhaps the YAML is malformed or commands are failing. The logs will show at which command it failed. For instance, if `docker build` failed, you’ll see the Docker error output (maybe a syntax error in Dockerfile or failing test).
  - **AWS Credentials in Build:** If the build needs to call AWS (like `aws ecr get-login-password` as we do), ensure the CodeBuild role has permissions. If you see access denied in logs when pushing to ECR, check the IAM policy for CodeBuild role includes ECR actions (e.g., missing `ecr:PutImage` or similar) ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=,an%20Amazon%20ECR%20private%20repository))  Adding the AmazonEC2ContainerRegistryFullAccess managed policy (if comfortable with broad access) or a specific policy for the repo can fix this.
  - **Docker not available:** If you forgot to enable privileged mode in CodeBuild for Docker, the `docker build` command will error out. The log will indicate permission denied or “Cannot connect to the Docker daemon”. Enable privileged mode and use the proper CodeBuild image that includes Docker.
  - **Dependency failures:** If building code, maybe `npm install` or similar failed due to network or a bad package. CodeBuild runs in a VPC or outside depending on config. If you put CodeBuild in private subnets with no internet, it can’t download dependencies. The fix is either give it a NAT Gateway for internet access or don’t enclose it in a private subnet. By default, CodeBuild without VPC config has internet access. So adjust accordingly.
  - **Tests failing:** If you have a test phase and tests fail, the build will be marked failed. That’s expected – fix the code or tests and push again.

**10.2 ECR & Image Issues:**  
- *Cannot Pull Image on ECS:* CodeDeploy or ECS service tries to start a task but fails to pull the image. The ECS event might say “Cannotpullcontainererror: access denied”. This usually means the ECS task execution role lacks permission to ECR. Ensure `AmazonECSTaskExecutionRolePolicy` is attached to the execution role ([Amazon ECS task execution IAM role - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#:~:text=,an%20Amazon%20ECR%20private%20repository))  Or if the image is in another account’s ECR, you need a repository policy to allow it.  
- *Image Not Found:* If the task definition references an image tag that doesn’t exist, tasks will fail to start. For instance, if CodePipeline didn’t update the taskdef or imagedef correctly, ECS might be looking for `:latest` but your pipeline pushed `:abcdef1`. If using CodeDeploy, ensure the placeholder substitution worked. Check the CodeDeploy AppSpec and taskdef that got deployed (CodeDeploy has a “deployment artifacts” section where you can see the final taskdef it used). If it still had `<IMAGE1_NAME>` or an old tag, something went wrong in pipeline configuration. Fix the pipeline to properly provide the new image URI.  
- *ECR Login Issues:* If the ECS container instance can’t get auth token for ECR (shouldn’t happen if roles set up). But if you see in ECS agent logs an error fetching ECR token, verify the container instance IAM role has `ecr:GetAuthorizationToken`. The managed instance role should have it; older ones might not (if created before ECR existed). The fix is to attach the updated policy or attach `AmazonEC2ContainerRegistryReadOnly` to the instance role.

**10.3 ECS and Service Issues:**  
- *Task Stopped Immediately (Exit 1):* ECS service tries to start tasks but they keep stopping. Check ECS console -> Tasks -> stopped tasks -> details. It will show a stop code and maybe container exit code. Exit Code 1 from your app means it likely crashed on startup. Now check the logs for that task in CloudWatch. Common reasons:
  - Misconfiguration: e.g., the app couldn’t connect to DB and it exits. The log might show a stack trace "Connection refused to DB". That could be because the DB env vars are wrong or security group not allowing connection. Solution: fix env vars, or security groups as needed, then update task definition and re-deploy.
  - Missing dependency: maybe the container can’t reach some external service and no retry logic. Ensure network routes are correct (if your container needs internet but is in private subnet without NAT, it can’t reach external APIs).
  - Code bug: null pointer exception causing crash, etc. Fix code, go through pipeline again.
  - If exit code is 137, that indicates OOM (out of memory killer). That means container tried to use more memory than allowed and kernel killed it. The ECS event would say `OutOfMemoryError`. Solution: allocate more memory in task def, or find memory leak in app. If it’s spiky usage, could set soft limit instead of hard limit so it can use more if available.
  - If exit was due to a `Essential container in task exited` – that just means the main app container stopped. Could also be caused by a dependency container failing if you had sidecars (not in our case).
- *Service Stabilization Timeout:* When deploying, ECS or CodeDeploy might report “Service did not stabilize in time” (like tasks didn’t reach steady state within deployment timeout). This often happens if new tasks fail health checks or can’t start. CodeDeploy would eventually rollback if it can’t get green tasks healthy. Investigate tasks as above. Maybe the health check endpoint is not correct. If container is healthy but perhaps the load balancer health check is failing (maybe you set `/health` but the app doesn’t have that route), then tasks will be killed as “unhealthy” even though app might actually be fine. Ensure the health check configuration matches the app. To debug, you can exec into a container or curl the health URL via the container’s host.
- *Load Balancer 5xx or No Response:* After deployment, if hitting the NLB DNS yields errors:
  - If it’s NLB with TCP, not much to go wrong except if no tasks are registered or tasks are closed. Check target group health: are targets healthy? If none, then no service behind LB -> likely tasks all failed. Or security groups not allowing the traffic from LB to tasks (target group health check would show failed). If target health check says “Timeout” or “Connection refused,” security group or app not listening is likely cause.
  - If using HTTP health check on NLB, a 5xx health result means app responded with error. Check app logs to see why.
  - DNS issues: ensure you’re using the correct NLB DNS or custom CNAME. Sometimes when NLB swaps target groups (CodeDeploy), the DNS stays same so that’s fine. But if you had multiple LBs (for multi-service), ensure you’re checking the right one.

- *One AZ Outage Scenario:* If one AZ’s tasks or instance failed, ECS should start replacement in other AZ (as long as at least one instance in other AZ). If not, check if your service placement constraints are too strict (by default it tries to balance across AZ, but if one AZ lost all instances and cluster had a minimum healthy percent, maybe it was waiting). Usually ECS is resilient here. If tasks in one AZ are unhealthy, NLB will stop sending to them (because they’d fail health checks), so the app still works via the other AZ tasks. Later, ASG would bring up new instance in that AZ. So normally handled, but it’s good to test by shutting down an instance: does traffic still flow? It should with two AZ NLB.
  
**10.4 CodeDeploy Issues:**  
- *CodeDeploy Deployment Failed & Rolled Back:* The CodeDeploy console will show at what phase it failed. Common ones:
  - **During AfterAllowTestTraffic hook** (if you had a validation test): We didn't set any, but you could. If a validation script or test fails, CodeDeploy will abort. Fix the test or code as needed.
  - **Target not ready in time:** means the new tasks didn't register healthy in target group within the wait period. Possibly the app took too long to start (exceeding health check grace). You can increase the timeout in deployment group settings (there is an option for how long to wait for tasks to be healthy). Or optimize app startup.
  - **Permissions:** If CodeDeploy couldn’t do something like modify target group or create task set, that’s an IAM issue with the CodeDeploy role. Ensure it has all needed perms (e.g., `ecs:CreateTaskSet`, `ecs:UpdateService`, `elasticloadbalancing:DeregisterTargets`, etc.) ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Application%20Load%20Balancer%20or%20Network,Load%20Balancer)) 
- *CodeDeploy Stuck in Progress:* Rarely, if something didn’t update but didn’t fail, you might see deployment in progress for a long time. Check ECS – are new tasks still launching? Possibly scale issues. If it’s clearly hung, you might need to stop deployment (which triggers rollback). Investigate afterwards.

**10.5 IAM and Security Issues:**  
- *Access Denied Errors:* We covered some specifically (ECR pull, CodeCommit access). The general fix is to identify which action is denied (the error message often says, e.g., `iam:PassRole was denied for role X`). That one is common: CodePipeline might be trying to pass a role to CodeDeploy or CodeBuild and not allowed. For instance, pipeline needs `iam:PassRole` on the CodeDeploy role ARN. Add that to pipeline role policy.
- *Too Open Permissions:* The opposite of an error is a potential security risk – e.g., you temporarily gave Admin access to CodeBuild role to fix an issue. Don’t forget to tighten it back. Use IAM Access Analyzer or CloudTrail logs to see what actions are actually used, then restrict accordingly.

**10.6 Aurora DB Issues:**  
- *Connection timeouts:* If the app can’t reach Aurora, double-check:
  - Security group rules (SG of DB allows SG of ECS tasks on port 3306).
  - If using hostnames, maybe DNS or VPC endpoint issues (shouldn’t, but if VPC has custom DNS disabled, need it on for AWS services).
  - Aurora status: is the cluster available? If failover happened, maybe a short outage occurred. The app should reconnect. If not using a driver that auto-retries, you may need to implement that.
- *Slow queries or high latency:* If app logs show DB queries slow, consider analyzing the Aurora performance (Performance Insights in AWS can help see heavy queries), add indexes, or scale up DB instance class if CPU pegged. Also ensure connections are reused (connection pooling) so that overhead is minimized. If database is the bottleneck, adding more app tasks won’t help – need to scale the DB read capacity by adding replicas or using caching layer (Elasticache).
- *Running out of connections:* If microservice creates too many DB connections (maybe on each request without closing), you might hit the max. Aurora MySQL by default might allow ~90 connections on a small instance. Use a connection pool and tune it. CloudWatch metric DatabaseConnections helps. If maxed out, you’ll see errors “Too many connections”. The solution might be to increase max_connections parameter (if instance can handle) or ideally fix app to use fewer.

**10.7 Docker and Container Issues:**  
- *Disk space on ECS instances:* Docker images and containers use space on the EC2 instance (usually the root EBS volume). Over time, old images or stopped containers could accumulate. The ECS optimized AMI comes with a cron to clean up, but sometimes you might need to manually prune if volume gets full. Monitor EC2 instance disk (CloudWatch can monitor instance free storage if you install CloudWatch agent or CloudWatch has basic metrics for certain AMIs). If it fills, tasks might fail to start (can't pull image).
- *Memory/CPU settings:* If tasks consistently use near their limits, consider adjusting resource settings. Also note that ECS EC2 mode scheduling: if a task has e.g. 256 MB reserved, ECS will ensure that is free on an instance before placing it. If you give tasks too high reservation, ECS might under-pack the instances. On the flip side, if you only give 256MB but they actually use 400MB, you'll see OOM. So calibrate tasks' resource needs.

**General Troubleshooting Tips:**  
- Use CloudWatch Logs and metrics as first go-to for any issue.
- AWS re:Post (forums) and documentation often have specific Q&As for error messages. For example, searching an error text often finds someone who faced it.
- **Reproduce systematically:** If something fails occasionally, try to identify pattern. E.g., only first deploy fails? Or fails under load? That can hint at race conditions or missing dependencies.
- **Divide and Conquer:** Isolate layers. If deployment fails, can you manually deploy that task def via ECS console (bypass CodeDeploy) to see if it works? If manual works, issue may be with CodeDeploy config.
- **Enable More Logging:** If app is a black box, add logging around the suspect area. For AWS services, some have debug modes (like ECS agent can log more if needed).
- **Rollback Capability:** In CodeDeploy we have automated rollback. But also consider keeping previous stable images tagged (don’t override latest if that’s stable version; use version tags). If pipeline deployed a bad version, being able to quickly redeploy the last known good image is important (either via CodeDeploy or manually updating service with old image). Our pipeline as set would require pushing a code revert to redeploy old code, which is fine in many cases (since CodeDeploy already rolled back to old tasks in event of failure).
- **Plan for Disaster Recovery:** Although not a typical “troubleshoot”, it’s related: think how to recover if an entire region outage or data loss occurs. E.g., Aurora global database or cross-region read replica for DR, or keeping an AMI of ECS instances to launch elsewhere, CodePipeline is region-specific so maybe have pipelines in backup region ready. This is beyond day-to-day issues but good for advanced planning.

By addressing issues methodically, one can maintain a robust CI/CD pipeline and deployment environment. Next, we’ll discuss how to optimize costs in this architecture, ensuring we use these AWS services cost-effectively.

## **11. Cost Optimization and AWS Billing Considerations**  
Running a full CI/CD pipeline and microservices architecture on AWS incurs costs across various services. As an advanced user, it’s important to understand the cost model of each component and implement strategies to optimize spending. Here, we break down the key cost contributors in our setup and provide optimization tips:

**11.1 AWS Service Costs Breakdown:**

- **CodeCommit:** AWS CodeCommit charges **$1 per active user per month** after the first 5 users ([Use Hosted Git Repositories from AWS CodeCommit with AWS Elastic Beanstalk](https://aws.amazon.com/about-aws/whats-new/2016/10/use-hosted-git-repositories-from-aws-codecommit-with-aws-elastic-beanstalk/#:~:text=AWS%20CodeCommit%20%20is%20a,to%20learn%20more%20about%20pricing))  For small teams or personal projects, CodeCommit is essentially free (up to 5 users, with generous storage and Git requests included). If you exceed 5 users, plan for $1/user. *Optimization:* If you have a large team, cost scales linearly per user, which is still typically cheaper than hosting a secure Git server. But if a team is extremely large, consider if consolidated repos or limiting user access (only active users count) helps. Remove any stale users.

- **CodePipeline:** CodePipeline costs **$1 per active pipeline per month** (first pipeline is free in free tier) ([Now Available – AWS CodePipeline | AWS News Blog](https://aws.amazon.com/blogs/aws/now-available-aws-codepipeline/#:~:text=You%E2%80%99ll%20pay%20%241%20per%20active,the%20course%20of%20a%20month))  “Active” means it ran at least once that month. Our single pipeline is $1/month, which is negligible. If you had many microservices each with its own pipeline, that could become a few dollars per month – still low. If you use more advanced features like higher frequency polling (for S3 sources) or multiple pipelines per environment (dev/prod), it marginally adds. *Optimization:* Keep pipeline count minimal by combining where practical (but don’t combine unrelated projects just to save $1 – clarity is more important). Inactive pipelines (no triggers) incur no charge.

- **CodeBuild:** Charged by the minute of build time. Different instance sizes have different per-minute rates. For example, a Linux small instance might be about $0.003/minute (check AWS pricing). Our build might take ~5 minutes, so that’s ~$0.015 per build. If you do 100 builds a month, it’s $1.5. Also, first 100 minutes per month are free on the small instance ([Managed Build Server - AWS CodeBuild Pricing](https://aws.amazon.com/codebuild/pricing/#:~:text=The%20CodeBuild%20AWS%20Free%20Tier,Using%20on))  *Optimization:* Use the smallest instance type that can handle your build. For instance, building our project on a small might be fine. Only go larger if builds are notably CPU/memory constrained. Also, avoid running builds more often than necessary (e.g., don’t trigger on every single commit if not needed; maybe trigger on merges to main or use test phases to gate). The free tier covers a lot for moderate use. If you have many microservices, you could reuse a CodeBuild project for multiple pipelines to reduce duplicate configuration, but cost wise it wouldn’t change much – you pay by build time consumed.

- **CodeDeploy:** CodeDeploy itself has **no additional charge**. It’s free to use for EC2, ECS, Lambda deployments. You pay for the resources it deploys. So no direct cost except minimal API calls. *Optimization:* N/A (free). Just be aware if you start using CodeDeploy hooks with Lambda functions, those Lambdas would be billed for invocation time (but likely negligible).

- **ECR (Elastic Container Registry):** Charges for storage and data transfer. Currently, AWS offers 500 MB-month storage free per month. Beyond that, it’s around $0.10 per GB-month (depending on region). Also, data transfer (pull/push) within same region from EC2 or Lambda is free; across regions or to internet costs extra. Our images might be, say, 200 MB each. If we keep many versions, it could accumulate. *Optimization:* Implement a **lifecycle policy** in ECR to delete old images (e.g., keep last N images or last N days). That ensures you don’t pay for storing outdated images. Also compress/minify images to reduce size (e.g., use alpine base images or multistage builds to remove dev dependencies). For data transfer, ensure ECS and CodeBuild pulling from ECR are in same region (they are, since it’s all in one environment). If you ever deploy cross-region, consider replicating images or accept transfer costs.

- **ECS (EC2 Launch) & EC2 Instances:** The ECS service (control plane) itself has no cost. Cost comes from the **EC2 instances** in the cluster and whatever EBS volumes they use, plus the data transfer. For instance, if we use 2 x t3.small instances (each maybe ~$0.02/hr in us-west-2), that’s $0.04/hr, about $29/month. Auto scaling can increase that if load increases; on average maybe it might scale between 2 and 4 instances, so budget accordingly. Also consider EBS volume: ECS-optimized AMI might default to 30GB gp2 volume ($3/month each). So another few dollars. *Optimization:* 
  - Use **Savings Plans or Reserved Instances** for EC2 if you have steady state usage. For example, if you commit to that baseline of 2 instances for 1 year, you can save ~30-40% compared to on-demand. 
  - If workloads are dev/test or can tolerate interruptions, consider using **Spot Instances** for ECS. These can be 70% cheaper. You might combine a base of on-demand and rest spot. But handle the fact they can be terminated (ECS will reschedule tasks).
  - Choose right **instance types**: burstable t3 family is cost-effective for spiky workloads (and with unlimited mode, might incur some surplus credits charges if constantly high CPU, so monitor CPU credits). If constant high CPU, a fixed performance instance like m5 might be more cost-efficient per CPU.
  - **Shut down non-production clusters** when not needed. If this pipeline is for production, it’s 24/7. But if you had a staging environment, you could downscale it to 0 at night or use smaller instances in staging.
  - Use auto scaling to only run needed instances as we set – that inherently optimizes cost by not running excess capacity.

- **Auto Scaling:** The Auto Scaling service itself doesn’t cost extra. You pay for the instances it launches. CloudWatch alarms used for auto scaling are small cost (first 10 alarms free, then ~$0.10 per alarm per month). We likely have a few alarms (CPU high/low, etc.), that’s negligible (<$1).

- **Network Load Balancer:** NLB pricing has two components:
  - An hourly fixed cost per NLB ($0.0225 per hour in us-west-2, roughly $16/month).
  - Data processing cost per GB (called an LCU charge, which for NLB is mainly based on new connections and data volume; typically $0.008 per LCU-hour, each LCU includes 1 GB traffic).
  If our traffic is moderate, say 100 GB/month, that might be < $1 in processing. So the bulk is the hourly charge. *Optimization:* If you had multiple microservices, each with their own NLB, you pay that per LB. You might consolidate services under one load balancer using different listener ports or using an ALB (with host/path routing) to cover multiple services behind one ALB (ALB costs are similar, ~$0.025/hr and LCUs for data). In our list, they specifically wanted NLB and two AZs. If cost is an issue and HTTP is main protocol, ALB could reduce number of LBs if many services can share it via domains or paths. But using one NLB for all is possible only if you multiplex by port (which is not user-friendly beyond a point).
  - Use AWS Free Tier: covers 750 hours and 15 GB for load balancers for 12 months (if this is a new account, the LB might be partly free for first year).
  - Ensure you enabled idle connection timeouts appropriately; but with NLB, not much tunable there.

- **CloudWatch Logs and Metrics:** Basic monitoring metrics (for EC2, ECS, ALB, etc.) are free. Detailed custom metrics or high-resolution metrics can incur cost. CloudWatch Logs costs about $0.50 per GB ingested and $0.03 per GB stored per month (after some free tier). Our application logs – if it generates 1MB of logs per hour, that’s ~720MB per month, costing ~$0.36 ingestion and if stored for say 1 month retention, negligible storage. If logs are very verbose (multiple MBs per minute), logs cost can outrun other costs – so be mindful of log volume. *Optimization:* 
  - Adjust log level in production (e.g., info or warn, not debug) to reduce volume.
  - Set log retention so old logs are deleted.
  - You can also use compression or offload logs to S3 for cheaper storage if needed. But likely not necessary unless huge.
  - Metrics: if you use custom metrics, each can incur $0.30/month per metric. But likely not many custom. Container Insights adds some cost (about $0.50 per ECS task per month plus $0.04 per GB of logs). If budget is tight, you can choose to not enable Container Insights and rely on default metrics.

- **Aurora MySQL:** This is usually a significant cost component:
  - Aurora pricing: Charged per instance hour + storage + I/O. Suppose we used 2x db.t3.small instances. On-demand, that’s ~$0.04 per hour combined (one writer, one reader), ~ $29/month each, so ~$58/month total for both. Storage is $0.1 per GB-month (but 10GB base costs $1). I/O throughput cost might add a bit but for light usage it’s small.
  - Aurora has a minimum of 2 instances for multi-AZ (with 1 reader). If you need to cut cost and can accept downtime during failover, you could run a single instance (Aurora without a replica) – it’s still fault-tolerant storage but if the instance/AZ fails, it has to reboot in another AZ which takes a bit longer.
  - Alternatively, use Amazon RDS MySQL with Multi-AZ (which charges for standby but not serving reads) – cost is similar though, or slightly cheaper than Aurora for small scales.
  - *Optimization:* If usage is low (dev/test), consider **Aurora Serverless v2** which can scale down to a fraction of a CPU when idle, charging per second. That can be cost-efficient for spiky or low-average workloads. But for a consistent load, provisioned might be better. For dev, even Aurora Serverless v1 or turning DB off when not needed (RDS can’t easily “stop” Aurora except in serverless mode).
  - Turn on storage auto-scaling so you don’t over-provision storage. It grows as needed (Aurora does this by default).
  - Use the smallest instance class that meets performance needs. If t3.small shows high CPU often, might need medium. But if small is enough, no need to use bigger.
  - If your microservice could use a simpler database for cheap (like DynamoDB or serverless db), that could save cost, but that’s a different architecture.

- **Data Transfer Costs:** 
  - Within same region and AZ, most traffic is free (like ECS tasks to Aurora, or tasks to NLB in same AZ). Cross-AZ traffic can incur charges: NLB cross-AZ feature (if enabled) duplicates traffic to both AZs, but by default NLB in each AZ will only send to targets in that AZ unless you enable cross-zone load balancing (which for NLB is off by default, and there's a charge if enabled). We likely left cross-zone off (which is fine, it means each LB node only targets same AZ instances).
  - Aurora’s replication traffic between AZs is not charged to you explicitly (it’s part of Aurora cost).
  - If clients of your service are on the internet, the data out from NLB to internet is charged as AWS Data Transfer Out (roughly $0.09/GB after first free GB). If this is an internal service, then minimal cost. *Optimization:* Use caching/CDN if applicable for large data to reduce output. Or if clients are within AWS (e.g., API calls from an EC2 in same region), that’s cheaper (intra-region typically $0.01/GB or free if same AZ and using private IP).
  - Keep an eye on **NAT Gateway** costs if you have one for instances (for pulling patches or Docker images). NAT has hourly + per GB fee ($0.045/hour + $0.045/GB). For our fairly static environment, NAT usage is small (just OS updates occasionally). If using NAT heavily for other stuff, consider that cost.

- **Other Minor Costs:** 
  - **IAM:** no direct cost for creating roles/policies.
  - **SNS/CloudWatch Alarms:** if we set up many alarms or notifications, each alarm after free tier is ~$0.10/month. A few alarms won’t exceed $1 or $2.
  - **CloudWatch Events rules:** small cost per rule per month ($1 or less).
  - **Parameter Store/Secrets Manager:** if we used Secrets Manager for DB password, that’s $0.40 per secret per month and small API call charges. Parameter Store standard parameters are free, advanced are $0.05/parameter-month. We didn’t explicitly use them in our design, but good to note.
  - **AWS Support Plan:** If you have a support plan (Business or higher), that’s a % of your AWS spend. So optimizing costs also reduces support cost proportionally.

**11.2 Right-Sizing and Efficiency:**

- We chose services appropriate for our scale. For example, using ECS on EC2 may be cheaper for steady usage than Fargate (Fargate charges per vCPU-minute and might cost more if instances are fully utilized). But for very small or sporadic workloads, Fargate can be more cost-efficient by not having idle EC2. Advanced users might weigh ECS+EC2 vs ECS+Fargate. At a small scale, the difference might be minor; Fargate avoids paying for unused capacity but per unit cost is higher. Since we have auto scaling to 0 tasks, we could even scale ECS to 0 tasks and 0 instances when no load, though CodeDeploy expects a service running, so probably not 0. 
- We used Aurora which is high-end; if cost was priority over performance, one could use a single-AZ RDS MySQL (cheaper, but not highly available) or even an EC2 self-managed MySQL (not recommended for production due to management overhead and less HA).
- Consider **multi-account strategy for cost allocation:** If this was part of an organization, use AWS Cost Explorer and cost allocation tags to track which service is incurring what. For example, tag resources with Project:MicroservicesDemo, Environment:Prod. That helps identify top cost drivers in reports.

**11.3 Monitoring Costs and Budgets:**

- Use **AWS Budgets** to set a monthly budget and get alerts if you approach thresholds. For instance, set a budget of $100 and notify if 80% used. This is prudent for any project to catch unexpected cost spikes (maybe if a bug causes infinite log loop generating huge logs, you’d see CloudWatch costs spike).
- Check the **AWS Cost and Usage Report** or Cost Explorer regularly. It can show costs by service over time. You might discover, e.g., “CloudWatch Logs suddenly $20, why?” and then find a noisy log.
- Remember that as usage grows (which is good, means you have more traffic), costs will grow too. Ensure your scaling strategy ties into cost control (scale-out gives better service but also costs more; sometimes put an upper limit to prevent runaway cost in case of a bug or abuse).

By applying these strategies, you can run this advanced architecture in a cost-effective manner. For instance, a rough cost estimation for a small production setup following our design (assuming moderate usage):
- 2 EC2 t3.smalls (~$30/mo), Aurora 2x small (~$60/mo), NLB ($16/mo), pipeline & build & misc ($5), total ~$111/month. With optimizations like reserved instances and turning off dev environment, you could reduce this further. For the value of an automated CI/CD and highly available app, this is quite reasonable. As you scale, each additional service or higher load will add costs linearly, but by then presumably the application is delivering value to justify it.

Finally, let’s conclude with a summary of best practices and key takeaways from this guide.

## **12. Conclusion and Best Practices Recap**  

In this extensive guide, we built a robust, project-based CI/CD pipeline for deploying microservices on AWS. Let’s summarize the journey and highlight the best practices we applied:

- **Infrastructure as Code & Automation:** Although we used the console for demonstration, in real scenarios consider defining all these resources in code (CloudFormation, Terraform, or AWS CDK). This allows versioning and reusing the setup. For instance, a CloudFormation template can define the CodePipeline with all stages, ECS cluster, etc. Automation reduces manual errors and makes spin-up of new environments faster.

- **Microservices Decoupling:** We deployed one service, but the architecture supports multiple independent microservices. Each should have its own repository, pipeline, and ECS service, allowing teams to deploy them separately. We emphasized how components like CodePipeline and ECS can scale to handle many services (with practices like one pipeline per service, or using monorepo with conditional builds – depending on team preference).

- **CI/CD Best Practices:** 
  - We enabled continuous integration by automatically building and testing code on each commit. Make sure to run unit tests in CodeBuild to catch issues early (we described where tests would go in buildspec). 
  - For continuous delivery, we used a **Blue/Green deployment** strategy via CodeDeploy to achieve zero downtime. This is a best practice for microservices to allow rapid, safe deployments ([Tutorial: Create a pipeline with an Amazon ECR source and ECS-to-CodeDeploy deployment - AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials-ecs-ecr-codedeploy.html#:~:text=In%20this%20tutorial%2C%20you%20configure,if%20there%20is%20an%20issue))  
  - We integrated health checks and automated rollback on failures. This minimizes the impact of bad deployments.
  - We also mentioned using staging environments or manual approvals (didn’t implement here, but for critical production deploys, you might have a manual approval step in CodePipeline between test and prod deploy).

- **Security Best Practices:** 
  - Used **IAM roles** with least privilege for each component (CodePipeline, CodeBuild, ECS tasks). This confines access scope if any component is compromised ([AWS Identity and Access Management (IAM) Best Practices](https://aws.amazon.com/iam/resources/best-practices/#:~:text=the%20goal%20of%20achieving%20least,IAM%20provides%20last%20accessed)) 
  - Ensured no credentials in code by using IAM roles for access (ECS pulling from ECR, etc.) and suggested using Secrets Manager/Parameter Store for sensitive data like DB passwords (so not plain text in task definitions).
  - The ECS tasks and Aurora DB are in private subnets (no direct internet access to them), and only the NLB (in public subnet) exposes the service to the internet. This reduces attack surface.
  - Enabled encryption at rest where applicable (ECR images are encrypted by default, Aurora storage is encrypted if chosen, etc.). Always consider encryption for sensitive data.
  - Enforced multi-AZ for resilience (which is also a part of security – high availability). Our cluster, load balancer, and DB all span at least 2 AZs, protecting against data center failure.
  - We didn’t explicitly implement WAF (Web Application Firewall) or API Gateway in front of the service, but that could be an additional best practice for public APIs to mitigate attacks and provide throttling, etc.

- **High Availability and Fault Tolerance:** 
  - Multi-AZ deployment of ECS and Aurora for fault tolerance ([Deployments on an Amazon ECS Compute Platform - AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html#:~:text=Application%20Load%20Balancer%20or%20Network,Load%20Balancer))  ([Relational Database – Amazon Aurora MySQL PostgreSQL Features – AWS](https://aws.amazon.com/rds/aurora/features/#:~:text=On%20instance%20failure%2C%20Aurora%20uses,RDS%20Proxy%20to%20reduce%20failover)) 
  - Load balancer distributes traffic across AZs, and auto scaling ensures replacement of failed components.
  - We set auto scaling so the system self-heals and adjusts to load, which is key for both reliability and cost.
  - In production, you’d test these: e.g., terminate an EC2 instance to see ECS tasks migrate (they will, due to desired count and ASG).
  - Also, we can use AWS Fault Injection Simulator to test resilience by creating failures, a newer best practice (chaos engineering).

- **Monitoring and Observability:** 
  - We set up CloudWatch logging for troubleshooting. Logs are invaluable for debugging issues in distributed systems.
  - Metrics and alarms help catch issues proactively (like high CPU or failing deployments).
  - Tracing (X-Ray) and structured logging can further help follow a request across microservices (for when there are multiple and they call each other).
  - We stressed the importance of reviewing pipeline and system logs after each deployment to ensure everything is as expected, and being alerted to anomalies.

- **Cost Management:** 
  - We broke down costs and provided optimizations – always review your AWS bills and identify any surprise charges early (e.g., a logging explosion or a forgotten large instance).
  - Use auto scaling to save cost when possible (scale in at low load).
  - Consider the pricing models: commit to 1-year or 3-year reserved instances or savings plans for ECS EC2 and Aurora if this is a long-running production (this can significantly reduce costs if you know the baseline resources you’ll always use).
  - If this environment is for dev/test, tear it down when not in use or use smaller resources. For example, developers can use on-demand ephemeral environments (there are tools like AWS Copilot or CDK pipelines that could create a test environment on a branch and destroy it after tests).
  - Evaluate alternative AWS services: for instance, AWS Fargate removes the EC2 management but might be slightly more cost for continuous loads; AWS Lambda (Serverless) could handle some microservices cheaper if they are event-driven and low-throughput. We chose ECS on EC2 to align with the requirement, but always tailor to your use case.

- **DevOps Culture:** 
  - Even though it’s not a tangible “AWS setting”, fostering a culture where developers can push code and see it deployed in minutes (as our pipeline allows) is immensely beneficial. It encourages frequent, small updates, which reduce risk (since changesets are small) and improve time to market.
  - Ensure to have rollback procedures in place (our CodeDeploy does auto-rollback on failure; also practice manual rollback, e.g., redeploy previous image, so team is confident in handling incidents).
  - Use this pipeline not just for deploying to prod, but also to automate deployments to staging or test environments with different triggers or branches. Perhaps have a pipeline per environment or a single pipeline with multiple deploy stages.

By following this guide, you’ve implemented a state-of-the-art deployment pipeline that leverages many AWS services in concert. You have source control, continuous integration, containerization, continuous delivery with blue/green deployments, scalable infrastructure, and robust monitoring – essentially covering the full DevOps cycle on AWS. This setup can serve as a strong foundation for running microservices in production.

Keep learning and iterating: AWS releases new features regularly (for example, new ECS features, Blue/Green deployments directly in CodePipeline without CodeDeploy, etc.), so stay updated. And always test changes in a safe environment.

**Final Tip:** Document your specific setup and configurations for your team. Use diagrams to map out this architecture (VPC diagram showing subnets, ECS, LB, etc., and a flow diagram of CodePipeline stages). This will help onboard others and serve as reference when making changes.

Congratulations on building an advanced AWS CI/CD pipeline for microservices! You have combined tools like CodePipeline, ECS, and Aurora into a cohesive system that is highly automated and resilient. With this knowledge, you can confidently deploy and manage complex cloud applications following DevOps best practices.

