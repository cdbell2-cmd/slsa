# SLSA Hands-On Lab

**Application repo:** https://github.com/spring-projects/spring-petclinic (fork this before starting)  
**Build tool:** Maven (./mvnw) · **Runtime:** Java 17 · **Artifact:** spring-petclinic-4.0.0-SNAPSHOT.jar  
**Environment:** CloudBees CI Modern on AWS EKS  
**Time:** ~90 minutes total  
**Prerequisite:** CloudBees CI test controller access · Git installed locally · cosign installed on your workstation

Read `SLSA-Feynman.md` before starting this lab.

---

## Lab overview

```
Pre-lab: Fork the repo and prepare your workspace
Lab 1:   Create a Multibranch Pipeline in CloudBees CI       (~15 min)
Lab 2:   Build the base Jenkinsfile                          (~15 min)
Lab 3:   Add SLSA Level 1 provenance                         (~20 min)
Lab 4:   Install cosign and prepare signing keys             (~10 min)
Lab 5:   Add SLSA Level 2 signed provenance                  (~20 min)
Lab 6:   Verify the signature end-to-end                     (~10 min)
Lab 7:   Lock down signing credentials with RBAC             (~10 min)
```

Each lab has a **Goal**, numbered steps, and an **Expected result**.

---

## Pre-lab — Fork the repo and clone it locally

**Goal:** You need your own fork so you can push Jenkinsfile changes without needing write access to the upstream repo. CloudBees CI will poll your fork.

### Steps

**P.1 Fork on GitHub**

1. Open https://github.com/spring-projects/spring-petclinic in your browser
2. Click **Fork** (top right) → **Create fork**
3. Note your fork URL — it will be `https://github.com/<YOUR-GITHUB-USERNAME>/spring-petclinic`

**P.2 Clone your fork locally**

```bash
git clone https://github.com/<YOUR-GITHUB-USERNAME>/spring-petclinic
cd spring-petclinic
```

**P.3 Understand the project layout**

```
spring-petclinic/
├── pom.xml               ← Maven build descriptor (Java 17, produces a JAR)
├── mvnw / mvnw.cmd       ← Maven wrapper — use this instead of system mvn
├── src/
│   ├── main/java/...     ← Spring Boot application source
│   └── test/java/...     ← JUnit tests
└── k8s/                  ← Kubernetes manifests (not used in this lab)
```

After `./mvnw package`, Maven produces:
```
target/spring-petclinic-4.0.0-SNAPSHOT.jar   ← the artifact we will fingerprint and sign
target/bom.json                                ← CycloneDX SBOM (generated automatically)
target/bom.xml                                 ← CycloneDX SBOM (XML format)
```

The pom.xml already includes two plugins that give SLSA a head start:
- **git-commit-id-plugin** — embeds the git SHA and branch into `META-INF/build-info.properties` inside the JAR
- **cyclonedx-maven-plugin** — generates a Software Bill of Materials (SBOM) listing every dependency during `mvn package`

These run automatically; you do not need to configure them.

---

## Lab 1 — Create a Multibranch Pipeline in CloudBees CI

**Goal:** Connect CloudBees CI to your spring-petclinic fork so every push to `main` triggers a build. This establishes the "hosted build service" requirement that underpins SLSA L1 and L2.

### Steps

**1.1 Add a GitHub credential to the test controller**

CloudBees CI needs read access to your fork and permission to post build statuses back to GitHub. Create a Personal Access Token (PAT) on GitHub first:

1. GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Scopes: tick the top-level **`repo`** checkbox (this includes `repo:status` as a sub-scope — both are required; `repo` alone for checkout, `repo:status` for CloudBees CI to post build results back to your PR)
4. Click **Generate token** and copy the value

> **Note:** If you see `Could not update commit status — 403` in the build log, the PAT is missing the `repo:status` scope. Regenerate it with the full `repo` checkbox selected.

Store the token in CloudBees CI:

1. On the test controller, go to **Manage Jenkins → Manage Credentials**
2. **System → Global credentials → Add Credentials**
3. Fill in:
   - **Kind:** Username with password
   - **Username:** your GitHub username
   - **Password:** the PAT you just generated
   - **ID:** `github-petclinic`
   - **Description:** `GitHub PAT for spring-petclinic fork`
4. Click **Create**

**1.2 Create a Multibranch Pipeline job**

A Multibranch Pipeline automatically discovers branches and pull requests in your repo and creates a sub-pipeline for each one. This is the standard CloudBees CI pattern for source-controlled pipelines.

1. From the controller dashboard, click **New Item**
2. Enter name: `spring-petclinic`
3. Select **Multibranch Pipeline** → click **OK**
4. Under **Branch Sources**, click **Add source → GitHub**
5. Set:
   - **Credentials:** `github-petclinic` (from step 1.1)
   - **Repository HTTPS URL:** `https://github.com/<YOUR-GITHUB-USERNAME>/spring-petclinic`
