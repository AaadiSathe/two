pipeline {
  agent any

  parameters {
    choice(name: 'TARGET_ENVIRONMENT', choices: ['blue', 'green'], description: 'Select the target environment')
  }

  stages {

    stage('Git Checkout') {
      steps {
        script {
          git branch: 'main', url: 'https://github.com/AaadiSathe/two'
        }
      }
    }

    stage('Maven Build') {
      steps {
        sh 'mvn clean install -DskipTests'
        sh 'cp target/SmartContactManger-0.0.1-SNAPSHOT.war .'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t your-spring-boot-image .'
      }
    }

    stage('Start Target Environment') {
      steps {
        script {
          def appPorts = [blue: 8081, green: 8082]
          def mysqlPorts = [blue: 3326, green: 3307]

          def appPort = appPorts[params.TARGET_ENVIRONMENT]
          def mysqlPort = mysqlPorts[params.TARGET_ENVIRONMENT]

          def removeContainer = { containerName ->
            sh """
              if docker ps -a -q -f name=${containerName} | grep -q .; then
                echo "Stopping and removing container: ${containerName}"
                docker stop ${containerName} || true
                docker rm ${containerName} || true
              fi
            """
          }

          def mysqlContainer = "mysql-${params.TARGET_ENVIRONMENT}"
          def appContainer = "app-${params.TARGET_ENVIRONMENT}"

          removeContainer(mysqlContainer)
          removeContainer(appContainer)

          sh "docker-compose -f docker-compose-${params.TARGET_ENVIRONMENT}.yml up -d"

          sleep time: 10, unit: 'SECONDS'

          // Uncomment the health check if needed (replace with your actual health check)
          // sh "curl http://localhost:${appPort}/actuator/health || exit 1"
        }
      }
    }

    stage('Verify Target Deployment') {
      steps {
        retry(3) {
        //   sh 'curl http://yourdomain.com || exit 1' 
        }
      }
    }

    stage('SWITCH TRAFFIC TO NEW CONTAINER') {
      steps {
        script {
          try {
            sh """
              set -x
              ansible-playbook -i inventory.ini switch_nginx.yml \
                --extra-vars "TARGET_ENVIRONMENT=${params.TARGET_ENVIRONMENT}"
            """
            echo "Traffic switched to ${params.TARGET_ENVIRONMENT} container"
          } catch (Exception e) {
            error("Traffic switch failed: ${e.message}")
          }
        }
      }
    }
  }

  post {
    always {
      script {
        def otherEnv = params.TARGET_ENVIRONMENT == 'green' ? 'blue' : 'green'
        echo "Stopping containers for ${otherEnv} environment"
        sh "docker-compose -f docker-compose-${otherEnv}.yml down || true"
      }
    }
  }
}
