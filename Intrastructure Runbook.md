# Infrastructure Deployment Runbook

**Template Version:** 20260923.a **Classification:** Internal

---

## 1. Document Control

| Field | Value |
| --- | --- |
| Runbook Title | AWS Infrastructure for HSU-Supported Static Websites Deployment |
| Author | Tae-Jun (TJ) Lee |
| Owner / Service Team | HSU |
| Date Created | 23.09.2026 |
| Last Updated |  |
| Reviewed By |  |
| Approved By |  |
| Related Change Ticket(s) |  |
| Related Design Doc / RFC |  |

---

## 2. Summary

**What is being deployed:** *One or two sentences describing the change — new service, version bump, config change, infra migration, etc.*

An AWS infrastructure that supports the hosting of several static websites that HSU supports.

**Why:** *Business or technical driver.*

To consolidate the hosting of several static websites into a single AWS infrastructure, with the aim of reducing the complexity of maintaining multiple static websites, reduce costs, and increase scalability.

**Environments affected:** ☐ Dev ☐ Staging ☐ UAT ☐ Production

**Deployment window:**

- Start: `YYYY-MM-DD HH:MM TZ`
- End (estimated): `YYYY-MM-DD HH:MM TZ`
- Maintenance window required? ☐ Yes ☐ No

**Change risk level:** ☐ Low ☐ Medium ☐ High ☐ Critical

---

## 3. Roles & Contacts

| Role | Name | Contact (Slack/Phone) | Responsibility |
| --- | --- | --- | --- |
| Deployment Lead |  |  | Executes runbook, calls go/no-go |
| Approver |  |  | Sign-off authority |
| On-call Engineer |  |  | Incident response during window |
| Secondary/Backup |  |  | Coverage if lead unavailable |
| Comms Lead |  |  | Stakeholder + status page updates |
| Database Owner (if applicable) |  |  | Schema/data changes |

**Escalation path:** *Who gets paged if things go wrong, in order.*

---

## 4. Pre-Deployment Checklist

### 4.1 Readiness

- [ ] Change ticket approved (CAB/change board if required)
- [ ] Code merged to release branch / tag cut
- [ ] All CI checks passing (build, unit, integration, security scan)
- [ ] Deployment artifact/image built and pushed to registry
- [ ] Artifact version/hash recorded: `_______________`
- [ ] Peer review of this runbook completed
- [ ] Dependencies confirmed deployed/available (list below)

**Dependencies:**

| Dependency | Required Version/State | Verified By | Status |
| --- | --- | --- | --- |
|  |  |  | ☐ |

### 4.2 Environment & Access

- [ ] Required credentials/secrets rotated and valid
- [ ] Access confirmed for all executing engineers (VPN, IAM roles, kubeconfig, etc.)
- [ ] Infrastructure-as-Code plan reviewed (`terraform plan` / equivalent) and diff sanity-checked
- [ ] No conflicting deployments scheduled in the same window

### 4.3 Safety Nets

- [ ] Current system state backed up (DB snapshot, config export, AMI/image snapshot)
- [ ] Backup verified restorable / restore tested recently
- [ ] Rollback procedure written and reviewed (Section 7)
- [ ] Feature flag(s) configured for kill-switch, if applicable: `_______________`
- [ ] Monitoring dashboards and alerts confirmed active for affected services
- [ ] Synthetic/health-check endpoints identified for validation

### 4.4 Communication

- [ ] Stakeholders notified of deployment window
- [ ] Status page updated (if customer-facing impact expected)
- [ ] On-call team briefed
- [ ] Go/no-go meeting held (if required by risk level)

**Go/No-Go Decision:** ☐ GO ☐ NO-GO — Decided by: \_\_\_\_\_\_\_\_\_\_ at \_\_\_\_\_\_\_\_\_\_

---

## 5. Architecture / Change Overview

*Diagram or brief description of what changes: components touched, traffic flow before/after, data migrations, network changes. Link to architecture diagram if available.*

**Blast radius:** *What breaks if this goes wrong? Which services/customers are affected?*

---

## 6. Deployment Procedure

> Write as literal, copy-pasteable steps. Assume the executor is not the author. Include expected output for each command where possible.

### Step 0 — Pre-checks
Ensure website is built using `Astro + Tailwind CSS` and saved in a `GitHub` repository.