6. Under **Build Configuration**, confirm **Mode = by Jenkinsfile** and **Script Path = Jenkinsfile**
7. Click **Save**

CloudBees CI will immediately scan the repo. Because there is no `Jenkinsfile` yet, it will find no buildable branches. That is expected — you will create the Jenkinsfile in Lab 2.

**1.3 Verify the scan completed**

Click **Scan Multibranch Pipeline Log** in the left nav. You should see:

```
Checking branches...
  Checking branch main
    'Jenkinsfile' not found
  Does not meet criteria
Finished: SUCCESS
```

### Expected result

A Multibranch Pipeline job named `spring-petclinic` exists on the controller and has scanned your fork successfully, reporting that no buildable branches were found (because Jenkinsfile does not exist yet).

---

## Lab 2 — Build the base Jenkinsfile

**Goal:** Create and push a working Jenkinsfile that builds and tests spring-petclinic using Maven. This is the scripted build required by SLSA L1.

### Steps

**2.1 Create the Jenkinsfile**

In the root of your local clone, create a file named `Jenkinsfile` with this content:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Capture git context for later stages
                    env.GIT_COMMIT_SHA = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                    env.GIT_REPO_URL = sh(
                        script: 'git remote get-url origin',
                        returnStdout: true
                    ).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    // cyclonedx:makeAggregateBom must be invoked explicitly —
                    // the plugin is in pom.xml but not bound to a lifecycle phase by default
                    sh './mvnw -B -DskipTests package cyclonedx:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

    }

    post {
        success {
            echo "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo "Build failed: check the stage logs above"
        }
    }
}
```

**2.2 Commit and push**

```bash
git add Jenkinsfile
git commit -m "Add base Jenkinsfile — Maven build and test"
git push origin main
```

**2.3 Trigger the scan and first build**

In CloudBees CI on the `spring-petclinic` Multibranch Pipeline:

1. Click **Scan Multibranch Pipeline Now** in the left nav
2. The scan will now find `Jenkinsfile` on `main` and create a `main` branch pipeline
3. A build will start automatically
4. Click **main → #1** to watch the live build log

The first build downloads Maven dependencies — expect it to take 3–5 minutes. Subsequent builds use the pod's cache volume and will be faster.

**2.4 Confirm the build log shows SCM checkout**

In the build log you should see:

```
[Pipeline] checkout
Cloning repository https://github.com/<YOUR-USERNAME>/spring-petclinic
 > git checkout -f <commit-sha>
...
[maven] ./mvnw -B -DskipTests package
...
[INFO] Building spring-petclinic 4.0.0-SNAPSHOT
...
[INFO] BUILD SUCCESS
```

And the test stage:

```
[maven] ./mvnw -B test
...
[INFO] Tests run: X, Failures: 0, Errors: 0
```

### Expected result

The `main` branch sub-pipeline runs and goes green. The build log shows the Maven wrapper producing `target/spring-petclinic-4.0.0-SNAPSHOT.jar` and all tests passing.

---

## Lab 3 — Add SLSA Level 1 provenance

**Goal:** Every build produces a fingerprinted JAR, an SBOM, and a provenance document. No signing yet — this is the auditable paper trail required for L1.

### What you're adding

```
Build (Lab 2)
  └── Archive JAR        ← Lab 3.1: fingerprint: true
  └── L1 Provenance      ← Lab 3.2: provenance-l1.json capturing git SHA, build URL, artifact hash
  └── SBOM               ← Already generated by pom.xml (target/bom.json) — just archive it
