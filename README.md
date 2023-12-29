# Hanabi-cicd

### Architecture Diagram 

  <div align="center">
  <p>
    <img src="https://lh7-us.googleusercontent.com/P5vpgCLKNNOoB7GLhGXN7boxRuXs9hTMnRNrjhaLBcmDWUaUl6uH9RICdkvKlQqB-QY80ByCB3AWYPX9oIeaiS6VyFj4UMdJl73QUeMOHuUQbKwMkTOLWJGsD2OKJQW2QGV9uFbwOKMHEPK2qzi-wCk" alt="Your Image">
  </p>
</div>


1. Basic directory structure
     
        |----.github/workflows
        |----Dockerfile
        |----app.py
        |----requirements.txt

3. requirements.txt  consists of

   ![](https://lh5.googleusercontent.com/OORtoO5gw-9Ytl6ST2h4qnRC9MGaldONxdtCvH9O5iL11HPc4PXWIJDqZdU8rXpmNXowtEWYpQMNdf5zAUelHySFT-9vqzcP12uD4QnQkRvSK6I-H3GcQV2XGCa_kxERd3yswlwj5Fe8B84LCSk_iZHVYz8mUBjWNRMRoL9oRrhJ3ROnd_5CpTR2J6qvkQ)

3. Make the flask application run locally using python app.py. Tested the root  from the browser and got the expected response

  ![](https://lh7-us.googleusercontent.com/Gq4h4HG6uONROLVWJmisDtlD4lBqmgLF9CEVpQ_WHB5N-Mfd0Ir1TFqn3nmjqsogCmHVY8CmnYs0P--hQUghhH53aBUObyV05eU1YlXB7dk-Vi87D1xeI9fJ78vmDBItb6JwecXibWPbcmBNeGjGd6M)

## Dockerization: Dockerize the Python application
4. Dockerfile is used to create a docker container and consists of 
```
FROM python:3.13-rc-slim

WORKDIR /python-docker

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

EXPOSE 8081

CMD [ "python3","app.py"]
```
Used python 3.8 version and chose rc-slim in order to reduce the size of the docker images. This can help us deploy the docker image faster. Created a working directory as ‘/python-docker’. Copied only requirements.txt because docker images are built layer by layer. Whenever the bottom layer gets changed, all the layers above them are rebuilt. So, application code changes often however, dependencies won't change much. Moreover, building the dependencies takes more time during the building process. Downloading and Installation of all the required dependencies is done using the command pip3 install. The source code is copied to the container and “CMD \[ "python3","app.py"]”. "python3" This is the executable or interpreter for the Python programming language. It tells Docker to use Python 3 to execute the script and "app.py" will be executed when the container starts. It is assumed to be in the current working directory or specified path within the container.

## GIT-HUB ACTION WORKFLOW
5. deploy.yml is used to execute CICD pipeline stages
   ```
   sast_scan:
     name: Run Bandit Scan
     runs-on: ubuntu-latest
  
     steps:
     - name: Checkout code
       uses: actions/checkout@v2
  
     - name: Set up Python
       uses: actions/setup-python@v2
       with:
         python-version: 3.8
  
     - name: Install Bandit
       run: pip install bandit
  
     - name: Run Bandit Scan
       run: bandit -ll -ii -r . -f json -o bandit-report.json -vv
  
     - name: Upload Artifact
       uses: actions/upload-artifact@v3
       if: always()
       with:
        name: bandit-findings
        path: bandit-report.json
   ```
   This GitHub Actions workflow checks Python code for security issues using Bandit, saving the results as an artifact. It helps ensure the project's security during development.

   ```
    image_scan:
     name: Build Image and Run Image Scan
     runs-on: ubuntu-latest
  
     steps:
     - name: Checkout code
       uses: actions/checkout@v2
  
     - name: Set up Docker
       uses: docker-practice/actions-setup-docker@v1
       with:
        docker_version: '20.10.7'
  
     - name: Build Docker Image
       run: docker build -f Dockerfile -t hameedakshal/hanabi-py:$GITHUB_RUN_NUMBER .
  
     - name: Docker Scout Scan
       run: |
         curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
         script -q -e -c "bash install-scout.sh" /dev/null
  
         echo ${{secrets.REPO_PASSWORD}} | docker login -u ${{secrets.REPO_USER}} --password-stdin
  
         docker scout quickview
          
         docker scout cves 2> error_list.txt
  
      - name: Upload Error List Artifact
        uses: actions/upload-artifact@v3
        with:
          name: error-list
          path: error_list.txt

   ```
   The above workflow performs a security scan using Docker Scout. Scans vulnerabilities of dockerfile layer by layer, saving the results as an artifact. It helps ensure the project's security during development.

   ```
    push:
      name: Build Image and Push Image 
      runs-on: ubuntu-latest
      needs: image_scan
    
      steps:
       - name: Checkout code
         uses: actions/checkout@v2
    
       - name: Set up Docker
         uses: docker-practice/actions-setup-docker@v1
         with:
          docker_version: '20.10.7'
    
       - name: Build and Push Docker Image with build tag
         run: |
          docker build -f Dockerfile -t hameedakshal/hanabi-py:$GITHUB_RUN_NUMBER .
          echo ${{secrets.REPO_PASSWORD}} | docker login -u ${{secrets.REPO_USER}} --password-stdin
          docker push  hameedakshal/hanabi-py:$GITHUB_RUN_NUMBER
         
       - name: Tag and Push Docker Image with latest tag
         run: |
          echo ${{secrets.REPO_PASSWORD}} | docker login -u ${{secrets.REPO_USER}} --password-stdin
          docker tag  hameedakshal/hanabi-py:$GITHUB_RUN_NUMBER hameedakshal/hanabi-py:latest
          docker push hameedakshal/hanabi-py:latest

   ```
   The above GitHub workflow uses Docker to package and share code. It builds and pushes two versions – one with a specific tag and another with the latest tag

   ```
   deploy:
    name: deploy to ebs
    runs-on: ubuntu-latest
    needs: push
  
    env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ secrets.AWS_DEFAULT_REGION }}
    steps:
        - uses: actions/checkout@v2
        - name: Install Python 3.9
          uses: actions/setup-python@v2
          with:
            python-version: 3.9
        - name: Install AWS CLI and Elastic Beanstalk CLI
          run: |
            sudo apt-get update
            sudo apt-get install -y awscli
            sudo apt-get install -y python3-pip
            python -m pip install --upgrade pip
            pip install awsebcli
        - name: Deploy to Elastic Beanstalk
          run: |
            eb init -r us-east-1 -p docker Hanabi-demo
            eb deploy Hanabi-demo-env 
   ```
   The above GitHub workflow deploys code to Amazon Elastic Beanstalk (EBS) once a push stage is successful. It configures AWS access, installs necessary tools, and initiates the deployment to Elastic Beanstalk in the US East region.

## Make sure to add the variables In Settings -> Secrets and Variables -> Actions
  ![](https://lh7-us.googleusercontent.com/ZHNIKjipWya7GVyBfKtiNxI3VgnBBQdZyYIDmC2DNl16Ip8FnAY3Vn6nywofCSIWGh2-C4z-UKVZ-uDssMfqYxaHdYQ-YydDwmlA2HKdMGJZB7IhX6t5wgNA9TstNsyREv-AO_GpZQ01cARSNSmasSY)
## Final Output
  ![](https://lh7-us.googleusercontent.com/CkcHcUg7S2qmkmKdVUTEwkbJGqNs8FalqvSnwVu9lgTrmEKcRRIGUc5ZpyHMBMWU_cdcxQw-v62WN70uIhE6pr1r3f_8isBORz2hGEowMPsGC6bevnhmce3xA_xuHcIIwwSnqFYKt0TcHGAuyc5HVGY)

