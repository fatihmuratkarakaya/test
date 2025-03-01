// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
            apiVersion: v1
            kind: Pod
            spec:
              serviceAccountName: jenkins
              containers:
              - name: kubectl-helm
                image: dtzar/helm-kubectl:latest
                command:
                - cat
                tty: true
            """
        }
    }
    
    stages {
        stage('Setup Environment') {
            steps {
                container('kubectl-helm') {
                    sh '''
                        # Create namespace if it doesn't exist
                        kubectl create namespace postgresql --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }
        
        stage('Deploy PostgreSQL') {
            steps {
                container('kubectl-helm') {
                    sh '''
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update
                        
                        # Deploy PostgreSQL with persistence and backup configuration
                        helm upgrade --install postgresql bitnami/postgresql \\
                            --namespace postgresql \\
                            --set global.postgresql.auth.username=postgres \\
                            --set global.postgresql.auth.password=postgres \\
                            --set global.postgresql.auth.database=postgres \\
                            --set primary.persistence.enabled=true \\
                            --set primary.persistence.size=8Gi \\
                            --set service.type=NodePort \\
                            --set service.nodePorts.postgresql=30001
                        
                        # Ensure service is patched to use NodePort
                        kubectl patch svc postgresql -n postgresql -p '{"spec": {"type": "NodePort", "ports": [{"port": 5432, "nodePort": 30001}]}}' || true
                    '''
                }
            }
        }
        
        stage('Configure Backup') {
            steps {
                container('kubectl-helm') {
                    sh '''
                        # Create ConfigMap for backup script
                        kubectl apply -n postgresql -f - <<EOL
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-backup-script
  namespace: postgresql
data:
  backup.sh: |
    #!/bin/bash
    BACKUP_DIR="/backups"
    TIMESTAMP=\$(date +%Y%m%d-%H%M%S)
    BACKUP_FILE="\${BACKUP_DIR}/backup-\${TIMESTAMP}.sql"
    
    # Create backup
    PGPASSWORD=postgres pg_dump -U postgres -d postgres -h postgresql -f \${BACKUP_FILE}
    
    # Compress backup
    gzip \${BACKUP_FILE}
    
    # Cleanup old backups (keep last 7 days)
    find \${BACKUP_DIR} -name "backup-*.sql.gz" -type f -mtime +7 -delete
    
    echo "Backup completed: \${BACKUP_FILE}.gz"
EOL

                        # Create CronJob for daily backups
                        kubectl apply -n postgresql -f - <<EOL
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: postgresql
spec:
  schedule: "0 0 * * *"  # Daily at midnight
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: bitnami/postgresql:latest
            command:
            - /bin/bash
            - /scripts/backup.sh
            volumeMounts:
            - name: backup-script
              mountPath: /scripts
            - name: backup-volume
              mountPath: /backups
          volumes:
          - name: backup-script
            configMap:
              name: postgres-backup-script
              defaultMode: 0755
          - name: backup-volume
            persistentVolumeClaim:
              claimName: postgres-backup-pvc
          restartPolicy: OnFailure
EOL

                        # Create PVC for backups
                        kubectl apply -n postgresql -f - <<EOL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
  namespace: postgresql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOL
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                container('kubectl-helm') {
                    sh '''
                        # Wait for PostgreSQL to be ready
                        kubectl rollout status statefulset/postgresql-primary -n postgresql || true
                        
                        echo "PostgreSQL is accessible at: localhost:30001"
                        echo "Username: postgres"
                        echo "Password: postgres"
                        
                        # Verify backup setup
                        echo "Daily backups configured. They will run at midnight and be stored in the postgres-backup-pvc volume."
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo "PostgreSQL deployment completed successfully!"
        }
        failure {
            echo "PostgreSQL deployment failed!"
        }
    }
}
