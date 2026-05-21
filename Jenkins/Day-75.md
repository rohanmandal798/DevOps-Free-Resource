### Module 1: Jenkins Installation & Setup

### Module 2: First Jenkins Job

### Module 3: Parameterized Builds

### Module 4: Build Periodically (Cron Jobs)

### Module 5: Git Integration with Jenkins

### Module 6: Maven Build Job

### Module 7: Maven Build Job with **Invoke top-level Maven targets**

---

# Automated Script to Install Jenkins

**NOTE: Check security group if not working. Allow port 8080**

### Cleanup Old Jenkins if Getting Error on Installing

Removes the old Jenkins repository configuration, old GPG keys and updates the package list.

### Copy the Cleanup Script to Your Kali Machine

1. **nano clean_jenkins.sh**
2. **Copy and paste**
3. **chmod +x clean_jenkins.sh**
4. **sudo ./clean_jenkins.sh**

```bash
#!/bin/bash

# Cleanup script to remove old/misconfigured Jenkins repository

echo "Cleaning up old Jenkins configuration..."

# Remove old repository files
rm -f /etc/apt/sources.list.d/jenkins.list
rm -f /etc/apt/sources.list.d/jenkins.list.save

# Remove old keys
rm -f /usr/share/keyrings/jenkins-keyring.asc
rm -f /etc/apt/keyrings/jenkins-keyring.asc

# Update package lists
apt-get update -y

echo "Cleanup complete! Now run: sudo bash install_jenkins.sh"
```

---

## Copy the Installation Script to Your Kali Machine

1. **nano install_jenkins.sh**
2. **Copy and paste**
3. **chmod +x install_jenkins.sh**
4. **sudo ./install_jenkins.sh**

