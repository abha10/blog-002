#!groovy

def releasedVersion

node('master') {
  def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
  withEnv(["DOCKER=${dockerTool}/bin"]) {
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }, 'Run Zalenium': {
            dockerCmd '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
        }
    }

    stage('Build') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                sh 'mvn clean package'
                dockerCmd 'build --tag sparktodo:SNAPSHOT .'
            }
        }
    }
  

    stage('Deploy') {
        dir('app') {
               dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT'
         }
    }

    stage('Tests') {
        try {
            dir('tests/rest-assured') {
                sh 'chmod a+rwx gradlew'
                sh './gradlew clean test'
            }
        } finally {
            junit testResults: 'tests/rest-assured/build/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/rest-assured/build/**'
        }

        dockerCmd 'rm -f snapshot'
        dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT'

        try {
            withMaven(maven: 'Maven 3') {
                dir('tests/bobcat') {
                    sh 'mvn clean test -Dmaven.test.failure.ignore=true'
                }
            }
        } finally {
            junit testResults: 'tests/bobcat/target/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/bobcat/target/**'
        }

       /* dockerCmd 'rm -f snapshot'
        dockerCmd 'stop zalenium'
        dockerCmd 'rm zalenium'*/
    }
     stage('Push Snapshot to JFrog Artifactory'){
       def server = Artifactory.server('abhaya-docker-artifactory')
        def rtDocker = Artifactory.docker server: server, host: "tcp://34.248.134.77:2375"
        //def rtDocker = Artifactory.docker server: server
      def buildInfo = rtDocker.push 'abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT', 'docker-local'
    //   Publish the build-info to Artifactory:
      server.publishBuildInfo buildInfo
      //  dockerCmd 'login -u admin -p 65VEySG41g abhaya-docker-local.jfrog.io'
        //dockerCmd 'push abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT'
    }

    /*stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email ghatkar.abhaya@gmail.com && git config user.name abha10"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag abhaya-docker-local.jfrog.io/sparktodo:${releasedVersion} ."
            }
        }
    }

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' abhaya-docker-local.jfrog.io/sparktodo:${releasedVersion}"
    }*/
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
