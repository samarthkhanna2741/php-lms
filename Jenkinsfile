/**
 * Jenkinsfile for a PHP & MySQL Library Management System
 *
 * This pipeline implements the design outlined in the CI/CD document.
 * It uses Docker for building and testing, and SSH for deployments.
 *
 * Prerequisites on Jenkins:
 * 1. Docker plugin installed.
 * 2. SSH Agent plugin installed.
 * 3. Credentials stored in Jenkins for:
 * - SSH access to staging ('staging-ssh-creds')
 * - SSH access to production ('prod-ssh-creds')
 */
pipeline {
    // Agent defines the execution environment. We use Docker to have a clean PHP environment.
    // We also run a MySQL container alongside it as a "sidecar" for testing.
    agent {
        docker {
            image 'php:8.1-fpm' // Use a specific PHP version
            args '-v /var/run/docker.sock:/var/run/docker.sock' // Mount docker sock to use docker from within docker
            // The 'mysql:8.0' container will be available on the hostname 'mysql'
            // inside the build network.
            registryUrl 'https://registry.hub.docker.com/'
            registryCredentialsId ''
        }
    }

    // Environment variables available to all stages
    environment {
        // Use Jenkins credentials for sensitive data
        DB_DATABASE      = 'test_library'
        DB_USER          = 'root'
        DB_PASSWORD      = 'password' // Use a secure password in a real scenario
        DB_HOST          = 'mysql'
        ARTIFACT_NAME    = "library-management-${env.BUILD_NUMBER}.zip"
    }

    stages {

        // =================================================================
        // STAGE 1: COMMIT - This is handled by Jenkins SCM trigger
        // =================================================================

        // =================================================================
        // STAGE 2: Build & Initial Test
        // =================================================================
        stage('Build & Unit Test') {
            steps {
                script {
                    // Start the MySQL sidecar container for this stage
                    sh 'docker run --name mysql -e MYSQL_ROOT_PASSWORD=${DB_PASSWORD} -e MYSQL_DATABASE=${DB_DATABASE} --network=host -d mysql:8.0'
                    
                    // Wait for MySQL to be ready
                    sh 'while ! docker exec mysql mysqladmin ping -h"127.0.0.1" --silent; do sleep 1; done'
                    
                    echo '--> Installing System Dependencies & PHP Extensions'
                    // Example: Installing git, zip, and common PHP extensions
                    sh 'apt-get update && apt-get install -y git zip unzip libzip-dev && docker-php-ext-install pdo pdo_mysql zip'

                    echo '--> Installing Composer Dependencies'
                    sh 'php -r "copy(\'https://getcomposer.org/installer\', \'composer-setup.php\');"'
                    sh 'php composer-setup.php --install-dir=/usr/local/bin --filename=composer'
                    sh 'composer install --no-interaction --prefer-dist --optimize-autoloader'

                    echo '--> Running Database Migrations on Test DB'
                    // Replace with your actual migration command
                    sh 'php artisan migrate --seed --force'

                    echo '--> Running Unit Tests'
                    // Assumes phpunit.xml is configured to run unit tests
                    sh './vendor/bin/phpunit --testsuite Unit'
                }
            }
            post {
                always {
                    // Cleanup: Stop and remove the MySQL container
                    sh 'docker stop mysql && docker rm mysql'
                }
            }
        }

        // =================================================================
        // STAGE 3: Quality & Security Gates
        // =================================================================
        stage('Quality & Security Gates') {
            steps {
                script {
                    // Start MySQL again for integration tests
                    sh 'docker run --name mysql -e MYSQL_ROOT_PASSWORD=${DB_PASSWORD} -e MYSQL_DATABASE=${DB_DATABASE} --network=host -d mysql:8.0'
                    sh 'while ! docker exec mysql mysqladmin ping -h"127.0.0.1" --silent; do sleep 1; done'
                    sh 'php artisan migrate --force' // Ensure schema is present

                    // Run tests in parallel to save time
                    parallel(
                        "Static Analysis": {
                            echo '--> Running Static Code Analysis (PHPStan)'
                            sh './vendor/bin/phpstan analyse'
                        },
                        "Coding Standards": {
                            echo '--> Checking Coding Standards (PHP_CodeSniffer)'
                            sh './vendor/bin/phpcs'
                        },
                        "Integration Tests": {
                            echo '--> Running Integration Tests'
                            sh './vendor/bin/phpunit --testsuite Integration'
                        },
                        "Security Scan": {
                            echo '--> Scanning Composer Dependencies (Snyk/Dependabot Placeholder)'
                            // In a real pipeline, you would integrate a tool like Snyk here.
                            // For example: `snyk test`
                            sh 'echo "Security scan placeholder"'
                        }
                    )
                }
            }
            post {
                always {
                    sh 'docker stop mysql && docker rm mysql'
                }
            }
        }

        // =================================================================
        // STAGE: Create and Archive Artifact
        // =================================================================
        stage('Archive Artifact') {
            steps {
                echo "--> Creating deployment artifact: ${ARTIFACT_NAME}"
                // Create a zip file excluding dev files and folders
                sh "zip -r ${ARTIFACT_NAME} . -x '*.git*' '*node_modules*' '*tests*' '*phpunit.xml*' '*Jenkinsfile*'"
                
                echo "--> Archiving artifact"
                archiveArtifacts artifacts: "${ARTIFACT_NAME}", fingerprint: true
            }
        }

        // =================================================================
        // STAGE 4: Deploy to Staging
        // =================================================================
        stage('Deploy to Staging') {
            steps {
                script {
                    // Retrieve the artifact from the archive
                    unarchiveMapping(
                        mapping: [source: "${ARTIFACT_NAME}", target: '.']
                    )
                    
                    // Use the SSH Agent plugin to securely connect to the staging server
                    // 'staging-ssh-creds' must be configured in Jenkins Credentials
                    sshagent(credentials: ['staging-ssh-creds']) {
                        def remote = [
                            user: 'your-staging-user',
                            host: 'staging.server.com',
                            releaseDir: "/var/www/releases/release-${env.BUILD_NUMBER}",
                            currentDir: '/var/www/html'
                        ]

                        echo "--> Deploying to Staging Server: ${remote.host}"
                        
                        // 1. Copy artifact to server
                        sh "scp ./${ARTIFACT_NAME} ${remote.user}@${remote.host}:/tmp/"
                        
                        // 2. Unzip, run migrations, and switch symlink on the remote server
                        sh """
                            ssh ${remote.user}@${remote.host} '
                                mkdir -p ${remote.releaseDir} &&
                                unzip /tmp/${ARTIFACT_NAME} -d ${remote.releaseDir} &&
                                cd ${remote.releaseDir} &&
                                php artisan migrate --force &&
                                php artisan cache:clear &&
                                ln -snf ${remote.releaseDir} ${remote.currentDir} &&
                                rm /tmp/${ARTIFACT_NAME} &&
                                echo "Staging deployment complete!"
                            '
                        """
                    }
                }
            }
        }

        // =================================================================
        // STAGE 5: Deploy to Production
        // =================================================================
        stage('Deploy to Production') {
            // This stage requires manual approval before proceeding.
            // It will timeout after 1 day if no one approves.
            input {
                message "Deploy to Production?"
                ok "Yes, Deploy!"
                submitter 'authorized-group, some-user' // Optional: Restrict who can approve
            }
            steps {
                script {
                    // Retrieve the artifact again for this stage
                    unarchiveMapping(
                        mapping: [source: "${ARTIFACT_NAME}", target: '.']
                    )
                    
                    sshagent(credentials: ['prod-ssh-creds']) {
                        // Using a Blue-Green strategy
                        def remote = [
                            user: 'your-prod-user',
                            host: 'prod-green-server.com', // Deploying to the INACTIVE server
                            releaseDir: "/var/www/releases/release-${env.BUILD_NUMBER}",
                            currentDir: '/var/www/html'
                        ]

                        echo "--> Deploying to Production (Green Environment): ${remote.host}"

                        sh "scp ./${ARTIFACT_NAME} ${remote.user}@${remote.host}:/tmp/"
                        
                        sh """
                            ssh ${remote.user}@${remote.host} '
                                mkdir -p ${remote.releaseDir} &&
                                unzip /tmp/${ARTIFACT_NAME} -d ${remote.releaseDir} &&
                                cd ${remote.releaseDir} &&
                                php artisan migrate --force &&
                                php artisan config:cache &&
                                php artisan route:cache &&
                                ln -snf ${remote.releaseDir} ${remote.currentDir} &&
                                rm /tmp/${ARTIFACT_NAME} &&
                                echo "Production deployment to Green environment complete!"
                            '
                        """
                        
                        // The final step of switching traffic (e.g., updating a load balancer)
                        // would be scripted here. This is highly dependent on your infrastructure.
                        echo "--> MANUAL STEP REQUIRED: Switch load balancer to point to the Green environment."
                    }
                }
            }
        }
    }
}