In the Amazon Console, esnure that the `Sydney` region is selected.

**Expected result:**

### Step 1 — Set Up Amazon Simple Storage Service (S3) Bucket
S3 is an object storage service that stores static files (e.g. HTML, CSS, JS, images). The astro website from the GitHub repository is compiled into static files, which will be stored in the S3 bucket. S3 bucket will serve the static files to users via CloudFront CDN (Content Delivery Network).

In the Amazon Console navigate to the `S3` service. Ensure that the `Sydney` region is selected. Click `Create bucket`.

Name the bucket `hsu-static-website-assets` under `Bucket name`.

Keep public access blocked by ensuring `Block all public access` is checked under the 'Block Public Access settings for this bucket' section.

Click `Create bucket`.

**Expected result:** You should be able to redirected to the `hsu-static-website-assets` bucket's page (you will see `hsu-static-website-assets` at the top of the page).

### Step 2 — Set Up Amazon CloudFront Distribution
The Amazon CloudFront Distribution is required to serve the static files to users via CloudFront CDN (Content Delivery Network). This means that users will be able to access the website via CloudFront CDN instead of S3 bucket directly, which improves the website's performance and scalability.

Note: This will need to be done per website. It is better to have a CloudFront distribution per domain (i.e., if you are hosting multiple static websites, you should create a CloudFront distribution for each website). You may have one domain to serve multiple static websites to reduce costs; however, this may result in a more complex management of the distribution (e.g. if you need to create a custom domain name and SSL certificate for the distribution).

In the Amazon Console, navigate to the `CloudFront` service. You will see that the region is `Global`. This is because CloudFront is a global service. Click `Create distribution`.

Add a distribution name that is related to the specific domain it serves (e.g., `hsu-com-au` if serving the `hsu.com.au` website). Ensure that `Distribution type` is `Single website configuration`. Click `Next`.

Select `Amazon S3` for `Origin type`. In the `Origin` section, enter `hsu-static-website-assets` under `S3 origin` (or select it from the `Browse S3` button). Ensure that in the `Settings` section, `Allow private S3 bucket access to CloudFront` is checked and leave the default settings for `Origin settings` and `Cache settings`. Click `Next`.

In the `Web Application Firewall (WAF)`, select `Do not enable security protections`. As these are simple static websites, we will not need the advanced security features of WAF and can save costs by implementing other security measures (e.g., AWS Certificate Manager (ACM) for SSL certificate management, Route 53 for domain name management, etc.). Click `Next`.

Click `Create distribution`. You will then be redirected to the newly created distribution's page. In the `General` tab, under the `Settings` section, click `Edit`.

TODO: "Skip Custom Domains (For Now): You will see a box for Alternate domain name (CNAME) and Custom SSL certificate. Leave these blank for now. We cannot fill these in until we create your free SSL certificate using AWS Certificate Manager (ACM). We will come back and edit this later."

In the `General` section, under `Default root object - option`, enter `index.html`. Click `Save changes`. You will be redirected to the distribution's page. 

To tell your S3 bucket to only allow your specific CloudFront distribution to look at its file, take note of the CloudFront distribution's `ARN` (Amazon Resource Name).

Go to the `hsu-static-website-assets` S3 bucket's page. Click on `Permissions` tab. In the `Bucket policy` page, ensure that the `ArnLike` is pointing to your CloudFront distribution's ARN. The JSON bucket policy should look like this:
```json
{
    "Version": "2008-10-17",
    "Id": "PolicyForCloudFrontPrivateContent",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::hsu-static-website-assets1/*",
            "Condition": {
                "ArnLike": {
                    "AWS:SourceArn": [
                        "arn:aws:cloudfront::123456789123:distribution/E1ABC123ABC", // This is one example, replace with your CloudFront distribution's ARN
                        "arn:aws:cloudfront::123456789123:distribution/E1ABC124ABC"  // This is another example, replace with subsequent CloudFront distribution's ARN
                    ]
                }
            }
        }
    ]
}
```

If it is not automatically there, click `Edit` and add the `ARN` that you noted earlier under the `AWD:SourceArn`. Click `Save changes`.

Now we will need to give CloudFront logic to automatically append `index.html` to the end of any folder path users click on so they are directed to the correct page (e.g., when users click `d298rvk2bhyrui.cloudfront.net/about`, CloudFront does not attempt to serve a file literally called `about/index.html`).

