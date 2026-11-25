
# Discovery Report: Storing and Serving Delivery Photos

| **Lab** | Storing and Serving Delivery Photos |
| --- | --- |
| **Name** | Effie Njoki |
| **Date** | 25 November 2026 |
| **Client** | Onward Logistics (see the Client Profile) |

# 1. What you found

## The client's need

Onward needs to keep every delivery photo safe as evidence for disputes and show it quickly only to the people it belongs to, at a cost that stays under control as photos and viewers grow every month.

## Solutions that exist

| **Solution** | **How it works** | **Trade off for this client** |
| --- | --- | --- |
| Object storage for the files, with details in a relational database | Photo files are kept in a storage service built for files. Each database record holds the photo's title, description and the name of its file. | Strength: storage grows without limit, costs only what is stored, and the database stays small and fast. Weakness: two services must be secured and backed up instead of one. |
| Photos stored inside the database | The image itself is saved in a database table next to its details. | Strength: everything is in one place. Weakness: the database becomes large and slow, backups become expensive, and the client explicitly requires photo files to stay out of the database. |
| Shared network file system | Photos are saved as files on a network drive that every application server opens like a normal folder. | Strength: simple for the application. Weakness: costs many times more per gigabyte than object storage and pays for file editing features that photos, which never change, do not need. |

## What happens with no solution

Delivery photos are personal data. They show customers' doors, addresses, parcels and sometimes people. Without a deliberate design, Onward faces four risks.

**Photos exposed to the public.** A storage location left open is a common and well documented failure. In 2020, security researchers found that PhotoSquared, a photo printing app, had left an Amazon S3 storage location online with no password. It held about 94.7 GB of customer photos, orders, receipts and shipping labels, including full names and home delivery addresses. That is very close to what Onward's delivery photos and records contain. In another case, the dating app Jack'd agreed to a US$240,000 settlement with the New York Attorney General after users' private photos sat in an unsecured S3 location for more than a year. For Onward, whose Client Profile says trust is the product, a similar leak would damage customer and merchant trust directly.

**Photos lost.** Photos are Onward's evidence in disputes. If they are lost, Onward cannot prove a delivery and may have to refund or compensate. The 2021 OVHcloud data centre fire in Strasbourg shows what happens when copies are kept in one place. Two customers had paid OVHcloud for backups, but the backups were stored in the same data centre as the live servers, so both were destroyed together. A French court later ordered OVHcloud to pay those two customers €250,000. The lesson for Onward: a backup in the same location as the original does not protect against losing that location.

**Legal exposure.** Data protection rules apply in every market Onward serves. In Nigeria, penalties under the Nigeria Data Protection Act 2023 can reach ₦10 million or 2% of annual gross revenue, whichever is greater. The regulator enforces this: it fined Multichoice Nigeria ₦766.2 million, partly for illegally transferring Nigerians' personal data across borders.

**Disruption to operations.** Without fast delivery of photos, customers on slow mobile connections wait for every image to travel from Europe, and every view adds load to Onward's servers. On peak days this risks slow pages or outages, which the Client Profile names as the top priority to avoid.

## Sources

- PhotoSquared data leak, Risk Based Security mailing list: https://seclists.org/dataloss/2020/q1/144
- Jack'd settlement over unsecured S3 photos, Trend Micro: https://www.trendmicro.com/vinfo/us/security/news/online-privacy/unsecured-aws-s3-servers-lead-to-steep-settlement-fee-for-dating-app-jack-d-exposed-data-for-fortune-100-companies
- OVHcloud ordered to pay customers who lost data, DatacenterDynamics: https://datacenterdynamics.com/en/news/ovhcloud-ordered-to-pay-250k-to-two-customers-who-lost-data-in-strasbourg-data-center-fire
- Nigeria Data Protection Act penalties, TheCable: https://www.thecable.ng/nigerias-data-privacy-moment-the-court-has-spoken-boardrooms-must-listen/
- NDPC enforcement and Multichoice fine, Nairametrics: https://nairametrics.com/2026/04/06/ndpc-probes-remita-sterling-bank-over-alleged-data-breach/
- Amazon S3 pricing: https://aws.amazon.com/s3/pricing/
- AWS storage options and prices compared, ComputerWeekly: https://computerweekly.com/feature/AWS-storage-Key-storage-options-in-the-Amazon-cloud
- Amazon VPC pricing (NAT gateways and endpoints): https://aws.amazon.com/vpc/pricing/
- Regional NAT gateways, AWS documentation: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html
- AWS edge location in Nigeria: https://aws.amazon.com/about-aws/whats-new/2023/06/aws-edge-location-nigeria
- CloudFront price classes: https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cloudfront.PriceClass.html
- How Git sync works with CloudFormation: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/git-sync-concepts-terms.html