```

### Steps

**3.1 Archive the JAR with fingerprinting**

Add a new `Archive` stage to your Jenkinsfile, after the `Test` stage:

```groovy
stage('Archive') {
    steps {
        // fingerprint: true records an MD5 hash of the JAR in Jenkins
        // allowing you to trace exactly which builds produced or consumed it
        archiveArtifacts artifacts: 'target/spring-petclinic-*.jar',
                         fingerprint: true
        // Archive the CycloneDX SBOM generated automatically by the pom.xml
        archiveArtifacts artifacts: 'target/bom.json, target/bom.xml',
                         allowEmptyArchive: true
    }
}
```

**3.2 Add L1 provenance generation**

Add this stage after `Archive`:

```groovy
stage('SLSA L1 Provenance') {
    steps {
        container('maven') {
            script {
                def buildTime  = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
                def jarFile    = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                def jarSha256  = sh(
                    script: "sha256sum ${jarFile} | awk '{print \$1}'",
                    returnStdout: true
                ).trim()

                def provenance = """{
  "slsa_level": "1",
  "subject": {
    "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
    "digest": {
      "sha256": "${jarSha256}"
    }
  },
  "source": {
    "repository": "${env.GIT_REPO_URL}",
    "commit": "${env.GIT_COMMIT_SHA}",
    "branch": "${env.BRANCH_NAME ?: 'main'}"
  },
  "builder": {
    "id": "${env.JENKINS_URL}",
    "platform": "CloudBees CI on EKS"
  },
  "build": {
    "url": "${env.BUILD_URL}",
    "job": "${env.JOB_NAME}",
    "number": "${env.BUILD_NUMBER}",
    "tool": "maven:3.9-eclipse-temurin-17"
  },
  "metadata": {
    "built_at": "${buildTime}",
    "sbom": "target/bom.json"
  }
}"""
                writeFile file: 'provenance-l1.json', text: provenance
                echo "=== SLSA L1 Provenance ==="
                sh 'cat provenance-l1.json'
            }
        }
        archiveArtifacts artifacts: 'provenance-l1.json', fingerprint: true
    }
}
```

**3.3 Your complete Jenkinsfile after Lab 3**

At this point your Jenkinsfile should look like this:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_REPO_URL   = sh(script: 'git remote get-url origin', returnStdout: true).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    // cyclonedx:makeAggregateBom must be invoked explicitly —
                    // the plugin is in pom.xml but not bound to a lifecycle phase by default
                    sh './mvnw -B -DskipTests package cyclonedx:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
                archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
            }
        }

        stage('SLSA L1 Provenance') {
            steps {
                container('maven') {
                    script {
                        def buildTime = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
                        def jarFile   = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                        def jarSha256 = sh(
                            script: "sha256sum ${jarFile} | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        def provenance = """{
  "slsa_level": "1",
  "subject": {
    "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
    "digest": { "sha256": "${jarSha256}" }
  },
  "source": {
    "repository": "${env.GIT_REPO_URL}",
    "commit": "${env.GIT_COMMIT_SHA}",
    "branch": "${env.BRANCH_NAME ?: 'main'}"
  },
  "builder": {
    "id": "${env.JENKINS_URL}",
    "platform": "CloudBees CI on EKS"
  },
  "build": {
    "url": "${env.BUILD_URL}",
    "job": "${env.JOB_NAME}",
    "number": "${env.BUILD_NUMBER}",
    "tool": "maven:3.9-eclipse-temurin-17"
  },
  "metadata": {
    "built_at": "${buildTime}",
    "sbom": "target/bom.json"
  }
}"""
                        writeFile file: 'provenance-l1.json', text: provenance
                        echo "=== SLSA L1 Provenance ==="
                        sh 'cat provenance-l1.json'
                    }
                }
                archiveArtifacts artifacts: 'provenance-l1.json', fingerprint: true
            }
        }

    }

    post {
        success { echo "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
```

**3.4 Commit, push, and run**

```bash
git add Jenkinsfile
git commit -m "Add SLSA L1: fingerprinting, SBOM archival, provenance document"
git push origin main
```

CloudBees CI will detect the push and start a build automatically (if you have a webhook configured) or wait for the next scan interval. Trigger manually via **Build Now** on the `main` branch if needed.

**3.5 Inspect the L1 artifacts**

After the build completes:

a. Click **Archived Artifacts** in the build left nav. You should see:
   - `spring-petclinic-4.0.0-SNAPSHOT.jar`
   - `bom.json` and `bom.xml`
   - `provenance-l1.json`

b. Click `provenance-l1.json`. Verify the `commit` field matches the git SHA in the build log and the `sha256` of the JAR is present.

c. Click **Fingerprints** in the build left nav. The JAR and provenance file should both appear with their MD5 fingerprint hash.

**3.6 Explore the SBOM (bonus)**

Click `bom.json` in Archived Artifacts. This file was generated automatically by the CycloneDX Maven Plugin that is already in `pom.xml`. It lists every dependency in the PetClinic application — Spring Boot, Thymeleaf, Hibernate, etc. — each with its version, group, and hash. This is complementary to SLSA provenance: provenance records _how_ the software was built; the SBOM records _what_ went into it.

### Expected result

The build goes green with five stages. Archived artifacts include the JAR, both SBOM files, and `provenance-l1.json`. The provenance file contains the actual git commit SHA from this build and the SHA256 hash of the JAR.

---

## Lab 4 — Install cosign and prepare signing keys

**Goal:** cosign installed on your workstation and available inside the maven build container; a key pair ready for use.

> **Why no cosign sidecar?** The official `gcr.io/projectsigstore/cosign` image is a _distroless_ image — it contains only the cosign binary with no shell, no `sleep`, and no other utilities. CloudBees CI's Kubernetes plugin keeps sidecar containers alive with `sleep infinity`, which distroless images cannot run. The solution is to download the cosign binary directly into the maven container as a pipeline step. This is simpler, more transparent, and works in any Kubernetes environment.

