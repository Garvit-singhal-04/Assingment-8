# Assignment 1 — Jenkins and Git Automation


---

# Objective

The objective of this assignment is to use Jenkins to automate Git operations and a simple file publishing workflow.

The assignment is divided into two parts.

### Part 1

Create a Jenkins job that can perform the following Git operations:

* Create a branch
* List all branches
* Merge one branch with another
* Rebase one branch with another
* Delete a branch
* Send Slack and Email notifications if any operation fails

### Part 2

Create two Jenkins jobs.

**Job 1** should:

* Accept a `Ninja Name` as a parameter
* Create a file
* Add the following content to the file:

```text
<Ninja Name> from DevOps Ninja
```

**Job 2** should:

* Retrieve the file created by Job 1
* Publish the file using a web server

The second job must automatically execute only when the first job completes successfully.

If the workflow succeeds, Slack and Email notifications should be sent.

If any step fails, Slack and Email notifications should also be sent.

---

# 2. Prerequisites

The following components were used for this assignment:

* Jenkins
* Git
* Linux/Ubuntu system
* Git repository
* Python 3 or Apache/Nginx web server
* Slack workspace/channel
* Email/SMTP configuration

Verify Git:

```bash
git --version
```

Verify Python:

```bash
python3 --version
```

Verify Jenkins is running:

```bash
sudo systemctl status jenkins
```

---

# 3. Part 1 — Git Operations Using Jenkins

## 3.1 Create the Jenkins Job

A Freestyle Jenkins project named:

```text
Git-Operations
```

was created.

### Steps

1. Open Jenkins.
2. Click **New Item**.
3. Enter the job name:

```text
Git-Operations
```

4. Select **Freestyle project**.
5. Click **OK**.

### Screenshot

> 📸 **SCREENSHOT 1 — Git-Operations Jenkins Job Creation**
>
> Insert screenshot here showing the creation of the `Git-Operations` Freestyle project.

---

## 3.2 Configure the Git Repository

The build step operates on a local Git repository.

The repository location used in the shell script is:

```bash
cd /home/garvit/git-demo
```

Replace the path with the actual Git repository location on the Jenkins machine.

The repository should contain a valid Git configuration.

Check it using:

```bash
git status
```

and:

```bash
git branch
```

### Screenshot

> 📸 **SCREENSHOT 2 — Git Repository**
>
> Insert screenshot showing the Git repository and existing branches.

---

# 3.3 Create a Branch

The following command creates a new branch:

```bash
git checkout -b devops-ninja
```

This creates the `devops-ninja` branch and switches to it.

The Jenkins shell script contains:

```bash
git checkout main
git checkout -b devops-ninja
```

### Screenshot

> 📸 **SCREENSHOT 3 — Branch Creation**
>
> Insert screenshot showing Jenkins console output where the `devops-ninja` branch is created successfully.

---

# 3.4 List All Branches

The following command lists the available branches:

```bash
git branch -a
```

This allows us to verify that the newly created branch exists.

### Screenshot

> 📸 **SCREENSHOT 4 — Branch Listing**
>
> Insert screenshot showing the Jenkins console output containing the available Git branches.

---

# 3.5 Merge One Branch With Another

The branch is merged into the `main` branch using:

```bash
git checkout main
git merge devops-ninja
```

The first command switches to the target branch.

The second command merges the changes from `devops-ninja` into `main`.

### Screenshot

> 📸 **SCREENSHOT 5 — Git Merge**
>
> Insert screenshot showing the successful merge operation in the Jenkins console.

---

# 3.6 Rebase One Branch With Another

A separate branch is created for demonstrating rebase:

```bash
git checkout -b rebase-test
```

A change is then committed:

```bash
echo "Rebase test content" >> rebase.txt
git add rebase.txt
git commit -m "Added rebase test file"
```

The branch is rebased using:

```bash
git checkout rebase-test
git rebase main
```

Rebase moves the branch's commits on top of the latest commit of the target branch.

### Screenshot

> 📸 **SCREENSHOT 6 — Git Rebase**
>
> Insert screenshot showing the successful rebase operation.

---

# 3.7 Delete a Branch

After the operations are completed, the temporary branches are deleted.

```bash
git checkout main
git branch -D rebase-test
git branch -D devops-ninja
```

The `-D` option forcefully deletes the local branch.

The final branch list can be checked with:

```bash
git branch
```

### Screenshot

