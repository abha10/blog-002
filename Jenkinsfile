#!groovy

def releasedVersion

node('master') {
  def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
  withEnv(["DOCKER=${dockerTool}/bin"]) {
    stage('Prepare') {
        deleteDir()
        parallel Checkout: {
            checkout scm
        }/*, 'Run Zalenium': {
            dockerCmd '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
        }*/
    }

    stage('Build') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                sh 'mvn clean package'
                dockerCmd 'build --tag automatingguy/sparktodo:SNAPSHOT .'
            }
        }
    }

    stage('Deploy') {
            dir('app') {
                dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'
            }
    }

    stage('Tests') {
      
      try{
       // dockerCmd 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'
      echo 'Testing Endpoint'
      /*def response = $(curl --write-out %{http_code} --silent --output /dev/null http://localhost:9999)
      def testing = (response == 200) ? echo "Testing Successfull" : echo "Testing Failed with status code ${response}"*/
      /* URL url = new URL('http://localhost:9999');    
        HttpURLConnection connection = url.openConnection();    
       
        connection.setRequestMethod("GET");
        //connection.setRequestProperty("Cookie", cookie); 
        connection.doOutput = true;   
 
        //get the request    
        connection.connect();    
        print $resp.statusCode
        //parse the response    
        //parseResponse(connection);    
 
        if(failure){    
            error("\nGET from URL: $requestUrl\n  HTTP Status: $resp.statusCode\n  Message: $resp.message\n  Response Body: $resp.body");    
        }    
 
        this.printDebug("Request (GET):\n  URL: $requestUrl");    
        this.printDebug("Response:\n  HTTP Status: $resp.statusCode\n  Message: $resp.message\n  Response Body: $resp.body");    
*/
// GET

      
 /*def get = new URL("http://localhost:9999").openConnection();
def getRC = get.getResponseCode();
println(getRC);
if(getRC.equals(200)) {
    println(get.getInputStream().getText());
}  */
      sh 'sleep 60'
      
     // def response = $(curl --write-out %{http_code} --silent --output /dev/null http://localhost:9999)
      def response = sh('curl --write-out %{http_code} --silent --output /dev/null http://localhost:9999')
      //def testing = (response == 200) ? (echo "Testing Successfull") : (echo "Testing Failed with status code ${response})"
        echo response
        
      }finally{
        dockerCmd 'rm -f snapshot'
      }
     
      
    }

    stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github-cred', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email test@automatingguy.com && git config user.name Jenkins"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                dockerCmd "build --tag automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }

    stage('Deploy @ Prod') {
        dockerCmd "run -d -p 9999:9999 --name 'production' automatingguy/sparktodo:${releasedVersion}"
    }
  }
}

def dockerCmd(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}

def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
}