In the Amazon Console, navigate to the `CloudFront` service. Click on `Functions` in the left menu. Click `Create function`. In the `Function details` section, under `Function name`, enter `add-index-html`. Leave `Runtime` as `cloudfront-js-2.0`. Click `Create`.

You will be redirected to the function's page. In the `Function code` section, replace the pre-filled code with the following code:

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    // If the URL ends with a slash, append index.html
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } 
    // If the URL doesn't have a file extension (like .css or .png), append /index.html
    else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }
    
    return request;
}
```

Click `Save changes`. Click the `Publish` tab and click `Publish function`.

Now we need to attach the function to the distribution we created earlier. In the Amazon Console, navigate to the `CloudFront` service. Click on `Distributions` in the left menu. Click on your website's distribution ID. Click the `Behaviors` tab. Check the checkbox next to your default behavior (Path pattern = `Default (*)`) and click `Edit`.

Scroll down to the `Function associations` section. For `Viewer request`, select `Function type = CloudFront Functions` and `Function ARN / Name = add-index-html`. Click `Save changes`.

**Expected result:** The `CloudFront` distribution should now be able to access the S3 bucket and serve the static files to users via CloudFront CDN. If you go to the CloudFront distribution's page and open the link under the `Details` secton called `Distribution domain name` in your browser, you should see an error that states `This XML file does not appear to have any style information associated with it. The document tree is shown below.` and should NOT return a `404 Server Not Found` error. This is expected as there would not be any content in the S3 bucket to serve at this point. Navigating between pages within a website should not throw the error `This XML file does not appear to have any style information associated with it`.

### Step 3 — Set Up Route 53 and Certificate Manager
NOTE: Skip this step if you have not purchased a domain name yet and wanted to use the default CloudFront distribution's domain name.

You will now need to link your custom domain to CloudFront with HTTPS mainly for trust and security — it avoids "Not Secure" browser warnings, enables features and integrations that require HTTPS, protects data in transit, and gives a small SEO boost. Linking your custom domain involves two AWS services working together: Amazon Route 53 (the internet's phonebook that manages your DNS records) and AWS Certificate Manager or ACM (which provides the free SSL/TLS certificates for HTTPS).

TODO: NOTE: If you already have not done so, make sure that you have purchased a domain. You can do this via AWS (using Route 53) or a third-party registrar (e.g., GoDaddy, Namecheap). If you are purchasing a domain via AWS, ensure that the domain is registered to the same AWS account that you are using to host the static website. You can check if you have a domain registered to your AWS account by navigating to the `Route 53` service in the Amazon Console and clicking on `Domains > Registered domains` in the left menu. If you do not see your domain listed, you will need to purchase one.

In the Amazon Console, navigate to the `Route 53` service. You will see that the region is `Global`. This is because Route 53 is a global service. Click `Hosted zones` in the left menu and then click `Create hosted zone`.

In the `Domain name` field, enter your custom domain name (e.g., `hsu.com.au`). Click `Create hosted zone`.

Note: If the domain was purchased through a third party (e.g., GoDaddy, Namecheap), you will need to copy the four `Name Servers` (NS) records Route 53 gives you in the `Hosted zone details` overview page in this next page and paste them into your domain registrar's DNS settings.

In the Amazon Console navigate to the `Certificate Manager` service. Ensure that the `Sydney` region is selected. Click `Request`.

Ensure that the default for `Certificate type` is `Request a public certificate`. Click `Next`.

In the `Domain names` section, enter your domain name (e.g., `hsu.com.au`) in the `Fully qualified domain name` field. Click `Add another name to this certificate` and enter `*.hsu.com.au` to cover all subdomains. In the `Allow export` section, ensure that `Validation method` is `DNS validation` (this should be the default). Leave the rest of the options as default and click `Request`.

TODO: Validate the certificate
```
The following is from Gemini:
AWS needs proof that you own the domain. Click on the certificate ID you just requested (its status will say "Pending validation"). In the "Domains" section, you will see a button that says Create records in Route 53. Click this button, then click Create records. AWS will automatically inject the required proof into your Route 53 DNS settings. Within a few minutes, the certificate status will turn green and say "Issued".
```

TODO: Attach the Certificate to CloudFront
```
The following is from Gemini:

