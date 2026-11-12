# Infrastructure as Code with CloudFormation and GitHub Actions

**Name:** Asif Istiaque
**Repository:** https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions
**Region:** US West (Oregon), us-west-2
**Date:** 12 November 2026

---

## 1. What I made

In this project I built a small server on AWS without clicking around in the AWS console. I wrote the server as a text file (CloudFormation). Then I made GitHub Actions deploy that file for me every time I push to the `main` branch.

The server is an Ubuntu EC2 machine that runs Nginx. When I open its public IP in the browser, it shows a small web page that says "Hello from bongoDev!".

I also made one small change, pushed it, and checked that AWS updated the same stack and did not make a new one. At the end I deleted everything.

Here are the names I used:

- Stack: `bongodev-github-iac-dev`
- IAM policy: `bongodev-github-iac-lab-policy`
- IAM role: `bongodev-github-iac-deploy-role`
- GitHub variables: `AWS_ROLE_ARN` and `AWS_REGION`

---

## 2. How it works

```text
git push  ->  GitHub Actions  ->  asks AWS for a short key (OIDC)
                                         |
                                         v
                              CloudFormation stack
                                         |
                                         v
                       VPC + subnet + security group + EC2 (Nginx)
```

When I push to `main`, GitHub starts a workflow. The workflow does not have any password. It shows AWS a special proof that says "I am a workflow from Asif's repo on the main branch". AWS checks the proof and gives a temporary key that lasts for one hour at most. With that key, the workflow runs `aws cloudformation deploy`, and CloudFormation builds or updates the server.

---

## 3. AWS setup

### The OIDC provider

First I told AWS to trust GitHub. I added an identity provider in IAM for `token.actions.githubusercontent.com`, with the audience `sts.amazonaws.com`.

![Figure 1: The OIDC provider in IAM](report-screenshots/01-iam-oidc-provider.png)

*Figure 1: AWS now knows GitHub as a trusted identity provider.*

### The policy

Next I made a policy with only the permissions this lab needs: work with CloudFormation, create the EC2 and network parts, and read the Ubuntu image name from Systems Manager. I did not give it full admin power.

![Figure 2: The IAM policy](report-screenshots/02-iam-policy-created.png)

*Figure 2: The policy `bongodev-github-iac-lab-policy`.*

### The role

Then I made the role that GitHub Actions uses. The important part is the trust policy. It says that only my repository, and only the `main` branch, can use this role. So another repository cannot use it, even if someone knows its name.

![Figure 3: The role with its permission and trust policy](report-screenshots/03-iam-role-permissions.png)

*Figure 3: The role has my policy attached, and the trust policy is updated.*

---

## 4. GitHub setup

In the repository settings I added two variables: the role address (`AWS_ROLE_ARN`) and the region (`AWS_REGION`). They are not passwords, so a variable is fine. I covered the role address in the screenshot because the repository is public.

![Figure 4: GitHub variables](report-screenshots/04-github-variables.png)

*Figure 4: The two repository variables.*

The repository has the CloudFormation file, the workflow file, the IAM policy files and this report.

![Figure 5: Files in the repository](report-screenshots/05-github-repo-files.png)

*Figure 5: The files in my GitHub repository.*

---

## 5. First deployment

After my first push, the workflow ran. Every step is green and the whole run took 1 minute 16 seconds.

![Figure 6: The first run, all steps green](report-screenshots/06-actions-run1-steps-green.png)

*Figure 6: Run #1 passed.*

At the end, the workflow prints the outputs of the stack: the instance ID, the VPC ID and the public IP.

![Figure 7: Outputs of the first run](report-screenshots/07-actions-run1-outputs.png)

*Figure 7: The instance ID is `i-0f9b03460f4eada58` and the public IP is 184.34.218.237.*

In the AWS console the stack shows `CREATE_COMPLETE`.

![Figure 8: The stack was created](report-screenshots/08-stack-create-complete.png)

