pipeline {
    agent {
        label 'docker'
    }
    stages {
        stage('Build docker image') {
            steps {
                 withCredentials([usernamePassword(credentialsId: 'hub.docker.com', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh '''
                        env
                        sg docker -c "
                            env
                            git rev-parse --abbrev-ref HEAD | sed -e 's^/^-^g; s^[.]^-^g;' | tr '[:upper:]' '[:lower:]'
                            docker login -u '${USER}' -p '${PASS}'
                            export GIT_BRANCH=$GIT_BRANCH
                            ./e2e-tests/build
                            docker logout
                        "
                        DOCKER_TAG=perconalab/percona-xtradb-cluster-operator:${GIT_BRANCH}
                        docker_tag_file='./results/docker/TAG'
                        mkdir -p $(dirname ${docker_tag_file})
                        echo ${DOCKER_TAG} > "${docker_tag_file}"
                        cat $docker_tag_file
                    '''
                }
                stash includes: 'results/docker/TAG', name: 'IMAGE'
                archiveArtifacts 'results/docker/TAG'
            }
        }
    }
    post {
        always {
            script {
                if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                    unstash 'IMAGE'
                    def IMAGE = sh(returnStdout: true, script: "cat results/docker/TAG | tr '[:upper:]' '[:lower:]'").trim()
                    if (env.CHANGE_URL) {
                        withCredentials([string(credentialsId: 'GITHUB_API_TOKEN', variable: 'GITHUB_API_TOKEN')]) {
                            sh """
                                curl -v -X POST \
                                    -H "Authorization: token ${GITHUB_API_TOKEN}" \
                                    -d "{\\"body\\":\\"PXC docker - ${IMAGE}\\"}" \
                                    "https://api.github.com/repos/\$(echo $CHANGE_URL | cut -d '/' -f 4-5)/issues/${CHANGE_ID}/comments"
                            """
                        }
                    }
                }
            }
            sh 'sudo rm -rf *'
            deleteDir()
        }
    }
}
