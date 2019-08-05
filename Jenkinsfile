def server = Artifactory.newServer url: 'http://172.17.0.3:8081/artifactory', credentialsId: 'mike_artifactory'
def rtMaven = Artifactory.newMavenBuild()
def buildInfo

pipeline {
    agent {
        docker {
            image 'maven:3-alpine'
            args '--mount "type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock" -u 0:0 --dns 192.168.17.1'
        }
    }
    stages {
        stage ('Link java') {
            steps {
                script {
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
                sh 'pwd'
                sh 'ls -la ./'
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
