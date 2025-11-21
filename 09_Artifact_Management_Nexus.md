# 09. Artifact Management: Nexus & Secure Supply Chain

Your code becomes a binary. That binary lives in Nexus.

## 1. Nexus Repository Manager: Deep Dive

### A. Repository Types
1.  **Proxy**: A mirror of the public internet.
    *   *Example*: `maven-central-proxy`.
    *   *Why*: Caching (Speed) and Stability (If Maven Central is down, you are fine).
2.  **Hosted**: Your private internal code.
    *   *Example*: `my-company-releases`.
    *   *Why*: Proprietary code that must not leak.
3.  **Group**: A virtual bucket combining Proxy + Hosted.
    *   *Example*: `maven-public`.
    *   *Config*: Developers point their `pom.xml` to this ONE URL.

### B. Cleanup Policies (Disk Space & Security)
Artifacts accumulate forever. You must clean them.
*   **Rule**: Keep last 5 snapshots. Delete anything older than 30 days.
*   **Implementation**:
    *   Nexus UI -> System -> Tasks -> "Remove Snapshots from Repository".
    *   **Groovy Scripting**: Nexus has a REST API that accepts Groovy scripts for advanced automation.

#### Groovy Script: Create Repo
```groovy
import org.sonatype.nexus.repository.config.Configuration

repository.createMavenHosted('my-private-repo')
```

---

## 2. Security: The Nexus Firewall

### A. Blocking Bad Components
Nexus Firewall (IQ Server) integrates with the Proxy repo.
*   **Policy**: "If Security-Risk >= 9 (Critical), BLOCK."
*   **Scenario**: Developer adds `log4j:2.14` to `pom.xml`.
*   **Result**: Maven build fails with `403 Forbidden`. Nexus refuses to download the jar.

### B. Authentication (Realms)
*   **Anonymous Access**: Turn this OFF.
*   **LDAP/Active Directory**: Map your AD groups to Nexus Roles.
    *   `AD_Developers` -> `nx-component-upload` (Can upload).
    *   `AD_Viewers` -> `nx-component-read` (Can download).

### C. User Token Support
Don't use AD passwords in CI/CD. Use User Tokens.
*   **Profile** -> **User Token** -> **Access User Token**.
*   Use this token in `settings.xml`.

---

## 3. Client Configuration (`settings.xml`)
Developers need to configure Maven to talk to Nexus.
File: `~/.m2/settings.xml`

```xml
<settings>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>admin</username>
      <password>{DES72...encrypted...}</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>http://nexus.company.com:8081/repository/maven-public/</url>
    </mirror>
  </mirrors>
</settings>
```
*   **Security Tip**: Use `mvn --encrypt-password` to generate the encrypted string. Never store plain text passwords here.

---

## 4. Docker Registry in Nexus
Nexus can host Docker images too.

### Setup
1.  Create a "docker (hosted)" repository.
2.  Enable an HTTP connector on port `8082`.
3.  **Nginx Reverse Proxy**:
    *   Docker requires HTTPS. Put Nginx in front of Nexus to handle SSL.

### Nginx Config Snippet
```nginx
server {
    listen 443 ssl;
    server_name docker.company.com;

    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto "https";
    }
}
```

---

## 5. Interview Preparation

### Q1: Why use a private repository manager like Nexus?
*   **Answer**:
    1.  **Speed**: Caching artifacts locally is faster than downloading from internet.
    2.  **Stability**: Builds don't break if internet is down.
    3.  **Security**: We can block bad libraries (Firewall) and host private code securely.
    4.  **Control**: We know exactly what versions are being used.

### Q2: How do you handle "Snapshot" vs "Release" versions?
*   **Answer**:
    *   **Snapshot** (`1.0-SNAPSHOT`): Mutable. Can be overwritten. Used during development.
    *   **Release** (`1.0`): Immutable. Once uploaded, CANNOT be changed. If you need to fix a bug, you must release `1.1`.
    *   **Security**: Never deploy Snapshots to Production.

---

## 6. Hands-On Lab
1.  **Install**: Run Nexus 3 in Docker.
    ```bash
    docker run -d -p 8081:8081 sonatype/nexus3
    ```
2.  **Login**: Find the admin password inside the container (`/nexus-data/admin.password`).
3.  **Create Repo**: Create a "maven-hosted" repo called `test-repo`.
4.  **Upload**: Use `curl` to upload a dummy text file to it.
5.  **Proxy**: Configure a proxy for `npmjs.org`. Try to `npm install` through Nexus.
