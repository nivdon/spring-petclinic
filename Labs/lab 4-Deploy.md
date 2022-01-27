# **DevOps Practitioner Lab 4**
 ## ***Setting up your CD Pipeline using GitHub Actions***
In this lab you will set up the upload and deploy phases for your CICD pipeline. For our demo we will be using S3 as an Artifacts store and Amazon EC2 as deployment server. 

Please Note that this lab is dependent on Lab 1 and Lab 2 and is to be built in continuation. 

Please check - 
* That you are on the feature/Student<your number> branch. 
* That you have created a .github/workflows/build.yml file.
* That you have a successful GitHub Actions workflow run from lab 4.
## **Prepare Application for Deployment**
### **Change the port number for your application.** 
* Open *application.properties file(at devopsfundamentals/src/main/resources)* files as shown below.

    ![](static/lab4-1.png)
Change the last digit of ```server.port:9090``` to match with your team number. </P>
i.e. 
  *team1* -  `server.port:9091`</p>
  *team2* - `server.port:9092` </p>
  *team1* -  `server.port:9093`</p>
  *team2* - `server.port:9094` and so on

<br>
* Open Stop.sh at the folder root (*devopsfundamentals*)
Change the port number to the same as in the previous file and to match with your team number</p>
`sudo kill -9 $(sudo lsof -t -i:9090)`</br>
change to </p>
i.e. 
  *team1* -  `sudo kill -9 $(sudo lsof -t -i:9091)` </p>
  *team2* - `sudo kill -9 $(sudo lsof -t -i:9092)` </p>
  *team3* -  `sudo kill -9 $(sudo lsof -t -i:9091)` </p>
  *team4* - `sudo kill -9 $(sudo lsof -t -i:9092)` and so on

    ![](static/lab4-2.png)

## **Add Upload Phase**
1. You will now add the upload phase to your pipeline. This phase is optional and can be used to upload your artifacts to the artifacts repository. 
2. In our example we are going to use Amazon S3 to store artifacts. 
3. You will add Following yml code in your .github/workflows/build.yml file
```
upload:
    runs-on: ubuntu-latest
    needs: [analyze]
    env:
      Region: us-east-1
      S3Bucket: cnbdevopsfundamentals
      S3Folder: githubactions<your team number> #change the number to match with your team number
    permissions:
      id-token: write
      contents: read
      
    steps:
      - name: Download an artifact
        uses: actions/download-artifact@v2
        with:
          name: jar-file

      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::286144240398:role/AssumeRoleForGithubUsers
          aws-region: ${{ env.Region }}
          
      - name: deploy to S3
        run: |
          ls
          aws s3 cp *.zip s3://${{ env.S3Bucket }}/${{ env.S3Folder }}/
```
**Rename the S3Folder section to match with your number. Search for the line** </p>     
> S3Folder: githubactions0 and update it with your team folder.  
S3Folder: githubactions`<your team number>`

Please see below the places marked in red where you need to change `S3Folder` value
```
name: Java CI with Maven

on:
  push:
    branches: [ main, feature/* ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
          cache: maven
  
      - name: Build with Maven Wrapper
        run: ./mvnw clean -B package
        
      - name: Create Artifacts
        run: |
          sudo apt-get install zip
          zip deploy_artifacts.zip target/*.jar appspec.yml run.sh setup.sh stop.sh
          
      - name: Upload Artifacts
        uses: actions/upload-artifact@v2
        with:
          name: jar-file
          path: deploy_artifacts.zip
        
  test:
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0       

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
          cache: maven

  analyze:
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - uses: actions/checkout@v2
      
      - name: Cache SonarCloud packages
        uses: actions/cache@v1
        with:
        
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Cache Maven packages
        uses: actions/cache@v1
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build and analyze
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=conceptsandbeyond_devopsfundamentals
      
  
  upload:
    runs-on: ubuntu-latest
    needs: [analyze]
    env:
      Region: us-east-1
      S3Bucket: cnbdevopsfundamentals
      S3Folder: githubactions<your team number> #change the number to match with your team number
    permissions:
      id-token: write
      contents: read
      
    steps:
      - name: Download an artifact
        uses: actions/download-artifact@v2
        with:
          name: jar-file

      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::286144240398:role/AssumeRoleForGithubUsers
          aws-region: ${{ env.Region }}
          
      - name: deploy to S3
        run: |
          ls
          aws s3 cp *.zip s3://${{ env.S3Bucket }}/${{ env.S3Folder }}/
```
4. Push the code
```
git add .
git commit -m "adding upload action"
git push 
```
*username- enter Student `<your number>` </p>
password enter the Personal access token provided to you.*

