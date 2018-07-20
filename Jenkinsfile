#!groovy

def releasedVersion

node('master') {
  def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
  withEnv(["DOCKER=${dockerTool}/bin"]) {
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }
    }

    stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email ghatkar.abhaya@gmail.com && git config user.name abha10"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag abhaya-docker-release-images.jfrog.io/sparktodo:${releasedVersion} ."
            }
        }
    }
    stage('Push Releases to Artifactory'){
       // Create an Artifactory server instance:
       def server = Artifactory.server('abhaya-docker-artifactory')
       def uploadSpec = """{
	"files": [
		{
		"pattern": "**/*.jar",
		"target": "ext-release-local/"
		}
	]
	}"""
	server.upload(uploadSpec)
	   
	   
       // Create an Artifactory Docker instance. The instance stores the Artifactory credentials and the Docker daemon host address:
       def rtDocker = Artifactory.docker server: server, host: "tcp://34.248.134.77:2375"
       
       // Push a docker image to Artifactory (here we're pushing hello-world:latest). The push method also expects
       // Artifactory repository name (<target-artifactory-repository>).
       def buildInfo = rtDocker.push "abhaya-docker-release-images.jfrog.io/sparktodo:${releasedVersion}", 'docker-release-images'

       //Publish the build-info to Artifactory:
       server.publishBuildInfo buildInfo
     
        //  dockerCmd 'login -u admin -p <pwf> abhaya-docker-local.jfrog.io'
        //dockerCmd 'push abhaya-docker-local.jfrog.io/sparktodo:SNAPSHOT'
		
		
    }

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' abhaya-docker-release-images.jfrog.io/sparktodo:${releasedVersion}"
    }
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
