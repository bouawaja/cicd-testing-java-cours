def ENV_NAME = getEnvName(env.BRANCH_NAME)
def CONTAINER_NAME = "calculator-" + ENV_NAME
def CONTAINER_TAG = getTag(env.BUILD_NUMBER, env.BRANCH_NAME)
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

        stage('Push to Nexus') {
            steps {
                script {
                    def nexusUrl = 'localhost:5000/my-docker-repository/'
                    def filePath = CONTAINER_NAME // Mettez à jour le chemin de votre artefact
                    def groupId = 'tech.zerofiltre.testing'
                    def artifactId = 'calculator'
                    def version = '0.0.1-SNAPSHOT'
                    def packaging = 'jar'


                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                        echo "Pushing the package to Nexus..."

                        // Commande pour envoyer le fichier
                        sh """
                            curl -u $NEXUS_USERNAME:$NEXUS_PASSWORD --upload-file ${filePath} \\
                            "${nexusUrl}${groupId.replace('.', '/')}/${artifactId}/${version}/${artifactId}-${version}.${packaging}"
                        """

                        // Vérification de l'envoi réussi
                        def response = sh (
                            script: """
                                curl -u $NEXUS_USERNAME:$NEXUS_PASSWORD --head \\
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

String getTag(String buildNumber, String branchName) {
    if (branchName == 'master') {
        return buildNumber + '-unstable'
    }
    return buildNumber + '-stable'
}
