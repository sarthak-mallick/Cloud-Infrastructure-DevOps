# Cloud-Infrastructure-DevOps

## Architecture

```mermaid
graph TD
    subgraph External["External Access"]
        User["User"]
        R53["Route 53 DNS"]
        IGW["Internet Gateway"]
    end

    subgraph VPC["VPC (Virtual Private Cloud)"]
        subgraph Public["Public Subnets"]
            ALB["Application Load Balancer<br/>(HTTPS/SSL/TLS)"]
        end

        subgraph Private["Private Subnets"]
            subgraph ASG["Compute Layer (Auto Scaling)"]
                EC2_1["EC2 Instance<br/>(Node.js App)"]
                EC2_2["EC2 Instance<br/>(Node.js App)"]
            end
            
            RDS[("RDS Database<br/>MySQL<br/>Encryption at Rest")]
        end
        
        S3["S3 Bucket<br/>(Object Storage<br/>Encryption at Rest)"]
    end

    subgraph Management["CI/CD & Observability"]
        GHA["GitHub Actions<br/>(CI/CD Pipeline)"]
        Packer["Packer"]
        AMI["AMI Registry"]
        CW["CloudWatch<br/>(Logs/Metrics/Alarms)"]
        StatsD["StatsD<br/>(Custom Metrics)"]
    end

    %% Network Flow
    User -- "HTTPS (Custom Domain)" --> R53
    R53 -- "Alias Record" --> ALB
    ALB -- "Traffic Distribution" --> EC2_1
    ALB -- "Traffic Distribution" --> EC2_2
    
    %% Application Flow
    EC2_1 -- "SQL/Data" --> RDS
    EC2_2 -- "SQL/Data" --> RDS
    EC2_1 -- "Object Storage" --> S3
    EC2_2 -- "Object Storage" --> S3

    %% Monitoring
    EC2_1 -. "Metrics (StatsD)" .-> StatsD
    EC2_2 -. "Metrics (StatsD)" .-> StatsD
    StatsD -. "Aggregated Metrics" .-> CW
    EC2_1 -. "Logs (CloudWatch Agent)" .-> CW
    EC2_2 -. "Logs (CloudWatch Agent)" .-> CW

    %% CI/CD Flow
    GHA -- "Build & Test" --> Packer
    Packer -- "Bake Image" --> AMI
    AMI -- "Rolling Deployment" --> EC2_1
    AMI -- "Rolling Deployment" --> EC2_2

    %% Styling
    classDef compute fill:#E1F5FE,stroke:#01579B,stroke-width:2px,color:#000000;
    classDef data fill:#E8F5E9,stroke:#1B5E20,stroke-width:2px,color:#000000;
    classDef network fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#000000;
    classDef cicd fill:#F3E5F5,stroke:#4A148C,stroke-width:2px,color:#000000;
    classDef external fill:#FAFAFA,stroke:#212121,stroke-width:2px,color:#000000;

    class EC2_1,EC2_2 compute;
    class RDS,S3 data;
    class ALB,R53,IGW network;
    class GHA,Packer,AMI,CW,StatsD cicd;
    class User external;
```


## Tech Stack
- **Backend**: Node.js, Express, Sequelize ORM
- **Database**: MySQL (RDS)
- **Infrastructure**: Terraform, AWS (VPC, EC2, S3, Route53, CloudWatch)
- **CI/CD**: GitHub Actions, Packer
- **Security**: IAM Roles, Security Groups, SSL/TLS, Encryption at Rest



## API Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/healthz` | Health check endpoint (Database connectivity). |
| `POST` | `/v1/file` | Upload a file (Authenticated). |
| `GET` | `/v1/file/:id` | Retrieve file metadata (Authenticated). |
| `DELETE` | `/v1/file/:id` | Delete a file (Authenticated). |
| `PUT` | `/v1/file/:id` | Update file metadata (Authenticated, if supported). |

## Deployment Guide

### Prerequisites
- **Node.js**: For local development and testing.
- **Terraform**: For infrastructure provisioning.
- **Packer**: For building AMIs (automated via CI/CD).
- **AWS CLI**: For managing AWS resources and credentials.

### Configuration

#### Application
1.  Create a `.env` file in `webapp/` with database and app credentials:
    ```
    DB_HOST=...
    DB_USER=...
    DB_PASSWORD=...
    DB_NAME=...
    PORT=...
    ```

#### Infrastructure
1.  Navigate to `tf-aws-infra/`.
2.  Create a `<env>.tfvars` file (e.g., `dev.tfvars`) with required variables:
    *   `region`
    *   `vpc_cidr`
    *   `profile`
    *   `route53_zone_id` (from AWS Route53)
    *   `ssl_certificate_arn` (from AWS ACM)

### Local Development
1.  Install dependencies: `npm install`
2.  Run tests: `npm test`
3.  Start server: `node server.js`

### Infrastructure Provisioning

1.  **Initialize Terraform**:
    ```bash
    terraform init
    ```
2.  **Plan Infrastructure**:
    ```bash
    terraform plan -var-file=<env>.tfvars
    ```
3.  **Apply Configuration**:
    ```bash
    terraform apply -var-file=<env>.tfvars
    ```

### SSL Certificate Setup (One-time)
1.  Generate CSR and Key using OpenSSL.
2.  Purchase/Obtain SSL Certificate.
3.  Import into AWS ACM:
    ```bash
    aws acm import-certificate --certificate fileb://cert.crt --private-key fileb://private.key --certificate-chain fileb://ca-bundle.crt --region us-east-1
    ```
4.  Update `ssl_certificate_arn` in your `.tfvars`.

### CI/CD & Deployment

Deployments are automated via GitHub Actions:
1.  **Build AMI**: Merging a PR triggers Packer to build an AMI with the application code.
2.  **Rolling Deployment**:
    *   The new AMI ID is updated in the Auto Scaling Group Launch Template.
    *   An Instance Refresh is triggered to perform a zero-downtime rolling update.

### Verification
1.  **Health Check**: Access `https://<your-domain>/healthz`.
2.  **Load Testing**:
    ```bash
    ab -n 1000 -c 100 https://<your-domain>/healthz
    ```
3.  **Monitoring**: Check CloudWatch for logs and custom metrics (StatsD).
