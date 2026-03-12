# Microsoft Azure AD Integration with AWS using SAML 2.0

A complete, step-by-step guide for integrating Microsoft Azure Active Directory (Azure AD) with AWS accounts using SAML 2.0 federation for Single Sign-On (SSO).

![Azure AD](https://img.shields.io/badge/Azure-AD-blue?style=for-the-badge&logo=microsoft-azure)
![AWS](https://img.shields.io/badge/AWS-SSO-orange?style=for-the-badge&logo=amazon-aws)
![SAML](https://img.shields.io/badge/SAML-2.0-green?style=for-the-badge)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [How It Works](#-how-it-works)
- [Prerequisites](#-prerequisites)
- [Architecture](#-architecture)
- [Setup Guide](#-setup-guide)
  - [Part 1: Configure AWS](#part-1-configure-aws-identity-provider)
  - [Part 2: Configure Azure AD](#part-2-configure-azure-ad-enterprise-application)
  - [Part 3: Configure Role Mapping](#part-3-configure-role-mapping)
  - [Part 4: Test SSO](#part-4-test-sso)
- [Multi-Account Setup](#-multi-account-aws-setup)
- [Advanced Configuration](#-advanced-configuration)
- [Troubleshooting](#-troubleshooting)
- [Sample Documents](#-sample-documents)
- [Security Best Practices](#-security-best-practices)

---

## 🎯 Overview

### What This Integration Enables

**Before Integration:**
```
Users → Individual AWS accounts
- Separate credentials per account
- No centralized management
- Difficult to audit
- Password fatigue
```

**After Integration:**
```
Users → Azure AD → SAML → AWS
- Single sign-on (SSO)
- Centralized user management
- MFA from Azure AD
- Conditional access policies
- Audit trail in Azure AD
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Single Sign-On (SSO)** | Users log in once with Azure AD credentials |
| **Centralized Management** | Manage AWS access from Azure AD |
| **Enhanced Security** | Enforce MFA, conditional access, password policies |
| **Simplified Auditing** | Single audit log in Azure AD |
| **No AWS IAM Users** | Eliminate long-term credentials |
| **Role-Based Access** | Map Azure AD groups to AWS IAM roles |

### Common Use Cases

1. **Enterprise SSO** - Single login for Microsoft 365 and AWS
2. **Multi-Account AWS** - Federate access across multiple AWS accounts
3. **Developer Access** - Grant temporary AWS console/CLI access
4. **Contractor Management** - Easy onboarding/offboarding
5. **Compliance** - Meet audit and security requirements

---

## 🔄 How It Works

### SAML 2.0 Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         STEP-BY-STEP FLOW                             │
└──────────────────────────────────────────────────────────────────────┘

Step 1: User initiates login
┌─────────┐
│  User   │  Clicks "AWS Console" in Azure AD portal
└────┬────┘
     │
     ▼
┌─────────────────┐
│   Azure AD      │  Step 2: Azure AD authenticates user
│                 │  - Username/password
│                 │  - MFA (if required)
│                 │  - Conditional access checks
└────┬────────────┘
     │
     │ Step 3: Azure AD generates SAML assertion
     │ Contains:
     │ - User identity
     │ - AWS role ARN
     │ - Session duration
     │
     ▼
┌─────────────────┐
│   SAML          │  Step 4: SAML assertion sent to AWS
│   Assertion     │  Signed with Azure AD certificate
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│   AWS STS       │  Step 5: AWS validates SAML assertion
│   (Security     │  - Verifies signature
│    Token        │  - Checks certificate
│    Service)     │  - Validates role ARN
└────┬────────────┘
     │
     │ Step 6: AWS STS issues temporary credentials
     │ - Access Key ID
     │ - Secret Access Key
     │ - Session Token
     │ - Expiration (1-12 hours)
     │
     ▼
┌─────────────────┐
│   AWS Console   │  Step 7: User gets access to AWS Console
│   or CLI        │  with assumed role permissions
└─────────────────┘
```

### Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                        AZURE AD (Identity Provider)                 │
│                                                                     │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐    │
│  │   Users      │      │   Groups     │      │ Enterprise   │    │
│  │              │◄────►│              │◄────►│ Application  │    │
│  │ john@co.com  │      │ AWS-Admins   │      │ (AWS SSO)    │    │
│  │ jane@co.com  │      │ AWS-Devs     │      │              │    │
│  └──────────────┘      └──────────────┘      └──────┬───────┘    │
│                                                      │             │
└──────────────────────────────────────────────────────┼─────────────┘
                                                       │
                                            SAML 2.0 Assertion
                                            (Signed with certificate)
                                                       │
┌──────────────────────────────────────────────────────▼─────────────┐
│                           AWS ACCOUNT                               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │              IAM Identity Provider (SAML)                 │     │
│  │  - Provider Name: AzureAD                                │     │
│  │  - Metadata: Azure AD Federation Metadata XML            │     │
│  └───────────────────────┬──────────────────────────────────┘     │
│                          │                                         │
│                          │ Trusts                                  │
│                          ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    IAM Roles                              │     │
│  │                                                           │     │
│  │  ┌────────────────┐         ┌────────────────┐          │     │
│  │  │ Azure-Admin    │         │ Azure-ReadOnly │          │     │
│  │  │ Role           │         │ Role           │          │     │
│  │  │                │         │                │          │     │
│  │  │ Trust: AzureAD │         │ Trust: AzureAD │          │     │
│  │  │ Permissions:   │         │ Permissions:   │          │     │
│  │  │ AdministratorAccess      │ ReadOnlyAccess │          │     │
│  │  └────────────────┘         └────────────────┘          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Prerequisites

### Azure AD Requirements

**1. Azure AD Tenant**
- Azure AD Premium P1 or P2 (recommended for conditional access)
- OR Azure AD Free (basic SSO works)

**2. Permissions Needed**
- **Global Administrator** or **Application Administrator** role
- Ability to create Enterprise Applications
- Ability to manage users and groups

**3. Users and Groups**
```
Recommended Setup:
├── Group: AWS-Admins
│   └── Users: admin1@company.com, admin2@company.com
├── Group: AWS-Developers
│   └── Users: dev1@company.com, dev2@company.com
└── Group: AWS-ReadOnly
    └── Users: analyst1@company.com, analyst2@company.com
```

### AWS Requirements

**1. AWS Account**
- Account ID (12-digit number)
- IAM administrative access

**2. Permissions Needed**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateSAMLProvider",
        "iam:UpdateSAMLProvider",
        "iam:GetSAMLProvider",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:UpdateAssumeRolePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**3. Information to Collect**

| Item | Example | Where to Find |
|------|---------|---------------|
| **AWS Account ID** | 123456789012 | AWS Console → Top right → Account |
| **Azure AD Tenant ID** | a1b2c3d4-e5f6-7890-abcd-ef1234567890 | Azure Portal → Azure AD → Properties |
| **Azure AD Domain** | company.onmicrosoft.com | Azure Portal → Azure AD → Custom domains |

---

## 🏗️ Architecture

### Single AWS Account Setup

```
Azure AD
  └── Enterprise Application (AWS)
      └── User Assignment
          ├── AWS-Admins → AWS AdministratorAccess Role
          ├── AWS-Devs → AWS PowerUserAccess Role
          └── AWS-ReadOnly → AWS ReadOnlyAccess Role
              ↓
          AWS Account (123456789012)
          ├── IAM Identity Provider (AzureAD)
          └── IAM Roles
              ├── Azure-Admin-Role
              ├── Azure-Dev-Role
              └── Azure-ReadOnly-Role
```

### Multi-Account AWS Setup

```
Azure AD
  └── Enterprise Application (AWS)
      └── User Assignment
          ├── Group: Production-Admins
          ├── Group: Production-Devs
          ├── Group: Staging-Admins
          └── Group: Dev-Full-Access
              ↓
      ┌───────────┬───────────┬───────────┐
      │           │           │           │
AWS Prod     AWS Staging   AWS Dev     AWS Shared
(111111)     (222222)      (333333)    (444444)
  │            │             │           │
Each has IAM Provider + Roles
```

---

## 🚀 Setup Guide

---

## Part 1: Configure AWS Identity Provider

### Step 1.1: Download Azure AD Metadata

**Option A: Via Azure Portal (Recommended)**

1. Go to **Azure Portal** → **Azure Active Directory**
2. Click **Enterprise applications** → **New application**
3. Search for **"AWS"** or **"Amazon Web Services"**
4. Click **Amazon Web Services (AWS)** → **Create**
5. Wait for application to be created
6. Go to **Single sign-on** → Select **SAML**
7. In **Section 3: SAML Signing Certificate**, download:
   - **Federation Metadata XML**

Save as: `AzureAD-Federation-Metadata.xml`

**Option B: Via Direct URL**

```bash
# Replace {tenant-id} with your Azure AD Tenant ID
https://login.microsoftonline.com/{tenant-id}/federationmetadata/2007-06/federationmetadata.xml?appid=86bf67d5-69b7-4736-9d9f-4b5b7a7f9e6a

# Example:
https://login.microsoftonline.com/a1b2c3d4-e5f6-7890-abcd-ef1234567890/federationmetadata/2007-06/federationmetadata.xml?appid=86bf67d5-69b7-4736-9d9f-4b5b7a7f9e6a
```

Download this XML file.

### Step 1.2: Create IAM SAML Identity Provider in AWS

**Via AWS Console:**

1. Sign in to **AWS Console**
2. Navigate to **IAM** → **Identity providers** → **Add provider**
3. **Configure provider:**
   - **Provider type:** SAML
   - **Provider name:** `AzureAD`
   - **Metadata document:** Upload `AzureAD-Federation-Metadata.xml`
4. Click **Add provider**
5. Note the **Provider ARN:**
   ```
   arn:aws:iam::123456789012:saml-provider/AzureAD
   ```
<img width="780" height="419" alt="image" src="https://github.com/user-attachments/assets/41b04204-cbdb-45f7-9d1d-69644745b55a" />

**Via AWS CLI:**

```bash
# Create SAML provider
aws iam create-saml-provider \
  --name AzureAD \
  --saml-metadata-document file://AzureAD-Federation-Metadata.xml

# Output will include ARN:
# arn:aws:iam::123456789012:saml-provider/AzureAD
```

### Step 1.3: Create IAM Roles for Federated Users

**Role 1: Admin Role**

**Via AWS Console:**

1. IAM → **Roles** → **Create role**
2. **Trusted entity type:** SAML 2.0 federation
3. **SAML provider:** AzureAD
4. **Attribute:** SAML:aud
5. **Value:** `https://signin.aws.amazon.com/saml`
6. **Allow programmatic and AWS Management Console access** (check both)
7. Click **Next**
8. **Attach permissions:** `AdministratorAccess`
9. **Role name:** `Azure-Admin-Role`
10. **Description:** `Administrator access via Azure AD SSO`
11. Click **Create role**

**Via AWS CLI:**

```bash
# Create trust policy file
cat > azure-admin-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/AzureAD"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name Azure-Admin-Role \
  --assume-role-policy-document file://azure-admin-trust-policy.json \
  --description "Administrator access via Azure AD SSO"

# Attach admin policy
aws iam attach-role-policy \
  --role-name Azure-Admin-Role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Get role ARN
aws iam get-role --role-name Azure-Admin-Role --query 'Role.Arn'
# Output: arn:aws:iam::123456789012:role/Azure-Admin-Role
```

**Role 2: Developer Role**

```bash
# Trust policy (same as admin)
cat > azure-dev-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/AzureAD"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name Azure-Developer-Role \
  --assume-role-policy-document file://azure-dev-trust-policy.json \
  --description "Developer access via Azure AD SSO"

# Attach PowerUser policy
aws iam attach-role-policy \
  --role-name Azure-Developer-Role \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess
```

**Role 3: Read-Only Role**

```bash
# Trust policy (same as above)
cat > azure-readonly-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/AzureAD"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name Azure-ReadOnly-Role \
  --assume-role-policy-document file://azure-readonly-trust-policy.json \
  --description "Read-only access via Azure AD SSO"

# Attach ReadOnly policy
aws iam attach-role-policy \
  --role-name Azure-ReadOnly-Role \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### Step 1.4: Collect Role ARNs

```bash
# List all role ARNs
aws iam list-roles --query 'Roles[?starts_with(RoleName, `Azure-`)].Arn'

# You should have:
# arn:aws:iam::123456789012:role/Azure-Admin-Role
# arn:aws:iam::123456789012:role/Azure-Developer-Role
# arn:aws:iam::123456789012:role/Azure-ReadOnly-Role

# Also save the SAML Provider ARN:
# arn:aws:iam::123456789012:saml-provider/AzureAD
```

**Save these ARNs - you'll need them in Azure AD configuration!**

---

## Part 2: Configure Azure AD Enterprise Application

### Step 2.1: Configure Basic SAML Settings

1. Go to **Azure Portal** → **Azure Active Directory** → **Enterprise applications**
2. Click on your **Amazon Web Services (AWS)** application
3. Click **Single sign-on** → **SAML**
4. Click **Edit** on **Section 1: Basic SAML Configuration**

**Configure:**

| Field | Value |
|-------|-------|
| **Identifier (Entity ID)** | `https://signin.aws.amazon.com/saml` |
| **Reply URL (Assertion Consumer Service URL)** | `https://signin.aws.amazon.com/saml` |
| **Sign on URL** | `https://signin.aws.amazon.com/saml` |
| **Relay State** | `https://console.aws.amazon.com/` |
| **Logout URL** | (Leave empty) |

Click **Save**

### Step 2.2: Configure User Attributes & Claims

1. In the same SAML configuration page, click **Edit** on **Section 2: User Attributes & Claims**

**Required Claims:**

| Claim name | Source | Value |
|------------|--------|-------|
| **Unique User Identifier (Name ID)** | Attribute | `user.userprincipalname` |
| **Name ID format** | | `Persistent` |

**Additional Claims to Add:**

Click **Add new claim** for each:

**Claim 1: RoleSessionName**
```
Name: https://aws.amazon.com/SAML/Attributes/RoleSessionName
Source: Attribute
Source attribute: user.userprincipalname
```

**Claim 2: SessionDuration**
```
Name: https://aws.amazon.com/SAML/Attributes/SessionDuration
Source: Transformation
Transformation: Choose one of:
  - 3600 (1 hour)
  - 7200 (2 hours)
  - 14400 (4 hours)
  - 28800 (8 hours)
  - 43200 (12 hours - maximum)

Recommended: 28800 (8 hours)

Configuration:
- Transformation: Extract substring
- Parameter 1: "28800"
- Parameter 2: 0
- Parameter 3: 5
```

**OR simpler:**
```
Source: Constant
Value: 28800
```

**Claim 3: Role (CRITICAL - Maps to AWS Roles)**
```
Name: https://aws.amazon.com/SAML/Attributes/Role
Source: Attribute
Source attribute: user.assignedroles

Note: We'll configure this mapping in the next step
```

Click **Save**

### Step 2.3: Configure Role Mapping

This is the **MOST IMPORTANT** step - mapping Azure AD groups to AWS roles.

**Option A: Using App Roles (Recommended)**

1. Go to **Azure AD** → **Enterprise applications** → Your AWS app → **App roles**
2. Click **Create app role**

**Role 1: AWS Admin**
```
Display name: AWS-Admin-Role
Value: arn:aws:iam::123456789012:role/Azure-Admin-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
Description: AWS Administrator Access
Allowed member types: Users/Groups
Enable this app role: Yes
```

**Role 2: AWS Developer**
```
Display name: AWS-Developer-Role
Value: arn:aws:iam::123456789012:role/Azure-Developer-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
Description: AWS Developer Access
Allowed member types: Users/Groups
Enable this app role: Yes
```

**Role 3: AWS Read-Only**
```
Display name: AWS-ReadOnly-Role
Value: arn:aws:iam::123456789012:role/Azure-ReadOnly-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
Description: AWS Read-Only Access
Allowed member types: Users/Groups
Enable this app role: Yes
```

**IMPORTANT FORMAT:**
```
{role-arn},{provider-arn}

Example:
arn:aws:iam::123456789012:role/Azure-Admin-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
```

**Option B: Using Azure AD Groups (Alternative)**

If you prefer group-based assignment:

1. Create Azure AD security groups:
   ```
   - AWS-Admins
   - AWS-Developers
   - AWS-ReadOnly
   ```

2. Add custom extension attribute to groups containing role ARN

### Step 2.4: Assign Users/Groups to Roles

1. Go to **Enterprise applications** → Your AWS app → **Users and groups**
2. Click **Add user/group**
3. **Select users or groups:**
   - Choose users or groups
4. **Select a role:**
   - Choose AWS-Admin-Role, AWS-Developer-Role, or AWS-ReadOnly-Role
5. Click **Assign**

**Example Assignments:**

| User/Group | App Role |
|------------|----------|
| admin@company.com | AWS-Admin-Role |
| Group: Developers | AWS-Developer-Role |
| Group: Analysts | AWS-ReadOnly-Role |

### Step 2.5: Download SAML Certificate

1. In the SAML configuration page, go to **Section 3: SAML Signing Certificate**
2. Download **Certificate (Base64)**
3. Save as `AzureAD-Certificate.cer`

This certificate is used by AWS to verify SAML assertions.

---

## Part 3: Configure Role Mapping

### Step 3.1: Verify Role Attribute Configuration

The role claim must be in this exact format:

```xml
<Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
  <AttributeValue>
    arn:aws:iam::123456789012:role/Azure-Admin-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
  </AttributeValue>
</Attribute>
```

### Step 3.2: Test SAML Assertion

1. In Azure AD SAML config, scroll to **Section 5: Test single sign-on**
2. Click **Test**
3. Click **Test sign in**
4. You should see a SAML response preview

**Check for:**
- Role attribute present
- Role ARN format correct
- RoleSessionName present
- SessionDuration present

### Step 3.3: Multi-Role Support (Optional)

If a user needs multiple roles, the Role attribute can contain multiple values:

```xml
<Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
  <AttributeValue>arn:aws:iam::111111111111:role/Prod-Admin,arn:aws:iam::111111111111:saml-provider/AzureAD</AttributeValue>
  <AttributeValue>arn:aws:iam::222222222222:role/Staging-Admin,arn:aws:iam::222222222222:saml-provider/AzureAD</AttributeValue>
  <AttributeValue>arn:aws:iam::333333333333:role/Dev-Admin,arn:aws:iam::333333333333:saml-provider/AzureAD</AttributeValue>
</Attribute>
```

User will be prompted to choose a role when logging in.

---

## Part 4: Test SSO

### Step 4.1: Test from Azure AD Portal

1. Sign in to https://myapps.microsoft.com as a test user
2. You should see **Amazon Web Services (AWS)** app tile
3. Click on the AWS tile
4. You should be redirected to AWS Console with the assigned role

**Expected Flow:**
```
My Apps Portal → Click AWS tile → Azure AD authenticates → 
SAML assertion sent → AWS validates → Console access granted
```

### Step 4.2: Test from AWS Direct SAML URL

```
https://signin.aws.amazon.com/saml
```

This will redirect to Azure AD for authentication.

### Step 4.3: Verify Role Assignment

Once logged in to AWS Console:

1. Top right corner should show:
   ```
   user@company.com @ Azure-Admin-Role
   ```

2. Click on your username → See assumed role details

3. Try performing an action (e.g., viewing EC2 instances)
   - Should work if role has permissions
   - Should fail if role doesn't have permissions

### Step 4.4: Test CLI Access (Optional)

For developers who need CLI/SDK access:

**Install aws-azure-login:**

```bash
npm install -g aws-azure-login
```

**Configure:**

```bash
aws-azure-login --configure

# Enter:
# - Azure Tenant ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
# - Azure App ID: (from Enterprise App)
# - Default Username: user@company.com
# - Default Role ARN: arn:aws:iam::123456789012:role/Azure-Developer-Role
# - Default Duration: 3600
```

**Login:**

```bash
aws-azure-login --profile azure-dev

# Browser opens for Azure AD authentication
# After auth, AWS credentials stored in ~/.aws/credentials
```

**Use AWS CLI:**

```bash
aws s3 ls --profile azure-dev
aws ec2 describe-instances --profile azure-dev
```

---

## 🏢 Multi-Account AWS Setup

### Scenario: Multiple AWS Accounts

```
Company has:
- Production Account (111111111111)
- Staging Account (222222222222)
- Development Account (333333333333)
```

### Step-by-Step Setup

**Step 1: Configure Each AWS Account**

Repeat Part 1 for each account:

**Production Account (111111111111):**
```bash
# Create SAML Provider
aws iam create-saml-provider \
  --name AzureAD \
  --saml-metadata-document file://AzureAD-Federation-Metadata.xml

# Create roles
aws iam create-role --role-name Azure-Prod-Admin-Role ...
aws iam create-role --role-name Azure-Prod-ReadOnly-Role ...
```

**Staging Account (222222222222):**
```bash
# Same process
aws iam create-saml-provider --name AzureAD ...
aws iam create-role --role-name Azure-Staging-Admin-Role ...
```

**Development Account (333333333333):**
```bash
# Same process
aws iam create-saml-provider --name AzureAD ...
aws iam create-role --role-name Azure-Dev-FullAccess-Role ...
```

**Step 2: Create App Roles in Azure AD**

```
AWS-Prod-Admin: arn:aws:iam::111111111111:role/Azure-Prod-Admin-Role,arn:aws:iam::111111111111:saml-provider/AzureAD

AWS-Prod-ReadOnly: arn:aws:iam::111111111111:role/Azure-Prod-ReadOnly-Role,arn:aws:iam::111111111111:saml-provider/AzureAD

AWS-Staging-Admin: arn:aws:iam::222222222222:role/Azure-Staging-Admin-Role,arn:aws:iam::222222222222:saml-provider/AzureAD

AWS-Dev-FullAccess: arn:aws:iam::333333333333:role/Azure-Dev-FullAccess-Role,arn:aws:iam::333333333333:saml-provider/AzureAD
```

**Step 3: Assign Users**

| User | Roles Assigned |
|------|----------------|
| SRE Team | AWS-Prod-Admin, AWS-Staging-Admin, AWS-Dev-FullAccess |
| Developers | AWS-Staging-Admin, AWS-Dev-FullAccess |
| QA Team | AWS-Staging-ReadOnly, AWS-Dev-FullAccess |

When users with multiple roles log in, they see a role selector:
```
Choose a role to access AWS:
○ Production - Azure-Prod-Admin-Role
○ Staging - Azure-Staging-Admin-Role
○ Development - Azure-Dev-FullAccess-Role
```

---

## 🔧 Advanced Configuration

### 1. Enforce MFA

**In Azure AD:**

1. **Azure AD** → **Security** → **Conditional Access** → **New policy**
2. **Name:** Require MFA for AWS
3. **Assignments:**
   - Users: Include all users
   - Cloud apps: Select Amazon Web Services (AWS)
4. **Access controls:**
   - Grant: Require multi-factor authentication
5. **Enable policy:** On

### 2. Set Session Duration

**In AWS IAM Role:**

```bash
# Maximum session duration (1-12 hours)
aws iam update-role \
  --role-name Azure-Admin-Role \
  --max-session-duration 43200  # 12 hours
```

**In Azure AD:**

In SAML SessionDuration claim, set value in seconds:
- 3600 = 1 hour
- 14400 = 4 hours
- 28800 = 8 hours
- 43200 = 12 hours (max)

### 3. Conditional Access Policies

**Block from Untrusted Locations:**

1. **Conditional Access** → **New policy**
2. **Name:** Block AWS from untrusted locations
3. **Assignments:**
   - Cloud apps: Amazon Web Services (AWS)
   - Locations: Any location → Exclude trusted IPs
4. **Access controls:** Block
5. **Enable policy:** Report-only (test first)

### 4. Custom Session Names

Instead of email, use employee ID:

**In Azure AD Claims:**
```
https://aws.amazon.com/SAML/Attributes/RoleSessionName
Value: user.employeeid
```

Result in AWS CloudTrail:
```
"userIdentity": {
  "type": "AssumedRole",
  "principalId": "AROA...:EMP123456",
  "arn": "arn:aws:sts::123456789012:assumed-role/Azure-Admin-Role/EMP123456"
}
```

### 5. Automated Role Provisioning

Use Microsoft Graph API to automate role assignment:

```powershell
# PowerShell example
$appRoleId = "..." # AWS-Admin-Role app role ID
$groupId = "..." # Azure AD group ID

New-AzureADGroupAppRoleAssignment `
  -ObjectId $groupId `
  -PrincipalId $groupId `
  -ResourceId $servicePrincipalId `
  -Id $appRoleId
```

---

## 🔍 Troubleshooting

### Issue 1: "Error: Your request included an invalid SAML response"

**Cause:** AWS cannot validate the SAML assertion

**Solutions:**

1. **Verify SAML Provider Metadata is up to date:**
   ```bash
   # Update metadata in AWS
   aws iam update-saml-provider \
     --saml-provider-arn arn:aws:iam::123456789012:saml-provider/AzureAD \
     --saml-metadata-document file://AzureAD-Federation-Metadata.xml
   ```

2. **Check certificate expiration:**
   - Download new certificate from Azure AD
   - Update in AWS

3. **Verify audience URL:**
   - Must be exactly: `https://signin.aws.amazon.com/saml`

---

### Issue 2: "User is not assigned to a role for the application"

**Cause:** User doesn't have an app role assigned

**Solution:**

1. Azure AD → Enterprise applications → AWS → Users and groups
2. Add user/group assignment
3. Select appropriate app role

---

### Issue 3: "Failed to assume role"

**Cause:** Role ARN format incorrect in Azure AD

**Check:**

Role attribute must be in exact format:
```
{role-arn},{provider-arn}

Correct:
arn:aws:iam::123456789012:role/Azure-Admin-Role,arn:aws:iam::123456789012:saml-provider/AzureAD

Wrong:
arn:aws:iam::123456789012:saml-provider/AzureAD,arn:aws:iam::123456789012:role/Azure-Admin-Role
(Order matters!)
```

**Solution:**

1. Azure AD → Enterprise App → App roles
2. Verify "Value" field has correct format
3. Order must be: role ARN, then provider ARN

---

### Issue 4: Multiple roles - no role selector appears

**Cause:** User has multiple roles but can't choose

**Solution:**

Ensure all role ARNs use the **same** provider ARN:

```
Correct:
Role 1: arn:aws:iam::111111:role/Admin,arn:aws:iam::111111:saml-provider/AzureAD
Role 2: arn:aws:iam::222222:role/Admin,arn:aws:iam::222222:saml-provider/AzureAD

Both use provider name "AzureAD"
```

---

### Issue 5: Session expires too quickly

**Cause:** SessionDuration set too low

**Solution:**

1. **In Azure AD** - Increase SessionDuration claim value
2. **In AWS** - Increase role max session duration:
   ```bash
   aws iam update-role \
     --role-name Azure-Admin-Role \
     --max-session-duration 43200
   ```

---

### Issue 6: Cannot access AWS CLI/SDK

**Cause:** SAML only works for console by default

**Solution:**

Use `aws-azure-login` or similar tool:

```bash
npm install -g aws-azure-login
aws-azure-login --configure
aws-azure-login --profile azure
```

OR use AWS SSO (separate setup from SAML federation)

---

### Issue 7: "Access Denied" after successful login

**Cause:** Role doesn't have required permissions

**Solution:**

1. Check IAM role policies:
   ```bash
   aws iam list-attached-role-policies --role-name Azure-Admin-Role
   ```

2. Test with a broader policy temporarily
3. Narrow down which permission is missing

---

## 📄 Sample Documents

### Sample 1: SAML Response (Decoded)

```xml
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol" 
                ID="_abc12345-6789-0def-1234-567890abcdef"
                Version="2.0"
                IssueInstant="2024-01-15T10:30:00.000Z"
                Destination="https://signin.aws.amazon.com/saml">
  
  <Issuer xmlns="urn:oasis:names:tc:SAML:2.0:assertion">
    https://sts.windows.net/a1b2c3d4-e5f6-7890-abcd-ef1234567890/
  </Issuer>
  
  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>
  
  <Assertion xmlns="urn:oasis:names:tc:SAML:2.0:assertion"
             ID="_1234abcd-5678-90ef-1234-567890abcdef"
             IssueInstant="2024-01-15T10:30:00.000Z"
             Version="2.0">
    
    <Issuer>https://sts.windows.net/a1b2c3d4-e5f6-7890-abcd-ef1234567890/</Issuer>
    
    <Subject>
      <NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:persistent">
        user@company.com
      </NameID>
      <SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
        <SubjectConfirmationData 
          NotOnOrAfter="2024-01-15T11:30:00.000Z"
          Recipient="https://signin.aws.amazon.com/saml"/>
      </SubjectConfirmation>
    </Subject>
    
    <Conditions NotBefore="2024-01-15T10:25:00.000Z"
                NotOnOrAfter="2024-01-15T11:30:00.000Z">
      <AudienceRestriction>
        <Audience>https://signin.aws.amazon.com/saml</Audience>
      </AudienceRestriction>
    </Conditions>
    
    <AttributeStatement>
      
      <!-- AWS Role mapping -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
        <AttributeValue>
          arn:aws:iam::123456789012:role/Azure-Admin-Role,arn:aws:iam::123456789012:saml-provider/AzureAD
        </AttributeValue>
      </Attribute>
      
      <!-- Session name -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/RoleSessionName">
        <AttributeValue>user@company.com</AttributeValue>
      </Attribute>
      
      <!-- Session duration (8 hours) -->
      <Attribute Name="https://aws.amazon.com/SAML/Attributes/SessionDuration">
        <AttributeValue>28800</AttributeValue>
      </Attribute>
      
      <!-- Additional user attributes -->
      <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress">
        <AttributeValue>user@company.com</AttributeValue>
      </Attribute>
      
      <Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name">
        <AttributeValue>John Doe</AttributeValue>
      </Attribute>
      
    </AttributeStatement>
    
    <AuthnStatement AuthnInstant="2024-01-15T10:30:00.000Z">
      <AuthnContext>
        <AuthnContextClassRef>
          urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
        </AuthnContextClassRef>
      </AuthnContext>
    </AuthnStatement>
    
  </Assertion>
  
</samlp:Response>
```

### Sample 2: AWS IAM Role Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/AzureAD"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

### Sample 3: Azure AD Federation Metadata (Excerpt)

```xml
<EntityDescriptor 
  xmlns="urn:oasis:names:tc:SAML:2.0:metadata"
  entityID="https://sts.windows.net/a1b2c3d4-e5f6-7890-abcd-ef1234567890/">
  
  <Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
    <!-- Digital signature -->
  </Signature>
  
  <RoleDescriptor 
    xmlns:fed="http://docs.oasis-open.org/wsfed/federation/200706"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    protocolSupportEnumeration="http://docs.oasis-open.org/wsfed/federation/200706"
    xsi:type="fed:SecurityTokenServiceType">
    
    <KeyDescriptor use="signing">
      <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
        <X509Data>
          <X509Certificate>
            MIIDPjCCAiqgAwIBAgIQVWmXY/+9RqFA/OG9kFulHDAJBgUrDgMCHQUAMC0xKzAp...
          </X509Certificate>
        </X509Data>
      </KeyInfo>
    </KeyDescriptor>
    
    <fed:SecurityTokenServiceEndpoint>
      <EndpointReference xmlns="http://www.w3.org/2005/08/addressing">
        <Address>
          https://login.microsoftonline.com/a1b2c3d4-e5f6-7890-abcd-ef1234567890/saml2
        </Address>
      </EndpointReference>
    </fed:SecurityTokenServiceEndpoint>
    
  </RoleDescriptor>
  
  <IDPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    
    <KeyDescriptor use="signing">
      <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
        <X509Data>
          <X509Certificate>
            MIIDPjCCAiqgAwIBAgIQVWmXY/+9RqFA/OG9kFulHDAJBgUrDgMCHQUAMC0xKzAp...
          </X509Certificate>
        </X509Data>
      </KeyInfo>
    </KeyDescriptor>
    
    <SingleLogoutService 
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      Location="https://login.microsoftonline.com/a1b2c3d4-e5f6-7890-abcd-ef1234567890/saml2"/>
    
    <SingleSignOnService 
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      Location="https://login.microsoftonline.com/a1b2c3d4-e5f6-7890-abcd-ef1234567890/saml2"/>
    
    <SingleSignOnService 
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
      Location="https://login.microsoftonline.com/a1b2c3d4-e5f6-7890-abcd-ef1234567890/saml2"/>
    
  </IDPSSODescriptor>
  
</EntityDescriptor>
```

---

## 🔒 Security Best Practices

### 1. Principle of Least Privilege

**DON'T:**
```
Everyone gets AdministratorAccess
```

**DO:**
```
- Admins → AdministratorAccess
- Developers → PowerUserAccess (no IAM changes)
- Analysts → ReadOnlyAccess
- Contractors → Custom policy (specific resources only)
```

### 2. Enable MFA

**In Azure AD:**
- Conditional Access → Require MFA for AWS app
- Per-user MFA settings

**In AWS:**
- IAM role policy to require MFA for sensitive actions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### 3. Monitor SAML Usage

**AWS CloudTrail:**
```json
{
  "eventName": "AssumeRoleWithSAML",
  "userIdentity": {
    "type": "SAMLUser",
    "principalId": "...:user@company.com"
  },
  "requestParameters": {
    "roleArn": "arn:aws:iam::123456789012:role/Azure-Admin-Role"
  }
}
```

**Set up CloudWatch alarm:**
```bash
# Alert on failed SAML authentications
aws cloudwatch put-metric-alarm \
  --alarm-name failed-saml-logins \
  --metric-name FailedSAMLAuthentication \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold
```

**Azure AD Sign-in Logs:**
- Monitor for failed sign-ins to AWS app
- Set up alerts for suspicious locations

### 4. Certificate Rotation

**Azure AD certificates expire!**

**Check expiration:**
1. Azure AD → Enterprise apps → AWS → Single sign-on
2. SAML Signing Certificate → Expiration date

**Rotate before expiry:**
1. Generate new certificate in Azure AD
2. Download new Federation Metadata XML
3. Update AWS SAML provider
4. Test
5. Remove old certificate

**Automate with reminder:**
- Set calendar reminder 30 days before expiry
- Or use Azure AD monitoring

### 5. Session Management

**Short sessions for production:**
```
Development: 8-12 hours
Staging: 4-8 hours
Production: 1-4 hours
```

**In Azure AD SessionDuration claim:**
```
Development: 43200 (12 hours)
Production: 14400 (4 hours)
```

### 6. Network Restrictions

**Limit AWS access to corporate IPs:**

In Azure AD Conditional Access:
```
Policy: Block AWS access from outside corporate network
Locations: Include "All", Exclude "Corporate IPs"
Cloud apps: Amazon Web Services
Access: Block
```

### 7. Regular Access Reviews

**In Azure AD:**
- Set up quarterly access reviews
- Review who has AWS access
- Remove inactive users

**In AWS:**
- Review CloudTrail for unused roles
- Delete roles not used in 90+ days

### 8. Break-Glass Accounts

**Always keep emergency access:**

In AWS:
- 1-2 IAM users with MFA
- Stored in secure vault
- Used only for emergency (SAML down)
- Monitored heavily

```bash
# Create break-glass user
aws iam create-user --user-name break-glass-admin

# Attach admin policy
aws iam attach-user-policy \
  --user-name break-glass-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Enable MFA
aws iam enable-mfa-device --user-name break-glass-admin ...

# Store credentials in vault (LastPass, 1Password, etc.)
```

---

## 📊 POC Checklist

### Planning Phase

- [ ] Collect AWS Account ID
- [ ] Collect Azure AD Tenant ID
- [ ] Identify test users (3-5 users)
- [ ] Create Azure AD test groups
- [ ] Define role requirements (Admin, Dev, ReadOnly)
- [ ] Schedule POC timeline (1-2 weeks)

### Setup Phase - Azure AD

- [ ] Create Enterprise Application (AWS)
- [ ] Configure Basic SAML settings
- [ ] Configure User Attributes & Claims
- [ ] Add RoleSessionName claim
- [ ] Add SessionDuration claim
- [ ] Add Role claim
- [ ] Download Federation Metadata XML
- [ ] Download SAML Certificate

### Setup Phase - AWS

- [ ] Create IAM SAML Provider
- [ ] Create IAM Role: Azure-Admin-Role
- [ ] Create IAM Role: Azure-Developer-Role
- [ ] Create IAM Role: Azure-ReadOnly-Role
- [ ] Collect all Role ARNs
- [ ] Collect SAML Provider ARN

### Configuration Phase

- [ ] Create App Roles in Azure AD
- [ ] Configure role mappings (role ARN + provider ARN)
- [ ] Assign test users to roles
- [ ] Verify SAML assertion format

### Testing Phase

- [ ] Test user 1: Admin role → Can access AWS console
- [ ] Test user 2: Developer role → Limited permissions work
- [ ] Test user 3: ReadOnly role → Cannot modify resources
- [ ] Test MFA enforcement (if configured)
- [ ] Test CLI access with aws-azure-login
- [ ] Test multi-account access (if applicable)
- [ ] Test role selector (for users with multiple roles)

### Validation Phase

- [ ] Verify session duration works correctly
- [ ] Check CloudTrail logs for SAML events
- [ ] Check Azure AD sign-in logs
- [ ] Test session expiration
- [ ] Test logout
- [ ] Verify no errors in logs

### Documentation Phase

- [ ] Document architecture
- [ ] Document role mappings
- [ ] Create user guide
- [ ] Document troubleshooting steps
- [ ] Plan rollout to production

---

## 🎯 Summary

This guide provides **complete, production-ready documentation** for integrating Azure AD with AWS using SAML:

✅ **Complete setup instructions** (AWS + Azure AD)  
✅ **Sample SAML documents** (response, metadata, trust policies)  
✅ **Multi-account configuration** (for enterprises)  
✅ **Advanced features** (MFA, conditional access, CLI)  
✅ **Troubleshooting guide** (7 common issues)  
✅ **Security best practices** (8 recommendations)  
✅ **POC checklist** (ready to execute)  

**POC Timeline:** 1-2 days for single account, 3-5 days for multi-account

**Next Steps:**
1. Follow Part 1-4 sequentially
2. Test with 1-2 users first
3. Expand to more roles/users
4. Implement security hardening
5. Roll out to production

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Maintained By:** Cloud Identity Team

For questions or issues, please open a GitHub issue.