# 2. The business value of your solution

## Your chosen approach

Photo files are stored in Amazon S3 and their details in an Amazon RDS for PostgreSQL database, linked by the photo's file name. The application runs as containers on Amazon ECS with Fargate in private subnets behind an Application Load Balancer. Photos are viewed through Amazon CloudFront using short lived signed links, and every photo is automatically copied to a second Region. Everything is built with CloudFormation from Git.

This approach suits Onward better than the alternatives because it protects the photo evidence in two Regions, keeps every photo private, and charges only for what is stored and viewed, which matters because delivery margins are thin.

### How a photo flows through the system

**Upload:** the rider submits a photo with its details to the application. The application saves the file in S3 under a unique name, then saves a database record containing the title, description, other details and that file name. S3 then copies the photo to the Frankfurt bucket.

**Viewing:** the customer opens the gallery. The application reads the records from the database and creates a signed link for each photo, valid for 5 minutes. The browser loads each photo from the nearest CloudFront edge location. The S3 location itself is never shown.

### Why Amazon S3 for the photo files

I compared four places to store the photo files against what the photos need:

| **Need (from the brief)** | **Amazon S3** | **Amazon EBS** | **Amazon EFS** | **Inside the database** |
| --- | --- | --- | --- | --- |
| Hold a growing number of files cheaply (Finance Lead) | Yes, pay per GB stored | No, pay for the full disk size reserved | No, much higher price per GB | No, makes the database large and slow |
| Reachable by every container in both Availability Zones (CTO) | Yes | No, a disk lives in one Availability Zone | Yes | Yes |
| Recover a deleted photo (Head of Operations) | Yes, versioning | Only through snapshots | Through backups | Through database restore |
| Copy every photo to a second Region (Head of Operations) | Yes, built in replication | No | Possible, at file system prices | Only by copying the whole database |
| Private, never directly viewable (Head of Security) | Yes, Block Public Access | Yes | Yes | Yes |
| Fast delivery to many cities (CTO, stretch) | Yes, works directly with CloudFront | No | No, every view passes through the application | No |

On price, S3 Standard costs $0.023 per GB per month for the first 50 TB, while EFS Standard in Europe is listed at about $0.33 per GB per month, roughly 14 times more for the same photos. S3 is also designed for eleven nines of durability and stores data across three Availability Zones by default.

**Decision:** Amazon S3 Standard. It is the only option that meets every need, and the cheapest. EBS is rejected because it lives in a single Availability Zone and is charged for its full size. EFS is rejected because it costs far more per gigabyte for editing features that photos, which never change after upload, do not need. The database is rejected because the client requires photo files to stay out of it, and it would make backups slow and expensive.

### Why Amazon RDS for PostgreSQL, Multi-AZ, for the photo details

The details are small, structured records that need to be searched and sorted, so a relational database fits. Each record has an id, the photo's S3 file name (unique, not null), title, description, delivery reference, uploader name and upload time. The S3 file name is the link between a record and its photo.

| **Option** | **Decision and reason** |
| --- | --- |
| RDS for PostgreSQL, Multi-AZ | **Chosen.** A standby copy in a second Availability Zone takes over automatically if the first fails. Lowest cost option that survives the loss of an Availability Zone. |
| Amazon Aurora | Rejected for now. Its readers and faster failover matter at high scale, but its smallest instance size is larger and costs more. Moving to Aurora later is a supported upgrade. |
| Single-AZ RDS | Rejected. The database would be unavailable if its Availability Zone failed. |
| Database on a self managed server | Rejected. The brief forbids it, and the small platform team would have to patch and back it up by hand. |
| Amazon DynamoDB | Rejected. The brief requires a relational database. |

PostgreSQL and MySQL would both work. PostgreSQL is a free, widely supported engine with full Java support.

### Keeping photos private and the database unreachable (Head of Security)

