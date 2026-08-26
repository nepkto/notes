# Azure vs AWS Terminology for Cloud Deployments

This reference maps the important deployment terms used in the Azure and AWS guides so readers can translate the same architecture across both environments.

## Comparison Table

| Category | Azure Term | AWS Term | What It Means |
| --- | --- | --- | --- |
| Cloud platform | Azure | AWS | The provider hosting the infrastructure. |
| Management portal | Azure Portal | AWS Management Console | Web UI used to create and manage resources. |
| Compute | Azure Virtual Machine | EC2 instance | The Linux server that hosts the application. |
| Public networking control | Network Security Group (NSG) | Security Group | Firewall rules controlling inbound and outbound traffic. |
| Public address | Public IP | Elastic IP / Public IPv4 | The internet-routable address used to reach the server. |
| Resource organization | Resource Group | Tags, accounts, and VPC organization | Azure groups related resources in one container; AWS usually organizes with tags and account structure rather than a direct equivalent. |
| Private network | Virtual Network (VNet) | VPC | The private network boundary for cloud resources. |
| Network segment | Subnet | Subnet | A network segment inside the VNet or VPC. |
| DNS / domain | Azure DNS or registrar DNS | Route 53 or registrar DNS | Where the domain points to the public server IP. |
| Operating system | Ubuntu on Azure VM | Ubuntu on EC2 | The guest OS running on the compute instance. |
| Package manager | `apt` | `apt` | The Linux package manager used on Ubuntu. |
| App runtime | Kestrel | Kestrel | The ASP.NET Core web server used by the application. |
| Reverse proxy | nginx | nginx | Handles incoming requests and proxies them to the app. |
| Process manager | systemd | systemd | Keeps the app running and restarts it if it fails. |
| Container runtime | Docker Engine | Docker Engine | The software used to run containers. |
| Multi-container orchestration | Docker Compose | Docker Compose | Defines and starts app, database, and proxy containers. |
| Application deployment model | Bare metal / VM process / container | Bare metal / VM process / container | The way the app is packaged and run. |
| Database engine | SQL Server 2022 | SQL Server 2022 | The relational database used by the application. |
| Database host | SQL Server on the VM | SQL Server on EC2 or in a container | Where SQL Server runs. |
| Certificate automation | certbot | certbot | Automates Let's Encrypt certificate issuance and renewal. |
| HTTPS / TLS | Let's Encrypt certificate on nginx | Let's Encrypt certificate on nginx | Secures browser-to-server traffic. |
| Firewall ports | 22, 80, 443 | 22, 80, 443 | SSH, HTTP, and HTTPS traffic. |
| SSH access | SSH over port 22 | SSH over port 22 | Secure shell access for administration. |
| App traffic port | Internal port such as 5000 | Internal port such as 5000 | The local port nginx forwards to for the ASP.NET Core app. |
| Public web ports | 80 and 443 | 80 and 443 | HTTP for validation and HTTPS for production traffic. |
| Connection string | appsettings.Production.json or environment variable | appsettings.Production.json or environment variable | The application setting used to reach SQL Server. |
| Secrets storage | Key Vault | AWS Secrets Manager / SSM Parameter Store | Where to store credentials and connection strings securely. |
| Identity / access | Managed Identity | IAM Role / Instance Profile | The cloud-native way to grant access without embedding credentials. |
| Monitoring logs | Azure Monitor / Log Analytics | CloudWatch Logs | Service and platform logs for troubleshooting. |
| Metrics / alerting | Azure Monitor alerts | CloudWatch alarms | Operational signals used to detect issues. |
| Backup / recovery | Azure Backup or database backups | AWS Backup or database backups | How data is protected and restored. |
| File transfer | SCP / SFTP | SCP / SFTP | Copying published app files to the server. |
| Public certificate authority | Let's Encrypt | Let's Encrypt | Free certificate authority used by certbot. |
| Web traffic flow | Browser -> nginx -> Kestrel -> SQL Server | Browser -> nginx -> Kestrel -> SQL Server | The request path stays the same across clouds. |

## How to Read This Table

- Azure and AWS use different names for similar building blocks.
- The app architecture stays the same: browser -> nginx -> Kestrel -> SQL Server.
- Only the cloud control-plane terminology changes.
- When moving between clouds, map the infrastructure terms first, then reuse the same application design.

## Practical Translation

- Azure Virtual Machine becomes EC2 instance.
- NSG becomes Security Group.
- Azure Portal becomes AWS Console.
- VNet becomes VPC.
- Azure Key Vault becomes AWS Secrets Manager or SSM Parameter Store.
- Azure Monitor becomes CloudWatch.
- Everything inside the guest OS stays the same: `systemd`, `nginx`, `certbot`, Docker, Kestrel, and SQL Server.