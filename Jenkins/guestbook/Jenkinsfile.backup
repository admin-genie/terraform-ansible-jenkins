import java.text.SimpleDateFormat
def TODAY = (new SimpleDateFormat("yyMMddHHmm")).format(new Date())

pipeline {
    agent any
    
    environment {
        strDockTag = "${TODAY}_${BUILD_ID}"
        strDockImg = "ofmw/guestbook:${strDockTag}"
    }

    stages {
        stage('Checkout') {
            steps {
                git (branch: 'main', url:'https://github.com/ofmw/jenkins-guestbook.git')
            }
        }
        stage('Build') {
            steps {
                sh "chmod +x ./mvnw; ./mvnw clean package -DskipTests"
            }
            post {
                success {
                    archiveArtifacts "target/*.jar"
                }
            }
        }
        stage('Unit Test') {
            steps {
                sh "./mvnw test"
            }
            post {
                always {
                    junit "**/target/surefire-reports/TEST-*.xml"
                }
            }
        }
        //stage('SonarQube Analysis') {
        //    steps {
        //        withSonarQubeEnv('sonarqube-server') {
        //            sh '''
        //            ./mvnw sonar:sonar \
        //                -Dsonar.projectKey=project \
        //                -Dsonar.host.url=http://172.31.16.120:9000 \
        //                -Dsonar.login=8a6115681f366535f98792c5620111690ed27797
        //            '''
        //        }
        //    }
        //}
        //stage('SonarQube Quality Gate') {
        //    steps {
        //        timeout(time: 1, unit: 'MINUTES') {
        //            script {
        //                def qg = waitForQualityGate()
        //                if(qg.status != 'OK') {
        //                    echo "NOT OK Status: ${qg.status}"
        //                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
        //                } else {
        //                    echo "OK Status: ${qg.status}"
        //                }
        //            }
        //        }
        //    }
        //}
        stage('Docker Image Build') {
            steps {
                script {
                    oDockImg = docker.build(strDockImg, "--build-arg VERSION=${strDockTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            steps {
                script {
                    oDockImg = docker.withRegistry('', 'dockerhub-token') {
                        oDockImg.push()
                    }
                }
            }
        }
        stage('SSH guestbook') {
            steps {
                sshagent(credentials: ['ssh-token']) {
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@10.0.10.187 \
                    docker container rm -f guestbook"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@10.0.10.187 \
                    docker container run -d -p 8888:80 --name guestbook \
                    -e MYSQL_IP=10.0.10.187 \
                    -e MYSQL_PORT=3306 \
                    -e MYSQL_USER=root \
                    -e MYSQL_PASSWORD=education \
                    -e MYSQL_DATABASE=guestbook \
                    ${strDockImg}"
                }
            }
            post {
                success {
                    slackSend(tokenCredentialId: 'slack-token',
                    channel: '#cicd-alarm',
                    color: 'good',
                    message: '배포성공')
                }
                failure {
                    slackSend(tokenCredentialId: 'slack-token',
                    channel: '#cicd-alarm',
                    color: 'danger',
                    message: '배포실패')
                }
            }
        }
    }
}   