> 📸 **SCREENSHOT 7 — Branch Deletion**
>
> Insert screenshot showing the branches being deleted and the final branch list.

---

# 3.8 Failure Handling

The shell script uses:

```bash
set -e
```

This causes the shell script to stop when a command returns a non-zero exit status.

For example, if a Git merge fails because of a conflict, Jenkins marks the build as failed.

This allows the configured Slack and Email notification mechanisms to detect the failure and send notifications.

### Screenshot

> 📸 **SCREENSHOT 8 — Failed Jenkins Build**
>
> Insert screenshot showing a Jenkins build marked as **FAILED**.

---

# 4. Part 2 — File Creation and Web Publishing

Part 2 consists of two Jenkins jobs:

```text
Ninja-File-Creation
          |
          | SUCCESS
          ↓
Ninja-Web-Publish
```

---

# 4.1 Job 1 — Ninja-File-Creation

A Freestyle project named:

```text
Ninja-File-Creation
```

was created.

### Steps

1. Click **New Item**.
2. Enter:

```text
Ninja-File-Creation
```

3. Select **Freestyle project**.
4. Click **OK**.

### Screenshot

> 📸 **SCREENSHOT 9 — Ninja-File-Creation Job**
>
> Insert screenshot showing the Job 1 configuration page.

---

# 4.2 Create the Ninja Name Parameter

The job was configured as a parameterized project.

Navigate to:

```text
General → This project is parameterized
```

Add:

```text
String Parameter
```

Parameter name:

```text
NINJA_NAME
```

Example value:

```text
Naruto
```

The parameter allows the user to enter different Ninja names whenever the Jenkins job is executed.

### Screenshot

> 📸 **SCREENSHOT 10 — Ninja Name Parameter**
>
> Insert screenshot showing the `NINJA_NAME` String Parameter configuration.

---

# 4.3 Create the File

An **Execute shell** build step was added.

The script is:

```bash
#!/bin/bash

set -e

echo "===== Ninja File Creation Started ====="

echo "Ninja Name: $NINJA_NAME"

mkdir -p "$WORKSPACE/output"

echo "$NINJA_NAME from DevOps Ninja" > "$WORKSPACE/output/ninja.txt"

echo "===== File Created ====="

cat "$WORKSPACE/output/ninja.txt"

echo "===== File Content ====="

echo "===== Job Completed Successfully ====="
```

The Jenkins parameter is accessed using:

```bash
$NINJA_NAME
```

For example, if the user enters:

```text
Naruto
```

the resulting file contains:

```text
Naruto from DevOps Ninja
```

### Screenshot

> 📸 **SCREENSHOT 11 — File Creation Console Output**
>
> Insert screenshot showing the Jenkins console output with the Ninja name and generated file content.

---

# 4.4 Archive the Generated File

The generated file is archived as a Jenkins build artifact.

Navigate to:

```text
Post-build Actions → Archive the artifacts
```

Enter:

```text
output/ninja.txt
```

This allows the file to be retrieved by the second Jenkins job.

### Screenshot

> 📸 **SCREENSHOT 12 — Artifact Configuration**
>
> Insert screenshot showing `output/ninja.txt` configured under **Archive the artifacts**.

---

# 4.5 Verify the Artifact

After a successful build, open the Jenkins build page.

Navigate to:

```text
Build → Artifacts
```

The following file should be available:

```text
output/ninja.txt
```

The file should contain:

```text
Naruto from DevOps Ninja
```

if `Naruto` was used as the input.

### Screenshot

> 📸 **SCREENSHOT 13 — Archived Artifact**
>
> Insert screenshot showing the `ninja.txt` artifact in Jenkins.

---

# 4.6 Job 2 — Ninja-Web-Publish

A second Freestyle project named:

```text
Ninja-Web-Publish
```

was created.

This job is responsible for retrieving the file generated by Job 1 and publishing it through a web server.

### Screenshot

> 📸 **SCREENSHOT 14 — Ninja-Web-Publish Job**
>
> Insert screenshot showing the Job 2 configuration page.

---

# 4.7 Configure Automatic Trigger

Job 2 must run automatically only after Job 1 succeeds.

In Job 2, navigate to:

```text
Build Triggers
```

Select:

```text
Build after other projects are built
```

Enter:

```text
Ninja-File-Creation
```

Select:

```text
Trigger only if build is stable
```

This creates the following dependency:

