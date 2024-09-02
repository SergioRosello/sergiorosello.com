---
title: What I learned after a year of setting up and running Hashicorp Vault in production 
date: '2024-04-23'
summary: In this blog post, I attempt to distill the key lessons learned from using Hashicorp Vault in integration and production environments during the course of  a year.
---

# Preface

At Scalefast we are transitioning from monolith to a service-based architecture. The addition of Vault as our de-facto secret management system into this new paradigm has been one of the biggest initiative I have contributed to the company to date. I am happy to see, in retrospective the amount of knowledge I accumulated during this period, both technical and personal. It's incredible to look back and recognize the growth and development that took place, not just in terms of skills and expertise, but also in understanding myself and others better.

# What is Hashicorp Vault

So for those that don’t know, Vault is a system that stores sensitive data (Secrets, certificates, keys, or any other data) aimed for application use. Its main features reside in the centralization of secrets in a single secure location (Vault) that can be accessed securely using a wide variety of systems.

# Why we decided to use Vault

One of the main pain points we were having in the security team at Scalefast was the management of secrets in our source code. As a startup grows it tends to develop tech debt. For the security team this materialized in uncontrollable secret management. When we started the transition from monolith to service-based, we knew secret handling was a topic we wanted to focus on, so we did not repeat past mistakes.

After researching several options on how to handle secrets, we decided to go with vault because at the time, it seemed the most mature, flexible and platform agnostic. 

# Key takeaways from the vault project

## Buy-in from other departments

I believe being an individual contributor consists on finding a balance between delivering the highest possible engineering solution to a problem in a given time frame. It is not easy, but it may be safe. You are not expected to defend the whole system from which you are building a part of, or obtain buy-in from other teams that will eventually interface with the system you are building. 

As I led the implementation and integration of Vault into our new platform, I completely underestimated the amount of energy that I would have to invest into establishing Vault as the de-facto secrets management solution for our new platform. 

In retrospect, it is just as natural as [Newton’s first law of motion.](https://en.wikipedia.org/wiki/Newton%27s_laws_of_motion)

### Lesson learned:

Be prepared to defend the idea I believed in, but also be open enough to take external feedback into consideration, and flexible enough to let go or alter this idea if it truly does not meet the requirements from the users.

## Solving for everything with Vault (Anti-pattern)

> If the only tool you have is a hammer, it is tempting to treat everything as if it were a nail.
[Abraham Maslow](https://en.wikipedia.org/wiki/Abraham_Maslow)
> 

Be sure to define specific use cases for the new tool you are bringing into the system. 

When we were designing the CI/CD strategy for our applications alongside the DevOps team, we had the challenge to re-create the `.env` file - from where the application fetches runtime configuration and secrets (dynamically generated) - into the Kubernetes pod with environment-specific data.

We ended up converting the application `.env` file into secret keys and uploading them into a specific deployment path inside Vault to later re-create the `.env` file with every secret key/value in the specific vault secret path into the application’s expected `.env` location. 

This provided the developers with a great advantage (we thought). They could now alter the application from within the vault web interface, without the need to re-deploy. It turns out this “feature” was also a single point of failure, as now, anyone with access to the production Vault application deployment path, would be able to change the value, and potentially break the application.

This also meant (to the security team’s pity) that now, a lot more developers had to be able to access the Vault service, as application configuration was intertwined with the application secrets.

Did it work? Yes, it did.

Was it the ideal solution? No.

If we had spoken to the application owners, we may have run into the fact that they did not want to  be able to dynamically change `.env` values, at the same time, we would also reduce the number of people that needed access to Vault secrets to just one or two per application.

The easier solution would have been to **only use Vault for secret management.** Syncing secrets to the application at a specific path, and using a Kubernetes `ConfigMap` to feed the application the `.env`  file at the requested location. This reduces security exposure to secrets, and provides developers with the features they wanted. It also decouples secret management from application lifecycle, which is a bonus, even in Kubernetes.

### Lesson learned:

Gather user input, and have a clear purpose for the tool. Think twice about adding anything that is not part of the initial tool’s purpose.

## Production-grading Vault

For the POC, we used bash scripts and feature-specific environments, so we could be certain to replicate anything we did. I learned this from Kris Nova on one of her streams.

For production however, we decided to go with the `IaC` approach. We created a Terraform module that held the core Vault configuration and two Terraform projects that imported this module with environment-specific configuration such as bound claims or `OIDC` groups ID’s.  Every project has its own staging environment, so we can test out releases and changes before going “live”. All this by simply tagging the commit in a specific way.

Up to now, this approach coupled with a strong CI/CD strategy has served us well. Updates to application configuration are always performed through Terraform, and updates to Vault versions have been automated through an update script run manually. Maybe we could automate it all, but honestly, the whole update process takes less than a minute a month, and we still have to check if there are any breaking changes in the update. I find  a bit more control in the update process is worth the time in this case.

One of the challenges we overcame to production-grade our Vault servers was to add `TLS` communications to every communication with the Vault servers. Of course, this was a must to have Vault running. Although it may not seem complicated at first, our security setup and our GitOps approach to deployments made it harder than initially anticipated. 

Our environment is protected under a VPN, and we run our own CA for our internal domains. These security layers are great but they have also introduced a layer of complexity we had to solve. 

We also deploy our applications in specific namespaces, created at deploy time for this application, therefore, we cannot rely on any data being stored in the namespace beforehand, but the Vault agent injector requires the CA in order to establish a secure `HTTPS` connection with the vault server, so we developed an application that listened to Kubernetes events, and synced the CA as a secret to the newly created namespace, so it would be available to the agent injector at application deployment time.

Having deployed Vault as High Availability both in integration and in production, we can confirm that when we proceed to update the Vault server’s version, there is still a 30 second down-time that we can’t resolve. We are not yet entirely sure to why this downtime occurs.

For us, production-grading Vault, and ensuring its security configuration is representative of our own security standards, has been where the majority of our efforts have resided. As it usually happens when the initial sprint of a project is over, it’s still not over, and maintenance and feature-addition is just as important.

### Lesson learned:

A project does not end at delivery time. It has to be maintained, adding maintenance time to the daily schedule. Factor in maintenance time reduction strategies into its development phase so they don’t come haunt you later on. 

## Conclusion

Every project has the ability to aggregate to the list of things you can do, but if you are willing to reflect a little, you can often see broader lessons to be learned.

Here is a TL;DR for those of you that are scarce of time.

- Gather user feedback early on into the development cycle
- Have a clear objective for the project you are building, don’t integrate anything that falls outside its same objective into it. There must be a better solution out there.
- Remember to factor in maintenance automation strategies into initial development time.