### Steps

**4.1 Install cosign on your workstation**

macOS:
```bash
brew install cosign
cosign version
# Expected: cosign version v2.x.x
```

Linux:
```bash
COSIGN_VERSION=v2.4.0
curl -Lo cosign "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
chmod +x cosign
sudo mv cosign /usr/local/bin/
cosign version
```

**4.2 Generate a key pair**

Run this on your workstation — never inside a CI job. When prompted for a passphrase, press Enter twice for no passphrase (acceptable for this lab; use a passphrase in production).

```bash
cd ~/  # run outside the petclinic repo so keys aren't accidentally committed
cosign generate-key-pair
```

This creates:
```
cosign.key   ← private key — KEEP THIS SECRET, never commit it
cosign.pub   ← public key — safe to share with anyone who needs to verify
```

**4.3 Store the private key in CloudBees CI Credentials**

Use **Secret file**, not **Secret text**. The cosign key is a PEM file with required newlines — pasting it into a text field strips or mangles those newlines and produces an `invalid pem block` error when cosign reads it. Uploading the file preserves the exact bytes.

1. On the test controller, go to **Manage Jenkins → Manage Credentials**
2. **System → Global credentials → Add Credentials**
3. Fill in:
   - **Kind:** Secret file
   - **File:** click **Browse** and upload `~/cosign.key` from your workstation
   - **ID:** `cosign-private-key`
   - **Description:** `SLSA L2 cosign signing key for spring-petclinic`
4. Click **Create**

Verify it appears in the credentials list. In the pipeline, a `file()` binding mounts the key at a Jenkins-managed temp path and removes it automatically when the `withCredentials` block exits — no manual `printf` or `rm` required.

**4.4 Add a Setup stage to install cosign in the build pod**

The maven container (`maven:3.9-eclipse-temurin-17`) is a Debian-based image with `curl` available. Add a `Setup` stage that downloads the cosign binary once per build, before any signing steps. The pod template stays exactly as it was in Lab 2 and 3 — **no sidecar needed**.

Add this stage immediately after `Checkout` in your Jenkinsfile:

```groovy
stage('Setup') {
    steps {
        container('maven') {
            sh '''
                COSIGN_VERSION=v2.4.0
                curl -sSfLo /usr/local/bin/cosign \
                    "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
                chmod +x /usr/local/bin/cosign
                cosign version
            '''
        }
    }
}
```

**4.5 Commit, push, and confirm**

```bash
git add Jenkinsfile
git commit -m "Add Setup stage: install cosign binary in maven container"
git push origin main
```

In the build log, the Setup stage should print:

```
  % Total    % Received ...
    cosign version v2.4.0
    ...
```

### Expected result

- `~/cosign.key` and `~/cosign.pub` exist on your workstation
- `cosign-private-key` credential exists in CloudBees CI
- The `Setup` stage in the build log prints the cosign version with no errors
- The pod template still has only the `maven` container — no sidecar

---

## Lab 5 — Add SLSA Level 2 signed provenance

**Goal:** The pipeline cryptographically signs the JAR using cosign. Anyone with the public key can verify the JAR came from this CloudBees CI build and has not been modified since.

At L2, we sign **the JAR file directly** using `cosign sign-blob`. This produces a detached signature file (`.sig`) that is archived alongside the JAR and can be verified offline.

### Steps

**5.1 Replace the SLSA L1 provenance stage with the L2 version**

Remove the `SLSA L1 Provenance` stage from Lab 3 and replace it with this combined L2 stage. All signing runs inside the `maven` container where cosign was installed by the `Setup` stage.

> **No `--output-certificate` flag:** that flag applies only to keyless/OIDC signing. With a local key (`--key`), cosign produces only a signature file — no certificate is generated or needed.

