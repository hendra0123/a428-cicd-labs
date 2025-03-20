# Practice Building a CI Pipeline with Jenkins

This exercise will teach you how to use Jenkins to build a Continuous Integration (CI) Pipeline with a React App. The application uses React and Node.js to display a web page with the content "Welcome to React" and includes tests to check the rendering process.

## Prerequisites

Before starting, make sure you have installed the following tools:

1. **Docker** - To run Jenkins and the React application in containers.
2. **Git** - To manage the code repository.
3. **Visual Studio Code** (or any other text editor) - To edit the code.
4. **Node.js** - To run the React application (optional, as we will use Docker).

You can use Windows, Linux, or macOS. However, in this guide, we will focus on Windows commands.

---

## Step 1: Running Jenkins in Docker

### 1.1 Creating a Docker Network

First, we need to create a Docker network to connect the Jenkins and Docker-in-Docker (dind) containers.

1. Open **Command Prompt** or **PowerShell**.
2. Run the following command to create a Docker network:

   ```bash
   docker network create jenkins
   ```

### 1.2 Running Docker-in-Docker (dind)

We will run Docker-in-Docker (dind) to allow Jenkins to use Docker from within the container.

1. Run the following command to start Docker-in-Docker:

   ```bash
   docker run ^
   --name jenkins-docker ^
   --detach ^
   --privileged ^
   --network jenkins ^
   --network-alias docker ^
   --env DOCKER_TLS_CERTDIR=/certs ^
   --volume jenkins-docker-certs:/certs/client ^
   --volume jenkins-data:/var/jenkins_home ^
   --publish 2376:2376 ^
   --publish 3000:3000 ^
   --restart always ^
   docker:dind ^
   --storage-driver overlay2
   ```

   **Explanation:**
   - `--name jenkins-docker`: Name of the container.
   - `--detach`: Runs the container in the background.
   - `--privileged`: Grants privileged access to run Docker inside Docker.
   - `--network jenkins`: Connects the container to the `jenkins` network.
   - `--volume`: Creates volumes to store Jenkins data and Docker certificates.
   - `--publish`: Exposes ports 2376 and 3000 to the host.

### 1.3 Running Jenkins with Blue Ocean

Blue Ocean is a modern user interface for Jenkins. We will run Jenkins with Blue Ocean using a Dockerfile.

1. Create a `Dockerfile` in your desired directory. You can use **Notepad** or **Visual Studio Code**.
2. Copy the following content into the `Dockerfile`:

   ```dockerfile
   FROM jenkins/jenkins:2.346.1-jdk11
   USER root
   RUN apt-get update && apt-get install -y lsb-release
   RUN curl -fsSlo /usr/share/keyrings/docker-archive-keyring.asc \
   https://download.docker.com/linux/debian/gpg
   RUN echo "deb [arch=$(dpkg --print-architecture) \
   signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
   https://download.docker.com/linux/debian \
   $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
   RUN apt-get update && apt-get install -y docker-ce-cli
   USER jenkins
   RUN jenkins-plugin-cli --plugins "blueocean:1.25.5 docker-workflow:1.28"
   ```

3. Save the file as `Dockerfile`.

4. Run the following command to build the Jenkins image with Blue Ocean:

   ```bash
   docker build -t myjenkins-blueocean:2.346.1-1 .
   ```

5. After the image is built, run the Jenkins container with the following command:

   ```bash
   docker run ^
   --name jenkins-blueocean ^
   --detach ^
   --network jenkins ^
   --env DOCKER_HOST=tcp://docker:2376 ^
   --env DOCKER_CERT_PATH=/certs/client ^
   --env DOCKER_TLS_VERIFY=1 ^
   --publish 8080:8080 ^
   --publish 50000:50000 ^
   --volume jenkins-data:/var/jenkins_home ^
   --volume jenkins-docker-certs:/certs/client:ro ^
   --volume "%USERPROFILE%":/home ^
   --restart=on-failure ^
   --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" ^
   myjenkins-blueocean:2.346.1-1
   ```

   **Note:** `%USERPROFILE%` is a Windows environment variable that refers to the user's home directory (e.g., `C:\Users\YourUsername`).

---

## Step 2: Setting Up Jenkins Wizard

1. Open your browser and go to `http://localhost:8080`.
2. You will be prompted to enter the initial Jenkins password. To retrieve the password, run the following command in Command Prompt:

   ```bash
   docker logs jenkins-blueocean
   ```

   Copy the password that appears between the `****` symbols.

3. Enter the password on the Jenkins page and click **Continue**.
4. Select **Install suggested plugins** to install the recommended plugins.
5. After the installation is complete, create an administrator account by filling in the username, password, and email.
6. Click **Save and Finish** to complete the setup.

---

## Step 3: Forking and Cloning the React App Repository

1. Log in to GitHub and fork the [React App](https://github.com/dicodingacademy/a428-cicd-labs) repository to your GitHub account.
2. Clone the forked repository to your computer using the following command:

   ```bash
   git clone -b react-app https://github.com/YOUR-GITHUB-USERNAME/a428-cicd-labs.git
   ```

   Replace `YOUR-GITHUB-USERNAME` with your GitHub username.

3. Open the `a428-cicd-labs` folder in Visual Studio Code.

---

## Step 4: Creating a Pipeline Project in Jenkins

1. Open Jenkins at `http://localhost:8080`.
2. Click **New Item** and name the project, for example, `react-app`.
3. Select **Pipeline** and click **OK**.
4. In the **Pipeline** section, select **Pipeline script from SCM**.
5. Select **Git** and enter the path to your local repository, for example:

   ```bash
   /home/a428-cicd-labs
   ```

6. In the **Branch Specifier** field, enter `*/react-app`.
7. Click **Save** to save the project.

---

## Step 5: Creating the Jenkinsfile

1. In Visual Studio Code, create a new file named `Jenkinsfile` in the root folder of `a428-cicd-labs`.
2. Copy the following code into the `Jenkinsfile`:

   ```groovy
   pipeline {
       agent {
           docker {
               image 'node:16-buster-slim'
               args '-p 3000:3000'
           }
       }
       stages {
           stage('Build') {
               steps {
                   sh 'npm install'
               }
           }
           stage('Test') {
               steps {
                   sh './jenkins/scripts/test.sh'
               }
           }
       }
   }
   ```

3. Save the file.
4. Commit the changes to your local repository with the following commands:

   ```bash
   git add .
   git commit -m "Add Jenkinsfile"
   ```

---

## Step 6: Running the Pipeline

1. Open Jenkins and click **Open Blue Ocean**.
2. Select the `react-app` project and click **Run**.
3. Jenkins will run the **Build** and **Test** stages. If successful, you will see a green status in the Blue Ocean interface.

---

## Step 7: Stopping Docker Containers

After completing the exercise, you can stop the Jenkins containers with the following command:

```bash
docker stop jenkins-blueocean jenkins-docker
```

To restart the containers, run the same commands as in Step 1.

---

By following the steps above, you have successfully created a CI Pipeline with Jenkins using Docker and a React App. Congratulations!

---
