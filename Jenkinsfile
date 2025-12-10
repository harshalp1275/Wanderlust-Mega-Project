@Library('Shared') _
pipeline {
  agent any

  environment {
    // Names must match what you configure in Manage Jenkins
    SONAR_TOOL_NAME = 'SonarScanner'   // name in Global Tool Configuration -> SonarQube Scanner
    SONAR_SERVER_NAME = 'Sonar'        // name in Manage Jenkins -> Configure System -> SonarQube servers
    DOCKERHUB_CRED_ID = 'DockerHubcred' // credential id for docker hub (username/password)
    GITHUB_CRED_ID = 'Github-cred'      // credential id for pushing updates
    GIT_REPO = 'https://github.com/harshalp1275/Wanderlust-Mega-Project.git'
    GIT_BRANCH = 'feature-test-10/12/2025-version-1'
  }

  parameters {
    string(name: 'FRONTEND_DOCKER_TAG', defaultValue: 'latest', description: 'Frontend Docker tag produced by CI')
    string(name: 'BACKEND_DOCKER_TAG', defaultValue: 'latest', description: 'Backend Docker tag produced by CI')
  }

  options {
    skipDefaultCheckout(true)
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Workspace cleanup') {
      steps { cleanWs() }
    }

    stage('Checkout') {
      steps {
        script {
          // use library function for checkout if provided
          code_checkout(GIT_REPO, GIT_BRANCH)
        }
      }
    }

    stage('Verify tags') {
      steps {
        echo "FRONTEND_DOCKER_TAG = ${params.FRONTEND_DOCKER_TAG}"
        echo "BACKEND_DOCKER_TAG  = ${params.BACKEND_DOCKER_TAG}"
      }
    }

    stage('Install Node Modules') {
      steps {
        script {
          // Install node deps for dependency-check
          dir('frontend') {
            sh 'if [ -f package-lock.json ] || [ -f package.json ]; then npm ci --silent || npm install --silent; else echo "no frontend package.json"; fi'
          }
          dir('backend') {
            sh 'if [ -f package-lock.json ] || [ -f package.json ]; then npm ci --silent || npm install --silent; else echo "no backend package.json"; fi'
          }
        }
      }
    }

    stage('Dependency Check') {
      steps {
        script {
          // call dependency check function from shared lib
          dependency_check()
        }
      }
      post {
        always {
          dependencyCheckPublisher pattern: '**/dependency-check-report.xml', failBuildOnCVSS: '7'
        }
      }
    }

    stage('Update Kubernetes manifests') {
      steps {
        script {
          dir('kubernetes') {
            // Use sed that tolerates periods in names and only replaces image name part
            sh """
              # backend
              sed -i -E 's|(image:[[:space:]]*harshalpatil1010/wanderlust-backend-beta)(:.*)?$|\\1:${params.BACKEND_DOCKER_TAG}|' backend.yaml || true
              # frontend
              sed -i -E 's|(image:[[:space:]]*harshalpatil1010/wanderlust-frontend-beta)(:.*)?$|\\1:${params.FRONTEND_DOCKER_TAG}|' frontend.yaml || true
            """
          }
        }
      }
    }

    stage('Commit & Push manifest updates') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: GITHUB_CRED_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh """
              git config user.email "ci-bot@yourdomain.com"
              git config user.name "CI Bot"
              git add kubernetes/backend.yaml kubernetes/frontend.yaml || true
              git commit -m "CI: update images backend=${params.BACKEND_DOCKER_TAG} frontend=${params.FRONTEND_DOCKER_TAG}" || echo "no changes to commit"
              git push https://${GIT_USER}:${GIT_PASS}@github.com/harshalp1275/Wanderlust-Mega-Project.git ${GIT_BRANCH} || echo "git push failed"
            """
          }
        }
      }
    }

    stage('SonarQube: Code Analysis') {
      steps {
        script {
          // Use Sonar shared lib wrapper or direct invocation
          // ensure Sonar tool exists in Global Tool Config with the name in SONAR_TOOL_NAME
          def scannerHome = tool name: env.SONAR_TOOL_NAME, type: 'hudson.plugins.sonar.SonarRunnerInstallation'
          withSonarQubeEnv(env.SONAR_SERVER_NAME) {
            sh """
              ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=wanderlust \
                -Dsonar.projectName=Wanderlust \
                -Dsonar.sources=. \
                -Dsonar.login=${SONAR_AUTH_TOKEN:-''}
            """
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        script {
          // will wait for quality gate result; requires SonarQube plugin and proper webhook or token
          timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
          }
        }
      }
    }

    stage('Docker: Build Images') {
      steps {
        script {
          // build both images (example using shared lib)
          dir('frontend') {
            docker_build('wanderlust-frontend-beta', params.FRONTEND_DOCKER_TAG, 'harshalpatil1010')
          }
          dir('backend') {
            docker_build('wanderlust-backend-beta', params.BACKEND_DOCKER_TAG, 'harshalpatil1010')
          }
        }
      }
    }

    stage('Docker: Push to DockerHub') {
      steps {
        script {
          docker_push('wanderlust-frontend-beta', params.FRONTEND_DOCKER_TAG, 'harshalpatil1010')
          docker_push('wanderlust-backend-beta', params.BACKEND_DOCKER_TAG, 'harshalpatil1010')
        }
      }
    }
  }

  post {
    success {
      script {
        emailext attachLog: true,
                from: 'ci-notify@yourdomain.com',
                subject: "Wanderlust CI SUCCESS - Build #${env.BUILD_NUMBER}",
                body: "<p>Project: ${env.JOB_NAME}</p><p>Build: ${env.BUILD_NUMBER}</p><p>URL: ${env.BUILD_URL}</p>",
                to: 'trainwithshubham@gmail.com',
                mimeType: 'text/html'
      }
    }
    failure {
      script {
        emailext attachLog: true,
                from: 'ci-notify@yourdomain.com',
                subject: "Wanderlust CI FAILED - Build #${env.BUILD_NUMBER}",
                body: "<p>Project: ${env.JOB_NAME}</p><p>Build: ${env.BUILD_NUMBER}</p><p>URL: ${env.BUILD_URL}</p>",
                to: 'harshalp1275@gmail.com',
                mimeType: 'text/html'
      }
    }
    always {
      cleanWs()
    }
  }
}