```groovy
stage('SLSA L2 Provenance — sign with cosign') {
    steps {
        container('maven') {
            script {
                def buildTime = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
                def jarFile   = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                def jarSha256 = sh(
                    script: "sha256sum ${jarFile} | awk '{print \$1}'",
                    returnStdout: true
                ).trim()

                // SLSA v1.0 provenance predicate
                def provenance = """{
  "buildDefinition": {
    "buildType": "https://cloudbees.com/jenkinsfile/v1",
    "externalParameters": {
      "repository": "${env.GIT_REPO_URL}",
      "ref": "${env.GIT_COMMIT_SHA}",
      "branch": "${env.BRANCH_NAME ?: 'main'}"
    },
    "internalParameters": {
      "buildUrl": "${env.BUILD_URL}",
      "controllerId": "${env.JENKINS_URL}",
      "jobName": "${env.JOB_NAME}",
      "buildNumber": "${env.BUILD_NUMBER}",
      "buildTool": "maven:3.9-eclipse-temurin-17"
    }
  },
  "runDetails": {
    "builder": {
      "id": "https://cloudbees-ci.proofpoint.com"
    },
    "metadata": {
      "invocationId": "${env.BUILD_URL}",
      "startedOn": "${buildTime}"
    }
  },
  "subject": [
    {
      "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
      "digest": { "sha256": "${jarSha256}" }
    }
  ]
}"""
                writeFile file: 'provenance-l2.json', text: provenance
                echo "=== SLSA L2 Provenance (pre-signing) ==="
                sh 'cat provenance-l2.json'

                // Capture cosign's base64 signature from stdout, then commit via Jenkins
                // writeFile — avoids TAR streaming failures caused by cosign's direct file
                // I/O on Kubernetes pod volumes not flushing before archiveArtifacts reads them.
                withCredentials([file(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY_PATH')]) {
                    def jarSig = sh(
                        script: '''
                            export COSIGN_PASSWORD=""
                            cosign sign-blob \
                                --key "${COSIGN_KEY_PATH}" \
                                --tlog-upload=false \
                                target/spring-petclinic-4.0.0-SNAPSHOT.jar
                        ''',
                        returnStdout: true
                    ).trim()
                    writeFile file: 'target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig', text: jarSig

                    def provSig = sh(
                        script: '''
                            export COSIGN_PASSWORD=""
                            cosign sign-blob \
                                --key "${COSIGN_KEY_PATH}" \
                                --tlog-upload=false \
                                provenance-l2.json
                        ''',
                        returnStdout: true
                    ).trim()
                    writeFile file: 'provenance-l2.json.sig', text: provSig
                }

                sh 'ls -lh target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig provenance-l2.json.sig'
            }

            archiveArtifacts artifacts: 'provenance-l2.json, provenance-l2.json.sig, target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig'
        }
    }
}
```

**5.2 Your complete Jenkinsfile after Lab 5**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: [sleep, infinity]
    env:
    - name: MAVEN_OPTS
      value: "-Xmx512m"
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  volumes:
  - name: maven-cache
    emptyDir: {}
