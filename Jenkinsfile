//def server = Artifactory.newServer url: 'http://172.17.0.3:8081/artifactory', credentialsId: 'mike_artifactory'
def server = Artifactory.newServer url: 'http://172.17.0.4:8081/artifactory', credentialsId: 'mike_artifactory'
def rtMaven = Artifactory.newMavenBuild()
def buildInfo
def remote = [:]
remote.name = "gate"
remote.host = "192.168.17.1"
remote.port = "3738"
remote.allowAnyHosts = true

pipeline {
    environment {
        def pom = readMavenPom file: 'maven-example/pom.xml'
        FOO = "BAR"
        BUILD_NUM_ENV = currentBuild.getNumber()
        ANOTHER_ENV = "${currentBuild.getNumber()}"
        INHERITED_ENV = "\${BUILD_NUM_ENV} is inherited"
        ACME_FUNC = pom.getArtifactId()
        ACME_VERS = pom.getVersion()
    }
    agent {
        docker {
            image 'maven:3-alpine'
            args '--mount "type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock" -u 0:0 --dns 192.168.17.1'
        }
    }
    
     stages {
      stage("Environment") {
          steps {
            // print whole ENV
            sh 'printenv'
              
            sh 'echo "FOO is $FOO"'
            // returns 'FOO is BAR'

            sh 'echo "BUILD_NUM_ENV is $BUILD_NUM_ENV"'
            // returns 'BUILD_NUM_ENV is 4' depending on the build number

            sh 'echo "ANOTHER_ENV is $ANOTHER_ENV"'
            // returns 'ANOTHER_ENV is 4' like the previous depending on the build number

            sh 'echo "INHERITED_ENV is $INHERITED_ENV"'
            // returns 'INHERITED_ENV is ${BUILD_NUM_ENV} is inherited'
            // The \ escapes the $ so the variable is not expanded but becomes a literal

            sh 'echo "ACME_FUNC is $ACME_FUNC"'
            // returns 'ACME_FUNC is spring-petclinic' or the name of the artifact in the pom.xml
          }
      }
        stage ('Link java and change maven workdir') {
            steps {
                script {
                    sh 'mv /root/.m2/settings-docker.xml /root/.m2/settings.xml'
                    sh 'ln -s /usr/lib/jvm/java-1.8-openjdk /usr/local/openjdk-8'
                    
                }
            }
        }
        stage ('Artifactory configuration') {
            steps {
                script {
                    rtMaven.tool = 'm3' // Tool name from Jenkins configuration
                    rtMaven.deployer releaseRepo: 'libs-release-local', snapshotRepo: 'libs-snapshot-local', server: server
                    rtMaven.resolver releaseRepo: 'libs-release', snapshotRepo: 'libs-snapshot', server: server
                    buildInfo = Artifactory.newBuildInfo()
                    buildInfo.env.capture = true
                }
            }
        }
    
        stage ('Exec Maven') {
            steps {
                script {
                    rtMaven.run pom: 'maven-example/pom.xml', goals: 'clean install', buildInfo: buildInfo
                }
            }
        }
    
        stage ('Publish build info') {
            steps {
                script {
                    server.publishBuildInfo buildInfo
                }
            }
        }

        stage ('SSH test case') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'gate_ssh_mike', passwordVariable: 'password', usernameVariable: 'userName')]) {
                    remote.user = userName
                    remote.password = password

                    stage("SSH Steps Rocks!") {
                        writeFile file: 'test.sh', text: 'ls'
                        sshCommand remote: remote, command: 'for i in {1..5}; do echo -n \"Loop \$i \"; date ; sleep 1; done'
                        sshScript remote: remote, script: 'test.sh'
                        sshPut remote: remote, from: 'test.sh', into: '.'
                        sshGet remote: remote, from: 'test.sh', into: 'test_new.sh', override: true
                        sshRemove remote: remote, path: 'test.sh'
                    }
                }
            }
        }
         
        stage('Deploy approval for test') {
            when {
                branch 'test-branch'
            }
            steps {
                input "Deploy to test?"
            }
        }
      
        stage('Deploy approval for master') {
            when {
                branch 'master'
            }
            steps {
                input "Deploy to prod?"
            }
        }
       
        stage('Prepare deployment environment') {
            when {
                branch 'master'
            }
            steps {
                script {
                    def pom = readMavenPom file: 'maven-example/pom.xml'
                 
                    print pom.version
                    print pom.getArtifactId()
                    sh 'pwd'
                    sh 'ls -la ./'
                    sh '[ -d "/var/jenkins_home/workspace/artifactory_test/maven-example/multi2/target" ] && \
                        ls -lah "/var/jenkins_home/workspace/artifactory_test/maven-example/multi2/target"'
                    sh 'echo http://dl-cdn.alpinelinux.org/alpine/edge/community >> /etc/apk/repositories && \
                        apk update && \
                        apk upgrade && \
                        apk add --no-cache bash git openssh curl "docker=17.05.0-r0"'
                    
                    sh 'docker ps -a && docker images -a && docker info'
                 }
            }
        }
                
         stage('Deploy to production') {

             when {
                branch 'master'
            }

            environment {
                def pom = readMavenPom file: 'maven-example/pom.xml'

                BUILD_NUM_ENV = currentBuild.getNumber()
                ART_ID = pom.getArtifactId()
                VER = pom.getVersion().toLowerCase()
                GIT_COMMIT = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
                //IMAGE = "registry2.mikronfs.ru:5000/test/maventest/${VER}-${BUILD_NUM_ENV}:${GIT_COMMIT}"
                //IMAGE = "registry2.mikronfs.ru:5000/test/maventest/${VER}:${BUILD_NUM_ENV}-${GIT_COMMIT}"
                IMAGE = "registry.mikronfs.ru:5000/test/maventest/${VER}:${BUILD_NUM_ENV}-${GIT_COMMIT}"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker_registry', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    //sh 'docker login -u ${USERNAME} -p ${PASSWORD} https://registry2.mikronfs.ru:5000'
                    sh 'docker login -u ${USERNAME} -p ${PASSWORD} https://registry.mikronfs.ru:5000'
                }
                sh 'docker build -t ${IMAGE} .'
                sh 'docker push ${IMAGE}'
            }
        }

        stage('Deploying t') {
            when {
                branch 'test-branch'
            }
            steps {
                sh "echo ${buildInfo}"
            }
        }

    }
}
