pipeline{
    agent any 
    environment {
        APP_REPO = 'https://github.com/AnnaPeretiatka/RepoB-app.git'
        IMAGE_NAME = "annapp" # name on ec2-docker
        REGISTRY = "992382545251.dkr.ecr.us-east-1.amazonaws.com" // AWS ECR registry
        ECR_REPO = "${REGISTRY}/${IMAGE_NAME}" // ECR repo for pushing
        EC2_IP = '34.226.124.69'
        REGION = 'us-east-1'
    }
    
    stages {
        // CI stages
        stage('chekout Pipline Repo'){ //neede if jenkinsfile is in another repo (not  with app)
            steps {
                git branch: 'main', url: 'https://github.com/AnnaPeretiatka/RepoA-infra.git' //clone repo with jenkins file so jenkins cacn load pipline definition
            }
        }
        stage('checkout App Repo') {
            steps{
                dir('annapp') {  //tells jenkins to run step inside 'annapp' subdir in jenkins workspace
                    git branch: 'main', url: "${APP_REPO}" // clone app-repo(B)
                }
            }
        }
        stage('Build Docker Image') { //builds docker image from app code
            steps {
                dir('annapp') { // run step inside 'annapp' subdir in jenkins workspace
                    sh "docker build -t ${IMAGE_NAME} ."
                }
            }
        }
        stage('Unit Tests') { // run unit tests inside docker container
            steps {
                dir('annapp') { // run step inside 'annapp' subdir in jenkins workspace
                    sh "docker run --rm ${IMAGE_NAME} pytest -q tests/test_app.py"
                }
            }
        }
        /*
        // CD STAGES (Option A: local-only deploy)
        // ---------------------------
        stage('Deploy Locally (EC2)') {
            // CD step: deploys container directly on the same EC2 that runs Jenkins.
            // Notice the port mapping: host 5000 → container 8080 (to avoid Jenkins 8080 clash).
            steps {
                sh '''
                  docker stop annapp || true
                  docker rm annapp || true
                  docker run -d --name annapp -p 5000:8080 ${ECR_REPO}:latest
                '''
            }
        }

        stage('Health Check Local') {
            // CD validation: verifies that app is up via curl on EC2 public IP:5000
            steps {
                sh "curl -f http://${EC2_IP}:5000/ || exit 1"
            }
        }
        */
         // CD stages (Option B: Push to ECR + deploy)
         stage('Login, Tag & push Docker Image to ECR') {
            // stage purpose: upload image from jenkins to ECR (image available on cloud)
            // This is CD preparation, but not yet running anything on the EC2 host.
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: '992382545251' // anna aws account id
                ]]) {
                    sh """
                        echo "Logging in to ECR..."
                        aws ecr get-login-password --region ${REGION} | \
                        docker login --username AWS --password-stdin ${REGISTRY}

                        echo "Tagging Docker image..."
                        docker tag ${IMAGE_NAME}:latest ${ECR_REPO}:latest

                        echo "Pushing Docker image to ECR..."
                        docker push ${ECR_REPO}:latest
                    """
                }
            }
        }
        stage('Deploy From ECR (EC2)') { 
            //Purpose: Actually run the container on your EC2 host
            //Stops old container if it exists, removes it, pulls latest image from ECR and runs it
            steps {
                // first stop and remove old container if it exists (docker run -d name annapp)
                // || true - prevents Jenkins from failing if the container doesn’t exist (first run)
                sh """
                    docker stop annapp || true
                    docker rm annapp || true

                    echo "Logging in again to ECR..."
                    aws ecr get-login-password --region ${REGION} | \
                    docker login --username AWS --password-stdin ${REGISTRY}

                    echo "Pulling and running container..."
                    docker pull ${ECR_REPO}:latest
                    docker run -d --name annapp -p 5000:8080 ${ECR_REPO}:latest
                """
            }
        }
        satge('Health Check ECR Deploy'){
            steps {
                // CD alidation again after ECR-based deployment
                sh "curl -f http://${EC2_IP}:5000/ || exit 1"
            }
        }
    }
}