```bash
#!/bin/bash

################################################################################
# Jenkins Installation and Setup Script for Kali Linux
# This script automates the installation and initial configuration of Jenkins
################################################################################

# Note: We don't use 'set -e' globally because apt-get update might fail
# if Jenkins repo is already misconfigured, which we handle gracefully

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging functions
log_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Check if script is run as root
check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "This script must be run as root (use sudo)"
        exit 1
    fi
    log_success "Running with root privileges"
}

# Check system requirements
check_system() {
    log_info "Checking system requirements..."
    
    # Check if running on Kali/Debian-based system
    if [[ ! -f /etc/debian_version ]]; then
        log_error "This script is designed for Debian-based systems (Kali Linux)"
        exit 1
    fi
    
    # Check available disk space (require at least 2GB free)
    available_space=$(df / | tail -1 | awk '{print $4}')
    if [[ $available_space -lt 2097152 ]]; then
        log_warning "Less than 2GB free disk space available"
    fi
    
    log_success "System check passed"
}

# Update system packages
update_system() {
    log_info "Updating system packages..."
    
    # Remove any existing Jenkins repository that might be misconfigured
    if [[ -f /etc/apt/sources.list.d/jenkins.list ]]; then
        log_info "Removing existing Jenkins repository configuration..."
        rm -f /etc/apt/sources.list.d/jenkins.list
    fi
    
    apt-get update -y || log_warning "Some repositories failed to update (this is OK if Jenkins repo was misconfigured)"
    log_success "System packages updated"
}

# Install Java (Jenkins requires Java 21)
install_java() {
    log_info "Checking for Java installation..."
    
    if command -v java &> /dev/null; then
        java_version=$(java -version 2>&1 | awk -F '"' '/version/ {print $2}')
        log_info "Java is already installed: $java_version"
        
        # Check if Java version is 21 or higher
        major_version=$(echo "$java_version" | cut -d. -f1)
        if [[ $major_version -lt 21 ]]; then
            log_warning "Jenkins requires Java 21 or later. Upgrading Java..."
            apt-get install -y fontconfig openjdk-21-jre
        fi
    else
        log_info "Installing OpenJDK 21 (required for Jenkins)..."
        apt-get install -y fontconfig openjdk-21-jre
        log_success "Java installed successfully"
    fi
    
    # Verify Java installation
    java -version
}

# Add Jenkins repository and key
add_jenkins_repo() {
    log_info "Adding Jenkins repository..."
    
    # Install prerequisites
    apt-get install -y wget gnupg2
    
    # Create keyrings directory if it doesn't exist
    mkdir -p /etc/apt/keyrings
    
    # Remove old Jenkins repository and keys if they exist
    rm -f /etc/apt/sources.list.d/jenkins.list
    rm -f /etc/apt/keyrings/jenkins-keyring.asc
    rm -f /usr/share/keyrings/jenkins-keyring.asc
    
    # Download Jenkins GPG key (2026 key for LTS stable)
    log_info "Downloading Jenkins GPG key..."
    wget -q -O /etc/apt/keyrings/jenkins-keyring.asc \
        https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
    
    if [[ ! -f /etc/apt/keyrings/jenkins-keyring.asc ]]; then
        log_error "Failed to download Jenkins GPG key"
        exit 1
    fi
    
    log_success "Jenkins GPG key downloaded"
    
    # Add Jenkins LTS repository with proper formatting
    log_info "Adding Jenkins repository..."
    echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
        tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    
    # Update package list
    log_info "Updating package lists..."
    apt-get update -y
    
    log_success "Jenkins repository added successfully"
}

# Install Jenkins
install_jenkins() {
    log_info "Installing Jenkins..."
    
    # Try to install Jenkins
    if apt-get install -y jenkins; then
        log_success "Jenkins installed successfully"
    else
        log_error "Jenkins installation failed"
        log_info "Checking repository configuration..."
        cat /etc/apt/sources.list.d/jenkins.list
        log_info "Checking GPG key..."
        ls -lh /etc/apt/keyrings/jenkins-keyring.asc
        exit 1
    fi
}

# Configure firewall (if UFW is active)
configure_firewall() {
    log_info "Checking firewall configuration..."
    
    if command -v ufw &> /dev/null && ufw status | grep -q "Status: active"; then
        log_info "Configuring UFW firewall for Jenkins (port 8080)..."
        ufw allow 8080/tcp
        ufw reload
        log_success "Firewall configured"
    else
        log_info "UFW firewall not active, skipping firewall configuration"
    fi
}

# Start Jenkins service
start_jenkins() {
    log_info "Starting Jenkins service..."
    systemctl daemon-reload
    systemctl enable jenkins
    systemctl start jenkins
    
    # Wait for Jenkins to start
    log_info "Waiting for Jenkins to start (this may take a minute)..."
    sleep 30
    
    if systemctl is-active --quiet jenkins; then
        log_success "Jenkins service is running"
    else
        log_error "Jenkins service failed to start"
        systemctl status jenkins
        exit 1
    fi
}

# Get initial admin password
get_admin_password() {
    log_info "Retrieving initial admin password..."

    if [[ -f /var/lib/jenkins/secrets/initialAdminPassword ]]; then
        initial_password=$(cat /var/lib/jenkins/secrets/initialAdminPassword)

        # Fetch public IP
        public_ip=$(curl -s https://api.ipify.org)

        # Fallback if API fails
        if [[ -z "$public_ip" ]]; then
            public_ip="localhost"
        fi

        echo ""
        echo "=========================================="
        echo -e "${GREEN}Jenkins Initial Setup Information${NC}"
        echo "=========================================="
        echo ""
        echo "Jenkins is now running!"
        echo ""

        echo -e "Local URL: ${BLUE}http://localhost:8080${NC}"
        echo -e "Public URL: ${BLUE}http://$public_ip:8080${NC}"

        echo ""
        echo -e "Initial Admin Password: ${YELLOW}${initial_password}${NC}"
        echo ""
        echo "Copy this password to complete the setup wizard in your browser."
        echo "=========================================="
        echo ""

    else
        log_warning "Initial admin password file not found. Jenkins may still be initializing."
        log_info "Check /var/lib/jenkins/secrets/initialAdminPassword in a moment."
    fi
}

# Display system information
display_info() {
    echo ""
    echo "=========================================="
    echo "Jenkins Installation Complete!"
    echo "=========================================="
    echo ""
    echo "Service Status:"
    systemctl status jenkins --no-pager | head -5
    echo ""
    echo "Jenkins Home Directory: /var/lib/jenkins"
    echo "Jenkins Logs: /var/log/jenkins/jenkins.log"
    echo ""
    echo "Useful Commands:"
    echo "  - Check status: sudo systemctl status jenkins"
    echo "  - Stop Jenkins: sudo systemctl stop jenkins"
    echo "  - Start Jenkins: sudo systemctl start jenkins"
    echo "  - Restart Jenkins: sudo systemctl restart jenkins"
    echo "  - View logs: sudo journalctl -u jenkins -f"
    echo ""
    echo "Next Steps:"
    echo "1. Open your browser and navigate to http://localhost:8080"
    echo "2. Enter the initial admin password shown above"
    echo "3. Install suggested plugins"
    echo "4. Create your first admin user"
    echo ""
}

# Main installation flow
main() {
    echo ""
    echo "=========================================="
    echo "Jenkins Installation Script for Kali Linux"
    echo "=========================================="
    echo ""
    
    check_root
    check_system
    update_system
    install_java
    add_jenkins_repo
    install_jenkins
    configure_firewall
    start_jenkins
    get_admin_password
    display_info
    
    log_success "Installation completed successfully!"
}

# Run main function
main
```