```text
Ninja-File-Creation
        |
        | SUCCESS
        ↓
Ninja-Web-Publish
```

If Job 1 fails:

```text
Ninja-File-Creation
        |
        | FAILURE
        ↓
Ninja-Web-Publish
       X
```

Therefore, Job 2 does not execute when Job 1 fails.

### Screenshot

> 📸 **SCREENSHOT 15 — Job Dependency**
>
> Insert screenshot showing **Build after other projects are built** and `Ninja-File-Creation`.

---

# 4.8 Copy the Artifact

Job 2 needs the file generated by Job 1.

The **Copy Artifact** build step is used to retrieve the archived file.

Configure:

```text
Project Name:
Ninja-File-Creation
```

Build selector:

```text
Last successful build
```

Artifact:

```text
output/ninja.txt
```

This copies the generated file into the workspace of Job 2.

### Screenshot

> 📸 **SCREENSHOT 16 — Copy Artifact Configuration**
>
> Insert screenshot showing the Copy Artifact configuration.

---

# 4.9 Publish the File Using a Web Server

For a simple demonstration, Python's built-in HTTP server can be used.

Example Jenkins shell script:

```bash
#!/bin/bash

set -e

echo "===== Web Publishing Started ====="

mkdir -p "$WORKSPACE/web"

cp "$WORKSPACE/output/ninja.txt" "$WORKSPACE/web/ninja.txt"

echo "===== File Content ====="

cat "$WORKSPACE/web/ninja.txt"

echo "===== Starting Web Server ====="

cd "$WORKSPACE/web"

nohup python3 -m http.server 8080 > "$WORKSPACE/webserver.log" 2>&1 &

echo $! > "$WORKSPACE/webserver.pid"

echo "Web server started on port 8080"

echo "===== Publishing Completed Successfully ====="
```

The file can then be accessed through:

```text
http://<server-ip>:8080/ninja.txt
```

For example:

```text
http://192.168.1.10:8080/ninja.txt
```

The exact IP address depends on the Jenkins/server machine.

### Screenshot

> 📸 **SCREENSHOT 17 — Web Server Console Output**
>
> Insert screenshot showing the web server starting successfully.

---

# 4.10 Verify the Published File

Open the file URL in a browser:

```text
http://<server-ip>:8080/ninja.txt
```

If the input was:

```text
Naruto
```

the browser should display:

```text
Naruto from DevOps Ninja
```

### Screenshot

> 📸 **SCREENSHOT 18 — Published File in Browser**
>
> Insert screenshot showing the `ninja.txt` file opened through the web server.

---

# 5. Notifications

Slack and Email notifications were configured for the Jenkins jobs.

Notifications are required for both:

### Successful Build

```text
Jenkins Build
      |
      ↓
 SUCCESS
   /   \
  ↓     ↓
Slack  Email
```

### Failed Build

```text
Jenkins Build
      |
      ↓
 FAILURE
   /   \
  ↓     ↓
Slack  Email
```

---

# 5.1 Slack Notification

Slack notification configuration can be added through the Jenkins Slack Notification plugin.

The Jenkins system configuration contains the Slack workspace/channel details.

The jobs are configured to send notifications for:

* Successful builds
* Failed builds

### Screenshot

> 📸 **SCREENSHOT 19 — Slack Configuration**
>
> Insert screenshot showing the Slack notification configuration in Jenkins.

### Screenshot

> 📸 **SCREENSHOT 20 — Slack Success Notification**
>
> Insert screenshot showing the successful Jenkins build notification in Slack.

### Screenshot

> 📸 **SCREENSHOT 21 — Slack Failure Notification**
>
> Insert screenshot showing the failed Jenkins build notification in Slack.

---

# 5.2 Email Notification

Email notification was configured using Jenkins' email notification settings.

The SMTP configuration depends on the email provider being used.

The Jenkins jobs are configured to send email notifications when:

* The build succeeds
* The build fails

### Screenshot

> 📸 **SCREENSHOT 22 — Email Configuration**
>
> Insert screenshot showing the Jenkins email/SMTP configuration.

### Screenshot

> 📸 **SCREENSHOT 23 — Email Success Notification**
>
> Insert screenshot showing the successful build email.

### Screenshot

> 📸 **SCREENSHOT 24 — Email Failure Notification**
>
> Insert screenshot showing the failure notification email.

---

# 6. Testing

The complete workflow was tested using different scenarios.