Navigate back to CloudFront and click on your distribution.
1. On the General tab, click Edit under Settings.
2. In the Alternate domain name (CNAME) box, click Add item and type your domain (e.g., yourdomain.com). Add another for [www.yourdomain.com](https://www.yourdomain.com).
3. In the Custom SSL certificate dropdown, select the ACM certificate you just created.
4. Scroll down and click Save changes.
```

TODO: Point Route 53 to CloudFront
```
The following is from Gemini:

Finally, go back to Route 53 and click on your Hosted zone.
1. Click Create record.
2. Leave the "Record name" blank (this routes the root domain).
3. Toggle the Alias switch to On.
4. For "Route traffic to", select Alias to CloudFront distribution.
5. Click the next box and select your CloudFront distribution URL (the .cloudfront.net address).
6. Click Create records.
```

### Step 4 — Connect GitHub repository using GitHub Actions
This step sets up the CI/CD pipeline. Whenever you push updates to GitHub, an automated workflow syncs your build files directly to your Amazon S3 bucket and triggers a cche invalidation in Amazon CloudFront.

In the Amazon Console, navigate to the `IAM` service. You will see that the region is `Global`. This is because IAM is a global service. In the left menu, click `Access Management > Identity providers`. Click `Add provider`.

In the `Provider details` section, select `OpenID Connect` for the `Provider type` field. Enter `https://token.actions.githubusercontent.com` into the `Provider URL` field and `sts.amazonaws.com` in the `Audience` field. Click `Add provider`.

Now we need to create the S3 and CloudFront deployment Policy. Firstly, you will need to have your CloudFront distribution's `ARN`. To get this (it is easier if this is done in a new window), in the Amazon Console, navigate to `CloudFront` service and click your distribution's ID in the `Distribution` column. The distribution's `ARN` (e.g., `arn:aws:cloudfront::123456789123:distribution/E1ABC123ABC`) will be shown in the `Details` section - Take a copy of this.

Back in the `IAM` service, click `Access Management > Policies` in the left menu. Click `Create policy`.

In the `Policy editor` section, select `JSON` and enter the following `JSON` script, replacing `YOUR_S3_BUCKET_NAME` with the name of your S3 bucket (e.g., `hsu-static-website-assets`, there should be two entries), `YOUR_ACCOUNT_NUMBER` with your account number (found at the top right-hand corner of the AWS console), and `YOUR_CLOUDFRONT_DISTRIBUTION_ID` with the ID of your CloudFront distribution (e.g., `E1ABC123ABC`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3SyncPermissions",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_S3_BUCKET_NAME", // Replace "YOUR_S3_BUCKET_NAME" with the name of your S3 bucket
        "arn:aws:s3:::YOUR_S3_BUCKET_NAME/*"  // Replace "YOUR_S3_BUCKET_NAME" with the name of your S3 bucket
      ]
    },
    {
      "Sid": "CloudFrontInvalidationPermissions",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::YOUR_ACCOUNT_NUMBER:distribution/YOUR_CLOUDFRONT_DISTRIBUTION_ID" // Replace "YOUR_ACCOUNT_NUMBER" with your account number (found at the top right-hand corner of the AWS console) and "YOUR_CLOUDFRONT_DISTRIBUTION_ID" with your distribution ID
    }
  ]
}
```

Click `Next`. In the `Policy details` section, enter `GitHubActionsStaticSiteDeployPolicy` in the `Policy name` field and click `Create policy`.

Now we will create a role for GitHub Actions to use. Still in the `IAM` service, click `Access Management > Roles` in the left menu. Click `Create role`.

In the `Trusted entity type` section, select `Web identity`. In the `Web identity` section, for `Identity provider`, select `token.actions.githubusercontent.com`. For `Audience`, select `sts.amazonaws.com`. For `GitHub Organization`, enter your username (e.g., `tlee149`) or the organisation name (e.g., `CNSGenomics`). For `GitHub repository`, enter the name of your repository (e.g., `hsu-astro-aws`). For `GitHub Branch`, enter `main`. Click `Next`.

In the `Permissions policies` section, search for the policy you created previously (e.g., `GitHubActionsStaticSiteDeployPolicy`) and select the corresponding checkbox. Click `Next`.

In the `Role details` section, under `Role name`, enter a meaningful name (e.g., `GitHubActions-S3-CloudFront-Role`). Click `Create role`.

You will be redirected to the `IAM > Access Management > Roles` page. Search for the role you just created (e.g., `GitHubActions-S3-CloudFront-Role`) and click on its name to view its details.

Now we need to create an IAM User that has permissions to assume the role we created above. In the left menu, click `Access Management > IAM Users`. Click `Create user`.

In the `User details` section, enter a meaningful name (e.g., `GitHubActionsDeployUser`). Do not check the `Provide user access to the AWS Management Console` checkbox. Click `Next`.

In the `Permissions options` section, select `Attach policies directly`. Search for the policy you created previously (e.g., `GitHubActionsStaticSiteDeployPolicy`) and check the corresponding checkbox. Click `Next`. Click `Create user`.

Click on your newly created user name (e.g., `GitHubActionsDeployUser`) and click the `Security credentials` tab. In the `Access keys` section, click `Create access key`. In the `Use case` section, select `Command Line Interface (CLI)` and select the checbkox for `I understand the above recommendation and want to proceed to create an access key`. Click `Next` and click `Create access key`. ***DO NOT CLOSE/LEAVE THE NEXT PAGE*** as you will not be able to access the `Access key` and `Secret access key` again after you close/leave it, so keep the page open or copy the keys in a secure location. These fields are important for the next step.

This next step will require you to enter AWS credential information into your GitHub repository. It is therefore easier to do this step in a new browser window. Open up the GitHub repository (e.g., `https://github.com/uqtlee17/hsu-astro-aws/`) in a new window. Navigate to `Settings` and click on `Secruity and quality > Secrets and variables > Actions` in the left menu.

