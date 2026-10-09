# AWS IAM Access Management and Governance Lab

A hands-on AWS IAM project that manages employee access by job role. Users get permissions through groups, MFA is enforced, and every change is ticketed, tested and reviewed.

> All employee names, managers and approvers are fictional. Account identifiers are redacted in the evidence.

**Full walkthrough with all screenshots:** [Project presentation (PDF)](Aws-iam-access-governance-lab.pdf.pdf)

## The problem

- Employees get more access than their job needs
- People who change teams keep old permissions (privilege creep)
- Accounts of people who leave stay active
- Nobody can show who approved access, or why

**Who has it:** security and IT teams, approving managers and auditors.

## The solution

- Groups (RBAC) instead of individual permissions
- Least-privilege custom policies in place of broad AWS managed policies
- MFA enforced for the department groups through an explicit-deny policy
- Joiner / Mover / Leaver process with tickets, approvals and verification
- CloudTrail event history and the IAM Policy Simulator for troubleshooting
- A temporary role for short-lived access, and a periodic access review

## What I built

| Item | Details |
|---|---|
| Users | `alice` (Finance, then Support), `bob` (Developer), `charlie` (Support, then offboarded), `aws-suri` (admin) |
| Groups | `Finance-Group`, `Developer-Group`, `Support-Group`, `LabAdmins` |
| Custom policies | `FinanceS3LabReadOnly`, `DeveloperEC2LabReadOnly`, `SupportCloudWatchLabReadOnly`, `RequireMFA`, `AllowAssumeFinanceS3ReadRole` |
| Role | `FinanceS3ReadRole`: temporary, read-only access to one S3 bucket |
| Controls | MFA, password policy (12+ characters, 90-day expiry, last 5 passwords blocked) |
| Tracking | Ticket log (IAM-001 to IAM-007), access review, screenshot evidence |

## Architecture

```
Users  ->  Groups  ->  Policies  ->  Resources (S3, EC2, CloudWatch)
                          |
                  RequireMFA (explicit deny without MFA)

bob --AssumeRole--> FinanceS3ReadRole --> finance S3 bucket (temporary, read-only)

CloudTrail event history records management activity
```

Rule: **User -> Group -> Policy -> Resource.** Nothing is attached directly to a user.

## Evidence

### Users, groups and least privilege

![Users and groups](evidence/01-groups-and-users.png)

*Four users and four groups. Access comes only from group membership.*

![Custom policies attached to groups](evidence/02-least-privilege-after.png)

*Custom least-privilege policies replaced the broad AWS managed policies.*

### MFA enforcement

![RequireMFA policy](evidence/03-requiremfa-policy.png)

*RequireMFA denies everything except MFA setup when `aws:MultiFactorAuthPresent` is false.*

![Bob blocked without MFA](evidence/04-mfa-blocked.png)

*Bob is blocked without MFA, and works after MFA is enrolled.*

### Joiner, mover, leaver

![IAM-001 joiner](evidence/05-iam001-joiner.png)

*IAM-001: alice joins Finance-Group. The finance bucket is allowed and EC2 is denied.*

![IAM-002 mover](evidence/06-iam002-mover.png)

*IAM-002: Finance-Group removed first, then Support-Group added.*

![IAM-003 leaver](evidence/07-iam003-leaver.png)

*IAM-003: charlie's console access is disabled and no groups or policies remain.*

### Troubleshooting

![IAM-004 CloudTrail event](evidence/08-iam004-cloudtrail.png)

*IAM-004: CloudTrail showed the denied call. `ec2:DescribeVolumes` was missing from the policy, so I added it.*

![FinanceS3ReadRole](evidence/09-iam005-role.png)

*IAM-005 and IAM-006: bob uses `FinanceS3ReadRole` for temporary access. The first switch failed for lack of `sts:AssumeRole`, and a later S3 read failed because the policy named one file. Both are fixed.*

### Access review

![RequireMFA re-attached to Developer-Group](evidence/10-access-review-f02.png)

*The review found `RequireMFA` missing from `Developer-Group`. I re-attached it.*

| ID | Finding | Status |
|---|---|---|
| F-01 | `bob` had `IAMUserChangePassword` attached directly | Closed |
| F-02 | `RequireMFA` was missing from `Developer-Group` | Closed (cause not traced) |
| F-03 | `charlie`'s disabled account still exists | Open (delete after retention period) |

## Tickets

| Ticket | Type | User | Result |
|---|---|---|---|
| IAM-001 | Joiner | alice | Added to Finance-Group, MFA set up |
| IAM-002 | Mover | alice | Finance-Group out, Support-Group in |
| IAM-003 | Leaver | charlie | Console, MFA and group removed |
| IAM-004 | Incident | bob | Added `ec2:DescribeVolumes` |
| IAM-005 | Role access | bob | Added `sts:AssumeRole` permission |
| IAM-006 | Incident | bob | Object resource set to the bucket's objects |
| IAM-007 | Security | all | `RequireMFA` and password policy |

## Results

- 7 tickets closed, 3 incidents resolved
- 2 control gaps found in the review and fixed; 1 pending (account deletion)
- No active access keys; MFA active for every user who can sign in

## What I learned

- An explicit deny always beats an allow
- Role access needs permission on both sides: the user and the role's trust policy
- Remove old access first when someone changes teams
- Periodic reviews catch gaps that tickets miss

## Limitations

- Single AWS account, small user set, manual work in the AWS Console
- Approvals are simulated; standing admin access remains and is reviewed, not removed
- Active Directory, Entra ID, SSO and SailPoint are studied as concepts only, with no hands-on claim

## Future improvements

- Automate joiner/mover/leaver with scripts or Terraform
- EventBridge alerts when IAM policies change
- Easier MFA self-enrolment, time-limited admin access, SSO and governance tooling

## Services used

AWS IAM, S3, EC2 (view only), CloudWatch (view only), CloudTrail, IAM Policy Simulator, IAM Credential Report, authenticator-app MFA
