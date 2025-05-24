pipeline {
    agent any

    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    stages {
        stage('Clone GitHub Repo') {
            steps {
                git url: 'git@github.com:yoctoalex/bigip-demo-jenkins.git', branch: env.BRANCH_NAME
            }
        }

        stage('Terraform Init & Action') {
            steps {
                withCredentials([
                    string(credentialsId: 'terraform-cloud-token', variable: 'TF_TOKEN_app_terraform_io'),
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN'),
                    string(credentialsId: 'bigip-admin-password', variable: 'TF_VAR_bigip_admin_password')
                ]) {
                    dir('terraform') {
                        script {
                            sh 'terraform init'

                            if (env.CHANGE_ID) {
                                echo "🔍 Pull request: plan only"
                                sh '''
                                    terraform plan -input=false -no-color > ../tfplan.txt
                                    terraform output -json > ../terraform-output.json
                                '''
                            } else {
                                echo "✅ Apply mode"
                                sh '''
                                    terraform apply -auto-approve -input=false -no-color
                                    terraform output -json > ../terraform-output.json
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Generate Ansible Inventory') {
            steps {
                script {
                    def tfOutput = readJSON file: 'terraform-output.json'
                    def mgmtIp = tfOutput.management_public_ip.value
                    def extIp = tfOutput.external_public_ip.value
                    def password = credentials('bigip-admin-password')

                    writeFile file: 'ansible/inventory.ini', text: """
[bigip]
${mgmtIp} ansible_host=${mgmtIp} ansible_user=admin ansible_password=${password}

[external]
${extIp}
"""
                }
            }
        }

        stage('Run Ansible') {
            when {
                allOf {
                    expression { fileExists('terraform-output.json') }
                }
            }
            steps {
                script {
                    sh 'ansible-galaxy install -r ansible/requirements.yml'
                    if (env.CHANGE_ID) {
                        echo 'Dry run: ansible-playbook --check'
                        sh "ansible-playbook -i ansible/inventory.ini ansible/playbook.yml --check"
                    } else {
                        echo 'Running full ansible-playbook'
                        sh "ansible-playbook -i ansible/inventory.ini ansible/playbook.yml"
                    }
                }
            }
        }

        stage('Comment on Pull Request') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def planOutput = readFile('tfplan.txt').take(6000)
                        def comment = """
        ### Terraform Plan Result
        ```hcl
        ${planOutput}
        ```

        _This is an automated dry run result._
        """

                        def payload = groovy.json.JsonOutput.toJson([body: comment])
                        def prNumber = env.CHANGE_ID

                        writeFile file: 'gh_comment.json', text: payload

                        sh """
                            curl -s -H "Authorization: token ${GITHUB_TOKEN}" \\
                                -H "Accept: application/vnd.github.v3+json" \\
                                -X POST \\
                                -d @gh_comment.json \\
                                https://api.github.com/repos/bigip-demo-jenkins/issues/${prNumber}/comments
                        """
                    }
                }
            }
        }

    }

    post {
        always {
            echo "Pipeline completed on branch: ${env.BRANCH_NAME}"
        }
    }
}
