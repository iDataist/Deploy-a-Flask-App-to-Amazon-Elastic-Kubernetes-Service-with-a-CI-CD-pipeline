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
