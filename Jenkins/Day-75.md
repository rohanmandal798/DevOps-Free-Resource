# Course Cover Today
### Module 1: Jenkins Installation & Setup

### Module 2: First Jenkins Job

### Module 3: Parameterized Builds

### Module 4: Build Periodically (Cron Jobs)

### Module 5: Git Integration with Jenkins

### Module 6: Maven Build Job
### Module 7: Maven Build Job with **Invoke top-level Maven targets**


# Automated Script to install jenkins

**NOTE : check security group if not working. allow port 8080**

### Cleanup Old jenkins if getting error on installing
Removes the old Jenkins repository configuration, old GPG keys and update the error
### Copy the script to your Kali machine

1. **nano install_jenkins.sh**
2. **copy and paste**
3. **chmod +x clean_jenkins.sh**
4. **sudo ./clean_jenkins.sh**

```
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

## Copy the script to your Kali machine

1. **nano install_jenkins.sh**
2. **copy and paste**
3. **chmod +x install_jenkins.sh**
4. **sudo ./install_jenkins.sh**

```
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



**Jenkins default Directory: /var/lib/jenkins**


## Create a simple job and run shell command

Great. Start with a very simple **Freestyle Job** that prints:

```bash
echo "welcome to jenkins class"
```

Steps inside Jenkins:

### 1. Open Jenkins

Go to:

```text
http://YOUR_PUBLIC_IP:8080
```

Unlock Jenkins and install the suggested plugins if you haven't already.

---

### 2. Create a new job

On the Jenkins dashboard:

```text
New Item
```

Enter:

```text
first-jenkins-job
```

Select:

```text
Freestyle project
```

Click:

```text
OK
```

---

### 3. Add build step

Scroll down to:

```text
Build Steps
```

Click:

```text
Add build step
```

Choose:

```text
Execute shell
```

Paste:

```bash
#!/bin/bash

echo "welcome to jenkins class"
```

Click:

```text
Save
```

---

### 4. Run the job

Click:

```text
Build Now
```

You should see:

```text
#1
```

appear on the left.

---

### 5. View output

Click:

```text
Build #1
→ Console Output
```

Expected:

```text
Started by user admin

Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/first-jenkins-job

+ echo welcome to jenkins class
welcome to jenkins class

Finished: SUCCESS
```


![[Pasted image 20260521021051.png]]

---

## create a **Parameterized Jenkins Job** so Jenkins asks for input and prints:

```text
FirstName: Rohan
LastName: Mandal
```

### 1. Create a new job

Dashboard → **New Item**

Name:

```text
parameter-demo
```

Choose:

```text
Freestyle project
```

Click **OK**

---

### 2. Enable parameters

Under **General**, check:

```text
This project is parameterized
```

Click:

```text
Add Parameter
```

Choose:

```text
String Parameter
```

Create parameter #1:

```text
Name: FirstName
Default Value: Rohan
Description: Enter First Name
```

Add another **String Parameter**

Parameter #2:

```text
Name: LastName
Default Value: Mandal
Description: Enter Last Name
```

---

### 3. Add build step

Go to:

```text
Build Steps
→ Add build step
→ Execute shell
```

Paste:

```bash
#!/bin/bash

echo "FirstName: $FirstName"
echo "LastName: $LastName"

echo ""
echo "welcome to jenkins class"
echo "Full Name: $FirstName $LastName"
```

Save.

---

### 4. Run job

Instead of **Build Now**, you'll now see:

```text
Build with Parameters
```

Click it.

You’ll see:

```text
FirstName: Rohan
LastName: Mandal
```

Change values or keep defaults:

```text
FirstName: Rohan
LastName: Mandal
```

Click:

```text
Build
```

---

### 5. View Console Output

Expected output:

```text
Started by user admin

Running as SYSTEM

+ echo FirstName: Rohan
FirstName: Rohan

+ echo LastName: Mandal
LastName: Mandal

welcome to jenkins class

Full Name: Rohan Mandal

Finished: SUCCESS
```

---

## **Build periodically**, which makes Jenkins automatically execute jobs using a cron schedule.

