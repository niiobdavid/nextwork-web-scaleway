// CI/CD pipeline for the NextWork web app on Scaleway.
// Replaces buildspec.yml (CodeBuild) + appspec.yml (CodeDeploy) + CodePipeline from the AWS version.
//
// Set these three in Jenkins: Manage Jenkins -> System -> Global properties -> Environment variables
//   WEB_SERVER_IP    public IP of the web server
//   ARTIFACT_BUCKET  Object Storage bucket name
//   SCW_REGION       e.g. nl-ams
//
// Credentials it uses (Manage Jenkins -> Credentials):
//   scw-object-storage  "Username with password"  = Scaleway access key / secret key
//   web-server-ssh      "SSH Username with private key" = user 'deploy' + deploy private key

pipeline {
    agent any

    environment {
        APP_NAME  = 'nextwork-web-project'
        // Build with Java 17 (Jenkins itself runs on Java 21)
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        // Newer AWS CLI checksum defaults can upset S3-compatible stores; only send them when required
        AWS_REQUEST_CHECKSUM_CALCULATION = 'WHEN_REQUIRED'
        AWS_RESPONSE_CHECKSUM_VALIDATION = 'WHEN_REQUIRED'
    }

    // Check GitHub for new commits every ~2 minutes (no webhook needed)
    triggers {
        pollSCM('H/2 * * * *')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Test') {
            steps {
                sh '''
                  if [ -f run-tests.sh ]; then
                    chmod +x run-tests.sh && ./run-tests.sh
                  else
                    echo "No run-tests.sh found - skipping tests"
                  fi
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B clean package'
            }
        }

        stage('Store artifact') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'scw-object-storage',
                                                  usernameVariable: 'AWS_ACCESS_KEY_ID',
                                                  passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                      aws s3 cp "target/$APP_NAME.war" "s3://$ARTIFACT_BUCKET/builds/$BUILD_NUMBER/$APP_NAME.war" \
                        --endpoint-url "https://s3.$SCW_REGION.scw.cloud" --region "$SCW_REGION"
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'web-server-ssh',
                                                   keyFileVariable: 'SSH_KEY',
                                                   usernameVariable: 'SSH_USER')]) {
                    sh '''
                      # Copy the key without Windows line endings (\\r) and make sure it ends with a newline
                      KEY_FILE="$(mktemp)"
                      trap 'rm -f "$KEY_FILE"' EXIT
                      { tr -d '\\r' < "$SSH_KEY"; echo; } > "$KEY_FILE"
                      chmod 600 "$KEY_FILE"

                      SSH_OPTS="-i $KEY_FILE -o StrictHostKeyChecking=accept-new"
                      scp $SSH_OPTS "target/$APP_NAME.war" "$SSH_USER@$WEB_SERVER_IP:/home/deploy/incoming/$APP_NAME.war"
                      ssh $SSH_OPTS "$SSH_USER@$WEB_SERVER_IP" "sudo /usr/local/bin/deploy-war.sh"
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                  for i in $(seq 1 12); do
                    if curl -fsS "http://$WEB_SERVER_IP/" > /dev/null; then
                      echo "Site is up"; exit 0
                    fi
                    sleep 5
                  done
                  echo "Site did not respond within 60 seconds"; exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "Build ${env.BUILD_NUMBER} is live at http://${env.WEB_SERVER_IP}"
        }
    }
}
