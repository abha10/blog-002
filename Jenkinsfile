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
                dockerCmd 'build --tag abhaya-docker-snapshot-images.jfrog.io/sparktodo:SNAPSHOT .'
            }
        }
    }
  

    stage('Deploy @ Test Envirnoment') {
        dir('app') {
               dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" abhaya-docker-snapshot-images.jfrog.io/sparktodo:SNAPSHOT'
         }
    }

    stage('Perform Tests') {
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
        dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" abhaya-docker-snapshot-images.jfrog.io/sparktodo:SNAPSHOT'

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

       dockerCmd 'rm -f snapshot'
        dockerCmd 'stop zalenium'
        dockerCmd 'rm zalenium'
    }
    stage('Push Snapshots to Artifactory'){
       // Create an Artifactory server instance:
       def server = Artifactory.server('abhaya-docker-artifactory')
       
	   def uploadSpec = """{
		"files": [
			{
			"pattern": "target/*.jar",
			"target": "ext-snapshot-local/"
			}
		]
		}"""
		server.upload(uploadSpec)
	   
	   
       // Create an Artifactory Docker instance. The instance stores the Artifactory credentials and the Docker daemon host address:
       def rtDocker = Artifactory.docker server: server, host: "tcp://34.248.134.77:2375"
       
       // Push a docker image to Artifactory (here we're pushing hello-world:latest). The push method also expects
       // Artifactory repository name (<target-artifactory-repository>).
       def buildInfo = rtDocker.push 'abhaya-docker-snapshot-images.jfrog.io/sparktodo:SNAPSHOT', 'docker-snapshot-images'

       //Publish the build-info to Artifactory:
       server.publishBuildInfo buildInfo
     
        //  dockerCmd 'login -u admin -p <pwf> abhaya-docker-local.jfrog.io'
        //dockerCmd 'push abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT'
		
		
		
    }

    stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email ghatkar.abhaya@gmail.com && git config user.name abha10"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag abhaya-docker-snapshot-images.jfrog.io/sparktodo:${releasedVersion} ."
            }
        }
    }
    

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' abhaya-docker-snapshot-images.jfrog.io/sparktodo:${releasedVersion}"
    }
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