## Test Case 1 — Part 1 Git Operations

The Jenkins job was executed to verify:

```text
Create Branch
      ↓
List Branches
      ↓
Merge Branch
      ↓
Rebase Branch
      ↓
Delete Branch
```

Expected result:

```text
BUILD SUCCESS
```

---

## Test Case 2 — Ninja File Creation

Input:

```text
NINJA_NAME = Naruto
```

Expected file:

```text
ninja.txt
```

Expected content:

```text
Naruto from DevOps Ninja
```

Expected result:

```text
BUILD SUCCESS
```

---

## Test Case 3 — Automatic Job Trigger

After Job 1 completes successfully:

```text
Ninja-File-Creation
        |
        ↓
     SUCCESS
        |
        ↓
Ninja-Web-Publish
```

Job 2 should automatically start.

Expected result:

```text
Job 1 → SUCCESS
Job 2 → SUCCESS
```

---

## Test Case 4 — Job 1 Failure

If Job 1 fails, Job 2 should not execute.

Expected:

```text
Ninja-File-Creation
        |
        ↓
      FAILED
        |
        X
Ninja-Web-Publish
```

Slack and Email notifications should be sent for the failure.

---

## Test Case 5 — Job 2 Failure

If the publishing process fails, Job 2 should become:

```text
BUILD FAILED
```

Slack and Email notifications should be sent.

---

# 7. Expected Workflow

The complete assignment workflow is:

```text
                    PART 1
                       |
                       ↓
              Git-Operations Job
                       |
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
     Create          Merge          Rebase
     Branch          Branch          Branch
        |              |              |
        └──────────────┼──────────────┘
                       ↓
                 Delete Branch
                       |
                       ↓
                 Jenkins Result
                  /          \
              SUCCESS       FAILURE
                 |             |
                 ↓             ↓
              Slack         Slack
              Email         Email


                    PART 2

             Ninja-File-Creation
                     |
              NINJA_NAME input
                     |
                     ↓
              Create ninja.txt
                     |
                     ↓
              Archive artifact
                     |
                     ↓
                  SUCCESS
                     |
                     ↓
             Ninja-Web-Publish
                     |
                     ↓
              Copy artifact
                     |
                     ↓
                Web Server
                     |
                     ↓
              Browser / URL
                     |
                     ↓
            Display file content
```

---

# 8. Screenshots Checklist

The following screenshots should be included in the final assignment report.

| No. | Screenshot                      | Status |
| --- | ------------------------------- | ------ |
| 1   | Git-Operations job creation     | ☐      |
| 2   | Git repository/branches         | ☐      |
| 3   | Branch creation                 | ☐      |
| 4   | Branch listing                  | ☐      |
| 5   | Merge operation                 | ☐      |
| 6   | Rebase operation                | ☐      |
| 7   | Branch deletion                 | ☐      |
| 8   | Failed Jenkins build            | ☐      |
| 9   | Ninja-File-Creation job         | ☐      |
| 10  | NINJA_NAME parameter            | ☐      |
| 11  | File creation console output    | ☐      |
| 12  | Artifact configuration          | ☐      |
| 13  | Archived artifact               | ☐      |
| 14  | Ninja-Web-Publish job           | ☐      |
| 15  | Automatic trigger configuration | ☐      |
| 16  | Copy Artifact configuration     | ☐      |
| 17  | Web server console output       | ☐      |
| 18  | Published file in browser       | ☐      |
| 19  | Slack configuration             | ☐      |
| 20  | Slack success notification      | ☐      |
| 21  | Slack failure notification      | ☐      |
| 22  | Email configuration             | ☐      |
| 23  | Email success notification      | ☐      |
| 24  | Email failure notification      | ☐      |

---

# 9. Conclusion

This assignment demonstrates how Jenkins can be used to automate Git operations and create a simple CI/CD-style workflow.

In Part 1, Jenkins was used to automate common Git operations including branch creation, branch listing, merging, rebasing, and branch deletion.

In Part 2, a parameterized Jenkins job was created to generate a file containing the Ninja name. The generated file was archived and passed to a second Jenkins job. The second job was configured to execute automatically only after the first job completed successfully and then publish the file through a web server.

Slack and Email notifications were also configured to provide feedback about successful and failed builds.

Overall, the assignment demonstrates Jenkins job configuration, build automation, parameterized builds, artifact management, job dependencies, Git automation, web publishing, and build notifications.
