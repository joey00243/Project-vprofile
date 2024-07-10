def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
    
	agent any

	tools {
        maven "MAVEN3"
        jdk "OracleJDK17"
        //
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUSIP = "172.31.26.165"
        NEXUSPORT = "8081"
        SNAP_REPO = "vprofile-snapshot"
        NEXUS_REPOSITORY = "vprofile-release"
	NEXUS_GRP_REPO = "vprofile-maven-group"
        CENTRAL_REPO = "vprofile-maven-central"
        NEXUS_CREDENTIAL_ID = "nexuslogin"
        NEXUS_URL = "172.31.26.165:8081"
        ARTVERSION = "${env.BUILD_ID}"
        SONAR_ORG = "jomab-projects"
        SONAR_PROJECT_KEY = "jomab-an"
        SONAR_TOKEN = "sonar-sonar"
        DOCKER_TAG = "${env.BUILD_ID}"
        DOCKER_IMAGE = "joey00243/vprofileapplication"
        DOCKERHUB_CREDENTIALS = "dockerhub"
    }
	
    stages{
        
        stage('BUILD'){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

	    stage('UNIT TEST'){
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

	    stage('INTEGRATION TEST'){
            steps {
                sh 'mvn -s settings.xml verify'
            }
        }
		
        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
          
		  environment {
             scannerHome = tool 'mysonarscanner4'
          }

          steps {
            withSonarQubeEnv('sonar-sonar') {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=jomab-an \
                   -Dsonar.projectName=jomab-an \
		   -Dsonar.organization=${SONAR_ORG} \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }
          }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    //echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: NEXUS_GRP_REPO,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } 
		            else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }

        stage ('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build DOCKER_IMAGE + ":V$DOCKER_TAG"
                }
            }
        }

        stage ('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                        dockerImage.push("V$DOCKER_TAG")
                    }
                }
            }
        }

        stage('Remove Unused docker image'){
            steps{
                sh "docker rmi $DOCKER_IMAGE:V$DOCKER_TAG"
            }
        }

        stage('Kubernetes Deploy') {
	    agent { label 'KOPS' }
            steps {
                    sh "sudo helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${DOCKER_IMAGE}:V${DOCKER_TAG} --namespace prod"
            }
        }

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

    }
    
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd00243',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"

        }
    }

}
