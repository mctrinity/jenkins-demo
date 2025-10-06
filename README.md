# 🧰 Jenkins + GitHub Setup Guide (Windows + Docker)

This guide walks through setting up **Jenkins in Docker** on a Windows machine and integrating it with **GitHub** for CI/CD pipelines.

---

## ⚙️ 1. Install and Run Jenkins on Docker

### 🧩 Prerequisites

* **Docker Desktop** installed and running on Windows
* **PowerShell 7+**

### 🐳 Run Jenkins in Docker

```powershell
docker run -d `
  --name jenkins `
  -p 8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  jenkins/jenkins:lts
```

Then open Jenkins at:

```
http://localhost:8080
```

Retrieve the initial admin password:

```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Log in with that password and install **suggested plugins**.

---

## 🔑 2. Install Required Plugins

Go to **Manage Jenkins → Plugins → Available** and ensure the following are installed and enabled:

| Plugin                            | Purpose                              |
| --------------------------------- | ------------------------------------ |
| Git plugin                        | Core Git integration                 |
| Git client plugin                 | Provides Git binary support          |
| GitHub plugin                     | Connects Jenkins to GitHub           |
| GitHub API plugin                 | Enables GitHub API access            |
| GitHub Branch Source plugin       | Multibranch / organization pipelines |
| Pipeline: GitHub Groovy Libraries | Allows shared library loading        |

---

## 🔐 3. Create GitHub Personal Access Token (PAT)

1. Go to [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Generate a new **Fine-grained PAT** (or classic token) with scopes:

   * `repo`
   * `admin:repo_hook`
   * (optional) `read:org`

Copy the token and keep it safe.

---

## 🧭 4. Add GitHub Credentials to Jenkins

1. Go to **Manage Jenkins → Credentials → System → Global credentials (unrestricted)**
2. Click **Add Credentials**
3. Fill in:

| Field       | Value                        |
| ----------- | ---------------------------- |
| Kind        | Username with password       |
| Username    | your GitHub username         |
| Password    | your GitHub PAT              |
| ID          | github-https                 |
| Description | GitHub HTTPS PAT for Jenkins |

Click **Create**.

---

## 🔗 5. Configure GitHub Server in Jenkins

1. **Manage Jenkins → System → GitHub section**
2. Click **Add GitHub Server**
3. Fill in:

   * **Name:** `GitHub`
   * **API URL:** `https://api.github.com`
   * **Credentials:** `github-https`
4. Click **Test Connection** → ✅ Should say *Credentials verified for user <username>*
5. **Save**

---

## 🧱 6. Create a GitHub Repository

1. Go to [https://github.com/new](https://github.com/new)
2. Create a repo named `jenkins-demo`
3. Check **Add a README**
4. Clone locally:

   ```powershell
   git clone https://github.com/<your-username>/jenkins-demo.git
   cd jenkins-demo
   ```

---

## 🧩 7. Add a Jenkinsfile

In the repo root, create a file named `Jenkinsfile`:

```groovy
pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo 'Hello, Jenkins + GitHub!'
            }
        }
    }
}
```

Push to GitHub:

```powershell
git add Jenkinsfile
git commit -m "Add initial Jenkinsfile"
git push
```

---

## 🚀 8. Create a Pipeline Job in Jenkins

1. In Jenkins Dashboard → **New Item** → name it `jenkins-demo-pipeline`
2. Choose **Pipeline** → OK
3. Under **Pipeline → Definition**, select **Pipeline script from SCM**
4. Configure:

   * **SCM:** Git
   * **Repository URL:** `https://github.com/<your-username>/jenkins-demo.git`
   * **Credentials:** `github-https`
   * **Branch Specifier:** `main`
5. Click **Save → Build Now**

✅ You should see:

```
Hello, Jenkins + GitHub!
Finished: SUCCESS
```

---

## 🔁 9. (Optional) Enable GitHub Webhook

To trigger builds automatically on pushes:

1. In GitHub repo → **Settings → Webhooks → Add webhook**
2. Payload URL: `http://localhost:8080/github-webhook/`
3. Content type: `application/json`
4. Select: *Just the push event*
5. Click **Add webhook**

Now Jenkins builds whenever you push to GitHub 🎉

---

## 🧱 10. (Optional) Allow Jenkins to Build Docker Images

Restart Jenkins container with Docker socket mounted:

```powershell
docker run -d `
  --name jenkins `
  -p 8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  -v /var/run/docker.sock:/var/run/docker.sock `
  jenkins/jenkins:lts
```

Then install the **Docker Pipeline** plugin.

---

## 🧰 Troubleshooting

### 🔹 Missing Credentials in Pipeline Job

If credentials don’t appear in the dropdown:

* Ensure they’re added under **System → Global credentials (unrestricted)**
* Use **Username with password** (not Secret text) for HTTPS GitHub URLs
* Restart Jenkins: `docker restart jenkins`

### 🔹 "Couldn't find remote ref refs/heads/master"

Change your branch name in job configuration from `master` → `main`.

### 🔹 GitHub Server form doesn’t expand

Reload Jenkins configuration from disk or restart the container.
If issue persists, reinstall the **GitHub API plugin**.

### 🔹 403 No valid crumb error

This occurs when CSRF protection blocks a form submission. Refresh the page and retry or disable the form temporarily under **Manage Jenkins → Configure Global Security → CSRF Protection** (only for local testing).

### 🔹 Jenkins can’t build Docker images

Recreate container with:

```powershell
-v /var/run/docker.sock:/var/run/docker.sock
```

Then install **Docker Pipeline** plugin.

---

## ✅ Summary

* Jenkins runs in Docker and connects securely to GitHub
* GitHub PAT added as a credential for SCM access
* Pipeline builds run directly from GitHub repos
* Webhooks optionally trigger builds automatically