In the `Secrets` tab, under the `Repository secrets` section, you will need to create two secrets by clicking `New repository secret`:

| Repository Secret Name | Value | Value Location in AWS | Example |
| --- | --- | --- | --- |
| AWS_ACCESS_KEY_ID | The Access Key ID of the IAM user you created | In the page you left open in the previous step, in the `Access key` section, under `Access key ID` | `AKIAIOSFODNN7EXAMPLE` |
| AWS_SECRET_ACCESS_KEY | The Secret Access Key of the IAM user you created | In the page you left open in the previous step, in the `Access key` section, under `Secret access key` | `Rs0cnuhi3LC81mHYu03hcd18A23Mrt3Y7vkVQNrt` |

In the `Variables` tab, you will need to create four variables by selecting `New repository secret`:

| Repository Variable Name | Value | Value Location in AWS | Example |
| --- | --- | --- | --- |
| AWS_REGION | Your AWS region | Region (top right dropdown in Console) | `ap-southeast-2` (Sydney) |
| AWS_ROLE_ARN | The ARN of the role you created previously | IAM > Access Management > Roles > Role name > Summary > ARN | `arn:aws:iam::123456789123:role/GitHubActions-S3-CloudFront-Role` |
| S3_BUCKET_NAME | The name of your S3 bucket | S3 > Buckets > General purpose buckets > Name | `hsu-static-website-assets` |
| CLOUDFRONT_DISTRIBUTION_ID | The ID of your CloudFront distribution | CloudFront > Distributions > ID | `E1ABC123ABC` |