"""
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHA = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    env.GIT_REPO_URL   = sh(script: 'git remote get-url origin', returnStdout: true).trim()
                    echo "Building commit: ${env.GIT_COMMIT_SHA}"
                }
            }
        }

        stage('Setup') {
            steps {
                container('maven') {
                    sh '''
                        COSIGN_VERSION=v2.4.0
                        curl -sSfLo /usr/local/bin/cosign \
                            "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
                        chmod +x /usr/local/bin/cosign
                        cosign version
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    // cyclonedx:makeAggregateBom must be invoked explicitly —
                    // the plugin is in pom.xml but not bound to a lifecycle phase by default
                    sh './mvnw -B -DskipTests package cyclonedx:makeAggregateBom'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh './mvnw -B test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'target/spring-petclinic-*.jar', fingerprint: true
                archiveArtifacts artifacts: 'target/bom.json, target/bom.xml', allowEmptyArchive: true
            }
        }

        stage('SLSA L2 Provenance') {
            steps {
                container('maven') {
                    script {
                        def buildTime = sh(script: 'date -u +"%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
                        def jarFile   = 'target/spring-petclinic-4.0.0-SNAPSHOT.jar'
                        def jarSha256 = sh(
                            script: "sha256sum ${jarFile} | awk '{print \$1}'",
                            returnStdout: true
                        ).trim()

                        def provenance = """{
  "buildDefinition": {
    "buildType": "https://cloudbees.com/jenkinsfile/v1",
    "externalParameters": {
      "repository": "${env.GIT_REPO_URL}",
      "ref": "${env.GIT_COMMIT_SHA}",
      "branch": "${env.BRANCH_NAME ?: 'main'}"
    },
    "internalParameters": {
      "buildUrl": "${env.BUILD_URL}",
      "controllerId": "${env.JENKINS_URL}",
      "jobName": "${env.JOB_NAME}",
      "buildNumber": "${env.BUILD_NUMBER}",
      "buildTool": "maven:3.9-eclipse-temurin-17"
    }
  },
  "runDetails": {
    "builder": {
      "id": "https://cloudbees-ci.proofpoint.com"
    },
    "metadata": {
      "invocationId": "${env.BUILD_URL}",
      "startedOn": "${buildTime}"
    }
  },
  "subject": [
    {
      "name": "spring-petclinic-4.0.0-SNAPSHOT.jar",
      "digest": { "sha256": "${jarSha256}" }
    }
  ]
}"""
                        writeFile file: 'provenance-l2.json', text: provenance
                        sh 'cat provenance-l2.json'

                        withCredentials([file(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY_PATH')]) {
                            def jarSig = sh(
                                script: '''
                                    export COSIGN_PASSWORD=""
                                    cosign sign-blob \
                                        --key "${COSIGN_KEY_PATH}" \
                                        --tlog-upload=false \
                                        target/spring-petclinic-4.0.0-SNAPSHOT.jar
                                ''',
                                returnStdout: true
                            ).trim()
                            writeFile file: 'target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig', text: jarSig

                            def provSig = sh(
                                script: '''
                                    export COSIGN_PASSWORD=""
                                    cosign sign-blob \
                                        --key "${COSIGN_KEY_PATH}" \
                                        --tlog-upload=false \
                                        provenance-l2.json
                                ''',
                                returnStdout: true
                            ).trim()
                            writeFile file: 'provenance-l2.json.sig', text: provSig
                        }

                        sh 'ls -lh target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig provenance-l2.json.sig'
                    }

                    archiveArtifacts artifacts: 'provenance-l2.json, provenance-l2.json.sig, target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig'
                }
            }
        }

    }

    post {
        success { echo "SLSA L2 build complete: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
        failure { echo "Build failed: check the stage logs above" }
    }
}
```

**5.3 Commit, push, and run**

```bash
git add Jenkinsfile
git commit -m "Add SLSA L2: cosign sign-blob on JAR and provenance document"
git push origin main
```

Watch the **SLSA L2 Provenance** stage. A successful run looks like this in the stage log:

```
Using payload from: target/spring-petclinic-4.0.0-SNAPSHOT.jar
[Pipeline] writeFile        ← Jenkins writes the captured signature to .jar.sig
Using payload from: provenance-l2.json
[Pipeline] writeFile        ← Jenkins writes the captured signature to .json.sig
-rw-r--r-- 1 root root 96 Jan 1 00:00 target/spring-petclinic-4.0.0-SNAPSHOT.jar.sig
-rw-r--r-- 1 root root 88 Jan 1 00:00 provenance-l2.json.sig
```

> **Why `returnStdout: true` + `writeFile`?** cosign's `--output-signature` writes directly to the Kubernetes pod volume. Jenkins' TAR transfer reads that file immediately after cosign exits, and a race between the OS flush and the TAR read can produce:
> `entry 'provenance-l2.json.sig' closed at '0' before the '96' bytes specified in the header were written`
> Capturing stdout and writing with `writeFile` puts Jenkins in control of the file write, eliminating the race.

**5.4 Download the artifacts for Lab 6**

From the build's **Archived Artifacts**, download to your local machine:
- `spring-petclinic-4.0.0-SNAPSHOT.jar`
- `spring-petclinic-4.0.0-SNAPSHOT.jar.sig`
- `provenance-l2.json`
- `provenance-l2.json.sig`

You will need these for the verification step in Lab 6.

### Expected result

The build goes green with six stages (Checkout, Setup, Build, Test, Archive, SLSA L2 Provenance). The signing stage archives three files: the JAR signature, the provenance document, and the provenance signature. The build log shows cosign output and confirms key cleanup.

---

## Lab 6 — Verify the signature end-to-end

**Goal:** Confirm that anyone with only the public key can verify the JAR and provenance — and that tampering is detected. This is the test that proves SLSA L2 is working.

### Steps

**6.1 Verify the JAR signature**

From your workstation, in the directory where you downloaded the artifacts (step 5.4):

```bash
cosign verify-blob \
    --key ~/cosign.pub \
    --signature spring-petclinic-4.0.0-SNAPSHOT.jar.sig \
    spring-petclinic-4.0.0-SNAPSHOT.jar
```

**Successful output:**

```
Verified OK
```

**6.2 Verify the provenance signature**

```bash
cosign verify-blob \
    --key ~/cosign.pub \
    --signature provenance-l2.json.sig \
    provenance-l2.json
```

**Successful output:**

```
Verified OK
```

**6.3 Read the provenance content**

```bash
cat provenance-l2.json | python3 -m json.tool
```

Confirm the output shows:
- `ref` field matches the git commit SHA from the build log
- `buildUrl` points to your CloudBees CI build
- `sha256` digest of the JAR matches what `sha256sum spring-petclinic-4.0.0-SNAPSHOT.jar` returns on your machine

This is the chain of custody: source commit → build on hosted platform → signed artifact.

**6.4 Tamper test — modify the JAR and re-verify**

This step shows what happens when an artifact is tampered with after signing.

```bash
# Make a tiny change to the JAR (append a byte)
echo "tampered" >> spring-petclinic-4.0.0-SNAPSHOT.jar

# Now try to verify — this must fail
cosign verify-blob \
    --key ~/cosign.pub \
    --signature spring-petclinic-4.0.0-SNAPSHOT.jar.sig \
    spring-petclinic-4.0.0-SNAPSHOT.jar
```

**Expected failure:**

```
Error: verifying blob spring-petclinic-4.0.0-SNAPSHOT.jar: invalid signature
main.go:69: error during command execution: verifying blob spring-petclinic-4.0.0-SNAPSHOT.jar: invalid signature
```

Restore the original JAR (re-download from CloudBees CI Archived Artifacts) and verify again — it passes. This confirms the signature is bound to the exact bytes of the artifact.

**6.5 Tamper test — forge a provenance document**

```bash
# Try to claim the build came from a different commit
sed 's/"ref": ".*"/"ref": "deadbeef0000"/' provenance-l2.json > fake-provenance.json

# Attempt verification — fails because the content hash has changed
cosign verify-blob \
    --key ~/cosign.pub \
    --signature provenance-l2.json.sig \
    fake-provenance.json
```

**Expected failure:**

```
Error: invalid signature
```

An attacker cannot alter the provenance and make it pass verification without the private key. The private key never left the CloudBees CI credential store.

### Expected result

- `cosign verify-blob` passes for the original JAR and provenance document
- Verification fails immediately when either file is modified
- The provenance document content matches the actual build metadata from CloudBees CI

---

## Lab 7 — Lock down signing credentials with RBAC

**Goal:** The `cosign-private-key` credential is currently in Global scope — any pipeline on the controller can reference it. Restrict it so only the SLSA pipeline folder can use it.

### Steps

**7.1 Create a dedicated folder for SLSA pipelines**

1. From the controller dashboard, click **New Item**
2. Enter name: `slsa-builds`
3. Select **Folder** → click **OK**
4. Click **Save**

**7.2 Move the pipeline into the folder**

1. From the controller dashboard, locate the `spring-petclinic` Multibranch Pipeline
2. Click the job → **Move** (in the left nav)
3. Select destination: `slsa-builds` folder → click **Move**

The pipeline is now at `slsa-builds/spring-petclinic`.

**7.3 Move the credential to folder scope**

1. Navigate to the `slsa-builds` folder
2. Click **Credentials** (left nav) → **Folder** → **Global credentials** → **Add Credentials**
3. Re-add the cosign key with the same values as step 4.3 in Lab 4:
   - **Kind:** Secret file
   - **File:** upload `~/cosign.key`
   - **ID:** `cosign-private-key`
4. Click **Create**

Now delete the global-scoped version:

1. Go back to **Manage Jenkins → Manage Credentials → System → Global credentials**
2. Find `cosign-private-key`, click the dropdown → **Delete**
3. Confirm deletion

**7.4 Configure RBAC to restrict credential use**

On the Operations Center:

1. Go to **Operations Center → Manage Jenkins → Configure Global Security**
2. Under **Authorization**, ensure **Role-Based Access Control** is enabled
3. Add a role named `slsa-builder` with permission: `Credentials/Use`
4. Assign `slsa-builder` to the `slsa-builds` folder, scoped to the CI service account
5. Ensure regular developer roles do NOT have `Credentials/Use` on this folder

**7.5 Test the restriction**

Log in as a developer user (not admin). Try to create a freestyle job outside the `slsa-builds` folder that references `cosign-private-key`. The build should fail with:

```
ERROR: No credentials found with id: cosign-private-key
```

**7.6 Confirm the pipeline still works**

Trigger a build of `slsa-builds/spring-petclinic/main`. The signing stage should complete successfully — the credential is accessible because the pipeline is inside the authorized folder.

**7.7 Confirm key cleanup**

Check the build log for the `post { always }` block output:

```
[Pipeline] sh
+ rm -f /tmp/cosign.key || true
```

This confirms the key does not persist on the build agent pod.

### Expected result

- `cosign-private-key` is scoped to the `slsa-builds` folder only
- Pipelines outside the folder cannot access the credential
- The `slsa-builds/spring-petclinic` pipeline continues to produce signed artifacts
- Build log confirms key removed from agent pod after signing

---

## Lab completion checklist

**SLSA Level 1**
- [ ] Jenkinsfile committed to SCM — pipeline runs from source (Lab 2)
- [ ] `spring-petclinic-4.0.0-SNAPSHOT.jar` archived with `fingerprint: true` (Lab 3)
- [ ] `bom.json` and `bom.xml` (CycloneDX SBOM) archived (Lab 3)
- [ ] `provenance-l1.json` archived with git SHA and artifact hash (Lab 3)
- [ ] Artifact fingerprints visible in Jenkins Fingerprints UI (Lab 3)

**SLSA Level 2**
- [ ] cosign installed on workstation and downloaded into maven container via Setup stage (Lab 4)
- [ ] Private key stored in CloudBees Credentials, not in source code (Lab 4)
- [ ] `spring-petclinic-4.0.0-SNAPSHOT.jar.sig` produced per build (Lab 5)
- [ ] `provenance-l2.json` and `provenance-l2.json.sig` produced per build (Lab 5)
- [ ] `cosign verify-blob` passes on original JAR (Lab 6)
- [ ] Tampered JAR fails verification (Lab 6)
- [ ] Provenance content matches actual build metadata (Lab 6)
- [ ] Signing credential scoped to `slsa-builds` folder only (Lab 7)
- [ ] Private key removed from agent pod after signing (Lab 7)

---

## Troubleshooting

**cosign container terminated with `exec: "sleep": executable file not found in $PATH`**  
The official `gcr.io/projectsigstore/cosign` image is distroless — it has no shell, no `sleep`, and no utilities beyond the cosign binary. CloudBees CI keeps sidecar containers alive using `sleep infinity`, so this image cannot be used as a Kubernetes pod sidecar. The fix used in this lab is to download the cosign binary directly into the maven container during the `Setup` stage — no sidecar is required.

**`Could not update commit status — 403` in the build log**  
The GitHub PAT is missing the `repo:status` scope. Go to GitHub → **Settings → Developer settings → Personal access tokens**, edit the token, and ensure the top-level **`repo`** checkbox is selected (not just `public_repo`). Re-save and update the `github-petclinic` credential in CloudBees CI with the new token. This error does not abort the build but means CloudBees CI cannot post ✓/✗ status back to GitHub pull requests.

**Multibranch scan finds no branches after pushing the Jenkinsfile**  
Check the scan log (**Scan Multibranch Pipeline Log**). If it reports `No credentials found`, verify the `github-petclinic` credential ID is spelled exactly as entered in step 1.1. If it says `branch filter excludes 'main'`, check the branch discovery strategy — by default, Multibranch includes all branches.

**`./mvnw` exits with `Permission denied`**  
The Maven wrapper must be executable. Fix with:
```bash
git update-index --chmod=+x mvnw
git commit -m "Make mvnw executable"
git push
```

**`sha256sum: command not found` in the maven container**  
The `maven:3.9-eclipse-temurin-17` image is Debian-based. Run `apt-get install -y coreutils` or switch to `openssl dgst -sha256`:
```bash
openssl dgst -sha256 target/spring-petclinic-4.0.0-SNAPSHOT.jar | awk '{print $2}'
```

**cosign fails with `invalid pem block`**  
The private key PEM file was stored as a **Secret text** credential, which strips or mangles the required newlines in the PEM format. Delete the credential and re-create it as **Secret file** (upload `~/cosign.key` directly). With a `file()` binding in `withCredentials`, Jenkins writes the exact bytes of the uploaded file to a managed temp path — no text processing, no newline corruption.

**cosign fails with `upload to tlog: user declined the prompt`**  
cosign v2 defaults to uploading a record to Sigstore's public Rekor transparency log and requires interactive consent (`y`). In a non-TTY container this prompt can never be answered. Add `--tlog-upload=false` to every `cosign sign-blob` call. This is correct for enterprise pipelines — the signatures are verified with your own public key, not via Rekor.

**cosign fails with `inappropriate ioctl for device` or `Enter password for private key:`**  
cosign is trying to prompt for the key passphrase on stdin, but there is no TTY in a container. Fix: set `export COSIGN_PASSWORD=""` before the `cosign sign-blob` calls (empty string = no passphrase). If a real passphrase was used when generating the key, store it as a separate **Secret text** credential and bind it:
```groovy
withCredentials([
    file(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY_PATH'),
    string(credentialsId: 'cosign-passphrase', variable: 'COSIGN_PASSWORD')
]) { ... }
```

**cosign sign-blob fails with `error: getting signer: reading key: no such file`**  
The `file()` binding failed to resolve. Confirm the credential ID is exactly `cosign-private-key` (no spaces, exact case) and that the credential **Kind** is **Secret file**.

**cosign verify-blob fails with `invalid signature` on the original (unmodified) JAR**  
The JAR downloaded from Archived Artifacts may have been re-zipped or had metadata stripped by the browser. Download using `curl` with authentication instead:
```bash
curl -u <username>:<api-token> \
    "<JENKINS_URL>/job/slsa-builds/job/spring-petclinic/job/main/lastSuccessfulBuild/artifact/target/spring-petclinic-4.0.0-SNAPSHOT.jar" \
    -o spring-petclinic.jar
```

**Credentials not found after moving to folder scope (Lab 7)**  
The pipeline's `withCredentials` block looks up credentials by ID in the folder containing the job, then walks up to Global. If the credential is missing from both the folder and global scope, the build fails. Confirm the credential was created inside the `slsa-builds` folder (step 7.3), not just at the system level.

**Build takes 10+ minutes**  
The `emptyDir` Maven cache is discarded when the pod terminates. For faster builds, replace `emptyDir` with a `PersistentVolumeClaim` backed by EBS. Ask your EKS admin to provision a `ReadWriteOnce` PVC named `maven-cache` in the build namespace.
