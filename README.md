# AWS CI/CD Pipeline — vProfile Application

A production-style CI/CD pipeline deploying a Java web application to **AWS Elastic Beanstalk**, automated end-to-end using **AWS CodePipeline** and **AWS CodeBuild**, with source code hosted on **Bitbucket**.

---

## Architecture Overview

``` 
                                                               AWS RDS (MySQL)
                                                                     |
Bitbucket (Source)   ──►   AWS CodeBuild (Build)   ──►   AWS Elastic Beanstalk (Deploy)
         └──────────────── AWS CodePipeline (Orchestration) ──────────────────┘                      
```

> **Pipeline Flow:** A push to the `aws-ci` branch on Bitbucket triggers CodePipeline, which kicks off a CodeBuild job to compile and package the app, then deploys the artifact to Elastic Beanstalk, which runs the app against an RDS MySQL database.

---

## Tech Stack

| Layer | Service / Tool |
|---|---|
| Source Control | Bitbucket |
| CI/CD Orchestration | AWS CodePipeline |
| Build | AWS CodeBuild (Corretto 17 + Maven 3.9.8) |
| Application Platform | AWS Elastic Beanstalk (Tomcat 11, Corretto 21) |
| Database | AWS RDS (MySQL 8.4.7) |
| Artifact Storage | AWS S3 |

---

## Setup Guide

### 1. Elastic Beanstalk

**Key Pair:** Create a dedicated EC2 key pair (`vpro-beankey`) before creating the environment.

**Application Config:**
- Platform: `Tomcat 11 Corretto 21`
- Domain: `vprofilee-production.us-east-1.elasticbeanstalk.com`

**IAM Roles:**

| Role | Attached Policies |
|---|---|
| `aws-elasticbeanstalk-service-role` (Service Role) | `AWSElasticBeanstalkEnhancedHealth`, `AWSElasticBeanstalkManagedUpdatesCustomerRolePolicy` |
| `vprofile-rearch-beanrole` (EC2 Instance Profile) | `AdministratorAccess-AWSElasticBeanstalk`, `AWSElasticBeanstalkCustomPlatformforEC2Role`, `AWSElasticBeanstalkRoleSNS`, `AWSElasticBeanstalkWebTier` |

**Networking:**
- VPC: Default (custom VPC recommended for production)
- Public IP enabled across all subnets

**Capacity & Scaling:**
- Type: Load Balanced
- Instances: Min 2 / Max 4 (t2.micro)
- Root Volume: gp3
- CloudWatch: 5-minute interval monitoring

> **Note on Session Stickiness:** Enabled intentionally. The vProfile application does not issue session tokens, so without stickiness, auth sessions would be lost when load is distributed across instances.

**Deployment:**
- Strategy: Rolling
- Batch Size: 50% *(10% is recommended for production)*

---

### 2. RDS (MySQL)

|    Setting   |     Value     |

| Engine       |  MySQL 8.4.7  |
| Template     |    Sandbox    |
| Instance     |   db.t3.micro |
| DB Name      |  `accounts`   |
| Username     |    `admin`    |
| VPC          |    Default    |
| Port         |     3306      |

**Security Group:** A new security group was created for the RDS instance allowing inbound traffic on port `3306` from the Elastic Beanstalk instances' security group.

**Database Initialisation:**

After verifying connectivity (SSH into a Beanstalk EC2 instance), initialise the schema:

```bash
# Download the schema
wget https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/aws-ci/src/main/resources/db_backup.sql

# Import into RDS
mysql -h <rds-endpoint> -u admin -p<password> accounts < db_backup.sql
```

---

### 3. RDS Connectivity — SSL Troubleshooting

When testing connectivity from an EC2 instance, MariaDB 11.4 enforces strict SSL verification by default, which breaks connections to RDS with a self-signed CA error.

**Quick fix (lab/testing only — skip SSL):**
```bash
mariadb -h <rds-endpoint> -u <user> -p<pass> --ssl=0
```

**Production fix (trust the AWS CA):**
```bash
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
mariadb -h <rds-endpoint> --ssl-ca=global-bundle.pem -u <user> -p<pass>
```

