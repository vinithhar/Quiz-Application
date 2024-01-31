def dockerUsername
def dockerPassword

pipeline  {
  agent { label 'slave-agent' }
  
  triggers {
    githubPush()
  }
  
  environment {
        DOCKER_USERNAME = credentials('prudhvi-docker-username')
        DOCKER_PASSWORD = credentials('prudhvi-docker-password')
  }
  
  stages  {
    stage('Initialize'){
      steps  {
        script  {
          def dockerHome = tool "myDocker"
          env.PATH = "${dockerHome}/bin:${env.PATH}"
        }
      }  
    }
    stage('Checkout') {
            steps {
                script {
                    git branch: 'development', url: 'https://github.com/prudhvitejad/Quiz-Application.git'
                }
            }
        }
    stage('Build and Push Docker Image') {
    steps {
        script {
            def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
            
            dockerUsername = getSecretText("prudhvi-docker-username")
            dockerPassword = getSecretText("prudhvi-docker-password")
            sh "echo dockerUsername=${dockerUsername}"
            sh "echo dockerPassword=${dockerPassword}"

            sh "docker build -t ${dockerUsername}/quiz-app:${commitId} ."
            sh "docker login -u ${dockerUsername} -p ${dockerPassword}"
            sh "docker push ${dockerUsername}/quiz-app:${commitId}"
        }
    }
}
    stage("Update Docker image in the ECS Task Definition") {
    steps {
        script {
            def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
            def imageName = "prudhvitejad/quiz-app:${commitId}"
            def taskDefinition = "Prudhvi-task-definition-family"
            def cluster = "Prudhvi-task1"
            def service = "Prudhvi-task1-service"
            def containerName = "quiz-app"
            
            // Pull the latest Docker image
            sh "docker login -u ${dockerUsername} -p ${dockerPassword}"
            sh "docker pull ${imageName}"
            
            // Register a new revision of the task definition
            def taskDefinitionInputJson = generateTaskDefinitionJson(taskDefinition, containerName, imageName)
            def registerTaskDefCmd = "aws ecs register-task-definition --cli-input-json '${taskDefinitionInputJson}'"
            def registerTaskDefOutput = sh(script: registerTaskDefCmd, returnStdout: true).trim()
            
            // Extract the revision number from the output
            def describeTaskDefinitionCmd = "aws ecs describe-task-definition --task-definition ${taskDefinition}"
            def taskDefinitionOutput = sh(script: describeTaskDefinitionCmd, returnStdout: true).trim()
            def revision = sh(script: "echo '${taskDefinitionOutput}' | jq -r '.taskDefinition.revision'", returnStdout: true).trim()
            
            // Update the ECS service to use the new task definition revision
            sh "aws ecs update-service --cluster ${cluster} --service ${service} --task-definition ${taskDefinition}:${revision}"
        }
    }
}
  }
  post {
  always {
    mail bcc: '', body: """'Project: ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER}, URL: ${env.BUILD_URL}'""", cc: '', from: '', replyTo: '', subject: """'${currentBuild.result}'""", to: 'prudhvi.teja@kansocloud.com'
  }
}
}

def getSecretText(credentialsId) {
    return Jenkins.instance.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0]
                           .credentials
                           .findAll { it.id == credentialsId }
                           .collect { it.secret.toString() }
                           .join(',')
}

def generateTaskDefinitionJson(taskDefinition, containerName, imageName) {
    def taskDefinitionInputJson = """
        {
          "family": "${taskDefinition}",
          "containerDefinitions": [
            {
              "name": "${containerName}",
              "image": "${imageName}",
              "cpu": 0,
              "portMappings": [
                {
                  "name": "${containerName}-81-tcp",
                  "containerPort": 81,
                  "hostPort": 81,
                  "protocol": "tcp",
                  "appProtocol": "http"
                }
              ],
              "essential": true,
              "environment": [],
              "environmentFiles": [],
              "mountPoints": [],
              "volumesFrom": [],
              "ulimits": [],
              "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                  "awslogs-create-group": "true",
                  "awslogs-group": "/ecs/${taskDefinition}",
                  "awslogs-region": "us-west-2",
                  "awslogs-stream-prefix": "ecs"
                },
                "secretOptions": []
              }
            }
          ],
          "executionRoleArn": "arn:aws:iam::022608205880:role/ecsTaskExecutionRole",
          "networkMode": "awsvpc",
          "requiresCompatibilities": [
            "FARGATE"
          ],
          "cpu": "1024",
          "memory": "3072",
          "runtimePlatform": {
            "cpuArchitecture": "X86_64",
            "operatingSystemFamily": "LINUX"
          }
        }
    """
    return taskDefinitionInputJson
}