---

**Jenkins Default Directory: /var/lib/jenkins**

---

## Module 2: Create Your First Simple Job

Start with a very simple **Freestyle Job** that prints:

```bash
echo "Welcome to Jenkins class"
```

### Steps Inside Jenkins:

#### 1. Open Jenkins

Go to:

```text
http://YOUR_PUBLIC_IP:8080
```

Unlock Jenkins and install the suggested plugins if you haven't already.

---

#### 2. Create a New Job

On the Jenkins dashboard:

1. Click **New Item**
2. Enter job name: `first-jenkins-job`
3. Select **Freestyle project**
4. Click **OK**

---

#### 3. Add Build Step

1. Scroll down to **Build Steps**
2. Click **Add build step**
3. Choose **Execute shell**
4. Paste:

```bash
#!/bin/bash

echo "Welcome to Jenkins class"
```

5. Click **Save**

---

#### 4. Run the Job

Click **Build Now**

You should see **#1** appear on the left side.

---

#### 5. View Output

1. Click **Build #1**
2. Click **Console Output**

Expected output:

```text
Started by user admin

Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/first-jenkins-job

+ echo Welcome to Jenkins class
Welcome to Jenkins class

Finished: SUCCESS
```

---

## Module 3: Parameterized Builds

Create a **Parameterized Jenkins Job** where Jenkins asks for input and prints:

```text
FirstName: Rohan
LastName: Mandal
```

### Steps:

#### 1. Create a New Job

1. Dashboard → **New Item**
2. Name: `parameter-demo`
3. Choose **Freestyle project**
4. Click **OK**

---

#### 2. Enable Parameters

1. Under **General**, check **This project is parameterized**
2. Click **Add Parameter**
3. Choose **String Parameter**

**Parameter #1:**

```text
Name: FirstName
Default Value: Rohan
Description: Enter First Name
```

4. Add another **String Parameter**

**Parameter #2:**

```text
Name: LastName
Default Value: Mandal
Description: Enter Last Name
```

---

#### 3. Add Build Step

1. Go to **Build Steps**
2. Click **Add build step** → **Execute shell**
3. Paste:

```bash
#!/bin/bash

echo "FirstName: $FirstName"
echo "LastName: $LastName"

echo ""
echo "Welcome to Jenkins class"
echo "Full Name: $FirstName $LastName"
```

4. Click **Save**

---

#### 4. Run Job

Instead of **Build Now**, you'll now see **Build with Parameters**

1. Click **Build with Parameters**
2. You'll see input fields for FirstName and LastName
3. Keep defaults or change values
4. Click **Build**

---

#### 5. View Console Output

Expected output:

```text
Started by user admin

Running as SYSTEM

+ echo FirstName: Rohan
FirstName: Rohan

+ echo LastName: Mandal
LastName: Mandal

Welcome to Jenkins class

Full Name: Rohan Mandal

Finished: SUCCESS
```

