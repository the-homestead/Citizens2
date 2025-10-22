pipeline {
    agent any
    
    tools {
        // Make sure Jenkins has Maven configured with this name
        maven 'Maven' // Adjust this to match your Jenkins Maven installation name
        jdk 'jdk21'        // Adjust this to match your Jenkins JDK 21 installation name
    }
    
    environment {
        // Set Maven options
        MAVEN_OPTS = '-Xmx2048m -Xms1024m'
        // Skip downloading sources and javadocs for faster builds
        MAVEN_CONFIG = '-Dmaven.repo.local=.m2/repository'
        // Set build number for Citizens
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }
    
    options {
        // Keep builds for 30 days
        buildDiscarder(logRotator(daysToKeepStr: '30', numToKeepStr: '50'))
        // Timeout after 45 minutes (Citizens2 can take longer to build)
        timeout(time: 45, unit: 'MINUTES')
        // Add timestamps to console output
        timestamps()
    }
    
    triggers {
        // Poll SCM every 5 minutes for changes (adjust as needed)
        pollSCM('H/5 * * * *')
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out Citizens2 source code...'
                checkout scm
            }
        }
        
        stage('Build Info') {
            steps {
                script {
                    echo "Building Citizens2 branch: ${env.BRANCH_NAME}"
                    echo "Build number: ${env.BUILD_NUMBER}"
                    echo "Target: Minecraft 1.21.1 compatible JAR"
                    echo "Java version check:"
                    bat 'java -version'
                    echo "Maven version check:"
                    bat 'mvn -version'
                }
            }
        }
        
        stage('Clean') {
            steps {
                echo 'Cleaning previous Citizens2 builds...'
                bat 'mvn clean'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling Citizens2 project...'
                // Use spigot-release profile which includes 1.21 modules
                bat 'mvn compile -P spigot-release'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running Citizens2 tests...'
                script {
                    try {
                        bat 'mvn test -P spigot-release'
                    } catch (Exception e) {
                        echo 'Tests failed, but continuing build...'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
            post {
                always {
                    // Publish test results if they exist
                    script {
                        if (fileExists('**/target/surefire-reports/*.xml')) {
                            junit testResultsPattern: '**/target/surefire-reports/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                echo 'Packaging Citizens2 for Minecraft 1.21.1...'
                // Build with spigot-release profile to include 1.21 support
                bat 'mvn package -P spigot-release -DskipTests -DBUILD_NUMBER=%BUILD_NUMBER%'
            }
        }
        
        stage('Verify Output JAR') {
            steps {
                echo 'Verifying Citizens2 JAR was created...'
                script {
                    // Check if the Citizens JAR was created in dist/target
                    def jarExists = fileExists('dist/target/Citizens-*.jar')
                    if (jarExists) {
                        echo 'Citizens2 JAR successfully created!'
                        bat 'dir "dist\\target\\Citizens-*.jar"'
                    } else {
                        error 'Citizens2 JAR was not found in dist/target!'
                    }
                }
            }
        }
        
        stage('Install') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                echo 'Installing Citizens2 to local repository...'
                bat 'mvn install -P spigot-release -DskipTests -DBUILD_NUMBER=%BUILD_NUMBER%'
            }
        }
    }
    
    post {
        always {
            echo 'Archiving Citizens2 build artifacts...'
            
            // Archive the main Citizens JAR (this is the output JAR you want)
            archiveArtifacts artifacts: 'dist/target/Citizens-*.jar', 
                           fingerprint: true, 
                           allowEmptyArchive: false,
                           caseSensitive: false
            
            // Archive plugin.yml for reference
            archiveArtifacts artifacts: '**/src/main/resources/plugin.yml', 
                           fingerprint: true, 
                           allowEmptyArchive: true
            
            // Archive individual module JARs for debugging if needed
            archiveArtifacts artifacts: 'main/target/citizens-main-*.jar, v1_21_R5/target/citizens-v1_21_R5-*.jar, v1_21_R6/target/citizens-v1_21_R6-*.jar', 
                           fingerprint: true, 
                           allowEmptyArchive: true
        }
        
        success {
            echo 'Citizens2 build completed successfully!'
            script {
                if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME == 'main') {
                    echo 'Master/Main branch build succeeded - Citizens2 JAR ready for Minecraft 1.21.1 servers!'
                    
                    // Display information about the built JAR
                    bat '''
                        echo "=== Citizens2 Build Summary ==="
                        echo "JAR Location: dist/target/"
                        dir "dist\\target\\Citizens-*.jar"
                        echo "Compatible with: Minecraft 1.21.1"
                        echo "==================================="
                    '''
                }
            }
        }
        
        failure {
            echo 'Citizens2 build failed!'
            // You can add notification steps here (email, Slack, Discord, etc.)
            script {
                bat '''
                    echo "=== Build Failure Debug Info ==="
                    echo "Check the following locations for clues:"
                    echo "- main/target/ for main module issues"
                    echo "- v1_21_R5/target/ and v1_21_R6/target/ for NMS issues"
                    echo "- dist/target/ for assembly issues"
                    if exist "main\\target" dir "main\\target"
                    if exist "dist\\target" dir "dist\\target"
                '''
            }
        }
        
        unstable {
            echo 'Citizens2 build completed with test failures but JAR was created'
        }
        
        cleanup {
            // Clean up workspace if needed (commented out to preserve artifacts for debugging)
            // deleteDir()
        }
    }
}