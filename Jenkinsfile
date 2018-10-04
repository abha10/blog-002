#!groovy

def releasedVersion
 
node('master') {
  //def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }
    }

    stage('Build') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
		    withSonarQubeEnv('sonar'){
		    	sh 'mvn clean verify sonar:sonar'
		    }
		
                sh 'mvn clean package'
		app = docker.build("digitaldemo-docker-snapshot-images.jfrog.io/sparktodo:SNAPSHOT",".")
                //dockerCmd 'build --tag  .'
            }
        }
    }
  
}


def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