Let's make your job run automatically and print:

```text
welcome to jenkins class
Full Name: Rohan Mandal
```

every minute.

### 1. Open your existing job

Go:

```text
Dashboard
→ parameter-demo
→ Configure
```

---

### 2. Enable periodic build

Scroll to:

```text
Build Triggers
```

Check:

```text
Build periodically
```

You will see a text box.

---

### 3. Add schedule

https://crontab.guru/

For every minute:

```text
* * * * *
```

or Jenkins recommended:

```text
H/1 * * * *
```

Explanation:

```text
MINUTE HOUR DAY MONTH WEEKDAY
```

Examples:

Run every 5 minutes:

```text
H/5 * * * *
```

Run every hour:

```text
H * * * *
```

Run daily at 9 AM:

```text
0 9 * * *
```

`H` means Jenkins spreads jobs intelligently instead of launching everything at the exact same second.

NOTE : check kali linux date with date command, and UTC time zone.

---

### 4. Keep your shell build step

```bash
#!/bin/bash

echo "welcome to jenkins class"
echo "Full Name: $FirstName $LastName"
echo "Build Time: $(date)"
```

Save.

---

### 5. Watch automatic builds

Go back:

```text
Dashboard
→ parameter-demo
```

After one minute you should see:

```text
#1
#2
#3
...
```

appearing automatically.

---

### 6. Verify

Open:

```text
Build #2
→ Console Output
```

Example:

```text
Started by timer

welcome to jenkins class
Full Name: Rohan Mandal
Build Time: Wed May 20 19:35:01 UTC

Finished: SUCCESS
```

Notice:

```text
Started by timer
```

instead of:

```text
Started by user admin
```

That confirms Jenkins triggered it automatically.

You just used one of the most important CI concepts: **scheduled automation**. Next useful steps: GitHub webhook triggers, pipelines (`Jenkinsfile`), and Docker builds.

---

## Jenkins pulls code from GitHub and builds it. 

