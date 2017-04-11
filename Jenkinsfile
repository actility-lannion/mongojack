pipeline {
	agent any
	
	tools {
		maven 'Maven 3.3.9'
		jdk 'jdk8'
	}
	
	parameters {
		// valid: booleanParam, choice, file, text, password, run, string
		
		booleanParam(name: 'RELEASE', defaultValue: false, description: 'To create a release')
		booleanParam(name: 'SONAR', defaultValue: false, description: 'Force the push of statistics on SonarQube (by default master and develop are automatically pushed)')
	}
	
	triggers {
		// valid: upstream, cron, gitlab, pollSCM
	
		cron('H 0,12 * * 1-5')
		// each 12 hours from monday to friday, build a snapshot
		
		gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All')
	}
	
	options {
		// The max number of logs to keep (valid: daysToKeep, numToKeep, artifactDaysToKeep, artifactNumToKeep)
	    buildDiscarder(logRotator(numToKeepStr: '10'))
	    
	    // Max timeout build
	    timeout(time: 60, unit: 'MINUTES')
	}
	
	stages {
		stage('Init') {
			steps {
				script {
					IS_RELEASE = "${params.RELEASE}".toBoolean()
					GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
					
					echo "RELEASE = $IS_RELEASE, BRANCH = $BRANCH_NAME / $GIT_BRANCH"
					
					if (IS_RELEASE) {
						env.RELEASE = " release"
					} else {
						env.RELEASE = ""
					}

					// environment block don't work (ClassCastException), need further analysis
					
					env.EMAIL_FROM = 'noreply.ci-lannion@actility.com'
					// env.EMAIL_TO = 'gilles.landel@actility.com'
					env.EMAIL_TO = 'all-lannion@actility.com'
					
					env.GIT_EMAIL = 'it+jenkins@actility.com'
					env.GIT_USER = 'Jenkins'
					
					// Configuration file provider identifier
					env.SETTINGS_ID = '039f3cd3-f61d-47b4-b2b5-9ff117d5bccf'
					
					// Git credential identifier
					env.CREDENTIALS_ID = '4b88a170-863d-4551-aaa1-f2b076a77f97'
					
					// Server identifier defined in Jenkins configuration
					env.SONAR_SERVER = 'SonarQube'
				}
			}
		}
		stage('Build') {
			steps {
				script {
					configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
						sh 'mvn clean compile -B -s $MAVEN_SETTINGS'
					}
				}
			}
		}
		stage('Test') {
			when {
				expression { !params.RELEASE }
			}
			
			steps {
				configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
					sh "mvn test -s $MAVEN_SETTINGS"
				}
			}
		}
		stage('Quality Scan') {
			when {
				expression { !params.RELEASE && (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'develop' || params.SONAR) }
			}
			
			steps {
				// withSonarQubeEnv only works with org.codehaus.mojo:sonar-maven-plugin:3.0.2
				// or with org.sonarsource.scanner.maven:sonar-maven-plugin:3.2 with an http sonar server (tried with SonarQube 6.2 in a docker)
				// maybe an https issue, try to inject cacert ???
			
				configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
					sh "mvn jacoco:report sonar:sonar -s $MAVEN_SETTINGS"
				}
			}
		}
		stage('Staging') {
			when {
				expression { !params.RELEASE }
			}
			
			steps {
				script {
					echo 'Deploying snapshot...'
					echo "RELEASE = $IS_RELEASE - $GIT_BRANCH"
				
					configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
						sh 'mvn deploy -B -s $MAVEN_SETTINGS -DskipTests=true'
					}
				}
			}
		}
		stage('Release') {
			when {
				expression { params.RELEASE }
			}
		
			steps {
				script {
					echo 'Releasing...'
				
					sshagent (credentials: ["${env.CREDENTIALS_ID}"]) {
						sh "git config --global user.email '${env.GIT_EMAIL}'"
						sh "git config --global user.name '${env.GIT_USER}'"
						
						// Jenkins checkout as detached head, but maven-release-plugin does not like working that way
						// sh "git checkout release"
						sh "git checkout master"
					
						configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
							
							echo "prepare"
							sh "mvn release:prepare -B -s $MAVEN_SETTINGS"
							
							echo "release"
							sh "mvn release:perform -B -s $MAVEN_SETTINGS -Darguments='-DskipTests=true'"
						}
						
						// FUTURE, FROM RELEASE BRANCH
						// echo 'Merging release to master...'
						// sh "git fetch origin +master:master"
						// sh "git checkout master"
						// sh "git merge release"
						// sh "git push origin master"
					}
				}
			}
			
			post {
				success {
					echo "send success email to ${env.EMAIL_TO}"
					
					mail(from: "${env.EMAIL_FROM}", to: "${env.EMAIL_TO}", subject: "[CI] SUCCESS: ${currentBuild.fullDisplayName} release", body: "Yeah, we succeeded.")
				}
			}
		}
	}
	
	post {
		failure {
			echo "send failure email to ${env.EMAIL_TO}"
			
			mail(from: "${env.EMAIL_FROM}", to: "${env.EMAIL_TO}", subject: "[CI] FAILED: ${currentBuild.fullDisplayName}${env.RELEASE}", body: "Boo, we failed.")
		}
		unstable {
			echo "send unstable email to ${env.EMAIL_TO}"
			
			mail(from: "${env.EMAIL_FROM}", to: "${env.EMAIL_TO}", subject: "[CI] UNSTABLE: ${currentBuild.fullDisplayName}${env.RELEASE}", body: "Huh, we're unstable.")
		}
	}
}