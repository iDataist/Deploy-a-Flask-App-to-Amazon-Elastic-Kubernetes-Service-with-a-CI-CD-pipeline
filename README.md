# Creating a CI/CD pipeline with AWS Code Build and Code Pipeline

## Overview
I containerized a Flask App using Docker and deployed the App to a Kubernetes cluster with a CI/CD pipeline using Cloudformation, Codebuild and CodePipeline. I associated the pipeline's one end to the Github repository (containing the application code), and connected the other end to the EKS cluster. Below is the architecture diagram of the deployed App with four main actions.
![](architecture.png)
1. Code check-in - The application code for the Flask app is hosted on Github. Multiple contributors can contribute to the repository collaboratively. 

2. Trigger Build - As soon as a commit happens in the Github repository, it will trigger the CodeBuild. This step requires connecting the Github account to the AWS CodeBuild service using a GitHub access token. Codebuld will build a new image for the application.

3. Automatic Deployment - The CodePipeline service will automatically deploy the application image to the Kubernetes cluster.

4. Service - Kubernetes cluster will start serving the application endpoints. To the end-user, there is an abstraction of the internal details of the cluster, such as nodes, PODs, containers, and the architecture.

## Flask App
The Flask app consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Dependencies
- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).
- AWS, EKSCTL and KUBECTL CLI

## Project Steps
### 1. Run the API Locally using the Flask Server
- Install python dependencies: These dependencies are kept in a `requirements.txt` file in the root directory of the repository. To install them, go to the project directory and run the command `pip install -r requirements.txt`.

- Set up the environment: You do not need to create an `.env` file to run locally but you do need two variables available in your terminal environment:
    - JWT_SECRET - The secret used to make the JWT, for the purpose of this course the secret can be any string.
    - LOG_LEVEL - It represents the level of logging. It is optional to be set. It has a default value as 'INFO', but when debugging an app locally, you may want to set it to 'DEBUG'.

    To add these to your terminal environment, run the following:
    ```
    export JWT_SECRET='myjwtsecret'
    export LOG_LEVEL=DEBUG
    echo $JWT_SECRET
    echo $LOG_LEVEL
    ```
- Run the app: Run the app using the Flask server, from the root directory of the downloaded repository, run `python main.py`.Open http://127.0.0.1:8080/ in a new browser OR run curl --request GET http://localhost:8080/ on the command-line terminal. It will give you a response as "Healthy".

- Install a command-line JSON processor: Before trying to access other endpoints of the app, we need the jq, a package that helps to pretty-print JSON outputs. In simple words, the JQ package helps you parse, filter, or modify JSON outputs. Open a new terminal window, and run the command below. For Linux, run `sudo apt-get install jq`. For Mac, run `brew install jq`. For Windows, run `chocolatey install jq`.

- Access endpoint /auth
To try the /auth endpoint, use the following command, replacing email/password as applicable to you 
    ```
    export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
    ```
    This calls the endpoint 'localhost:8080/auth' with the email/password as the message body. The return value is a JWT token based on the secret string you supplied. We are assigning that secret to the environment variable 'TOKEN'. To see the JWT token, run `echo $TOKEN`.

- Access endpoint /contents: To try the /contents endpoint which decrypts the token and returns its content, run `curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .` You should see the email id that you passed in as one of the values.

### Containerize the Flask App and Run Locally

- Prerequisite - Docker Desktop: If you haven't installed Docker already, you should install now using [these installation instructions](https://docs.docker.com/get-docker/).

- Storing Environment variables: Create a file named .env_file and save both JWT_SECRET and LOG_LEVEL into .env_file. This .env_file is only for the purposes of running the container locally, you do not want to push it into the Github or other public repositories. You can prevent this by adding it to your .gitignore file, which will cause git to ignore it. To safely store and use secrets in the cloud, use a secure solution such as AWS’s parameter store.

- Build an image: Build a local Docker image, by running the commands below from the directory where your Dockerfile resides.
    ```
    docker build -t myimage .
    # Check the list of images
    docker image ls
    # Remove any image
    docker image rm <image_id>
    ```