---

## Module 4: Build Periodically (Cron Jobs)

Use **Build Periodically** to make Jenkins automatically execute jobs using a cron schedule.

Let's make your job run automatically every minute.

### Steps:

#### 1. Open Your Existing Job

1. Go to Dashboard
2. Click **parameter-demo**
3. Click **Configure**

---

#### 2. Enable Periodic Build

1. Scroll to **Build Triggers**
2. Check **Build periodically**
3. You will see a text box for schedule

---

#### 3. Add Schedule

**For every minute:**

```text
* * * * *
```

**Or Jenkins recommended:**

```text
H/1 * * * *
```

**Cron Format:**

```text
MINUTE HOUR DAY MONTH WEEKDAY
```

**Examples:**

- Run every 5 minutes: `H/5 * * * *`
- Run every hour: `H * * * *`
- Run daily at 9 AM: `0 9 * * *`

**Note:** `H` means Jenkins spreads jobs intelligently instead of launching everything at the exact same second.

**Resource:** https://crontab.guru/

**Important:** Check your Kali Linux date and time zone with the `date` command to ensure timing is correct.

---

#### 4. Update Your Shell Build Step

```bash
#!/bin/bash

echo "Welcome to Jenkins class"
echo "Full Name: $FirstName $LastName"
echo "Build Time: $(date)"
```

5. Click **Save**

---

#### 5. Watch Automatic Builds

1. Go back to Dashboard → **parameter-demo**
2. After one minute you should see builds appearing automatically:

```text
#1
#2
#3
...
```

---

#### 6. Verify

1. Open **Build #2**
2. Click **Console Output**

Example output:

```text
Started by timer

Welcome to Jenkins class
Full Name: Rohan Mandal
Build Time: Wed May 21 19:35:01 UTC 2026

Finished: SUCCESS
```

**Notice:** `Started by timer` instead of `Started by user admin`

This confirms Jenkins triggered it automatically.

---

## Module 5: Git Integration with Jenkins

Jenkins pulls code from GitHub and builds it.

