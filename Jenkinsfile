def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-" + ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
def HTTP_PORT = getHTTPPort(env.BRANCH_NAME)
def EMAIL_RECIPIENTS = "drivexpresse@gmail.com"

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
                sh " mvn sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
            }
            timeout(time: 1, unit: 'MINUTES') {
                def qg = waitForQualityGate()
                if (qg.status != 'OK') {
                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                }
            }
        }

        stage('Push to Nexus') {
            steps {
                script {
                    def nexusUrl = 'localhost:5000/my-docker-repository/'
                    def filePath = CONTAINER_NAME // Mettez à jour le chemin de votre artefact
                    def groupId = 'tech.zerofiltre.testing'
                    def artifactId = 'calculator'
                    def version = '0.0.1-SNAPSHOT'
                    def packaging = 'jar'

                    echo "Pushing the package to Nexus..."

                    sh """
                        curl -u \$NEXUS_USERNAME:\$NEXUS_PASSWORD --upload-file ${filePath} \\
                        "${nexusUrl}${groupId.replace('.', '/')}/${artifactId}/${version}/${artifactId}-${version}.${packaging}"
                    """

                    def response = sh (
                        script: """
                            curl -u \$NEXUS_USERNAME:\$NEXUS_PASSWORD --head \\
                            "${nexusUrl}${groupId.replace('.', '/')}/${artifactId}/${version}/${artifactId}-${version}.${packaging}"
                        """,
                        returnStatus: true
                    )

                    if (response == 0) {
                        echo "Package successfully uploaded to Nexus."
                    } else {
                        error "Failed to upload the package to Nexus."
                    }
                }
            }
        }
    } finally {
        deleteDir()
        //sendEmail(EMAIL_RECIPIENTS);
    }
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
