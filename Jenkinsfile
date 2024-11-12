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
            initializeTools()
        }

        stage('Checkout') {
            checkout scm
        }

        stage('Build with test') {
            buildPackage()
        }

        stage('Sonarqube Analysis') {
            sonarqubeAnalysis()
        }

        stage('Create TAR package') {
            createTarPackage()
        }

        stage('Upload TAR to Nexus') {
            uploadToNexusTar(NEXUS_URL, TAR_FILE_PATH, ENV_NAME, ARTIFACT_ID)
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

// Initialize tools like Docker and Maven
def initializeTools() {
    def dockerHome = tool 'DockerLatest'
    def mavenHome = tool 'MavenLatest'
    env.PATH = "${dockerHome}/bin:${mavenHome}/bin:${env.PATH}"
}

// Build project with tests
def buildPackage() {
    sh "mvn clean package"
    echo "JAR file generated at: ${pwd()}/${JAR_FILE_PATH}"
}

// SonarQube analysis for the project
def sonarqubeAnalysis() {
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

// Create TAR package with JAR, entrypoint.sh, and Dockerfile
def createTarPackage() {
    echo "Creating TAR package with JAR, entrypoint.sh, and Dockerfile"
    def timestamp = new Date().format("yyyyMMdd_HHmmss")
    def tarDir = "calculator-${timestamp}" // This will be the inner folder name

    sh """
        mkdir -p ${tarDir}
        cp target/calculator-*.jar ${tarDir}/
        cp Dockerfile ${tarDir}/
        cp entrypoint.sh ${tarDir}/
        tar -czf ${TAR_FILE_NAME} -C ${tarDir} .
        echo "Tar package created: ${TAR_FILE_NAME}"
    """
}

// Upload the TAR package to Nexus
def uploadToNexusTar(NEXUS_URL, TAR_FILE_PATH, ENV_NAME, ARTIFACT_ID) {
    def DATE_FORMAT = new Date().format("yyyyMMdd_HHmmss")
    def FILE_NAME = "${ARTIFACT_ID}-${DATE_FORMAT}.tar.gz"
    def FULL_PATH = "${pwd()}/$TAR_FILE_PATH"

    echo "Uploading TAR package: ${FULL_PATH} as ${FILE_NAME} to Nexus..."

    sh """
        curl -v -u $USERNAME:$PASSWORD --upload-file ${FULL_PATH} \
        '${NEXUS_URL}/${ARTIFACT_ID}/${ENV_NAME}/${FILE_NAME}'
    """
}

// Helper functions to get values based on the branch name
String getEnvName(String branchName) {
    return branchName == 'master' ? 'prod' : (branchName == 'develop' ? 'preprod' : 'dev')
}

String getHTTPPort(String branchName) {
    return branchName == 'master' ? '9003' : (branchName == 'develop' ? '9002' : '9001')
}

String getTag(String buildNumber, String branchName) {
    return branchName == 'master' ? "${buildNumber}-unstable" : "${buildNumber}-stable"
}
