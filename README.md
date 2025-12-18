<h1 align="center">🚀 PayMyBuddy – CI/CD with Jenkins, Docker & Deployment Pipeline</h1>

<p align="center">
  <strong>Automated build, test, Docker image creation, and multi-environment deployment (Staging & Production)</strong>
</p>

<hr/>

<h2>📌 Project Overview</h2>

<p>
PayMyBuddy is a Spring Boot application packaged with Docker and deployed automatically through a complete Jenkins pipeline.
This project demonstrates:
</p>

<ul>
  <li>✔ Continuous Integration (unit tests, build, validation)</li>
  <li>✔ Docker image creation for both <strong>backend app</strong> and <strong>MySQL DB</strong></li>
  <li>✔ Automatic pushes to Docker Hub</li>
  <li>✔ Deployment to remote VM servers via SSH</li>
  <li>✔ Health checks </li>
  <li>✔ Conditional deployments:
    <ul>
      <li>Staging → all branches except <code>main</code></li>
      <li>Production → <code>main</code> only</li>
    </ul>
  </li>
  <li>✔ Slack notifications for success or failure</li>
</ul>

<hr/>

<h2>📦 Project Architecture</h2>

<pre>
├── Dockerfile                # Backend application image
├── Dockerfile-db             # Custom MySQL image
├── Jenkinsfile               # CI/CD Pipeline
├── src/                      # Spring Boot project
└── src/main/ressources/database/create.sql         # DB schema automatically injected at container startup
</pre>

<hr/>

<h2>🐳 Docker Build Instructions</h2>

<h3>Backend Image</h3>
<pre>
docker build -t kingsley95/paymybuddy:latest .
</pre>

<h3>Database Image</h3>
<pre>
docker build -f Dockerfile-db -t kingsley95/paymybuddy-db:latest .
</pre>

<hr/>

<h2>🚀 Deployment Pipeline Summary</h2>

<h3>1️⃣ CI Steps</h3>
<ul>
  <li>Run Maven tests</li>
  <li>Build JAR</li>
  <li>Build Docker images</li>
  <li>Local test with temporary containers</li>
</ul>

<h3>2️⃣ Push to DockerHub</h3>
<pre>
docker push kingsley95/paymybuddy:latest
docker push kingsley95/paymybuddy-db:latest
</pre>

<h3>3️⃣ Deployment Logic</h3>

<h4>🌍 Staging (all branches except main)</h4>
<ul>
  <li>Pull images on remote server</li>
  <li>Restart containers</li>
  <li>Wait for health endpoint</li>
</ul>

<h4>🔥 Production (main only)</h4>
<ul>
  <li>Deploy only after staging success</li>
  <li>Same process on PROD server</li>
</ul>

<hr/>

<h2>⚙ Example Deployment Command (remote server)</h2>

<pre>
docker stop paymybuddy || true
docker rm paymybuddy || true

docker pull kingsley95/paymybuddy:latest
docker pull kingsley95/paymybuddy-db:latest

docker run -d --name mysql -p 3306:3306 --network paymybuddy-net \
 -e MYSQL_ROOT_PASSWORD=password \
 -e MYSQL_DATABASE=db_paymybuddy \
 -e MYSQL_USER=test \
 -e MYSQL_PASSWORD=password \
 kingsley95/paymybuddy-db:latest

docker run -d --name paymybuddy -p 8080:8080 --link mysql kingsley95/paymybuddy:latest
</pre>

<hr/>

<h2>✨ Jenkinsfile Features</h2>

<ul>
  <li>Conditionnal stages</li>
  <li>SSH deployments with credentials</li>
  <li>Slack notifications</li>
  <li>Artifact storage</li>
  <li>Startup health validation</li>
</ul>

<hr/>

<h2>📬 Notifications</h2>

<p>Slack alerts are sent on each pipeline result:</p>

<ul>
  <li>🟢 Success message with deployment URLs</li>
  <li>🔴 Failure message with error details</li>
  
</ul>

**![PayMyBuddy Overview](https://github.com/kingsleyDeve/PayMyBuddy-Jenkins/blob/main/img/jenkins.png)**
<hr/>


<h2>✅ Deployment Validation</h2>

<p>Each deployment is automatically validated to ensure the application is running correctly.</p>

<ul>
  <li>🔍 The application health is checked using an HTTP request after deployment</li>
  <li>⏳ The pipeline waits until the service is fully available before continuing</li>
  <li>🚫 If the application does not respond, the pipeline fails automatically</li>
</ul>

<p>This validation step guarantees that only successful and healthy deployments reach the next environment.</p>

<p><strong>Example:</strong> the pipeline checks the <code>/login</code> or <code>/actuator/health</code> endpoint before confirming deployment success.</p>

<strong>pipeline results</strong>

  <img src="https://github.com/kingsleyDeve/PayMyBuddy-Jenkins/blob/main/img/jenkins-pipeline.PNG" alt="PayMyBuddy Jenkins Pipeline Overview">

  <strong>AWS EC2</strong>

  <img src="https://github.com/kingsleyDeve/PayMyBuddy-Jenkins/blob/main/img/jenkins_aws.PNG" alt="PayMyBuddy Jenkins Pipeline Overview">

  
  <strong>PayMyBuddy-app</strong>

  <img src="https://github.com/kingsleyDeve/PayMyBuddy-Jenkins/blob/main/img/paymybuddy.PNG" alt="PayMyBuddy Jenkins Pipeline Overview">
<hr/>




<h2>👤 Author</h2>

<p>
<b>Kingsley Williams</b><br/>
DevOps & Cloud Engineer<br/>
Docker • Jenkins • Kubernetes • Terraform • Linux • Ansible 
</p>

<hr/>