- Create and run a container by running:
    ```
    docker run --name myContainer --env-file=.env_file -p 80:8080 myimage

    # List running containers
    docker container ls
    docker ps
    # Stop a container
    docker container stop <container_id>
    # Remove a container
    docker container rm <container_id>
    ```
- Check the endpoints: To use the endpoints, you can use the same curl commands as before, except using port 80 this time. Open a new terminal window, and try the following command:
    ```
    # Flask server running inside a container
    curl --request GET 'http://localhost:80/'

    # Flask server running locally (only the port number is different)
    curl --request GET 'http://localhost:8080/'

    # Calls the endpoint 'localhost:80/auth' with the email/password as the message body. 
    # The return JWT token assigned to the environment variable 'TOKEN' 
    export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"WindowsPwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
    echo $TOKEN

    # Decrypt the token and returns its content
    curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
    ```
### Create an EKS Cluster and IAM Role
1. Create an EKS (Kubernetes) Cluster
    - Create an EKS cluster named “simple-jwt-api” in a region of your choice:
        ```
        # Create cluster in the default region
        eksctl create cluster --name simple-jwt-api
        # Create a cluster in a specific region, such as us-east-2
        eksctl create cluster --name simple-jwt-api --region=us-east-2
        ```
    - Verify: You can go to the CloudFormation or EKS web-console to view the progress. If you don’t see any progress, be sure that you are viewing clusters in the same region that they are being created. Once the status is CREATE_COMPLETE in your command line, check the health of your clusters nodes `kubectl get nodes`.

    - Delete by running `eksctl delete cluster simple-jwt-api  --region=<REGION>`

2. Create an IAM Role
    - Get your AWS account id by running `aws sts get-caller-identity --query Account --output text`. 

    - Create a trust relationship. To do this, create a blank trust.json file, add the following content to it and replace the <ACCOUNT_ID> with your actual account Id. This policy file defines the actions allowed by whosoever assumes the new Role.
        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
        ```
    - Create a role, 'FlaskDeployCBKubectlRole', using the trust.json trust relationship:
        ```
        aws iam create-role --role-name FlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
        ```
    - Create a policy document, `iam-role-policy.json`, that allows (whosoever assumes the role) to perform specific actions (permissions) - "eks:Describe*" and "ssm:GetParameters" as:
        ```
        {
            "Version": "2012-10-17",
            "Statement":[{
                "Effect": "Allow",
                "Action": ["eks:Describe*", "ssm:GetParameters"],
                "Resource":"*"
            }]
        }
        ```
    - Attach the iam-role-policy.json policy to the 'FlaskDeployCBKubectlRole' by running:
        ```
        aws iam put-role-policy --role-name FlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
        ```
    - Verify the newly created role in the IAM service
3. Allowing the new role access to the cluster: Before you assign the new-role to the CodeBuild service (so that CodeBuild can also administer the cluster) you will have to add an entry of this new role into the 'aws-auth ConfigMap'. The aws-auth ConfigMap is used to grant role-based access control to your cluster. When your cluster is first created, the user who created it is given sole permission to administer it. Therefore, to grant any AWS service/user who will assume this role the ability to interact with your cluster, you must edit the 'aws-auth ConfigMap' within Kubernetes.

- Fetch: Get the current configmap and save it to a file:
    ```
    # Mac/Linux
    # The file will be created at `/System/Volumes/Data/private/tmp/aws-auth-patch.yml` path

    kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml

    # Windows 
    # The file will be created in the current working directory

    kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml
    ```
- Edit: Open the aws-auth-patch.yml file using any editor, such as VS code editor:
    ```
    # Mac/Linux
    code /System/Volumes/Data/private/tmp/aws-auth-patch.yml
    # Windows
    code aws-auth-patch.yml
    ```
    Add the following group in the data → mapRoles section of this file. YAML is indentation-sensitive, therefore refer to the snapshot below for a correct indentation:
    ```
    mapRoles: |
        - groups:
        - system:masters
        rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
        username: build      
    ```

- Update: Update your cluster's configmap:
    ```
    # Mac/Linux
    kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

    # Windows
    kubectl patch configmap/aws-auth -n kube-system --patch "$(cat aws-auth-patch.yml)"
    ```
    The command above must show you `configmap/aws-auth patched` as a response.



