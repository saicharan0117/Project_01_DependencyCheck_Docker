<img width="940" height="385" alt="image" src="https://github.com/user-attachments/assets/28010ee4-0ff2-4ce5-8f14-b59d09f523be" />


# Spring Boot based Java web application
 
This is a simple Sprint Boot based Java application that can be built using Maven. Sprint Boot dependencies are handled using the pom.xml 
at the root directory of the repository.

This is a MVC architecture based application where controller returns a page with title and message attributes to the view.


# Take instance type as t2.large and i attched volume 50gb

## After the server was configured initially install jenkins using below commands
      "sudo yum install wget ",
      "sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo",
      "sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key",	    
      "sudo yum upgrade -y",
      "sudo dnf install java-17-amazon-corretto -y",	  
      "sudo yum install jenkins -y",
 	    "sudo systemctl start jenkins",
      "sudo systemctl enable jenkins",


## Then install docker using below commands
      sudo yum update -y
      sudo yum install -y docker
      sudo service docker start
      sudo usermod -aG docker ec2-user
      sudo usermod -a -G docker jenkins



## Plugins to install
   sonar sonar

   docker

   stage view

   OWASP Dependency-Check


## credentails that can be used 
   dockerHub

   soanrqube


## Run maven commands to build and test the code, use below commands
    mvn compile && mvn test


## Use SONARQUBE to do code quality checks

    create sonarqube container from docker image by using below command
      "sudo docker run -itd --name sonar -p 9000:9000 sonarqube"

      
    After the creation of sonarqube container we have to intigrate it with jenkins pipeline
    for that follow below process.

##     Step 1: Install Plugins in Jenkins

        Login to Jenkins → Manage Jenkins → Plugins
        Install:
        SonarQube Scanner for Jenkins
        (Optional for quality gate check) Quality Gates Plugin

##     Step 2: Configure SonarQube in Jenkins
        IN jenkins Go to Manage Jenkins → System Configuration → SonarQube Servers
        Add a new SonarQube server:
        Name: MySonarQube (any identifier)
        Server URL: http://<sonarqube-server-ip>:9000

        here we need credentials,
        for that we go to settings and select Credentials → System → Global credentials (unrestricted)
        Click Add Credentials, then select Secret text → for tokens
        to get the token we have to generate in sonarqube
        Authentication Token:
        Generate from SonarQube → My Account → Security → Generate Token
        Paste it into Jenkins Credentials as "Secret Text"
        Select that credential here.
        Save the configuration.

##      Step 4: Configure SonarQube Scanner

        Go to Manage Jenkins → Tools
        Add SonarQube Scanner installation:
        Name: SonarScanner
        Select “Install automatically” OR provide the path if already installed.
        Save.


##      Then add below code in pipeline to connect to the sonarqube

    stage('sonar') {
    steps {
        echo 'scanning project'
        sh 'ls -ltr'
        sh '''cd java-cicd-project/spring-boot-app && mvn sonar:sonar \\
              -Dsonar.host.url=http://localhost:9000 \\
              -Dsonar.login=squ_d519f9f045bdd07c9fb4866a7614f18155c09308'''
        }
    }


    or


    stage('SonarQube Analysis') {
    steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
            sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
        }
    }
}

         

##  Do the security scan to the dependencies of the code, for that follow below process

    
##        Step1: Install OWASP Dependency-Check using below commands

            sudo yum update -y

            Download Dependency-Check:
            cd /opt
            sudo curl -L -O https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.2/dependency-check-8.4.2-release.zip

            nstall unzip & extract:
            sudo yum install unzip -y
            sudo unzip dependency-check-8.4.2-release.zip
            sudo mv dependency-check /opt/dependency-check

            Add to PATH:
            echo 'export PATH=$PATH:/opt/dependency-check/bin' | sudo tee -a /etc/profile.d/dependency-check.sh
            source /etc/profile.d/dependency-check.sh



    
##        step2: integrated OWASP with jenkins using below process

                Step 1: Install Dependency-Check Plugin in Jenkins
                Go to Jenkins Dashboard → Manage Jenkins → Plugins → Available.
                Search for “OWASP Dependency-Check”.
                Install it and restart Jenkins.


                Step 2: Configure Dependency-Check Tool
                Go to Manage Jenkins → Tools.
                Under Dependency-Check installations, click Add Dependency-Check.
                Give it a name: dependency-check
                Set the installation directory if you installed it manually (/opt/dependency-check).


                step3: Add below code in your pipeline
                stage('Dependency Check') {
                    steps {
                        sh '''
                        /opt/dependency-check/bin/dependency-check.sh \
                        --project "SpringBootApp" \
                        --scan java-cicd-project/spring-boot-app \
                        --format HTML \
                        --data /var/lib/jenkins/odc-data \
                        --out dependency-check-report
                        '''
                    }
                }



## Then build the source code using below command

            mvn clean package'

            
            stage('Building the code') {
            steps {
                sh 'ls -ltr'
                // build the project and create a JAR file
                sh 'cd java-cicd-project/spring-boot-app && mvn clean package'
            }
        }       



## Then use docker to build the image from the artifact

            docker build -t adarshbarkunta/java:${BUILD_NUMBER}

                stage('Build docker image'){
                steps{
                    script{
                        echo 'docker image build'
                        sh 'cd java-cicd-project/spring-boot-app && docker build -t adarshbarkunta/java:${BUILD_NUMBER} .'
                    }
                }
            }





## Install the trivy to scan the image in jenkins server:


            "sudo rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.rpm",


            stage('docker image scan'){
                steps{
                    sh "trivy image adarshbarkunta/java:${BUILD_NUMBER}"
                }
            }




## Push the build image to docker hub

            First intigrate docker hub credentials to jenkins to do that follow below process

            for that we go to settings and select Credentials → System → Global credentials (unrestricted)
            Click Add Credentials, then select Secret text → and add dockerhub password

            Then use below stage in pipeline to push the docker image

            stage('Push image to Hub'){
                steps{
                    script{
                        withCredentials([string(credentialsId: 'dockerhub', variable: 'dockerhub')]) {
                        sh 'docker login -u adarshbarkunta -p ${dockerhub}'

                        }
                        sh 'docker push adarshbarkunta/java:${BUILD_NUMBER}'
                        }
                    }
                }



## Deploy the pushed image to docker container for that use below process


            stage('Deploying image to docker container'){
            steps{
                script{
            
                    sh 'docker run -itd --name java-app -p 8000:8080 adarshbarkunta/java:${BUILD_NUMBER}'
                    }
                }
            }   
























   I 
   




   





   









