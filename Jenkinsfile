def server = Artifactory.newServer url: 'http://172.17.0.3:8081/artifactory', credentialsId: 'mike_artifactory'
def rtMaven = Artifactory.newMavenBuild()
def buildInfo

pipeline {
    environment {
        def pom = readMavenPom file: 'maven-example/pom.xml'

        FOO = "BAR"
        BUILD_NUM_ENV = currentBuild.getNumber()
        ANOTHER_ENV = "${currentBuild.getNumber()}"
        INHERITED_ENV = "\${BUILD_NUM_ENV} is inherited"
        ACME_FUNC = pom.getArtifactId()
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
       
        stage('Deploying m') {
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
                    sh '[ -d "/var/jenkins_home/workspace/artifactory_test/maven-example/multi2/target" ] && ls -lah "/var/jenkins_home/workspace/artifactory_test/maven-example/multi2/target"'
                }
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