| **Setting** | **What it does** |
| --- | --- |
| All four S3 Block Public Access settings on, written in the template | S3 refuses any setting that would make a photo public, even by mistake. Written in code so it is visible in Git and provable in an audit. |
| Bucket policy allowing only our CloudFront distribution (Origin Access Control) | CloudFront can read photos. A direct request to S3 is refused. |
| CloudFront signed links valid for 5 minutes | Photos load only through pages the application serves. A link copied out of the application stops working after 5 minutes. |
| Database in private data subnets with no route to the internet, not publicly accessible | The database cannot be reached from the internet in either direction. |
| Security group chain: load balancer accepts web traffic, application accepts traffic only from the load balancer, database accepts its port only from the application | Nothing else, even inside the network, can reach the database. Rules reference security groups, not IP addresses, so they keep working when containers are replaced. |
| Database password created and rotated by RDS in Secrets Manager | No password exists in code or templates. |
| CloudFront signing key generated inside AWS and stored in Secrets Manager | No person handles the private key and it never appears in Git. |
| ECS injects settings from Parameter Store and secrets from Secrets Manager at startup | No credentials or environment specific values in the application code. |
| ECS task role with a customer managed policy allowing only upload and read on the photos folder of one bucket | Least privilege. AWS managed S3 policies cover every bucket in the account, and the company standard forbids inline policies. |
| Encryption at rest for S3 and RDS, encrypted connections to the database | Data is protected on disk and in transit inside AWS. |

Options rejected: S3 presigned links, because they give direct access to storage. Signed cookies, because one leaked cookie would open every photo. Plain CloudFront links, because they could be shared forever.

### Recovering from deletion or failure (Head of Operations)

| **Setting** | **What it does** |
| --- | --- |
| S3 Versioning | A deleted photo is kept under a delete marker. Removing the marker restores it. |
| Lifecycle rule: delete old versions after 90 days | Caps the cost of keeping history. 90 days is chosen to cover how long a dispute might take to appear, and can be changed if disputes take longer. |
| RDS automated backups kept for 7 days | The database can be restored to any second in the last 7 days. The CloudFormation default is only 1 day, so it is set explicitly. |
| Recovery targets for the photo details | Recovery point about 5 minutes, recovery time under an hour. Acceptable because the photo files are protected separately. |
| RDS deletion protection and a final snapshot if the stack is deleted | The database cannot be deleted by accident, and a full copy survives even if it is. |
| Multi-AZ database and two application containers in two Availability Zones | Losing one Availability Zone does not stop the service. |

Option considered: AWS Backup, a central backup service. Rejected for now because one database is fully covered by RDS's own backups, and the brief only requires Region protection for photos. Worth adopting as the number of databases grows.

### Keeping every photo if a Region is lost (Head of Operations)

S3 Cross-Region Replication copies every new photo from the Ireland bucket to a bucket in Frankfurt (eu-central-1). Frankfurt is the second Region approved for this account, and far enough from Ireland that one event is very unlikely to affect both. The Frankfurt bucket is private, versioned and blocks public access. Deletions in Ireland are not replicated, so a mistaken or malicious delete never removes the Frankfurt copy.

If Ireland is lost: every photo still exists in Frankfurt, apart from any uploaded in the last few minutes before replication finished. The application and database are in Ireland only, as the brief allows, so the gallery and the photo details would be unavailable until Ireland recovers. The application can be rebuilt from Git, but the photo details cannot be recovered from Frankfurt.

### Staying available on busy days (CTO)

The ECS service always runs at least two containers, one in each Availability Zone, behind the load balancer. If a container fails its health check, the load balancer stops sending it traffic and ECS replaces it. Auto scaling adds containers when CPU use is high, up to a maximum of 4, which also caps cost. New versions are released with a blue/green deployment: the new version starts beside the old one and receives traffic only once healthy, so a bad release can be rolled back quickly.

### Fast, low cost viewing in different cities (stretch objective)

| **Setting** | **Choice and reason** |
| --- | --- |
| Amazon CloudFront in front of the photo bucket | Keeps copies of photos at edge locations near customers, including the AWS edge location opened in Lagos, Nigeria in 2023. Cached photos load quickly on slow mobile connections and do not use Onward's containers. |
| Price Class 200 | Includes edge locations in Nigeria, Kenya, South Africa and Egypt. Price Class 100 is cheaper but excludes Africa, so West African customers would be served from Europe. Price Class All adds regions such as South America and Australia where Onward has no customers. |
| AWS managed CachingOptimized policy, photos uploaded with a one year cache lifetime | Photos never change after upload, so long caching is safe and keeps most views at the edge. Signed links do not affect caching, because CloudFront checks the signature and then serves the same cached copy for every valid link. |
| Origin Access Control and signed links | Keeps the bucket private while CloudFront delivers photos (see the Head of Security section). |

