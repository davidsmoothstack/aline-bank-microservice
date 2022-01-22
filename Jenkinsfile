// Leave blank
def jenkinsImage = ""

final repositoryUrl = "https://github.com/davidsmoothstack/aline-bank-microservice.git"
final branchName = "dev"
final ecrUrl = "862167864120.dkr.ecr.us-east-1.amazonaws.com"
final ecrRepoName = "dw-bank-microservice"

pipeline {
    agent any

    tools {
        maven "Maven 3.8.4"
    }

    parameters {
        string description: "Url of the repository. Example: https://github.com/davidsmoothstack/aline-bank-microservice.git", defaultValue: "${repositoryUrl}", name: "REPO_URL", trim: true
        string description: "Remote location of the repository", defaultValue: "origin", name: "REMOTE_LOCATION", trim: true
        string description: "Name of the branch to checkout", defaultValue: "${branchName}", name: "BRANCH_NAME", trim: true
        string description: "ECR Url. Example 862167864120.dkr.ecr.us-east-1.amazonaws.com", defaultValue: "${ecrUrl}",  name: "ECR_URL", trim: true
        string description: "ECR Repository Name. Example dw-bank-microservice", deafultValue: "${ecrRepoName}", name: "ECR_REPOSITORY_NAME", trim: true
        booleanParam defaultValue: true, description: "Run the tests during the build?", name: "RUN_TESTS"
    }

    stages {
        stage("Clone") {
            steps {
                git "${params.REPO_URL}"
            }
        }

        stage("Checkout") {
            steps {
                sh "git checkout ${params.REMOTE_LOCATION}/${params.BRANCH_NAME}"
                sh "git submodule init && git submodule update"
            }
        }

        stage("Build Maven Project") {
            steps {
                sh "mvn clean package -Dmaven.test.skip=true"
            }
        }

        stage("Test") {
            when {
                expression { params.RUN_TESTS == true }
            }

            steps {
               sh "mvn test"
            }
        }

        stage("Build Docker Image") {
            environment {
                COMMIT_HASH = "`git log -1 --pretty=format:\"%h\"`"
            }

           
            steps {
                script {
                    jenkinsImage = "jenkins-build:${COMMIT_HASH}"
                }

                sh "docker build -t ${jenkinsImage} ."
            }
        }

        stage("Deploy Image") {
            environment {
                COMMIT_HASH = "`git log -1 --pretty=format:\"%h\"`"
                ECR_IMAGE_TAG_LATEST = "${ECR_URL}/${ECR_REPOSITORY_NAME}"
                ECR_IMAGE_TAG_HASH = "${ECR_IMAGE_TAG_LATEST}:${COMMIT_HASH}"
            }
            
            steps {
                echo "Deploying image to ECR..."

                withCredentials([[
                    $class: "AmazonWebServicesCredentialsBinding",
                    credentialsId: "AWS-Credentials",
                    accessKeyVariable: "AWS_ACCESS_KEY_ID",
                    secretKeyVariable: "AWS_SECRET_ACCESS_KEY"
                ]]) {
                    sh "aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ECR_URL}"
                    sh "docker tag ${jenkinsImage} ${ECR_IMAGE_TAG_LATEST}"
                    sh "docker tag ${jenkinsImage}  ${ECR_IMAGE_TAG_HASH}"
                    sh "docker push -a ${ECR_IMAGE_TAG_LATEST}"
                }
            }
        }
    }
}