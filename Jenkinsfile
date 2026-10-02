pipeline {
    agent any
    environment {
        BACKUP_HOST = 'keycloak'
        S3_BUCKET = 'my-keycloak-backups'
        S3_PROFILE = 'keycloak-s3'
        S3_REGION = 'us-east-1'
        GIT_REPO = 'https://github.com/praveenaws74/keycloak-backups.git'
        GIT_BRANCH = 'main'
        BACKUP_DIR = "${WORKSPACE}/keycloak-backup"
    }
    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(
            logRotator(
                numToKeepStr: '30'
            )
        )
    }
    stages {
        stage('Prepare') {
            steps {
                sh '''
                    set -e
                    echo "=============================================="
                    echo "        KEYCLOAK BACKUP"
                    echo "=============================================="
                    echo "Jenkins Job    : ${JOB_NAME}"
                    echo "Build Number   : ${BUILD_NUMBER}"
                    echo "Build Timestamp: ${BUILD_TIMESTAMP}"
                    echo "Backup Host    : ${BACKUP_HOST}"
                    echo "=============================================="
                    rm -rf "${BACKUP_DIR}"
                    mkdir -p "${BACKUP_DIR}"
                    mkdir -p "${BACKUP_DIR}/config"
                    chmod 700 "${BACKUP_DIR}"
                '''
            }
        }
        stage('Test Keycloak Connection') {
            steps {
                sh '''
                    set -e
                    echo "Testing SSH connection to ${BACKUP_HOST}..."
                    ssh \
                      -o BatchMode=yes \
                      -o ConnectTimeout=10 \
                      -o StrictHostKeyChecking=no \
                      "${BACKUP_HOST}" \
                      'hostname'

                    echo "SSH connection successful."
                '''
            }
        }
        stage('Backup PostgreSQL') {
            steps {
                sh '''
                    set -e
                    echo "Creating PostgreSQL backup..."
                    ssh "${BACKUP_HOST}" "
                        sudo -u postgres pg_dump \
                          -Fc \
                          keycloak \
                          > /tmp/keycloak-${BUILD_NUMBER}.dump
                    "
                    echo "Copying PostgreSQL backup to Jenkins..."
                    scp \
                      "${BACKUP_HOST}:/tmp/keycloak-${BUILD_NUMBER}.dump" \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump"
                    echo "Removing temporary PostgreSQL dump..."
                    ssh "${BACKUP_HOST}" \
                      "rm -f /tmp/keycloak-${BUILD_NUMBER}.dump"
                    echo "Compressing PostgreSQL backup..."
                    zstd -T0 \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump" \
                      -o "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst"
                    rm \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump"
                    echo "PostgreSQL backup completed."
                '''
            }
        }
        stage('Backup Keycloak Configuration') {
            steps {
                sh '''
                    set -e
                    echo "Backing up Keycloak configuration..."
                    ssh "${BACKUP_HOST}" "
                        sudo tar \
                          -czf /tmp/keycloak-config-${BUILD_NUMBER}.tar.gz \
                          /etc/keycloak \
                          /etc/systemd/system/keycloak.service
                    "
                    echo "Copying Keycloak configuration backup..."
                    scp \
                      "${BACKUP_HOST}:/tmp/keycloak-config-${BUILD_NUMBER}.tar.gz" \
                      "${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz"
                    echo "Removing temporary configuration archive..."
                    ssh "${BACKUP_HOST}" \
                      "sudo rm -f /tmp/keycloak-config-${BUILD_NUMBER}.tar.gz"
                    echo "Keycloak configuration backup completed."
                '''
            }
        }
        stage('Validate Backups') {
            steps {
                sh '''
                    set -e
                    echo "Validating PostgreSQL backup..."
                    test -s \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst"
                    echo "Validating configuration backup..."
                    test -s \
                      "${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz"
                    echo "Checking PostgreSQL archive..."
                    zstd -t \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst"
                    echo "Checking configuration archive..."
                    tar -tzf \
                      "${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz" \
                      > /dev/null
                    echo "Backup validation successful."
                '''
            }
        }
        stage('Create Manifest') {
            steps {
                sh '''
                    set -e
                    echo "Creating manifest..."
                    cat > "${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt" <<EOF
====================================================
Keycloak Backup Manifest
====================================================
Jenkins Job:
${JOB_NAME}

Build Number:
${BUILD_NUMBER}

Build Timestamp:
${BUILD_TIMESTAMP}

Keycloak Host:
${BACKUP_HOST}

Keycloak Version:
26.7.4
====================================================
Files
====================================================
PostgreSQL:
keycloak-${BUILD_TIMESTAMP}.dump.zst

Configuration:
config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz
====================================================
SHA256
====================================================
EOF

                    sha256sum \
                      "${BACKUP_DIR}/keycloak-${BUILD_TIMESTAMP}.dump.zst" \
                      "${BACKUP_DIR}/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz" \
                      >> "${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt"

                    echo ""
                    echo "Manifest:"
                    cat "${BACKUP_DIR}/manifest-${BUILD_TIMESTAMP}.txt"
                '''
            }
        }
        stage('Upload Database Backup to S3') {
            steps {
                s3Upload(
                    profileName: 'keycloak-s3',
                    entries: [[
                        sourceFile: "keycloak-backup/keycloak-${BUILD_TIMESTAMP}.dump.zst",
                        bucket: "${S3_BUCKET}",
                        path: "keycloak-backup/"
                    ]]
                )
            }
        }
        stage('Upload Configuration to S3') {
            steps {
                s3Upload(
                    profileName: 'keycloak-s3',
                    entries: [[
                        sourceFile: "keycloak-backup/config/keycloak-config-${BUILD_TIMESTAMP}.tar.gz",
                        bucket: "${S3_BUCKET}",
                        path: "keycloak-backup/config/"
                    ]]
                )
            }
        }
        stage('Upload Manifest to S3') {
            steps {
                s3Upload(
                    profileName: 'keycloak-s3',
                    entries: [[
                        sourceFile: "keycloak-backup/manifest-${BUILD_TIMESTAMP}.txt",
                        bucket: "${S3_BUCKET}",
                        path: "keycloak-backup/"
                    ]]
                )
            }
        }
        stage('Prepare GitHub Repository') {
            steps {
                sh '''
                    set -e
                    echo "Cloning GitHub repository..."
                    rm -rf git-backup
                    git clone \
                      --branch "${GIT_BRANCH}" \
                      "${GIT_REPO}" \
                      git-backup
                    cd git-backup
                    mkdir -p \
                      config \
                      systemd \
                      scripts

                    echo "GitHub repository prepared."
                '''
            }
        }
        stage('Update GitHub Configuration') {
            steps {
                sh '''
                    set -e
                    cd git-backup
                    echo "Copying Keycloak systemd service..."
                    scp \
                      "${BACKUP_HOST}:/etc/systemd/system/keycloak.service" \
                      systemd/keycloak.service
                    echo "Creating sanitized keycloak.env.example..."
                    cat > config/keycloak.env.example <<'EOF'
KC_DB=postgres
KC_DB_URL=jdbc:postgresql://127.0.0.1:5432/keycloak
KC_DB_USERNAME=keycloak
KC_DB_PASSWORD=CHANGE_ME
EOF
                    echo "Creating sanitized bootstrap.env.example..."
                    cat > config/bootstrap.env.example <<'EOF'
KC_BOOTSTRAP_ADMIN_USERNAME=admin
KC_BOOTSTRAP_ADMIN_PASSWORD=CHANGE_ME
EOF
                    echo "Creating .gitignore..."
                    cat > .gitignore <<'EOF'
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


                    echo "Checking files that will be committed..."
                    git status --short
                '''
            }
        }
        stage('Commit GitHub Configuration') {
            steps {
                sh '''
                    set -e
                    cd git-backup
                    git add \
                        .gitignore \
                        config/keycloak.env.example \
                        config/bootstrap.env.example \
                        systemd/keycloak.service
                    if git diff --cached --quiet; then
                        echo "No GitHub configuration changes."
                    else
                        git commit -m "Update Keycloak configuration - ${BUILD_TIMESTAMP}"
                        git push origin "${GIT_BRANCH}"
                        echo "GitHub repository updated."
                    fi
                '''
            }
        }
        stage('Backup Summary') {
            steps {
                sh '''
                    echo ""
                    echo "=================================================="
                    echo "        KEYCLOAK BACKUP COMPLETED"
                    echo "=================================================="
                    echo ""
                    echo "Build:"
                    echo "${BUILD_NUMBER}"
                    echo ""
                    echo "Timestamp:"
                    echo "${BUILD_TIMESTAMP}"
                    echo ""
                    echo "PostgreSQL:"
                    echo "s3://${S3_BUCKET}/postgres/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "Configuration:"
                    echo "s3://${S3_BUCKET}/config/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "Manifest:"
                    echo "s3://${S3_BUCKET}/manifests/${BUILD_TIMESTAMP}/"
                    echo ""
                    echo "GitHub:"
                    echo "${GIT_REPO}"
                    echo ""
                    echo "=================================================="
                '''
            }
        }
    }
    post {
        always {
            sh '''
                echo "Cleaning temporary files..."
                rm -rf "${BACKUP_DIR}"
                rm -rf git-backup
            '''
        }
        success {
            echo "Keycloak backup completed successfully."
        }
        failure {
            echo "=================================================="
            echo "KEYCLOAK BACKUP FAILED"
            echo "=================================================="
            echo "Check the Jenkins console output."
        }
    }
}