### Network access for the private containers

The containers have no route to the internet. They reach the AWS services they need through VPC endpoints: a free S3 gateway endpoint for photos and container image layers, and interface endpoints for Amazon ECR, CloudWatch Logs, Secrets Manager and Parameter Store.

| **Option** | **Decision and reason** |
| --- | --- |
| VPC endpoints, no NAT gateway | **Chosen.** The containers can reach only the AWS services they need and nothing else on the internet, which serves the Head of Security. The hourly cost is in the same range as the NAT options and is confirmed in the Pricing Calculator. The client ranks protecting data above cost, so the stronger security wins. |
| One zonal NAT gateway | Rejected. Cheapest NAT option, but if its Availability Zone fails, containers in the other zone lose access to AWS services. |
| Two zonal NAT gateways, one per Availability Zone | Rejected. Highly available, but gives the containers a general path to the internet they do not need. |
| One regional NAT gateway | Rejected for the same reason. Simpler to manage than two zonal gateways, but it is billed per hour for each Availability Zone it serves, so its hourly cost is similar to one zonal gateway per zone. |

### How the system is built and changed

| **Choice** | **Reason** |
| --- | --- |
| Separate infrastructure and application repositories | Required by the brief. Infrastructure changes rarely and needs careful review. The application changes weekly. Each has its own history and permissions. |
| Several small CloudFormation stacks, each deployed by its own Git sync deployment file | Each layer (network, storage, database, application) can change on its own. Rejected: one large template, where an application change risks the database; and nested stacks, which need child templates uploaded to S3 by a second tool, while Git sync deploys straight from the repository and nested stacks cannot cross Regions. |
| Values passed between stacks through Parameter Store, under paths like /onward/dev/network/vpc-id | No resource IDs typed into templates, so the whole setup can be rebuilt from Git. Rejected: CloudFormation exports, which lock stacks together. |
| One template per stack, with a deployment file per environment | Values that differ by environment are set in each environment's file. Conditions are used only for resources that exist in some environments and not others. |
| Pipeline for the application: CodePipeline, CodeBuild, CodeDeploy | Every push builds and deploys the same way, with no credentials on laptops. Supports the move from monthly to weekly releases, and every running version traces back to a Git commit. |
| Container base image Amazon Corretto 21 from Amazon ECR Public, pinned version | AWS supported and regularly patched Java. Pinning the version makes every build identical. Rejected: the deprecated openjdk image and any image tagged latest. |
| Company naming (onward-dev-purpose) and tags (Project, Owner, Environment) on every resource | Every resource has an owner and its cost can be traced. |
| AWS Budgets alert | Unexplained spend is flagged early, as the company standard requires. |

### Cost

The main cost drivers are the RDS Multi-AZ database, the Fargate containers, the load balancer, the VPC interface endpoints, S3 storage in two Regions, replication transfer between Regions, and CloudFront delivery. Storage is the part that grows with every photo, which is why the cheapest suitable storage was chosen. CloudFront reduces the cost of viewing as it grows, because cached photos are served from the edge rather than by Onward's containers.

Full monthly estimate: [AWS Pricing Calculator link, added after the cost step]

## Services you will use

