def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-" + ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def NEXUS_URL = 'http://localhost:8081/repository/maven-releases/'
def EMAIL_RECIPIENTS = "drivexpresse@gmail.com"
def GROUP_ID = "tech.zerofiltre.testing"
def ARTIFACT_ID = "calculator"
def VERSION = "1.0.0"
def FILE_NAME = "${ARTIFACT_ID}.jar"
def FILE_PATH = "target/${FILE_NAME}"
def REPO_NAME = 'maven-releases'

node {
    try {
        stage('Initialize') {
            def dockerHome = tool 'DockerLatest'
            def mavenHome = tool 'MavenLatest'
            env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Build with test') {
            sh "mvn clean install"
        }

        stage('Sonarqube Analysis') {
            withSonarQubeEnv('SonarQubeLocalServer') {
                sh "mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
            }
            timeout(time: 1, unit: 'MINUTES') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }

        stage("Image Prune") {
            imagePrune(CONTAINER_NAME)
        }

        stage('Image Build') {
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
        }

    /*    stage('Push Docker Image to Nexus') {
            withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                pushToImageToNexus(CONTAINER_NAME, CONTAINER_TAG, USERNAME, PASSWORD, NEXUSURL)
            }
        }*/
          stage('Check and Create Repo if Needed') {
                    checkAndCreateRepo(NEXUS_URL, REPO_NAME, USERNAME, PASSWORD)
                }

        stage('Upload JAR to Nexus') {
         withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
          uploadToNexusJar(USERNAME, PASSWORD, NEXUS_URL,FILE_NAME, GROUP_ID, FILE_PATH, VERSION, ENV_NAME)
            }
        }

    } finally {
        deleteDir()
        //sendEmail(EMAIL_RECIPIENTS);
    }
}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName"
    } catch (ignored) {
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag --pull --no-cache ."
    echo "Image build complete"
}

def pushToImageToNexus(containerName, tag, nexusUser, nexusPassword, nexusUrl) {
    sh "docker tag $containerName:$tag $nexusUrl/$containerName:$tag"
    sh "docker login localhost:5000 -u $nexusUser -p $nexusPassword"
    sh "docker push $nexusUrl/$containerName:$tag"
    echo "Image push to Nexus complete"
}

def uploadToNexusJar(USERNAME, PASSWORD, NEXUS_URL,FILE_NAME, GROUP_ID, FILE_PATH, VERSION, ENV_NAME) {

         sh """
                 curl -u $USERNAME:$PASSWORD --upload-file $FILE_PATH \
                 "${NEXUS_URL}${GROUP_ID.replace('.', '/')}/${ENV_NAME}/${VERSION}/${FILE_NAME}"
             """
}

def checkAndCreateRepo(NEXUS_URL, REPO_NAME, USERNAME, PASSWORD) {
    // Vérifier si le dépôt existe
    def repoExists = sh(script: """
        curl -u $USERNAME:$PASSWORD -s -o /dev/null -w "%{http_code}" "$NEXUS_URL/service/rest/v1/repositories/$REPO_NAME"
    """, returnStdout: true).trim()

    // Si le dépôt n'existe pas (code HTTP 404), le créer
    if (repoExists == "404") {
        echo "Le dépôt $REPO_NAME n'existe pas. Création en cours..."

        // Créer le dépôt (exemple pour un dépôt Maven hosted)
        def createRepoResponse = sh(script: """
            curl -u $USERNAME:$PASSWORD -X POST "$NEXUS_URL/service/rest/v1/repositories/maven/hosted" \
            -H "Content-Type: application/json" \
            -d '{
                "name": "$REPO_NAME",
                "online": true,
                "storage": {
                    "blobStoreName": "default",
                    "strictContentTypeValidation": false
                },
                "maven": {
                    "versionPolicy": "RELEASE",
                    "layoutPolicy": "STRICT"
                },
                "cleanup": {
                    "policyName": "none"
                }
            }'
        """, returnStdout: true).trim()

        echo "Réponse de la création du dépôt: $createRepoResponse"
    } else {
        echo "Le dépôt $REPO_NAME existe déjà."
    }
}


def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}

String getEnvName(String branchName) {
    if (branchName == 'master') {
        return 'prod'
    }
    return (branchName == 'develop') ? 'preprod' : 'dev'
}

String getHTTPPort(String branchName) {
    if (branchName == 'master') {
        return '9003'
    }
    return (branchName == 'develop') ? '9002' : '9001'
}

String getTag(String buildNumber, String branchName) {
    if (branchName == 'master') {
        return buildNumber + '-unstable'
    }
    return buildNumber + '-stable'
}
