pipeline {
    agent any
    triggers {
        cron('H 7 * * 0')
    }
    environment {
        BACKUP_HOST = 'keycloak'
        S3_BUCKET = 'my-keycloak-backups'
        S3_PROFILE = 'keycloak-s3'
        S3_REGION = 'us-east-1'
        GIT_REPO = 'https://github.com/praveenaws74/keycloak-backups.git'
        GIT_BRANCH = 'main'
        BACKUP_DIR = "${WORKSPACE}/keycloak-backup"
        // ANSI colors
        RESET = '\033[0m'
        RED = '\033[31m'
        GREEN = '\033[32m'
        YELLOW = '\033[33m'
        BLUE = '\033[34m'
        MAGENTA = '\033[35m'
        CYAN = '\033[36m'
        WHITE = '\033[37m'
        BOLD = '\033[1m'
    }
    options {
        ansiColor('xterm')
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    echo "${CYAN}${BOLD}"
                    echo "╔══════════════════════════════════════════════════════════════╗"
                    echo "║                 KEYCLOAK BACKUP PIPELINE                    ║"
                    echo "╚══════════════════════════════════════════════════════════════╝"
                    echo "${RESET}"
                    echo "${BLUE}Build Number    : ${BUILD_NUMBER}${RESET}"
                    echo "${BLUE}Build Timestamp : ${BUILD_TIMESTAMP}${RESET}"
                    echo "${BLUE}Backup Host     : ${BACKUP_HOST}${RESET}"
                    echo "${BLUE}S3 Bucket       : ${S3_BUCKET}${RESET}"
                    echo "${BLUE}Git Repository  : ${GIT_REPO}${RESET}"
                    sh '''
                        set -e
                        rm -rf "${BACKUP_DIR}"
                        mkdir -p "${BACKUP_DIR}/config"
                        chmod 700 "${BACKUP_DIR}"
                    '''
                    echo "${GREEN}✓ Backup workspace prepared${RESET}"
                }
            }
        }
        stage('Test Keycloak Connection') {
            steps {
                script {
                    echo "${CYAN}${BOLD}━━━ SSH Connectivity ━━━${RESET}"
                    sh '''
                        set -e
                        echo "Testing SSH connection to ${BACKUP_HOST}..."
                        ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=no pk@${BACKUP_HOST} hostname
                    '''
                    echo "${GREEN}✓ SSH connection successful${RESET}"
                }
            }
        }
        stage('Backup PostgreSQL') {
            steps {
                script {
                    echo "${CYAN}${BOLD}━━━ PostgreSQL Backup ━━━${RESET}"
                    sh '''
                        set -e
                        echo "→ Creating PostgreSQL dump..."
                        ssh pk@${BACKUP_HOST} "sudo -u postgres pg_dump -Fc keycloak > /tmp/keycloak-${BUILD_NUMBER}.dump"
                        echo "→ Copying dump to Jenkins..."
                        scp pk@${BACKUP_HOST}:/tmp/keycloak-${BUILD_NUMBER}.dump ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump
                        echo "→ Removing remote temporary dump..."
                        ssh pk@${BACKUP_HOST} "rm -f /tmp/keycloak-${BUILD_NUMBER}.dump"
                        echo "→ Compressing PostgreSQL dump..."
                        zstd -T0 ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump -o ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst
                        rm -f ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump
                        test -s ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst
                    '''
                    echo "${GREEN}✓ PostgreSQL backup created${RESET}"
                }
            }
        }
        stage('Backup Keycloak Configuration') {
            steps {
                script {
                    echo "${CYAN}${BOLD}━━━ Keycloak Configuration Backup ━━━${RESET}"
                    sh '''
                        set -e
                        echo "→ Backing up /etc/keycloak..."
                        echo "→ Backing up keycloak.service..."
                        ssh pk@${BACKUP_HOST} "sudo tar -czf /tmp/keycloak-config-${BUILD_NUMBER}.tar.gz /etc/keycloak /etc/systemd/system/keycloak.service"
                        echo "→ Copying configuration archive..."
                        scp pk@${BACKUP_HOST}:/tmp/keycloak-config-${BUILD_NUMBER}.tar.gz ${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz
                        echo "→ Removing remote temporary archive..."
                        ssh pk@${BACKUP_HOST} "sudo rm -f /tmp/keycloak-config-${BUILD_NUMBER}.tar.gz"
                        test -s ${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz
                    '''
                    echo "${GREEN}✓ Keycloak configuration backup created${RESET}"
                }
            }
        }
        stage('Validate Backups') {
            steps {
                script {
                    echo "${CYAN}${BOLD}━━━ Backup Validation ━━━${RESET}"
                    sh '''
                        set -e
                        echo "→ Validating PostgreSQL archive..."
                        zstd -t ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst
                        echo "→ Validating configuration archive..."
                        tar -tzf ${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz > /dev/null
                        echo "→ Calculating checksums..."
                        sha256sum ${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst > ${BACKUP_DIR}/postgres.sha256
                        sha256sum ${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz > ${BACKUP_DIR}/config.sha256
                    '''
                    echo "${GREEN}✓ Backup validation successful${RESET}"
                }
            }
        }
        stage('Create Manifest') {
            steps {
                script {
                    echo "${CYAN}${BOLD}━━━ Backup Manifest ━━━${RESET}"
                    sh '''
                        set -e
                        cat > ${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt <<EOF
============================================================
Keycloak Backup Manifest
============================================================

Jenkins Job       : ${JOB_NAME}
Build Number      : ${BUILD_NUMBER}
Build Timestamp   : ${BUILD_TIMESTAMP}
Keycloak Host     : ${BACKUP_HOST}
Keycloak Version  : 26.7.4

============================================================
Backup Files
============================================================

PostgreSQL:
keycloak-${BUILD_TIMESTAMP}.dump.zst

Configuration:
/etc/keycloak
/etc/systemd/system/keycloak.service

Archive:
config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz
============================================================
SHA256
============================================================
EOF
                        cat ${BACKUP_DIR}/postgres.sha256 >> ${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt
                        cat ${BACKUP_DIR}/config.sha256 >> ${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt
                    '''
                    echo "${GREEN}✓ Manifest generated${RESET}"
                }
            }
        }
        stage('Upload PostgreSQL to S3') {
            steps {
                script {
                    echo "${MAGENTA}${BOLD}━━━ S3: PostgreSQL Backup ━━━${RESET}"
                    s3Upload(
                        profileName: "${S3_PROFILE}",
                        consoleLogLevel: 'INFO',
                        dontSetBuildResultOnFailure: false,
                        dontWaitForConcurrentBuildCompletion: false,
                        pluginFailureResultConstraint: 'FAILURE',
                        entries: [[
                            bucket: "${S3_BUCKET}/postgres/${BUILD_TIMESTAMP}",
                            sourceFile: "keycloak-backup/keycloak-${BUILD_TIMESTAMP}.dump.zst",
                            excludedFile: '',
                            selectedRegion: "${S3_REGION}",
                            storageClass: 'STANDARD',
                            noUploadOnFailure: true,
                            uploadFromSlave: true,
                            managedArtifacts: false,
                            flatten: true,
                            gzipFiles: false,
                            keepForever: true,
                            showDirectlyInBrowser: false,
                            useServerSideEncryption: true
                        ]],
                        userMetadata: [
                            [key: 'backup-type', value: 'keycloak-postgresql'],
                            [key: 'build-number', value: "${BUILD_NUMBER}"],
                            [key: 'build-timestamp', value: "${BUILD_TIMESTAMP}"]
                        ]
                    )

                    echo "${GREEN}✓ PostgreSQL backup uploaded to S3${RESET}"
                }
            }
        }
        stage('Upload Configuration to S3') {
            steps {
                script {
                    echo "${MAGENTA}${BOLD}━━━ S3: Keycloak Configuration ━━━${RESET}"
                    s3Upload(
                        profileName: "${S3_PROFILE}",
                        consoleLogLevel: 'INFO',
                        dontSetBuildResultOnFailure: false,
                        dontWaitForConcurrentBuildCompletion: false,
                        pluginFailureResultConstraint: 'FAILURE',
                        entries: [[
                            bucket: "${S3_BUCKET}/config/${BUILD_TIMESTAMP}",
                            sourceFile: "keycloak-backup/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz",
                            excludedFile: '',
                            selectedRegion: "${S3_REGION}",
                            storageClass: 'STANDARD',
                            noUploadOnFailure: true,
                            uploadFromSlave: true,
                            managedArtifacts: false,
                            flatten: true,
                            gzipFiles: false,
                            keepForever: true,
                            showDirectlyInBrowser: false,
                            useServerSideEncryption: true
                        ]],
                        userMetadata: [
                            [key: 'backup-type', value: 'keycloak-configuration'],
                            [key: 'build-number', value: "${BUILD_NUMBER}"],
                            [key: 'build-timestamp', value: "${BUILD_TIMESTAMP}"]
                        ]
                    )
                    echo "${GREEN}✓ Keycloak configuration uploaded to S3${RESET}"
                }
            }
        }
        stage('Upload Manifest to S3') {
            steps {
                script {
                    echo "${MAGENTA}${BOLD}━━━ S3: Backup Manifest ━━━${RESET}"
                    s3Upload(
                        profileName: "${S3_PROFILE}",
                        consoleLogLevel: 'INFO',
                        dontSetBuildResultOnFailure: false,
                        dontWaitForConcurrentBuildCompletion: false,
                        pluginFailureResultConstraint: 'FAILURE',
                        entries: [[
                            bucket: "${S3_BUCKET}/manifests/${BUILD_TIMESTAMP}",
                            sourceFile: "keycloak-backup/manifest-${BUILD_TIMESTAMP}.txt",
                            excludedFile: '',
                            selectedRegion: "${S3_REGION}",
                            storageClass: 'STANDARD',
                            noUploadOnFailure: true,
                            uploadFromSlave: true,
                            managedArtifacts: false,
                            flatten: true,
                            gzipFiles: false,
                            keepForever: true,
                            showDirectlyInBrowser: false,
                            useServerSideEncryption: true
                        ]],
                        userMetadata: [
                            [key: 'backup-type', value: 'keycloak-manifest'],
                            [key: 'build-number', value: "${BUILD_NUMBER}"],
                            [key: 'build-timestamp', value: "${BUILD_TIMESTAMP}"]
                        ]
                    )
                    echo "${GREEN}✓ Manifest uploaded to S3${RESET}"
                }
            }
        }
        stage('Push to GitHub') {
            steps {
                script {
                    echo "${BLUE}${BOLD}━━━ GitHub Configuration Backup ━━━${RESET}"
                    withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh '''
                            set -e
                            echo "→ Cloning GitHub repository..."
                            rm -rf repo
                            git clone "https://${GIT_USER}:${GIT_TOKEN}@$(echo ${GIT_REPO} | sed 's#https://##')" repo
                            cd repo
                            mkdir -p config systemd scripts
                            echo "→ Copying Keycloak systemd service..."
                            scp pk@${BACKUP_HOST}:/etc/systemd/system/keycloak.service systemd/keycloak.service
                            echo "→ Creating sanitized keycloak.env.example..."
                            cat > config/keycloak.env.example <<EOF
KC_DB=postgres
KC_DB_URL=jdbc:postgresql://127.0.0.1:5432/keycloak
KC_DB_USERNAME=keycloak
KC_DB_PASSWORD=CHANGE_ME
EOF
                            echo "→ Creating sanitized bootstrap.env.example..."
                            cat > config/bootstrap.env.example <<EOF
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=CHANGE_ME
EOF

                            cat > .gitignore <<EOF
*.env
*.env.*
!*.env.example
*.dump
*.dump.*
*.tar
*.tar.gz
*.zst
*.zip
*.tmp
*.bak
workspace/
EOF

                            git config user.email "jenkins@homelab.local"
                            git config user.name "jenkins-self-backup"
                            git add .gitignore config/keycloak.env.example config/bootstrap.env.example systemd/keycloak.service
                            if git diff --cached --quiet; then
                                echo "No GitHub changes detected."
                                exit 0
                            fi
                            git commit -m "Keycloak configuration backup ${BUILD_TIMESTAMP}"
                            git push origin HEAD:${GIT_BRANCH}
                        '''
                    }
                    echo "${GREEN}✓ GitHub configuration backup completed${RESET}"
                }
            }
        }
        stage('Backup Summary') {
            steps {
                script {
                    echo ""
                    echo "${CYAN}${BOLD}"
                    echo "╔══════════════════════════════════════════════════════════════╗"
                    echo "║                  BACKUP COMPLETED ✓                         ║"
                    echo "╚══════════════════════════════════════════════════════════════╝"
                    echo "${RESET}"

                    echo "${GREEN}Build Number    : ${BUILD_NUMBER}${RESET}"
                    echo "${GREEN}Timestamp       : ${BUILD_TIMESTAMP}${RESET}"
                    echo "${GREEN}Keycloak Host   : ${BACKUP_HOST}${RESET}"
                    echo ""
                    echo "${MAGENTA}PostgreSQL:${RESET}"
                    echo "  s3://${S3_BUCKET}/postgres/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "${MAGENTA}Configuration:${RESET}"
                    echo "  s3://${S3_BUCKET}/config/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "${MAGENTA}Manifest:${RESET}"
                    echo "  s3://${S3_BUCKET}/manifests/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "${BLUE}GitHub:${RESET}"
                    echo "  ${GIT_REPO}"
                    echo ""
                    echo "${CYAN}Backup contents:${RESET}"
                    echo "  ✓ PostgreSQL database dump"
                    echo "  ✓ /etc/keycloak"
                    echo "  ✓ /etc/systemd/system/keycloak.service"
                    echo "  ✓ SHA256 checksums"
                    echo "  ✓ Backup manifest"
                    echo "  ✓ Sanitized GitHub configuration"
                    echo ""
                    echo "${GREEN}${BOLD}All backup operations completed successfully.${RESET}"
                }
            }
        }
    }
    post {
        always {
            sh 'rm -rf "${BACKUP_DIR}" repo'
        }
        success {
            echo "${GREEN}${BOLD}✓ KEYCLOAK BACKUP JOB SUCCESSFUL${RESET}"
        }
        failure {
            echo "${RED}${BOLD}✗ KEYCLOAK BACKUP JOB FAILED${RESET}"
            echo "${YELLOW}Check the Jenkins console output for the failed stage.${RESET}"
            mail to: 'praveenkumarhg8@gmail.com', subject: "Failed Pipeline: ${currentBuild.fullDisplayName}", body: "Servers Maintenance Failed ! ${env.BUILD_URL}"
        }
        aborted {
            echo "${YELLOW}${BOLD}⚠ KEYCLOAK BACKUP JOB ABORTED${RESET}"
        }
    }
}