<br>

5. Check github actions workflow to see the Upload phase added

    ![](static/lab4-3.png)


## **Add Deploy Phase**
1. AWS provides virtual servers to host applications.
2. In this phase you can deploy the application to any destination of your choice
 In this example we are going to use AWS CodeDeploy to handle our deployment to Amazon EC2 servers. Open your .github/workflows/build.yml file.
3. You will add the following yml code in your .github/workflows/build.ymlfile.


```
deploy:
    runs-on: ubuntu-latest
    needs: [upload]
    env:
      Region: us-east-1
      S3Bucket: cnbdevopsfundamentals
      S3Folder: githubactions<your team number> #change the number to match with your team number
    permissions:
      id-token: write
      contents: read
      
    steps:
      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::286144240398:role/AssumeRoleForGithubUsers
          aws-region: ${{ env.Region }}
          
          
      - name: CodeDeploy to EC2
        id: deploy
        run: |
          aws deploy create-deployment \
            --application-name CnBDevOpsApp \
            --deployment-group-name CnBDevOpsDG \
            --deployment-config-name CodeDeployDefault.AllAtOnce \
            --s3-location bucket=${{ env.S3Bucket }},bundleType=zip,key=${{ env.S3Folder }}/deploy_artifacts.zip
```

**Rename the S3Folder section to match with your number. Search for the line**      
`S3Folder: githubactions0` **and update it with your team number.**
`S3Folder: githubactions<your team number>` </p>

``` hl_lines="2 3 5"
name: Java CI with Maven

on:
  push:
    branches: [ main, feature/* ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
          cache: maven
  
      - name: Build with Maven Wrapper
        run: ./mvnw clean -B package
        
      - name: Create Artifacts
        run: |
          sudo apt-get install zip
          zip deploy_artifacts.zip target/*.jar appspec.yml run.sh setup.sh stop.sh
          
      - name: Upload Artifacts
        uses: actions/upload-artifact@v2
        with:
          name: jar-file
          path: deploy_artifacts.zip
        
  test:
    runs-on: ubuntu-latest
    needs: [build]
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0       

      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
          cache: maven

  analyze:
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - uses: actions/checkout@v2
      
      - name: Cache SonarCloud packages
        uses: actions/cache@v1
        with:
        
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Cache Maven packages
        uses: actions/cache@v1
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build and analyze
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Needed to get PR information, if any
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=conceptsandbeyond_devopsfundamentals
      
  
  upload:
    runs-on: ubuntu-latest
    needs: [analyze]
    env:
      Region: us-east-1
      S3Bucket: cnbdevopsfundamentals
      S3Folder: githubactions<your team number> #change the number to match with your team number
    permissions:
      id-token: write
      contents: read
      
    steps:
      - name: Download an artifact
        uses: actions/download-artifact@v2
        with:
          name: jar-file

      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::286144240398:role/AssumeRoleForGithubUsers
          aws-region: ${{ env.Region }}
          
      - name: deploy to S3
        run: |
          ls
          aws s3 cp *.zip s3://${{ env.S3Bucket }}/${{ env.S3Folder }}/
          
  deploy:
    runs-on: ubuntu-latest
    needs: [upload]
    env:
      Region: us-east-1
      S3Bucket: cnbdevopsfundamentals
      S3Folder: githubactions<your team number> #change the number to match with your team number
    permissions:
      id-token: write
      contents: read
      
    steps:
      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::286144240398:role/AssumeRoleForGithubUsers
          aws-region: ${{ env.Region }}
          
          
      - name: CodeDeploy to EC2
        id: deploy
        run: |
          aws deploy create-deployment \
            --application-name CnBDevOpsApp \
            --deployment-group-name CnBDevOpsDG \
            --deployment-config-name CodeDeployDefault.AllAtOnce \
            --s3-location bucket=${{ env.S3Bucket }},bundleType=zip,key=${{ env.S3Folder }}/deploy_artifacts.zip
```
Push the code
```
git add .
git commit -m "adding Deploy action"
git push 
```
*username- enter Student`<your number>`
password enter the Personal access token provided to you.*
5. Check github actions workflow to see the Deploy phase added

   ![](static/lab4-4.png)

**Congratulations, your application is now deployed. you can view it at below url. Change the port number to match with your port number.**

http://54.173.148.82:9090/

  ![](static/lab4-5.png)