Note: In the GitHub repository, create a file called `.github/workflows/deploy.yml`. Inside this file, paste the following exactly as is:
```yml
name: Deploy Astro Website to AWS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Install dependencies
        run: npm ci

      - name: Build Astro site
        run: npm run build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Sync files to S3
        run: |
          aws s3 sync ./dist s3://${{ vars.S3_BUCKET_NAME }} --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ vars.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

Websites built on Astro typically look like this:
```
hsu-astro-aws/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── apps/
│   └── hsu/
|       └── src/
│           ├── layouts/
│           │   └── Layout.astro
│           ├── pages/
│           │   ├── index.astro
│           │   └── about.astro
│           ├── content/    
│           │   └── blog/
│           │       ├── my-first-post.md
│           │       └── another-post.md
│           ├── styles/
│           │   └── global.css
│           └── assets/
│              ├── logo.png
│              └── hero.jpg
│   └── [second static website]/     
│       └── src/
│           ├── layouts/
│           │   └── Layout.astro
│           ├── pages/
│           │   ├── index.astro
│           │   └── about.astro
│           ├── content/    
│           │   └── blog/
│           │       ├── my-first-post.md
│           │       └── another-post.md
│           ├── styles/
│           │   └── global.css
│           └── assets/
│              ├── logo.png
│              └── hero.jpg
├── public/
│   └── favicon.ico
├── astro.config.mjs
├── package.json
├── README.md
└── .gitignore
```

`[repo_name]/apps/hsu/src/pages/index.astro` is the file that gets compiled into `index.html` at the root of your output folder when built.

**Expected result:** If you commit changes to the GitHub repository, under the `Actions` tab in the GitHub repository, you will see that the `Deploy Astro Website to AWS` workflow has started automatically. The status will start off as a yellow circle indicating that the status is `In progress`, then turn into a green circle with a white checkmark once completed. If you navigate to the domain name in a browser (e.g., `d298rvk2bhyrui.cloudfront.net`), you will see the changes you made to the website.

### Step 5 — Build the Contact Form Submission Flow

#### Step 5.1 — Create SES Identity
First, we will set up the `Amazon SES`. This is the mail carrier that takes the processed data and sends it to your specified inbox. We will have to verify our email address(es) here to prove that we own the email address(es).

In the Amazon Console navigate to the `Amazon Simple Email Service` service. Ensure that the `Sydney` region is selected. 

Click `Configuration > Identities` in the left menu. Click `Create identity`. In the `Identity details` section, under `Identity type`, select `Email address`. In the `Email address` field, enter your email address (e.g. `hsu.it@imb.uq.edu.au`). Click `Create identity`.

The specified email address will receive an automated email. Open that email and click the URL to verify the address.

**Expected result:** Under `Configuration > Identities`, the status of your email address should change from `Pending verification` to `Verified` (it will have a green checkmark next to it).

#### Step 5.2 — Create the Lambda Function and Assign Execution Role

Next, we will set up the `Lambda` service. The `Lambda` service executes the backend code to process the form data. It is akin to a small server that only "wakes up" for a few seconds when someone clicks `Submit`. It executes the script to format the email and then turns itself off.

In the Amazon Console navigate to the `Lambda` service. Ensure that the `Sydney` region is selected. Click `Create function`.

Select `Author from scratch`. In the `Basic information` section, under `Function name`, enter `ProcessContactForm`. In `Runtime`, ensure that the default, `Node.js 24.x` is selected. Leave all the other settings as default and click `Create function`.

You will be redirected to the `ProcessContactForm` function overview page. Click the `Configuration` tab. Click `Configuration > Permissions`. In the `Execution role`, under `Role name`, select the hyperlinkd role name (e.g., `ProcessContactForm-role-xxxxxxxx`).

In the new tab that gets opened, in the `Permissions policies` section, click the `Add permissions` dropdown and select `Attach policies`.

In `Other permissions policies` section, in the search box, type `AmazonSESFullAccess`. Check the corresponding checkbox and click `Add permissions`.

You will be redirected back to the execution role's page but this time the `AmazonSESFullAccess` policy will be listed under the `Permissions policies` section. You may close this tab.

Back in the `ProcessContactForm` overview page, click the `Code` tab. In the `EXPLORER` pane, double click `index.mjs` (this will open up the file in the editor pane). Replace the content with the following, replacing the placeholders (i.e., 2x `YOUR_VERIFIED_EMAIL@EXAMPLE.COM`) with your SES verified email (which was verified in Step 5.1):

```js
import { SESClient, SendEmailCommand } from "@aws-sdk/client-ses";
// Set the region to match your SES setup (Sydney)
const ses = new SESClient({ region: "ap-southeast-2" });

