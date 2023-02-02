pipeline{
  agent {
        kubernetes {
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: joseandresdevops/nuevojava:6.0
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-socket-volume
    securityContext:
      privileged: true
  volumes:
  - name: docker-socket-volume
    hostPath:
      path: /var/run/docker.sock
      type: Socket
    command:
    - sleep
    args:
    - infinity
        '''
            defaultContainer 'shell'
        }
    }

  environment {
    registryCredential='docker-hub-credentials'
    registryBackend = 'joseandresdevops/spring-boot-app'
    NEXUS_VERSION = "nexus3"
    NEXUS_PROTOCOL = "http"
    NEXUS_URL = "hot-teams-go-213-0-57-163.loca.lt"//"localhost:8081/repository/bootcamp/"//
    NEXUS_REPOSITORY = "bootcamp"
    NEXUS_CREDENTIAL_ID = "nexus"
    DOCKER_IMAGE_NAME="joseandresdevops/spring-boot-app"
	DOCKERHUB_CREDENTIALS=credentials("docker-hub-credentials")
  }

    stages {

        stage('build') {
            steps {
                //*sh 'mvn --version'
                sh "mvn clean package -DskipTest"
                sh "mvn compile"
                sh "mvn package"
            }
        }
        
        stage("test"){
            steps{
                sh "mvn test"
                jacoco()
                junit "target/surefire-reports/*.xml"
            }
        }


//********************************************* FUNCIONA SUBIR IMAGEN DOCKER
        stage('Push Image to Docker Hub') {
            steps {
                script {
                    dockerImage = docker.build registryBackend + ":$BUILD_NUMBER"
                    docker.withRegistry( '', registryCredential) {
                    dockerImage.push()
                    }
                }
            }
        }

        stage('Push Image latest to Docker Hub') {
            steps {
                script {
                    dockerImage = docker.build registryBackend + ":latest"
                    docker.withRegistry( '', registryCredential) {
                    dockerImage.push()
                    }
                }
            }
        }
//********************************************/

 //FUNCIONA ES EL JMETER

        stage ("Setup Jmeter") {
            steps{
                script {

                    if(fileExists("jmeter-docker")){
                        sh 'rm -r jmeter-docker'
                    }

                    sh 'git clone https://github.com/joseandresdevops/jmeter-docker.git'

                    dir('jmeter-docker') {

                        if(fileExists("apache-jmeter-5.5.tgz")){
                        sh 'rm -r apache-jmeter-5.5.tgz'
                        }
                    sh 'wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.5.tgz'
                    sh 'tar xvf apache-jmeter-5.5.tgz'
                    sh 'cp plugins/*.jar apache-jmeter-5.5/lib/ext'
                    sh 'mkdir test'
                    sh 'mkdir apache-jmeter-5.5/test'
                    sh 'cp ../src/main/resources/*.jmx apache-jmeter-5.5/test/'
                    sh 'chmod +775 ./build.sh && chmod +775 ./run.sh && chmod +775 ./entrypoint.sh'
                    sh 'rm -r apache-jmeter-5.5.tgz'
                    sh 'tar -czvf apache-jmeter-5.5.tgz apache-jmeter-5.5'
                    sh './build.sh'
                    sh 'rm -r apache-jmeter-5.5 && rm -r apache-jmeter-5.5.tgz'
                    sh 'cp ../src/main/resources/perform_test.jmx test'
                    }
                }
            }
        }

        stage ("Run Jmeter Performance Test") {
            steps{
                script {
                    dir('jmeter-docker') {
                        if(fileExists("apache-jmeter-5.5.tgz")){
                            sh 'rm -r apache-jmeter-5.5.tgz'
                        }
                    sh './run.sh -n -t test/perform_test.jmx -l test/perform_test.jtl'
                    sh 'pwd'
                    sh 'docker cp jmeter:/home/jmeter/apache-jmeter-5.5/test/perform_test.jtl $(pwd)/test'
                    perfReport 'test/perform_test.jtl'
                    }
                }
            }
        }

//TAURUS/BLAZER
stage ("Generate Taurus Report") {
   steps{
       script {
            dir('jmeter-docker') {
               sh 'pip install bzt'
               sh 'export PATH=$PATH:/home/jenkins/.local/bin'

               BlazeMeterTest: {
                   sh 'bzt test/perform_test.jtl -report'  //si le cambio la ruta de prncipio me dice permiso denegado
               }
            }
       }
   }
}


/*
//SONAR NO FUNCIONA PQ LAS CREDENCIALES NO PUEDEN VERSE
        stage('NPM build') {
            steps {
                script {
                    sh 'mvn install'
                    sh 'mvn run build'
                }
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv(credentialsId: "sonarqube-server3",installationName: "sonarqube-server"){
                sh 'mvn run sonar'
                }
            }
        }
*/


        //FUNCIONA NEXUS - PUBLICAR PUERTO 8081
        
        stage("Publish to Nexus") {
            steps {
                script {
                    // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                    pom = readMavenPom file: "pom.xml"
                    // Find built artifact under target folder
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    // Print some info from the artifact found
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    // Extract the path from the File found
                    artifactPath = filesByGlob[0].path
                    // Assign to a boolean response verifying If the artifact name exists
                    artifactExists = fileExists artifactPath

                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
                        versionPom = "${pom.version}"

                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: pom.version,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                // Artifact generated such as .jar, .ear and .war files.
                                [artifactId: pom.artifactId,
                                classifier: "",
                                file: artifactPath,
                                type: pom.packaging],

                                // Lets upload the pom.xml file for additional information for Transitive dependencies
                                [artifactId: pom.artifactId,
                                classifier: "",
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        )

                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }



        stage('Deploy to K8s') {

            steps{
                script {
                    if(fileExists("configuracion")){
                        sh 'rm -r configuracion'
                    }
                }
                    sh 'git clone https://github.com/JoseAndresDevOps/kubernetes-helm-docker-config.git configuracion --branch test-implementation'
                    sh 'kubectl apply -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yml -n default --kubeconfig=configuracion/kubernetes-config/config'
            }
        }
        
        
    }
}
