pipeline {

    agent {
        label "master"
    }

    environment {
        // GLobal Vars
        PIPELINES_NAMESPACE = "labs-ci-cd"
        NAME = "dev"

        // Job name contains the branch eg my-app-feature%2Fjenkins-123
        JOB_NAME = "${JOB_NAME}".replace("%2F", "-").replace("/", "-")
        IMAGE_REPOSITORY= 'image-registry.openshift-image-registry.svc:5000'

        GIT_SSL_NO_VERIFY = true

        // Credentials bound in OpenShift
        GIT_CREDS = credentials("${PIPELINES_NAMESPACE}-git-auth")
        NEXUS_CREDS = credentials("${PIPELINES_NAMESPACE}-nexus-password")
        ARGOCD_CREDS = credentials("${PIPELINES_NAMESPACE}-argocd-token")

        // Nexus Artifact repo
        NEXUS_REPO_NAME="labs-static"
        NEXUS_REPO_HELM = "helm-charts"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage('Perpare Environment') {
            failFast true
            parallel {
                stage("Release Build") {
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("master") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "labs-dev"
                            env.STAGING_NAMESPACE = "labs-staging"
                            env.APP_NAME = "${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
                stage("Sandbox Build") {
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("dev") || GIT_BRANCH.startsWith("feature") || GIT_BRANCH.startsWith("fix") || GIT_BRANCH.startsWith("nsfw") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "labs-dev"
                            // ammend the name to create 'sandbox' deploys based on current branch
                            env.APP_NAME = "${GIT_BRANCH}-${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
                stage("Pull Request Build") {
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("PR-") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "labs-dev"
                            env.APP_NAME = "${GIT_BRANCH}-${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
            }
        }

        stage("Build (Compile App)") {
            parallel {
                stage("Build App") {
                    agent {
                        node {
                            label "master"
                        }
                    }
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject("${TARGET_NAMESPACE}") {
                                    echo '### Running app build ###'
                                    openshift.selector("bc", "${NAME}-build").startBuild("--wait=true")
                                }
                            }
                        }
                    }
                }
                stage("Build Api") {
                    agent {
                        node {
                            label "master"
                        }
                    }
                    steps {
                        script {
                            openshift.withCluster() {
                                openshift.withProject("${TARGET_NAMESPACE}") {
                                    echo '### Running api build ###'
                                    openshift.selector("bc", "${NAME}-api").startBuild("--wait=true")
                                }
                            }
                        }
                    }
                }
            }
        }

        // triggered by app build above
        stage("Build Runtime App") {
            agent {
                node {
                    label "master"
                }
            }
            steps {
                echo '### Waiting for runtime app build to complete ###'
                script {
                    openshift.withCluster() {
                        openshift.withProject("${TARGET_NAMESPACE}") {
                            openshift.selector("bc", "${NAME}-runtime").untilEach(1) {
                                return it.object().status.phase == "Complete"
                            }
                        }
                    }
                }
            }
        }
    }
}