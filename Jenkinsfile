pipeline {
    agent any
    
    tools {
        maven 'Maven 3.3.9'
        jdk 'jdk7'
    }
    
    // !!! with jdk 7, the property https.protocols is required to be compatible with the Actility repository
    
    parameters {
    	booleanParam(name: 'RELEASE', defaultValue: false, description: 'To create a release')
    }
    
    triggers { 
        cron('H 0,12 * * 1-5')
        // each 12 hours from monday to friday, build a snapshot 
    }
    
    environment {
    	EMAIL_FROM = 'noreply.ci-lannion@actility.com'
    	// EMAIL_TO = 'gilles.landel@actility.com'
    	EMAIL_TO = 'all-lannion@actility.com'
    	
    	GIT_EMAIL = 'all-lannion@actility.com'
    	GIT_USER = 'actility-lannion'
    	
    	// Configuration file provider identifier
    	SETTINGS_ID = '039f3cd3-f61d-47b4-b2b5-9ff117d5bccf'
    	
    	// Git credential identifier
    	CREDENTIALS_ID = '4b88a170-863d-4551-aaa1-f2b076a77f97'
    	
    	// Server identifier defined in Jenkins configuration
    	SONAR_SERVER = 'SonarQube'
    	
    	// The max number of logs to keep
    	LOGS_NUMBER = '10'
    }
    
    stages {
        stage('Init') {
            steps {
            	script {
            		IS_RELEASE = "${params.RELEASE}".toBoolean()
            		GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
            		
            		echo "RELEASE = $IS_RELEASE, BRANCH = $GIT_BRANCH"
            		
            		if (IS_RELEASE) {
            			env.RELEASE = " release"
            		} else {
            			env.RELEASE = ""
            		}
            		
            		// Remove old builds logs (disabled, jenkins don't show the build with parameters if uncommented)
            		// properties([ 
				    //  [$class: 'jenkins.model.BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: "${env.LOGS_NUMBER}"]]
				    // ])
            	}
            }
        }
        stage('Build') {
            steps {
            	script {
            		configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
						sh 'mvn clean package -B -s $MAVEN_SETTINGS'
				    }
                }
            }
        }
        stage('Quality Scan') {
        	when {
                expression { !params.RELEASE }
            }
            
			steps {
				// withSonarQubeEnv only works with sonar-maven-plugin:3.0.2
				// or with sonar-maven-plugin:3.2 with an http sonar server (tried with SonarQube 6.2 in a docker)
				// maybe an https issue, try to inject cacert ???
			
				// JDK8 is required for the maven sonar plugin
				withEnv(["JAVA_HOME=${ tool 'jdk8' }"]) {
					configFileProvider([configFile(fileId: "${env.SETTINGS_ID}", variable: 'MAVEN_SETTINGS')]) {
						sh "mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -s $MAVEN_SETTINGS"
					}
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
            			sh 'mvn deploy -B -s $MAVEN_SETTINGS -Dhttps.protocols=TLSv1.2'
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
                			sh "mvn release:prepare -B -s $MAVEN_SETTINGS -Dhttps.protocols=TLSv1.2"
                			
                			echo "release"
                			sh "mvn release:perform -B -s $MAVEN_SETTINGS -Darguments='-DskipTests=true -s $MAVEN_SETTINGS -Dhttps.protocols=TLSv1.2'"
                		}
                		
                		// FUTURE, FROM RELEASE BRANCH
                		// echo 'Merging release to master...'
                		// sh "git fetch origin +master:master"
						// sh "git checkout master"
						// sh "git merge release"
						// sh "git push origin/master"
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