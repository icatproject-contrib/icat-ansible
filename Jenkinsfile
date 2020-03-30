#!groovy
pipeline {
    agent { label 'sl7'}
    environment {
        ANSIBLE_VAULT_PASSWORD_FILE = credentials('ansible-vault-password-file')
    }
    stages {
        stage('Preparation') {
            steps {
                checkout scm
                sh 'pip install -r requirements.txt'
                sh 'echo -e "[hosts-all]\nlocalhost ansible_connection=local" > hosts'
                sh 'mv ./vault.yml ./group_vars/all'
                sh 'sed -i -e "s/^payara_user: ''glassfish''/payara_user: ''jenkins''/" ./group_vars/all/vars.yml'
            }
        }
        stage('Test') {
            steps {
                sh 'ansible-playbook --inventory ./hosts ./hosts-all.yml'
            }
        }
    }
}