*Figure 8: `CREATE_COMPLETE` for `bongodev-github-iac-dev`.*

The stack made 9 resources, and all of them are `CREATE_COMPLETE`.

![Figure 9: The 9 resources](report-screenshots/09-stack-resources-created.png)

*Figure 9: VPC, subnet, internet gateway, route table, security group and the EC2 instance are all there.*

Finally I opened the public IP in my browser. Nginx answers with my page.

![Figure 10: The live website](report-screenshots/10-website-live.png)

*Figure 10: "Hello from bongoDev!" with Environment: dev.*

---

## 6. Second push: updating the stack

For the second change I did not touch the web page. I only changed one tag on the EC2 instance. The tag `Course` was changed from `bongoDev-DevOps` to `bongoDev-DevOps-Homework-V2`. I pushed this as commit `9072ad4`.

I will explain below why I changed a tag and not the HTML.

![Figure 11: The commits on main](report-screenshots/11-commits-page.png)

*Figure 11: Three commits. `81f16dc` is the first deployment, `9072ad4` is the tag change and `a8ed5d3` is the fix I explain next.*

### What went wrong

After the push, the new run (Run #2) stayed in the `Queued` state and never started. I waited a long time, then I tried to cancel it, but the cancel did not work.

![Figure 12: Run #2 stuck in the queue](report-screenshots/12-actions-run2-queued.png)

*Figure 12: Run #2 was queued for many minutes, while Run #1 was already a success.*

![Figure 13: Cancel failed](report-screenshots/13-actions-run2-cancel-failed.png)

*Figure 13: GitHub said "Failed to cancel workflow".*

Then I looked at the GitHub status page. It was not my mistake. GitHub Actions had an incident with degraded availability that day.

![Figure 14: GitHub status page](report-screenshots/14-github-status-incident.png)

*Figure 14: The GitHub incident page.*

I also tried to force cancel the run with the GitHub API, but it answered with a `409` error saying the run "has not been queued yet". So the run was a ghost: it was visible, but GitHub could not really cancel it.

### The fix

My workflow has a `concurrency` group, which makes sure only one deployment runs at a time. The ghost run was still holding that group, so my new runs had to wait behind it. I renamed the group from `bongodev-cloudformation-dev` to `bongodev-cloudformation-dev-v2` and pushed again (commit `a8ed5d3`). Because the new group was free, Run #3 started at once and passed in 48 seconds.

I want to be honest here: the tag change itself is from commit `9072ad4`. Run #3 deployed it, because it used the latest code on `main`.

![Figure 15: Run #3 outputs](report-screenshots/15-actions-run3-outputs.png)

*Figure 15: Run #3 passed and printed the same instance ID `i-0f9b03460f4eada58`.*

### Proof that it was an update

The stack now shows `UPDATE_COMPLETE`. Only the EC2 instance has an update. The network parts were not touched.

![Figure 16: The stack was updated](report-screenshots/16-stack-update-complete.png)

*Figure 16: `UPDATE_COMPLETE`. It is the same stack and not a new one.*

On the EC2 page the new tag is there, and the instance ID is still the same. So AWS changed the tag on the same machine. It did not make a new server. The instance is Running with 3/3 checks passed.

![Figure 17: The new tag on the same instance](report-screenshots/17-ec2-tag-v2.png)

*Figure 17: `Course = bongoDev-DevOps-Homework-V2` on `i-0f9b03460f4eada58`.*

---

## 7. Clean up

I did not want to leave anything running, so I deleted the lab at the end.

First I deleted the stack. CloudFormation removes all 9 resources by itself. The stack list is now empty.

![Figure 18: No stacks left](report-screenshots/18-stacks-empty.png)

*Figure 18: Stacks (0).*

The EC2 instance is terminated.

![Figure 19: The instance is terminated](report-screenshots/19-ec2-terminated.png)

*Figure 19: The instance state is Terminated. The old tags are still shown for a short time.*

Then I deleted the IAM role. Without this role, no workflow, old or stuck, can create the stack again.

![Figure 20: The role is deleted](report-screenshots/20-iam-role-deleted.png)

*Figure 20: AWS shows "Role deleted".*

---

## 8. Problems I had

**A stuck workflow.** This was my biggest problem, and I explained it in section 6. The cause was a GitHub incident, and the fix was a new concurrency group name.

**The GitHub CLI.** I installed `gh` with winget, but the terminal said `command not found`. I had to open a new terminal. The login also failed the first time, because I pressed keys while it was waiting. I ran the login again and waited, and it worked.

**A wrong region name in the README.** The README says "us-west-2 (N. Virginia)". That is wrong. `us-west-2` is Oregon, and N. Virginia is `us-east-1`. I used Oregon.

**Private information.** My screenshots had my AWS account ID and the role address in them. Because the repository is public, I covered them before I added the pictures here.

---

## 9. Questions and answers

### 1. What is the difference between GitHub Actions and CloudFormation?

GitHub Actions is the worker that runs steps when something happens, like a push. CloudFormation is the tool that reads my template file and builds the AWS resources. In this project GitHub Actions starts the work, and CloudFormation does the building.

### 2. Why can a push deploy without me logging in to the AWS console?

Because the workflow logs in by itself. It uses the role I made, and AWS trusts my repository. So I only push my code, and the workflow does the rest.

### 3. What is OIDC, and why is it safer than saving access keys in GitHub?

OIDC lets GitHub prove who it is to AWS, and then AWS gives a temporary key. Nothing secret is saved in GitHub.

Access keys are long-lived. If someone steals them, they work until I notice and delete them. A temporary key from OIDC stops working after at most one hour, and the trust policy also limits it to my repository and my branch.

### 4. Why does the second push update the same stack?

The workflow always uses the same stack name, `bongodev-github-iac-dev`. CloudFormation sees that this stack already exists, so it compares my new file with the old one and changes only the difference. Here the difference was one tag, so only the EC2 instance was updated. Figure 16 and Figure 17 show this.

### 5. Is `CREATE_COMPLETE` the same as "Nginx is ready"?

No. `CREATE_COMPLETE` only means that AWS made the resources. The Nginx install runs later, inside the server, from the UserData script. So the server can exist for a minute or two before the web page works. If the page does not open at once, I wait a little and try again.

### 6. Why did I change a tag and not the HTML page?

The UserData script runs only the first time the server boots. If I only change the HTML in UserData, CloudFormation would not run it again on the old server. A tag change can be updated safely on the running server, so it is a clean way to prove that the update works without replacing the machine.

### 7. What should happen to the resources at the end?

They should be deleted, so there are no extra costs. I deleted the stack, which removes the VPC, the EC2 and the other resources, and then I deleted the role. The IAM policy and the GitHub workflow are the last things to clean up. The policy can be deleted in IAM, and the workflow can be disabled in the Actions tab. I did not take screenshots of those two steps.

---

## 10. Links

- Repository: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions
- Commit `81f16dc`: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/commit/81f16dc
- Commit `9072ad4`: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/commit/9072ad4
- Commit `a8ed5d3`: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/commit/a8ed5d3
- Run #1: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/actions/runs/43091447803
- Run #2 (the stuck one): https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/actions/runs/43096891087
- Run #3: https://github.com/Asifistiaque777/bongodev-cloudformation-github-actions/actions/runs/43187997019

---

## 11. Conclusion

The project worked from start to end. One push built a full server on AWS, and a second push updated the same stack without making a new machine. The stuck run was annoying, but it taught me how the concurrency group works and how to check the GitHub status page.

The most useful thing I learned is that OIDC gives a deploy pipeline a short-lived key, so I never have to save a secret in GitHub, and that the same template file can build, update and remove a whole setup.