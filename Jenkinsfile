label = "${UUID.randomUUID().toString()}"
BUILD_FOLDER = "/go"
expired=240
git_project = "nuclio-templates"
git_project_user = "iguazio"
git_deploy_user_token = "iguazio-prod-git-user-token"
git_deploy_user_private_key = "iguazio-prod-git-user-private-key"

podTemplate(label: "${git_project}-${label}", yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: "${git_project}-${label}"
  labels:
    jenkins/kube-default: "true"
    app: "jenkins"
    component: "agent"
spec:
  shareProcessNamespace: true
  containers:
    - name: jnlp
      image: jenkins/jnlp-slave
      resources:
        limits:
          cpu: 1
          memory: 2Gi
        requests:
          cpu: 1
          memory: 2Gi
      volumeMounts:
        - name: go-shared
          mountPath: /go
    - name: docker-cmd
      image: docker
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run
        - name: go-shared
          mountPath: /go
    - name: golang
      image: golang:1.11
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
        - name: go-shared
          mountPath: /go
  volumes:
    - name: docker-sock
      hostPath:
          path: /var/run
    - name: go-shared
      emptyDir: {}
"""
) {
    node("${git_project}-${label}") {
        withCredentials([
                string(credentialsId: git_deploy_user_token, variable: 'GIT_TOKEN')
        ]) {
            def TAG_VERSION
            pipelinex = library(identifier: 'pipelinex@DEVOPS-204-pipelinex', retriever: modernSCM(
                    [$class: 'GitSCMSource',
                     credentialsId: git_deploy_user_private_key,
                     remote: "git@github.com:iguazio/pipelinex.git"])).com.iguazio.pipelinex
            multi_credentials=[pipelinex.DockerRepo.ARTIFACTORY_IGUAZIO, pipelinex.DockerRepo.DOCKER_HUB, pipelinex.DockerRepo.QUAY_IO]

            common.notify_slack {
                stage('get tag data') {
                    container('jnlp') {
                        TAG_VERSION = github.get_tag_version(TAG_NAME)
                        PUBLISHED_BEFORE = github.get_tag_published_before(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)

                        echo "$TAG_VERSION"
                        echo "$PUBLISHED_BEFORE"
                    }
                }

                if (TAG_VERSION != null && TAG_VERSION.length() > 0 && PUBLISHED_BEFORE < expired) {
                    stage('prepare sources') {
                        container('jnlp') {
                            dir("${BUILD_FOLDER}/src/github.com/iguazio/${git_project}") {
                                git(changelog: false, credentialsId: git_deploy_user_private_key, poll: false, url: "git@github.com:${git_project_user}/${git_project}.git")
                                sh("git checkout ${TAG_VERSION}")
                            }
                        }
                    }

                    parallel(
                            'linux binaries': {
                                container('golang') {
                                    dir("${BUILD_FOLDER}/src/github.com/iguazio/${git_project}") {
                                        sh("AVREZ_TAG=${TAG_VERSION} GOARCH=amd64 GOOS=linux make bin")
                                    }
                                }
                                container('jnlp') {
                                    RELEASE_ID = github.get_release_id(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)
                                    github.upload_asset(git_project, git_project_user, "avrez-${TAG_VERSION}-linux-amd64", RELEASE_ID, GIT_TOKEN)
                                }
                                container('jnlp') {
                                    withCredentials([
                                            string(credentialsId: pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[2], variable: 'PACKAGES_ARTIFACTORY_PASSWORD')
                                    ]) {
                                        common.upload_file_to_artifactory(pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[0], pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[1], PACKAGES_ARTIFACTORY_PASSWORD, "iguazio-devops/avrez", "avrez-${TAG_VERSION}-linux-amd64")
                                    }
                                }
                            },

                            'darwin binaries': {
                                container('golang') {
                                    dir("${BUILD_FOLDER}/src/github.com/iguazio/${git_project}") {
                                        sh("AVREZ_TAG=${TAG_VERSION} GOARCH=amd64 GOOS=darwin make bin")
                                    }
                                }
                                container('jnlp') {
                                    RELEASE_ID = github.get_release_id(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)
                                    github.upload_asset(git_project, git_project_user, "avrez-${TAG_VERSION}-darwin-amd64", RELEASE_ID, GIT_TOKEN)
                                }
                                container('jnlp') {
                                    withCredentials([
                                            string(credentialsId: pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[2], variable: 'PACKAGES_ARTIFACTORY_PASSWORD')
                                    ]) {
                                        common.upload_file_to_artifactory(pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[0], pipelinex.PackagesRepo.ARTIFACTORY_IGUAZIO[1], PACKAGES_ARTIFACTORY_PASSWORD, "iguazio-devops/avrez", "avrez-${TAG_VERSION}-darwin-amd64")
                                    }
                                }
                            }
                    )

                    stage('update release status') {
                        container('jnlp') {
                            github.update_release_status(git_project, git_project_user, "${TAG_VERSION}", GIT_TOKEN)
                        }
                    }
                } else {
                    stage('warning') {
                        if (PUBLISHED_BEFORE >= expired) {
                            currentBuild.result = 'ABORTED'
                            error("Tag too old, published before $PUBLISHED_BEFORE minutes.")
                        } else {
                            currentBuild.result = 'ABORTED'
                            error("${TAG_VERSION} is not release tag.")
                        }
                    }
                }
            }
        }
    }
}
