pipeline {

    agent any

    parameters {

        booleanParam(
            name: 'RUN_STABILITY',
            defaultValue: true,
            description: 'Run Code Stability / Unit Tests'
        )

        booleanParam(
            name: 'RUN_QUALITY',
            defaultValue: true,
            description: 'Run Code Quality Analysis'
        )

        booleanParam(
            name: 'RUN_COVERAGE',
            defaultValue: true,
            description: 'Run Code Coverage Analysis'
        )
    }

    environment {
        JAVA_TOOL_OPTIONS = '--add-opens=java.base/java.lang=ALL-UNNAMED'
    }

    stages {

        // =========================================================
        // 1. CODE CHECKOUT
        // =========================================================

        stage('Code Checkout') {

            steps {

                git branch: 'master',
                    url: 'https://github.com/yogiindoria/spring3hibernate.git'
            }
        }


        // =========================================================
        // 2. BUILD
        // =========================================================

        stage('Build') {

            steps {

                sh '''
                    echo "===== JAVA VERSION ====="
                    java -version

                    echo "===== MAVEN VERSION ====="
                    mvn -version

                    echo "===== FIXING JAVA COMPILER SETTINGS ====="

                    sed -i 's/<source>1.6<\\/source>/<source>1.8<\\/source>/g' pom.xml
                    sed -i 's/<target>1.6<\\/target>/<target>1.8<\\/target>/g' pom.xml

                    echo "===== BUILDING APPLICATION ====="

                    mvn clean package \
                        -DskipTests \
                        -Dfindbugs.skip=true

                    echo "===== BUILD COMPLETED ====="
                '''

                // Source code for parallel stages
                stash(
                    name: 'source-code',
                    includes: '**/*',
                    excludes: 'target/**,.git/**'
                )

                // WAR file for final publication
                stash(
                    name: 'war-artifact',
                    includes: 'target/Spring3HibernateApp.war'
                )
            }
        }


        // =========================================================
        // 3. PARALLEL ANALYSIS
        // =========================================================

        stage('Parallel Analysis') {

            parallel {


                // -------------------------------------------------
                // CODE STABILITY
                // -------------------------------------------------

                stage('Code Stability') {

                    when {
                        expression {
                            params.RUN_STABILITY
                        }
                    }

                    steps {

                        ws("${env.WORKSPACE}@stability") {

                            deleteDir()

                            unstash 'source-code'

                            sh '''
                                echo "===== CODE STABILITY ANALYSIS ====="
                            
                                sed -i 's/<source>1.6<\\/source>/<source>1.8<\\/source>/g' pom.xml
                                sed -i 's/<target>1.6<\\/target>/<target>1.8<\\/target>/g' pom.xml
                            
                                mvn clean test -Dfindbugs.skip=true
                            
                                echo "===== SUREFIRE REPORTS ====="
                                find target -type f -name "*.xml" -o -name "*.txt" 2>/dev/null || true
                            
                                echo "===== SUREFIRE DIRECTORY ====="
                                ls -lah target/surefire-reports/ 2>/dev/null || echo "surefire-reports directory NOT FOUND"
                            
                                echo "===== UNIT TESTS COMPLETED ====="
                            '''
                            stash(
                                name: 'stability-report',
                                includes: 'target/surefire-reports/**',
                                allowEmpty: true
                            )
                        }
                    }
                }


                // -------------------------------------------------
                // CODE QUALITY
                // -------------------------------------------------

                stage('SonarQube Quality Analysis') {

                    when {
                        expression {
                            params.RUN_QUALITY
                        }
                    }
                
                    steps {
                
                        ws("${env.WORKSPACE}@quality") {
                
                            deleteDir()
                
                            git branch: 'master',
                                url: 'https://github.com/yogiindoria/spring3hibernate.git'
                
                            sh '''
                                echo "===== SONARQUBE CODE QUALITY ANALYSIS ====="
                
                                sed -i 's/<source>1.6<\\/source>/<source>1.8<\\/source>/g' pom.xml
                                sed -i 's/<target>1.6<\\/target>/<target>1.8<\\/target>/g' pom.xml
                
                                echo "===== COMPILING PROJECT FOR SONARQUBE ====="
                
                                mvn clean package \
                                    -DskipTests \
                                    -Dfindbugs.skip=true
                
                                echo "===== RUNNING SONARQUBE ANALYSIS ====="
                            '''
                
                            withSonarQubeEnv('SonarQube') {
                
                                sh '''
                                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar \
                                        -Dsonar.projectKey=Spring3Hibernate \
                                        -Dsonar.projectName=Spring3Hibernate
                                '''
                            }
                
                            echo "===== SONARQUBE ANALYSIS COMPLETED ====="
                        }
                    }
                }

                // -------------------------------------------------
                // CODE COVERAGE
                // -------------------------------------------------

                stage('Code Coverage') {

                    when {
                        expression {
                            params.RUN_COVERAGE
                        }
                    }

                    steps {

                        ws("${env.WORKSPACE}@coverage") {

                            deleteDir()

                            unstash 'source-code'

                            sh '''
                                echo "===== CODE COVERAGE ANALYSIS ====="

                                sed -i 's/<source>1.6<\\/source>/<source>1.8<\\/source>/g' pom.xml
                                sed -i 's/<target>1.6<\\/target>/<target>1.8<\\/target>/g' pom.xml

                                mvn test -Dfindbugs.skip=true

                                mvn jacoco:report \
                                    -Dfindbugs.skip=true

                                echo "===== CODE COVERAGE ANALYSIS COMPLETED ====="
                            '''

                            stash(
                                name: 'coverage-report',
                                includes: 'target/site/jacoco/**',
                                allowEmpty: true
                            )
                        }
                    }
                }
            }
        }

	
        // =========================================================
        // 4. GENERATE REPORTS
        // =========================================================

                     stage('Generate Reports') {
                    
                        steps {
                    
                            script {
                    
                                if (params.RUN_STABILITY) {
                    
                                    unstash 'stability-report'
                    
                                    junit(
                                        allowEmptyResults: true,
                                        testResults: 'target/surefire-reports/*.xml'
                                    )
                                }
                    
                                if (params.RUN_COVERAGE) {
                    
                                    unstash 'coverage-report'
                    
                                    archiveArtifacts(
                                        artifacts: 'target/site/jacoco/**',
                                        allowEmptyArchive: true
                                    )
                                }
                            }
                        }
                    }

                    // -------------------------------
                    // Coverage Report
                    // -------------------------------

                   

        // =========================================================
        // 5. APPROVAL
        // =========================================================

        stage('Approval') {

            steps {

                script {

                    timeout(
                        time: 5,
                        unit: 'MINUTES'
                    ) {

                        def decision = input(
                            message: 'Approve artifact publication?',
                            parameters: [
                                choice(
                                    name: 'DECISION',
                                    choices: [
                                        'APPROVE',
                                        'DENY'
                                    ],
                                    description: 'Choose whether the artifact should be published'
                                )
                            ]
                        )

                        if (decision != 'APPROVE') {

                            error(
                                'Artifact publication denied by user.'
                            )
                        }

                        echo 'Artifact publication approved.'
                    }
                }
            }
        }


        // =========================================================
        // 6. PUBLISH ARTIFACT
        // =========================================================

        stage('Publish Artifact') {

            steps {

                // Get WAR created during Build stage
                unstash 'war-artifact'

                echo '===== PUBLISHING ARTIFACT ====='

                archiveArtifacts(
                    artifacts: 'target/Spring3HibernateApp.war',
                    fingerprint: true
                )

                echo 'WAR artifact published successfully.'
            }


            // =====================================================
            // 7. NOTIFICATIONS AFTER PUBLICATION
            // =====================================================

            post {

                success {

                    echo 'Artifact publication successful.'
                    
                    slackSend(
                        channel: '#jenkins',
                        message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} - WAR artifact published successfully."
                    )
                    
                    emailext(
                        subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """\
                            Jenkins Pipeline completed successfully.
                    
                            Job: ${env.JOB_NAME}
                            Build: #${env.BUILD_NUMBER}
                            Artifact: Spring3HibernateApp.war
                    
                            Build URL:
                            ${env.BUILD_URL}
                        """,
                        to: 'tanushirana081@gmail.com'
                    )
                }


                failure {

                    echo 'Artifact publication failed.'
                    
                    slackSend(
                        channel: '#jenkins',
                        message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} - artifact publication failed."
                    )

                    emailext(
                        subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: """\
                            Artifact publication failed.
                    
                            Job: ${env.JOB_NAME}
                            Build: #${env.BUILD_NUMBER}
                    
                            Check Jenkins console output:
                            ${env.BUILD_URL}
                        """,
                        to: 'tanushirana875@gmail.com'
                    )
                }
            }
        }
    }


    // =============================================================
    // PIPELINE LEVEL POST ACTIONS
    // =============================================================

    post {

        success {

            echo '========================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
            echo '========================================'
        }

        failure {

            echo '========================================'
            echo 'PIPELINE FAILED'
            echo '========================================'
        }

        aborted {

            echo '========================================'
            echo 'PIPELINE ABORTED'
            echo '========================================'
        }
    }
}
