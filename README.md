# jenkins_bcp_code
We will need a plugin for this named Pipeline(Aws: steps) 
and IAM user privilages will need same for GCP as well like service account

cd /var/lib/
aws s3 cp s3://jenkins-backup-0/backup/jenkinsBackup.tar.gz ./
ls
tar -xzvf jenkinsBackup.tar.gz -C ./
systemctl restart jenkins