export const handler = async (event) => {
    // Parse the incoming data from the website form
    let body;
    try {
        body = JSON.parse(event.body);
    } catch (e) {
        body = event; 
    }

    const { name, email, message } = body;

    // Construct the email format
    const params = {
        Destination: {
            ToAddresses: ["YOUR_VERIFIED_EMAIL@example.com"], // CHANGE THIS
        },
        Message: {
            Body: {
                Text: { Data: `Name: ${name}\nEmail: ${email}\nMessage: ${message}` },
            },
            Subject: { Data: `New Contact Form Submission from ${name}` },
        },
        // The "Source" must be verified in SES to prove you aren't sending spam
        Source: "YOUR_VERIFIED_EMAIL@example.com", // CHANGE THIS
    };

    try {
        // Send the email via Amazon SES
        await ses.send(new SendEmailCommand(params));
        return {
            statusCode: 200,
            headers: {
                "Access-Control-Allow-Origin": "*", // Required for CORS (allows your website to talk to the API)
            },
            body: JSON.stringify({ message: "Email sent successfully!" }),
        };
    } catch (error) {
        console.error("Error sending email:", error);
        return {
            statusCode: 500,
            headers: { "Access-Control-Allow-Origin": "*" },
            body: JSON.stringify({ message: "Failed to send email." }),
        };
    }
};
```

In the `EXPLORER` pane, under `DEPLOY`, click `Deploy`.

#### Step 5.3 — Create API Gateway

Finally, we will create the `API Gateway` front door. This front door is what receives the `POST /contact` request from the contact form and passes it to the `Lambda` service.

In the Amazon Console navigate to the `API Gateway` service. Ensure that the `Sydney` region is selected. Click `Create HTTP API`.

In the `API details` section, under `API name`, enter `ContactFormAPI`. Under `Integrations`, click `Add integration`. Select `Lambda` from the dropdown.  `AWS Region` should be `ap-southeast-2` by default. In `Lambda function`, search for and select the `ARN` of your Lambda function (e.g., `ProcessContactForm`) you created in Step 5.2. Click `Next`.

In the `Configure routes` section, click `Add route`. In the `Method` column, select `POST`. In the `Resource` column, enter `/contact`. In the `Integration` column, select the `Lambda` function you just selected (e.g., `ProcessContactForm`). Click `Next`. Leave `Configure stages > Stage name` as its default (i.e., `$default`) and click `Next`. Click `Create`.

You will be redirected to the `ContactFormAPI` route's overview page. We now have to give explicit permission to allow the static website(s) to send data to your new API Gateway URL (this is blocked by default). To do so, click `Develop > CORS` on the left menu.

In the `Configure CORS` section, click `Configure`. In the `Access-Control-Allow-Origin` field, enter `*` and click the `Add` button next to it. In the `Access-Control-Allow-Headers` field, enter `content-type` and click the `Add` button next to it. In the `Access-Control-Allow-Methods` field, select `POST`. Click `Save`.

Click your API (e.g., `API: ContactFormAPI...(4d6r188pwc)`) in the left menu and select `Stages` on the left menu. In the `Stages for ContactFormAPI` section, there will be a `Invoke URL` (e.g., `https://4d6r188pwc.execute-api.ap-southeast-2.amazonaws.com`). Copy this URL. This is what you'll use in the static website's contact form to send data to the API Gateway.

#### Step 5.4 — Update Contact Page in Astro Website
Navigate to your GitHub repository (e.g., `https://github.com/uqtlee17/hsu-astro-aws/`). Navigate to the `apps/hsu/src/pages` folder and edit the `contact.astro` file. In the `script` section, replace the URL in the `fetch` call with the `Invoke URL` you copied from the API Gateway. The script should look similar to the following:

```js
<form id="contactForm">
  <label for="name">Name:</label>
  <input type="text" id="name" required />

  <label for="email">Email:</label>
  <input type="email" id="email" required />

  <label for="message">Message:</label>
  <textarea id="message" required></textarea>

  <button type="submit">Send Message</button>
</form>

<!-- A simple div to show success or error messages to the user -->
<div id="formStatus" style="margin-top: 15px; font-weight: bold;"></div>
```

It is important that your input fields have matching `id` attributes (`name`, `email`, `message`) because the `AWS Lambda` backend will be looking specifically for those exact `id`s to extract the data.

In the `contact.astro` file, add the following script above the `ContactUs` widget to present the `Sending...` or `Message sent successfully!` notification:
```html
<div id="formStatus" style="text-align: center; color: #d97706; font-size: 1.2rem; margin-top: 2rem; min-height: 1.5rem;"></div>
```

At the very bottom of `contact.astro`, paste the following script to stop the page from refreshing and skilently sends the data to AWS in the background, replacing the `YOUR_API_GATEWAY_INVOKE_URL` placeholder with the actual Invoke URL of your API Gateway from the previous step (e.g., `https://abcdefg123.execute-api.ap-southeast-2.amazonaws.com/contact`):

