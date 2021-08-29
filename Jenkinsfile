// This pipeline revolves around detecting secrets:
// - lint: Lints Docker immage
// - detect new secrets: Detect new secrets
// - Push: Pushes the image to the registry

pipeline {
    agent any

    environment { // Environment variables defined for all steps
        TOOLS_IMAGE = "registry.demo.local:5000/tools-image"
        SONAR_KEY = "juice-shop"
    }

    stages {
        stage("Lint") {
            agent {
                docker {
                    image "docker.io/hadolint/hadolint:v1.18.0"
                    reuseNode true
                }
            }
            steps {
                script {
                    def result = sh label: "Lint Dockerfile",
                        script: """\
                            hadolint Dockerfile > hadolint-results.txt
                        """,
                    returnStatus: true
                    if (result > 0) {
                        unstable(message: "Linting issues found")
                    }
                }
            }
        }

        // Detect new secrets added since last successful build
        stage("detect new secrets"){
            agent {
                docker {
                    image "${TOOLS_IMAGE}"
                    //make sure that username can be mapped correctly
//                     args "--volume /etc/passwd:/etc/passwd:ro"
                    reuseNode true
                }
            }
            steps {
                // Determine commit of previous successful build when this is master
                script{
                    def  result = sh label: "detect-secrets",
                        script: """ \
                            detect-secrets-hook --no-verify \
                                                -v \
                                                -- baseline .secrets.baseline.json \
                            \$(git diff-tree --no-commit-id --name-only -r ${GIT_COMMIT} | xargs -n1)
                            """,
                            returnStatus: true
                    // Exit code 1 is generated when secrets are detected or no baseline is present
                    // Exit code 3 is generated only when .secrets.baseline.json is updated
                    // eg. when the line numbers don't match anymore
                    if (result == 1) {
                        // There are (unaudited) secrets detected: fail stage
                        error("unaudited secrets have been found")
                    }
                }
            }
        }

        stage("sonarscanner") {
            agent {
                docker {
                    image "${TOOLS_IMAGE}"
                    reuseNode true
                }
            }
            steps {
                withSonarQubeEnv("sonarqube.demo.local") {
//                     sh label: "install prerequisites",
//                         script: "npm --registry https://registry.npm.taobao.org install -D typescript"
                    sh label: "sonar-scanner",
                        script: """\
                            sonar-scanner \
                            '-Dsonar.buildString=${BRANCH_NAME}-${BUILD_ID}' \
                            '-Dsonar.projectKey=${SONAR_KEY}' \
                            '-Dsonar.projectVersion=${BUILD_ID}' \
                            '-Dsonar.sources=${WORKSPACE}'
                            """
                }
            }
        }
    }

    post {
        always{
            archiveArtifacts artifacts: "*-results.txt"
        }
    }
}