Repository: 
[simple-admin-template repository](https://github.com/rohanmandal798/simple-admin-template?utm_source=chatgpt.com)

Let's create a Git-based Jenkins job.

### 1. Create a new job

Dashboard →

```text
New Item
```

Name:

```text
github-build-demo
```

Choose:

```text
Freestyle project
```

Click **OK**

---

### 2. Connect Git repository

Scroll to:

```text
Source Code Management
```

Choose:

```text
Git
```

Repository URL:

```text
https://github.com/rohanmandal798/simple-admin-template.git
```

If Jenkins says Git is missing, install it on the server:

```bash
sudo apt install git -y
```

---

### 3. Add a build step

Go:

```text
Build Steps
→ Add build step
→ Execute shell
```

Paste:

```bash
#!/bin/bash

echo "welcome to jenkins class"

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

---

### 4. Save and build

Click:

```text
Build Now
```

Open:

```text
Build #1
→ Console Output
```

Expected flow:

```text
Started by user admin

Cloning the remote Git repository

Fetching upstream changes from:
https://github.com/rohanmandal798/simple-admin-template.git

Checking out Revision ...

welcome to jenkins class

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
Wed May ...

Finished: SUCCESS
```

This teaches an important Jenkins concept:

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

After this, a good next exercise for your DevOps portfolio is: build the HTML template and deploy it automatically to `/var/www/html` so Jenkins acts like a mini CI/CD pipeline.

---


Nice choice. You're now moving from simple shell jobs into a real **Java + Maven CI build pipeline**. Maven projects use a `pom.xml` file to define dependencies and build steps. Jenkins commonly builds Maven projects using commands like `mvn clean package`. ([GitHub](https://github.com/apache/maven?utm_source=chatgpt.com "Apache Maven core"))

Your repository:

[java-project-maven-new repository](https://github.com/rohanmandal798/java-project-maven-new?utm_source=chatgpt.com)

Let's build it in Jenkins step by step.

## Step 1: Install Git + Maven on EC2

SSH into your Jenkins server:

```bash
sudo apt update
sudo apt install git maven -y
```

Verify:

```bash
git --version
mvn --version
```

Expected:

```text
Apache Maven 3.x
Java version: 21
```

Maven requires Java and uses the `pom.xml` build file. ([GitHub](https://github.com/apache/maven?utm_source=chatgpt.com "Apache Maven core"))

---

## Step 2: Create Jenkins Job

Dashboard →

```text
New Item
```

Name:

```text
java-maven-build
```

Choose:

```text
Freestyle project
```

Click:

```text
OK
```

---

## Step 3: Connect GitHub

Go to:

```text
Source Code Management
```

Select:

```text
Git
```

Repository URL:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

Leave credentials empty because the repo is public.

---

## Step 4: Add Build Step

Scroll:

```text
Build Steps
→ Add build step
→ Execute shell
```

Paste:

```
mvn clean package
```

Important:

```bash
mvn clean package
```

means:

```text
clean    → remove old build files
package  → compile + test + create JAR
```

Maven commonly generates artifacts in:

```text
target/
```

([GitHub](https://github.com/Shubh2-0/Maven-Projects?utm_source=chatgpt.com "Shubh2-0/Maven-Projects"))

---

## Step 5: Save and Build

Click:

```text
Build Now
```

Open:

```text
Build #1
→ Console Output
```

Expected:

```text
Started by user admin

Cloning repository...

welcome to jenkins class

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

## Step 6: Verify Artifact

SSH into EC2:

```bash
cd /var/lib/jenkins/workspace/java-maven-build
ls target
```

You should see a generated `.jar` file.

You just completed:

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

This is the classic beginner CI pipeline used in many DevOps tutorials. Jenkins itself uses Maven-based workflows heavily in examples. ([GitHub](https://github.com/jenkins-docs/simple-java-maven-app?utm_source=chatgpt.com "jenkins-docs/simple-java-maven-app"))

After this, a good next step is:

```text
GitHub → Jenkins → Maven → Docker → Deploy
```

That starts resembling real-world DevOps pipelines.

---





For your repository:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

using **Invoke top-level Maven targets** is actually cleaner than `Execute shell`.

Follow these steps:

### 1. Create Job

Dashboard →

```text
New Item
```

Name:

```text
java-maven-build
```

Choose:

```text
Freestyle project
```

Click **OK**

---

### 2. Configure Git Repository

Go to:

```text
Source Code Management
```

Select:

```text
Git
```

Repository URL:

```text
https://github.com/rohanmandal798/java-project-maven-new.git
```

Leave credentials empty (public repo).

---

### 3. Configure Maven in Jenkins

Go:

```text
Manage Jenkins
→ Tools
```

Find:

```text
Maven installations
```

Click:

```text
Add Maven
```

Name:

```text
Maven-3
```

Check:

```text
Install automatically
```

Choose latest Maven version.

Save.

This tells Jenkins where Maven exists.

---

### 4. Add Maven Build Step

Back to your job:

```text
Configure
→ Build
→ Add build step
```

Choose:

```text
Invoke top-level Maven targets
```

You will see fields like:

```text
Maven Version:
Goals:
POM:
Options:
```

Fill:

**Maven Version**

```text
Maven-3
```

**Goals**

```text
clean package
```

Leave others empty.

Meaning:

```text
clean   → remove old files
package → compile + create JAR
```

---

### 5. Save and Build

Click:

```text
Build Now
```

Open:

```text
Build #1
→ Console Output
```

Expected:

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

The generated artifact should appear here:

```bash
/var/lib/jenkins/workspace/java-maven-build/target
```

Check manually:

```bash
cd /var/lib/jenkins/workspace/java-maven-build
ls target
```

You should see:

```text
java-project-maven-new-1.0.jar
```

Difference between shell vs Maven build step:

**Execute shell**

```bash
mvn clean package
```

You manually type commands.

**Invoke top-level Maven targets**

```text
Goals: clean package
```

Jenkins handles Maven internally.

For Java projects, many teams prefer the Maven build step because it integrates better with Jenkins tooling.

---
