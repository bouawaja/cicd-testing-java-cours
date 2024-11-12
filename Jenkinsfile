def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-" + ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def NEXUS_URL = 'http://172.17.0.2:8081/repository/my-project-release'
def EMAIL_RECIPIENTS = "drivexpresse@gmail.com"
def GROUP_ID = "tech.zerofiltre.testing"
def ARTIFACT_ID = "calculator"
def VERSION = "0.0.1"
def JAR_FILE_PATH = "target/${ARTIFACT_ID}.jar"
def TAR_FILE_NAME = "${ARTIFACT_ID}-${VERSION}.tar.gz"
def TAR_FILE_PATH = "target/${TAR_FILE_NAME}"

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
            sh "mvn clean package"
            echo "JAR file generated at: ${pwd()}/${JAR_FILE_PATH}"
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


        stage('Create TAR package') {
           echo "Creating TAR package with JAR, entrypoint.sh, and Dockerfile"
           createTarPackage()
        }

        stage('Upload TAR to Nexus') {
            withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                uploadToNexusTar(USERNAME, PASSWORD, NEXUS_URL, TAR_FILE_PATH, ENV_NAME, ARTIFACT_ID)
            }
        }

        /* Optional: Uncomment if Docker image build and push is needed
        stage("Image Build and Push") {
            imageBuild(CONTAINER_NAME, CONTAINER_TAG)
            pushToImageToNexus(CONTAINER_NAME, CONTAINER_TAG, NEXUS_URL, USERNAME, PASSWORD)
        }
        */

    } finally {
        // Cleanup and notifications
        // deleteDir()
        // sendEmail(EMAIL_RECIPIENTS)
    }
}

def uploadToNexusTar(USERNAME, PASSWORD, NEXUS_URL, TAR_FILE_PATH, ENV_NAME, ARTIFACT_ID) {
    def DATE_FORMAT = new Date().format("yyyyMMdd_HHmmss")
    def FILE_NAME = "${ARTIFACT_ID}-${DATE_FORMAT}.tar.gz"
    def FULL_PATH = "${pwd()}/$TAR_FILE_PATH"

    echo "Uploading TAR package: ${FULL_PATH} as ${FILE_NAME} to Nexus..."

    sh """
        curl -v -u $USERNAME:$PASSWORD --upload-file ${FULL_PATH} \
        '${NEXUS_URL}/${ARTIFACT_ID}/${ENV_NAME}/${FILE_NAME}'
    """
}

def imagePrune(containerName) {
    try {
        sh "docker image prune -f"
        sh "docker stop $containerName"
    } catch (ignored) {
        echo 'No existing container to remove.'
    }
}

def imageBuild(containerName, tag) {
    sh "docker build -t $containerName:$tag --pull --no-cache ."
    echo "Image build complete"
}

def sendEmail(recipients) {
    mail(
            to: recipients,
            subject: "Build ${env.BUILD_NUMBER} - ${currentBuild.currentResult} - (${currentBuild.fullDisplayName})",
            body: "Check console output at: ${env.BUILD_URL}/console" + "\n")
}
// Refactored function to create the TAR package
def createTarPackage() {
    echo "Creating TAR package with JAR, entrypoint.sh, and Dockerfile"
    def timestamp = new Date().format("yyyyMMdd_HHmmss")
    def tarDir = "calculator-${timestamp}" // This will be the inner folder name

    // Creating the tar file with the structure as: calculator-20241112_153920.tar.gz/calculator-20241112_153920.tar/calculator-20241112_153920/
    sh """
        mkdir -p ${tarDir}
        cp target/calculator-*.jar ${tarDir}/  // Assuming the .jar file is generated in target/
        cp Dockerfile ${tarDir}/
        cp entrypoint.sh ${tarDir}/
        tar -czf calculator-${timestamp}.tar.gz -C ${tarDir} .
        echo "Tar package created: calculator-${timestamp}.tar.gz"
    """
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
