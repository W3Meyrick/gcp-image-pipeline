pipeline {
    agent any

    environment {
        images_changed = false
        build_trigger = 'OTHER'
    }

    triggers {
        issueCommentTrigger('^/testpass$')
    }

    stages {
        stage('Determine what caused the build') {
            when {
                changeRequest target: 'master'
            }

            steps {
                script {
                    def triggerIssueComment = currentBuild.rawBuild.getCause(org.jenkinsci.plugins.pipeline.github.trigger.IssueCommentCause)

                    if (triggerIssueComment) {
                        // Determine which issue comment caused the build
                        if (triggerIssueComment.triggerPattern == '^/testpass$') {
                            build_trigger = 'ISSUE_COMMENT_TEST_PASS'
                        } else {
                            build_trigger = 'ISSUE_COMMENT_OTHER'
                        }
                    }
                }
            }
        }

        stage('Determine which images changed') {
            when {
                anyOf {
                    branch 'master'
                    changeRequest target: 'master'
                }
            }

            steps {
                script {
                    def previous_commit = ''

                    if (env.GIT_BRANCH == 'master' && env.GIT_PREVIOUS_SUCCESSFUL_COMMIT != null) {
                        previous_commit = " ${env.GIT_PREVIOUS_SUCCESSFUL_COMMIT}"
                    } else {
                        def previous_commit_id = sh (
                            script: "git merge-base ${env.GIT_BRANCH} refs/remotes/origin/master",
                            returnStdout: true
                        ).trim()

                        previous_commit = " ${previous_commit_id}"
                    }

                    def changed_images_list = sh (
                        script: "for dir in \$(ls -d */); do chdir=\$(git diff-tree --name-only ${env.GIT_COMMIT}${previous_commit} \$dir); if ! [ -z \$chdir ] && [ \"\$chdir\" != \"ansible\" ]; then echo \$chdir; fi; done",
                        returnStdout: true
                    ).trim().split('\n')

                    if (changed_images_list[0].length() > 0) {
                        images_changed = true
                        changed_images = []
                        changed_images_list.each { image ->
                            def image_map = [:]
                            def lockdown = sh (
                                script: "if [ -f \"./${image}/lockdown\" ]; then echo \"TRUE\"; else echo \"FALSE\"; fi;",
                                returnStdout: true
                            ).trim()
                            def base_image = sh (
                                script: "cat ./${image}/base-image",
                                returnStdout: true
                            ).trim().split('/')

                            image_map['name'] = image
                            image_map['project'] = sh (
                                script: "cat ./${image}/project",
                                returnStdout: true
                            ).trim()
                            image_map['image_project'] = base_image[0]
                            image_map['image_family'] = base_image[1]

                            if (lockdown == "TRUE") {
                                image_map['lockdown'] = true
                            } else {
                                image_map['lockdown'] = false
                            }

                            try {
                                image_map['service_account'] = sh (
                                    script: "cat ./${image}/service-account",
                                    returnStdout: true
                                ).trim()
                            } catch(Exception e) {
                                image_map['service_account'] = ""
                            }
                            
                            try {
                                image_map['tags'] = sh (
                                    script: "cat ./${image}/tags",
                                    returnStdout: true
                                ).trim().split('\n')
                            } catch(Exception e) {
                                image_map['tags'] = []
                            }

                            changed_images.add(image_map)
                        }
                    }
                }
            }
        }

        stage('Cleanup prior runs') {
            when {
                equals expected: 'OTHER', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    changed_images.each { image ->
                        try {
                            sh "gcloud --project=${image['project']} compute instances delete ${image['name']}-build --zone=europe-west1-b --quiet"
                        } catch(Exception e) {}

                        try {
                            sh "gcloud --project=${image['project']} compute images delete ${image['name']}-tmp --quiet"
                        } catch(Exception e) {}

                        try {
                            sh "gcloud --project=${image['project']} compute instances delete ${image['name']}-gmitest --zone=europe-west1-b --quiet"
                        } catch(Exception e) {}
                    }
                }
            }
        }

        stage('Create instances from source image') {
            when {
                equals expected: 'OTHER', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image ->
                        def tags = "http-server,https-server"
                        def service_account = ""

                        image['tags'].each { tag ->
                            tags += ",${tag}"
                        }

                        if (image['sa'] != "") {
                            service_account = "--service-account=${image['service_account']} --scopes=cloud-platform"
                        }

                        instances[image['name']] = {
                            sh "gcloud --project=${image['project']} compute instances create ${image['name']}-build --zone=europe-west1-b --image-project=${image['image_project']} --image-family=${image['image_family']} --tags=${tags} ${service_account} --metadata-from-file=ssh-keys=/opt/terraform/littleterra.pub --quiet"
                        }
                    }

                    parallel instances
                    sh 'sleep 30s'
                }
            }
        }

        stage('Apply custom Ansible configuration') {
            when {
                equals expected: 'OTHER', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {          
                script {

                    def whoisrunning = sh (
                        script: "whoami >> workspace/user.txt",
                        returnStdout: true
                    )
                    
                    def instances = [:]

                    changed_images.each { image ->
                        image['private_ip'] = sh (
                            script: "gcloud --project=${image['project']} compute instances describe ${image['name']}-build --zone=europe-west1-b --format=json | jq '.networkInterfaces[0].networkIP' -r",
                            returnStdout: true
                        ).trim()

                        instances[image['name']] = {
                            sh "ansible-playbook -i ${image['private_ip']}, -u jenkinsbuild --private-key=/var/lib/jenkins/ssh/jenkins-ssh --ssh-common-args='-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no' ./${image['name']}/ansible/main.yml"
                        }
                    }

                    parallel instances
                }
            }
        }

        stage('Test stage3 instances') {
            when {
                equals expected: 'OTHER', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instance_table = "| Instance | IP address |\n| --- | --- |"

                    changed_images.each { image ->
                        instance_table += "\n| ${image['name']}-build | ${image['private_ip']} |"
                    }

                    pullRequest.comment("Please test the instances that have been created and confirm that your tests have passed by commenting with **/testpass**. Recreate the images by pusing further commits.\n\n${instance_table}")
                }
            }
        }

        stage('Apply Ansible configuration') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image ->
                        if (image['lockdown']) {
                            instances[image['name']] = {
                                sh "ansible-playbook -i ${image['private_ip']}, -u jenkinsbuild --private-key=/var/lib/jenkins/ssh/jenkins-ssh --ssh-common-args='-o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no' ./ansible/main.yml"
                            }
                        }
                    }

                    parallel instances
                }
            }
        }

        stage('Shutdown instances') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image ->
                        instances[image['name']] = {
                            sh "gcloud --project=${image['project']} compute instances stop ${image['name']}-build --zone=europe-west1-b --quiet"
                        }
                    }

                    parallel instances
                }
            }
        }

        stage('Create configured images') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def images = [:]

                    changed_images.each { image ->
                        images[image['name']] = {
                            sh "gcloud --project=${image['project']} compute images create ${image['name']}-tmp --source-disk=${image['name']}-build --source-disk-zone=europe-west1-b --quiet"
                        }
                    }

                    parallel images                  
                }
            }
        }

        stage('Delete instances') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image -> 
                        instances[image['name']] = {
                            sh "gcloud --project=${image['project']} compute instances delete ${image['name']}-build --zone=europe-west1-b --quiet"
                        }
                    }

                    parallel instances
                }
            }
        }
        
        stage('Create instances from configured image to vulnerability scan') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image ->
                        if (image['lockdown']) {
                            instances[image['name']] = {
                                sh "gcloud --project=${image['project']} compute instances create ${image['name']}-gmitest --zone=europe-west1-b --image=${image['name']}-tmp --tags=http-server,https-server --quiet"
                            }
                        }
                    }

                    parallel instances
                    sh 'sleep 30s'
                }
            }
        }

        stage('Check compliance scan results') {
            when {
                equals expected: 'ISSUE_COMMENT_TEST_PASS', actual: build_trigger
                equals expected: true, actual: images_changed
                changeRequest target: 'master'
            }

            steps {
                script {
                    def instance_table = "| Instance | IP address |\n| --- | --- |"

                    changed_images.each { image ->
                        if (image['lockdown']) {
                            instance_table += "\n| ${image['name']}-gmitest | ${image['private_ip']} |"
                        }
                    }

                    pullRequest.comment("The following images need to be vulnerability scanned. Please check the scan result. When the scans have passed, please approve this PR and merge in to master.\n\n${instance_table}")
                }
            }
        }

        stage('Delete scanned instances') {
            when {
                equals expected: true, actual: images_changed
                branch 'master'
            }

            steps {
                script {
                    def instances = [:]

                    changed_images.each { image ->
                        if (image['lockdown']) {
                            instances[image['name']] = {
                                sh "gcloud --project=${image['project']} compute instances delete ${image['name']}-gmitest --zone=europe-west1-b --quiet"
                            }
                        }
                    }

                    parallel instances
                }
            }
        }

        stage('Assign image family') {
            when {
                equals expected: true, actual: images_changed
                branch 'master'
            }

            steps {
                script {
                    def instances = [:]
                    def now = new Date()

                    changed_images.each { image ->
                        instances[image['name']] = {
                            sh "gcloud --project=${image['project']} compute images create ${image['name']}-${now.format("yyyyMMddHHmmss")} --family ${image['name']} --source-image=${image['name']}-tmp --quiet"
                            sh "gcloud --project=${image['project']} compute images delete ${image['name']}-tmp --quiet"
                        }
                    }

                    parallel instances
                }
            }
        }
    }
}