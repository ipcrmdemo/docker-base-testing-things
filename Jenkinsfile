import groovy.json.JsonOutput

/**
 * Notify the Atomist services about the status of a build based from a
 * git repository.
 */
def notifyAtomist(
  String workspaceIds,
  String buildStatus,
  String gitUrl,
  String gitBranch,
  String gitCommitSha,
  String buildPhase="FINALIZED"
) {
    if (!workspaceIds) {
        echo 'No Atomist workspace IDs, not sending build notification'
        return
    }

    def payload = JsonOutput.toJson(
        [
            name: env.JOB_NAME,
            duration: currentBuild.duration,
            build: [
                number: env.BUILD_NUMBER,
                phase: buildPhase,
                status: buildStatus,
                full_url: env.BUILD_URL,
                scm: [
                    url: gitUrl,
                    branch: gitBranch,
                    commit: gitCommitSha
                ]
            ]
        ]
    )
    workspaceIds.split(',').each { workspaceId ->
        String endpoint = "https://webhook.atomist.com/atomist/jenkins/teams/${workspaceId}"
        echo "Sending ${payload} to ${endpoint}"
        sh "curl --silent -X POST -H 'Content-Type: application/json' -d '${payload}' ${endpoint}"
        echo "Success"
    }
}

/**
 * Send Image-link event to Atomist to associate the new image to the commit
 */
def sendImageLink(
    String workspaceIds,
    String owner,
    String repo,
    String commit,
    String image
) {
    if (!workspaceIds) {
        echo 'No Atomist workspace IDs, not sending image-link event'
        return
    }

    def payload = JsonOutput.toJson(
        [
            git: [ owner: owner, repo: repo, sha: commit ],
            docker: [ image: image ],
            type: "link-image",
        ]
    )
    workspaceIds.split(',').each { workspaceId ->
        String endpoint = "https://webhook.atomist.com/atomist/link-image/teams/${teamId}"
        sh "curl --silent -X POST -H 'Content-Type: application/json' -d '${payload}' ${endpoint}"
    }
};

node {
      try {
          withCredentials([[$class: 'StringBinding', credentialsId: 'atomist-workspace', variable: 'ATOMIST_WORKSPACES']]) {
            final scmVars = checkout(scm)
            def url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
            echo "scmVars: ${scmVars}"
            echo "scmVars.GIT_COMMIT: ${scmVars.GIT_COMMIT}"
            echo "scmVars.GIT_BRANCH: ${scmVars.GIT_BRANCH}"
            echo "url: ${url}"

            echo 'Sending build start...'
            notifyAtomist(
                ATOMIST_WORKSPACES,
                'STARTED',
                url,
                scmVars.GIT_BRANCH,
                scmVars.GIT_COMMIT,
                'STARTED'
            )

            stage('Build Image') {
                def customImage = docker.build("ipcrm/docker-base-testing-things:${env.BUILD_ID}")
                docker.withRegistry('https://registry.hub.docker.com', 'dockerhub-creds') {
                    customImage.push("${env.BUILD_NUMBER}")
                    customImage.push("latest")
                }

                def repoName = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/')[3].split("\\.")[0]
                def ownerName = scm.getUserRemoteConfigs()[0].getUrl().tokenize('/')[2]

                echo "Found repo name to be ${repoName} and owner to be ${ownerName}"
                sendImageLink(
                  ATOMIST_WORKSPACES,
                  ownerName,
                  repoName,
                  scmVars.GIT_COMMIT,
                  "ipcrm/docker-base-testing-things:${env.BUILD_ID}"
                )
            }

            echo 'Sending build success...'
            currentBuild.result = 'SUCCESS'
            notifyAtomist(
                ATOMIST_WORKSPACES,
                currentBuild.result,
                url,
                scmVars.GIT_BRANCH,
                scmVars.GIT_COMMIT
            )
        }
      } catch (Exception err) {
          withCredentials([[$class: 'StringBinding', credentialsId: 'atomist-workspace', variable: 'ATOMIST_WORKSPACES']]) {
            echo "Failure discovered! ${err}"
            echo 'Sending build failure...'
            currentBuild.result = 'FAILURE'
            def url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
            final scmVars = checkout(scm)
            notifyAtomist(
              ATOMIST_WORKSPACES,
              currentBuild.result,
              url,
              scmVars.GIT_BRANCH,
              scmVars.GIT_COMMIT
            )
        }
      }
}
