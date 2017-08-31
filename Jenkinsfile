#!/usr/bin/env groovy

@Library('thingpark/shared-jenkins-library') _
// @Library('thingpark/shared-jenkins-library@1.0.0') _

// for all parameters, see example: https://git.int.actility.com/thingpark/shared-jenkins-library/blob/master/Jenkinsfile

thingparkPipeline {

    // the main branch of the project (master, develop)
    // [MANDATORY]
    mainBranch = 'master'

    // if true, post release a merge is performed into master branch (from mainBranch)
    // [MANDATORY]
    mergeToMaster = false

    // java tool (jdk7, jdk8)
    // [MANDATORY]
    javaTool = 'jdk8'

    // the project name used in email notifications
    // [MANDATORY]
    projectName = 'MongoJack'

    // Email configuration
    
    // Email receiver if release in success
    // [OPTIONAL, default: RD_SW_release_delivery@actility.com]
    emailToRelease = 'all-lannion@actility.com'

    // Email in copy receiver if release in success
    // [OPTIONAL, default: all-lannion@actility.com, info: to avoid CC email, just specify a text not blank without the character '@' like 'None']
    emailCcRelease = 'None'

    // Others configuration

    // Maven credentials identifier
    // [OPTIONAL, default: 4b88a170-863d-4551-aaa1-f2b076a77f97]
    credentialsId = 'github-lannion'
}