> **DevOps Takeaway:** Always pin package versions in CI/CD scripts (e.g., `dnf install mariadb105`). Upgrading a client tool can silently introduce stricter security defaults that break existing pipelines.

---

### 4. Bitbucket Source Setup

```bash
# Generate SSH key and add public key to Bitbucket (Account Settings → SSH Keys)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Test the connection
ssh -T git@bitbucket.org

# Clone the repository
git clone git@bitbucket.org:<workspace>/vproapp.git
```

**Troubleshooting — Push Failing with "User Limit Exceeded":**

If `git push` fails with a read-only/user-limit error even though account limits are fine, the issue is likely a stale metadata or session cache. Fix:

```bash
ssh -T git@bitbucket.org          # refresh the SSH handshake
git remote rm origin               # clear cached remote config
git remote add origin git@bitbucket.org:<workspace>/vproapp.git
git push origin --all
```

---

### 5. CodeBuild — `buildspec.yml`

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto17
  pre_build:
    commands:
      - apt-get update
      - apt-get install -y jq
      - wget https://archive.apache.org/dist/maven/maven-3/3.9.8/binaries/apache-maven-3.9.8-bin.tar.gz
      - tar xzf apache-maven-3.9.8-bin.tar.gz
      - ln -s apache-maven-3.9.8 maven
      - sed -i 's/jdbc.password=admin123/jdbc.password=<YOUR_RDS_PASSWORD>/' src/main/resources/application.properties
      - sed -i 's/jdbc.username=admin/jdbc.username=admin/' src/main/resources/application.properties
      - sed -i 's/db01:3306/<YOUR_RDS_ENDPOINT>:3306/' src/main/resources/application.properties
  build:
    commands:
      - mvn install
  post_build:
    commands:
      - mvn package

artifacts:
  files:
    - '**/*'
  base-directory: 'target/vprofile-v2'
```

> **Common Mistake:** The `sed` substitution requires a closing `/` — `'s/old/new/'` not `'s/old/new'`. Missing it causes a build failure.

**S3 Artifact Bucket:** Create an S3 bucket before setting up the CodeBuild project to store build artifacts.

---

### 6. CodePipeline

|  Stage  |          Provider           |               Notes                       |

| Source  | Bitbucket (`aws-ci` branch) | Triggers automatically on push            |
| Build   |        AWS CodeBuild        | Uses the project configured above         |
| Test    |     *(not configured)*      | A separate CodeBuild project can be added |
| Deploy  |    AWS Elastic Beanstalk    | Targets the `vprofile` application        |

> **IAM Note:** The auto-created CodePipeline service role may not include Elastic Beanstalk permissions. Add `AWSElasticBeanstalkFullAccess` (or a scoped equivalent) to the role if deployment fails.

---

## Known Issues & Fixes

|               Issue                 |                 Root Cause                     |                           Fix                          |

| `sed` build failure                 | Missing closing `/` in substitution expression | Ensure `'s/old/new/'` syntax                           |
| MariaDB SSL error (11.4)            | Stricter default SSL verification              | Use `--ssl=0` for testing or provide `--ssl-ca` bundle |
| Bitbucket push rejected (read-only) | Stale SSH session / cached remote config       | Re-run `ssh -T`, remove and re-add remote              |
| Session lost between requests       | No session token in vProfile app               | Enable Session Stickiness on Beanstalk load balancer   |

---

## Repository Structure

```
vprofile-project/
├── src/
│   └── main/
│       └── resources/
│           ├── application.properties   # DB connection config (patched by buildspec)
│           └── db_backup.sql            # Initial schema for RDS
├── buildspec.yml                        # CodeBuild build definition
└── ...
```

---

## References

- [AWS Elastic Beanstalk Docs](https://docs.aws.amazon.com/elasticbeanstalk/)
- [AWS CodePipeline Docs](https://docs.aws.amazon.com/codepipeline/)
- [AWS RDS SSL/TLS Certificates](https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem)
- [vProfile Project Source](https://github.com/hkhcoder/vprofile-project/tree/aws-ci)