**Repository:** [simple-admin-template](https://github.com/rohanmandal798/simple-admin-template)

### Steps:

#### 1. Create a New Job

1. Dashboard → **New Item**
2. Name: `github-build-demo`
3. Choose **Freestyle project**
4. Click **OK**

---

#### 2. Connect Git Repository

1. Scroll to **Source Code Management**
2. Choose **Git**
3. Repository URL:

```text
https://github.com/rohanmandal798/simple-admin-template.git
```

**If Jenkins says Git is missing, install it on the server:**

```bash
sudo apt install git -y
```

---

#### 3. Add a Build Step

1. Go to **Build Steps**
2. Click **Add build step** → **Execute shell**
3. Paste:

```bash
#!/bin/bash

echo "Welcome to Jenkins class"

echo "Current directory:"
pwd

echo ""
echo "Listing project files:"
ls -la

echo ""
echo "Git commit:"
git log --oneline -1

echo ""
echo "Build time:"
date
```

4. Click **Save**

---

#### 4. Build and View Output

1. Click **Build Now**
2. Open **Build #1** → **Console Output**

Expected flow:

```text
Started by user admin

Cloning the remote Git repository

Fetching upstream changes from:
https://github.com/rohanmandal798/simple-admin-template.git

Checking out Revision ...

Welcome to Jenkins class

Current directory:
/var/lib/jenkins/workspace/github-build-demo

Listing project files:
index.html
assets/
css/
js/

Git commit:
abc123 initial commit

Build time:
Wed May 21 ...

Finished: SUCCESS
```

---

**Workflow:**

```text
GitHub Repo
     ↓
Jenkins clones repository
     ↓
Workspace created
     ↓
Build executes
     ↓
Output generated
```

**Next Step:** Build the HTML template and deploy it automatically to `/var/www/html` for a complete CI/CD pipeline.

---

## Module 6: Maven Build Job

Building a **Java + Maven CI build pipeline**.

Maven projects use a `pom.xml` file to define dependencies and build steps. Jenkins commonly builds Maven projects using commands like `mvn clean package`.

**Repository:** [java-project-maven-new](https://github.com/rohanmandal798/java-project-maven-new)

### Step 1: Install Git + Maven on EC2

SSH into your Jenkins server:

```bash
sudo apt update
sudo apt install git maven -y
```

Verify installation:

```bash
git --version
mvn --version
```

Expected output:

```text
Apache Maven 3.x
Java version: 21
```

---

### Step 2: Create Jenkins Job

1. Dashboard → **New Item**
2. Name: `java-maven-build`
3. Choose **Freestyle project**
4. Click **OK**

---

### Step 3: Connect GitHub

1. Go to **Source Code Management**
2. Select **Git**
3. Repository URL:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

Leave credentials empty (public repo).

---

### Step 4: Add Build Step

1. Scroll to **Build Steps**
2. Click **Add build step** → **Execute shell**
3. Paste:

```bash
mvn clean package
```

**What it does:**

```text
clean   → Remove old build files
package → Compile + test + create JAR
```

Maven generates artifacts in the `target/` directory.

4. Click **Save**

---

### Step 5: Build and Verify

1. Click **Build Now**
2. Open **Build #1** → **Console Output**

Expected output:

```text
Started by user admin

Cloning repository...

Welcome to Jenkins class

Current workspace:
/var/lib/jenkins/workspace/java-maven-build

Project files:
pom.xml
src/

Building Maven project...

[INFO] BUILD SUCCESS

Generated files:
target/
project-name-1.0.jar

Finished: SUCCESS
```

---

### Step 6: Verify Artifact

SSH into EC2:

```bash
cd /var/lib/jenkins/workspace/java-maven-build
ls target
```

You should see a generated `.jar` file.

---

**Workflow:**

```text
GitHub
   ↓
Jenkins
   ↓
Git Clone
   ↓
Maven Build
   ↓
JAR Artifact
```

---

## Module 7: Maven Build Job with Invoke Top-Level Maven Targets

For the repository:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

Using **Invoke top-level Maven targets** is cleaner than `Execute shell`.

### Steps:

#### 1. Create Job

1. Dashboard → **New Item**
2. Name: `java-maven-build-v2`
3. Choose **Freestyle project**
4. Click **OK**

---

#### 2. Configure Git Repository

1. Go to **Source Code Management**
2. Select **Git**
3. Repository URL:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

Leave credentials empty (public repo).

---

#### 3. Configure Maven in Jenkins

1. Go to **Manage Jenkins** → **Tools**
2. Find **Maven installations**
3. Click **Add Maven**
4. Name: `Maven-3`
5. Check **Install automatically**
6. Choose the latest Maven version
7. Click **Save**

This tells Jenkins where Maven exists.

---

#### 4. Add Maven Build Step

1. Go back to your job → **Configure**
2. Scroll to **Build**
3. Click **Add build step**
4. Choose **Invoke top-level Maven targets**

**Fill in:**

- **Maven Version:** `Maven-3`
- **Goals:** `clean package`
- Leave other fields empty

**What it does:**

```text
clean   → Remove old files
package → Compile + create JAR
```

5. Click **Save**

---

#### 5. Build and Verify

1. Click **Build Now**
2. Open **Build #1** → **Console Output**

Expected output:

```text
Started by user admin

Cloning repository...

Parsing POMs

Executing Maven:
mvn clean package

[INFO] Compiling...

[INFO] BUILD SUCCESS

Finished: SUCCESS
```

---

#### 6. Check Generated Artifact

SSH into EC2:

```bash
cd /var/lib/jenkins/workspace/java-maven-build-v2
ls target
```

You should see:

```text
java-project-maven-new-1.0.jar
```

---

### Difference Between Execute Shell vs Maven Build Step:

**Execute Shell:**

```bash
mvn clean package
```

You manually type commands.

**Invoke Top-Level Maven Targets:**

```text
Goals: clean package
```

Jenkins handles Maven internally with better integration.

---

**For Java projects, many teams prefer the Maven build step because it integrates better with Jenkins tooling.**
