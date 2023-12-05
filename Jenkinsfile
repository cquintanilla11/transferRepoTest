@Library('bd-pipelines')
import com.blackdiamond.*

// Force all projects to build if the last build was a failure. This ensures no projects are missed 
// during the commit change scan
def forceBuild = false
def buildVersion = "1.0.${new Date().format('yyyyMMddHH')}.${env.BUILD_NUMBER}-${env.BRANCH_NAME}"
def gitops = new gitops()
def projects = [
    [
        jiraComponentName: "BlackDiamond.GitOpsServer", 
        path: ".",
        buildContext: ".",
        isNodejs: true,
        isInternalComponent: true,
        isWebComponent: true,
    ],
]

pipeline {
    options { disableConcurrentBuilds() }
    agent {
        docker {
            label 'docker-linux'
            image "proget.bdreporting.com/dreamboat/jenkins-agent-base:${NETCORE3_BUILD_TOOLS_TAG}"
            registryUrl 'https://proget.bdreporting.com'
            registryCredentialsId 'dreamboat-docker-registry-user'
            args '-u root --privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    } 
	stages {
        stage('Build and Push Images') {
            steps {
                checkout scm
                script {
                    projects.each {
                        stage("Build ${it.jiraComponentName}") {
                            def dockerPushRegistry = 'proget.bdreporting.com'
                            def dockerImageName = "${dockerPushRegistry}/blackdiamond/${it.jiraComponentName}".toLowerCase()
                            def tag = gitops.GenerateImageTag()
                            def dockerImageNameWithTag = "${dockerImageName}:${tag}".toLowerCase()
                            def existingTags = gitops.GetImageTags(dockerPushRegistry, dockerImageName)
                            
                            if(!forceBuild && tag in existingTags ){
                                 echo "Found tag \"${tag}\" on remote image registry. Skipping build!"
                            }
                            else {
                                echo "Now building ${dockerImageNameWithTag}"
                                docker.withRegistry('https://proget.bdreporting.com', 'docker_registry_publisher') {
                                    def image = docker.build(dockerImageNameWithTag, "-f ${it.path}/Dockerfile ${it.buildContext}")

                                    echo "Now pushing ${dockerImageNameWithTag}"
                                    image.push()

                                    ['latest'].each {
                                        echo "Pushing additional tag ${it} for ${dockerImageNameWithTag}"
                                        image.push(it)
                                    }

                                    gitops.UpdateGitOpsComponentImageTag([
                                        new GitOpsComponent(
                                            targetBranch: "dev",
                                            jiraComponentName: it.jiraComponentName, 
                                            dockerImageNameWithTag: dockerImageNameWithTag,
                                        )
                                    ], true)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
