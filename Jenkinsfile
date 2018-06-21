#!/usr/bin/env groovy


openshift.withCluster() { // Use "default" cluster or fallback to OpenShift cluster detection

    stage('environment') {
        echo "Print environment vars for debug purposes"
        sh 'env > env.txt'
        for (String i : readFile('env.txt').split("\r?\n")) {
            println i
        }
    }

    String projectName = encodeName("${JOB_NAME}")
    echo "name=${projectName}"

    //Stages outside a node declaration runs on the jenkins host
    //GO to a node with ruby
    //    https://github.com/redhat-cop/containers-quickstarts/tree/master/jenkins-slaves/jenkins-slave-ruby
//    node('jenkins-slave-ruby') {
    stage('checkout') {
        checkout scm
        url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
    }

    stage('Create test project') {
        recreateProject(projectName)

        openshift.withProject(projectName) {

            APPLICATION_NAME = projectName
            stage("Start rails app") {
                url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()

                def rubyApp = openshift.newApp("${url}#development", "--strategy=source")

                echo "new-app created ${rubyApp.count()} objects named: ${rubyApp.names()}"

                rubyApp.describe()

                def build = rubyApp.narrow("bc")
                build.logs("-f")

                def deployment = rubyApp.narrow("dc")
                deployment.logs("-f")

                //We have to create a route, as the maven node is not part of the same project as the tomcat pod,
                // and thus cannot connect directly to the service

                openshift.raw("expose", "service/${rubyApp.narrow("svc").object().metadata.name}")
                route = openshift.selector("route/${rubyApp.narrow("svc").object().metadata.name}")
                routeURL = "http://${route.object().spec.host}"
                echo "Webserver host exposes as ${routeURL}"
            }

            stage("test rails app") {
                sh "curl ${routeURL}"
            }

        }
    }
}


private void recreateProject(String projectName) {
    echo "Delete the project ${projectName}, ignore errors if the project does not exist"
    try {
        openshift.selector("project/${projectName}").delete()
    } catch (e) {

    }

    openshift.selector("project/${projectName}").watch {
        echo "Waiting for the project ${projectName} to be deleted"
        return it.count() == 0
    }
//
//    //Wait for the project to be gone
//    sh "until ! oc get project ${projectName}; do date;sleep 2; done; exit 0"

    echo "Create the project ${projectName}"
    openshift.newProject(projectName)
}

/**
 * Encode the jobname as a valid openshift project name
 * @param jobName the name of the job
 * @return the jobname as a valid openshift project name
 */
private static String encodeName(groovy.lang.GString jobName) {
    def name = jobName
            .replaceAll("\\s", "-")
            .replaceAll("_", "-")
            .replaceFirst("^[^/]+/", '')
            .replace("/", '-')
            .replaceAll("^openshift-", "")
            .toLowerCase()
    return name
}