| **Service** | **The job it does** | **Why this service, used this way** |
| --- | --- | --- |
| Amazon S3 (Ireland) | Stores the photo files | Cheapest storage that grows without limit, copies data across three Availability Zones, and supports versioning, replication and CloudFront. Private, versioned, encrypted. |
| Amazon S3 (Frankfurt) and Cross-Region Replication | Keeps a copy of every photo in a second Region | Meets the Head of Operations' Region loss requirement with no application code. Deletions are not replicated. |
| Amazon RDS for PostgreSQL, Multi-AZ | Stores the photo details | Managed relational database with automatic failover, 7 day point in time recovery and deletion protection, at the lowest cost that survives an Availability Zone loss. |
| Amazon ECS with AWS Fargate | Runs the application containers | Required by the brief. No servers to manage, which suits a small platform team. Two containers across two Availability Zones, scaling to 4. |
| Amazon ECR | Stores the application's container images | Private image registry inside AWS, reachable from private subnets through endpoints. |
| Application Load Balancer | Receives customer traffic and spreads it across containers | The only public entry point to the application. Health checks remove failed containers. |
| Amazon VPC with public, private application and private data subnets in two Availability Zones | Separates what is public from what is private | Only the load balancer is public. The database has no internet route at all. |
| VPC endpoints (S3 gateway, and interface endpoints for ECR, CloudWatch Logs, Secrets Manager, Parameter Store) | Let private containers reach AWS services without the internet | Containers can reach only what they need, which is stronger security than a NAT gateway at a similar hourly cost. |
| Amazon CloudFront | Delivers photos from edge locations near customers | Fast viewing in West Africa through the Lagos edge, lower viewing cost as views grow, signed links to stop photos being passed around. Price Class 200. |
| AWS Secrets Manager | Holds the database password and the CloudFront signing key | RDS creates and rotates the password itself. Secrets never appear in code. |
| AWS Systems Manager Parameter Store | Holds settings such as the bucket name, database address and network IDs | Free for standard parameters. Removes hardcoded values from templates and code. |
| AWS IAM roles with customer managed policies | Give the application, ECS, S3 replication and the pipeline only the access they need | Temporary credentials, no long lived keys, least privilege, no inline policies. |
| AWS Lambda (inside the CloudFormation stack) | Generates the CloudFront signing key pair | Keeps the setup fully rebuildable from Git, with no manual key handling. |
| AWS CloudFormation with Git sync | Builds and updates all infrastructure from the repository | Required by the brief. Every change is reviewed and traceable in Git. |
| AWS CodePipeline, CodeBuild, CodeDeploy | Builds and releases the application | Repeatable weekly releases with blue/green rollback and no credentials on laptops. |
| Amazon CloudWatch Logs | Keeps the application's logs | Needed to investigate problems and support audits. |
| AWS Budgets | Alerts when spend passes a set amount | Company standard: keep budget alerts active. |

## Who benefits, and how you will know

| **Stakeholder** | **Benefit** | **How you would know** |
| --- | --- | --- |
| Chief Technology Officer | Photos load quickly in every city, and the service keeps running if a container or an Availability Zone fails | CloudFront cache hit ratio above 90%. Stopping one container in the review leaves the website working. |
| Head of Security | Photos and the database cannot be reached from outside the system, and no secrets are stored in code | A direct S3 request returns Access Denied. A copied photo link fails after 5 minutes. A search of both repositories finds no passwords or keys. |
| Head of Operations | Every photo survives a deletion, a failure or the loss of the Region | A deleted photo and a deleted record are restored in the review. A photo uploaded today is shown in the Frankfurt bucket. |
| Finance Lead | A clear, controlled cost for storing, protecting and showing photos | Pricing Calculator estimate with a shared link. Actual monthly spend stays within the budget alert. |
| Small platform team and engineers | Infrastructure and releases are automated and repeatable | The whole setup can be rebuilt from the two repositories. Releases take a git push. |
| Customers and merchants | Their delivery photos are private, quick to view and never lost | Photos appear in the gallery with their details within seconds of upload. |

## Limits and risks of your design

**No sign-in.** The lab does not require sign-in, so anyone with the application's address can open the gallery. Photos are protected from direct storage access and from link sharing, but are not yet tied to a specific customer. The fix is to add sign-in so the application checks who is asking before creating a signed link.

**The application and database are in one Region.** As the brief allows, losing Ireland makes the gallery and photo details unavailable until it recovers, and the photo details cannot be rebuilt from Frankfurt. Copying database snapshots to Frankfurt would be a low cost next step.

**Replication is not instant.** Photos uploaded in the last few minutes before a Region failure may not have reached Frankfurt yet.

**Deletions are not replicated.** This protects evidence, but if Onward must erase a photo, for example under a data protection request, it must be deleted in both Regions.

**Signed links can be shared for 5 minutes.** A shorter window reduces this further at the cost of more link creation.

**Cached copies of deleted photos.** A deleted photo may remain in CloudFront's cache, though it cannot be requested once existing links expire. If removal must be immediate, a CloudFront invalidation is required.

**Where the data lives.** Customer data from West Africa is stored in Ireland and Frankfurt. Data protection rules in Onward's markets restrict cross-border transfers, so Onward's legal team should confirm the lawful basis for this transfer. Hosting in an African Region would reduce this risk but is not approved for this account.

**The application is served over plain HTTP.** An encrypted HTTPS connection to the load balancer needs a domain name and certificate, which this lab does not provide. Photos themselves are delivered over HTTPS by CloudFront.

**Cost grows with use.** Storage, replication and viewing costs rise as photos and viewers grow. The budget alert, lifecycle rule and CloudFront caching limit this, but the estimate should be reviewed monthly.

**Only the dev environment is built.** Test and prod would reuse the same templates with their own deployment files.