```html
<script>
  const form = document.getElementById('contactForm');
  const statusDiv = document.getElementById('formStatus');

  form.addEventListener('submit', async (e) => {
    e.preventDefault(); // Prevents the default page refresh

    // 1. Gather the data from the input fields
    const data = {
      name: document.getElementById('name').value,
      email: document.getElementById('email').value,
      message: document.getElementById('message').value
    };

    statusDiv.innerText = "Sending message...";

    try {
      // 2. Send the POST request to API Gateway
      // Example: https://abcdefg123.execute-api.ap-southeast-2.amazonaws.com/contact
      const response = await fetch('YOUR_API_GATEWAY_INVOKE_URL/contact', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(data)
      });

      // 3. Handle the response
      if (response.ok) {
        statusDiv.innerText = "Message sent successfully!";
        statusDiv.style.color = "green";
        form.reset(); // Clear the form fields
      } else {
        statusDiv.innerText = "Failed to send. Please try again.";
        statusDiv.style.color = "red";
      }
    } catch (error) {
      console.error(error);
      statusDiv.innerText = "An error occurred while sending the message.";
      statusDiv.style.color = "red";
    }
  });
</script>
```

Click `Commit changes`.

**Expected result:** You should be able to submit a contact form, and the form should submit successfully and the data should be sent to the listed inbox.

### Step 6 — Custom Domain Integration
Now you can link a custom domain to your static website. 

TODO











### Final Step — Confirm deployment complete

- [ ] Access the website(s) using the purchased (or CloudFront distribution's) domain name(s).
- [ ] Push changes to GitHub repository and view the changes in the website(s).
- [ ] Submit a contact form and receive the email in listed inbox.

---

## 7. Validation & Smoke Testing

| # | Check | Command / Method | Expected Result | Pass/Fail |
| --- | --- | --- | --- | --- |
| 1 | Service health endpoint | `curl .../health` | `200 OK` | ☐ |
| 2 | Key user flow (e.g., login) |  |  | ☐ |
| 3 | Downstream integration |  |  | ☐ |
| 4 | Error rate (last 15 min) | Dashboard link | Below baseline threshold | ☐ |
| 5 | Latency (p95/p99) | Dashboard link | Within SLO | ☐ |
| 6 | Logs free of new error signatures |  |  | ☐ |

**Bake time before declaring success:** `___ minutes`

---

## 8. Rollback Plan

**Rollback trigger criteria** *(be specific — don't leave this to judgment under pressure):*

- Error rate exceeds `___%` for `___` minutes
- p99 latency exceeds `___ms`
- Any data-integrity alert fires
- Health checks failing on `___%` of instances

**Rollback procedure:**

```bash
# exact commands to revert — pinned to prior known-good version/hash
```

**Expected result after rollback:** **Rollback validation:** *re-run relevant checks from Section 7* **Data/schema rollback (if applicable):** *migration reversal steps, backup restore procedure*

**Rollback decision authority:** who can call it, and can anyone on-call call it unilaterally?

---

## 9. Post-Deployment

- [ ] Deployment marked complete in change ticket
- [ ] Version/config recorded in deployment log (Section 11)
- [ ] Monitoring watched for extended bake period: `___ hours`
- [ ] Temporary access/credentials revoked
- [ ] Feature flags cleaned up / defaults restored
- [ ] Stakeholders and status page updated with completion
- [ ] Runbook updated with any deviations encountered (Section 10)

---

## 10. Deviations / Notes From This Run

*Filled in during/after execution — capture anything that didn't go per plan so the template improves next time.*

| Time | Step | Deviation / Issue | Resolution |
| --- | --- | --- | --- |
|  |  |  |  |

---

## 11. Deployment Log

| Timestamp | Action | Executed By | Result |
| --- | --- | --- | --- |
|  | Go/No-Go called |  |  |
|  | Deployment started |  |  |
|  | Step X completed |  |  |
|  | Validation passed |  |  |
|  | Deployment closed |  |  |

---

## 12. Post-Incident (only if rollback or failure occurred)

- [ ] Incident ticket opened
- [ ] Postmortem scheduled
- [ ] Root cause documented
- [ ] Action items tracked

---

## Appendix

- **Useful links:** dashboards, logs, architecture docs, prior runbooks
- **Glossary:** define any service-specific terms/acronyms used above
- **Runbook change history:**

| Version | Date | Author | Change |
| --- | --- | --- | --- |
| 1.0 |  |  | Initial template |
