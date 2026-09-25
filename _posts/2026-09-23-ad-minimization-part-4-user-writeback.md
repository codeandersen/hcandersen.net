---
title: "AD Minimization Part IV: User Writeback - Provisioning Cloud-Native Users to On-Premises Active Directory"
date: 2026-09-23 08:00:00 +0200
categories: [Microsoft, Entra ID, Active Directory]
tags: [entraId, activedirectory, cloudsync, writeback, cloudmigration, minimization, hybrididentity, provisioning]
published: false
---

*This is Part IV of the Active Directory Minimization series.*

In [Part I](/posts/ad-minimization-part-1-exchange-soa-conversion/) I converted Exchange mailbox attributes to cloud management, in [Part II](/posts/ad-minimization-part-2-group-soa-conversion/) I moved Group Source of Authority to Entra ID, and in [Part III](/posts/ad-minimization-part-3-exchange-writeback/) I closed the loop for Exchange attributes with writeback to on-premises Active Directory. Today, we take on the next natural step: **User Writeback**.

With Entra Cloud Sync's **Microsoft Entra ID to AD sync** configuration, you can now provision users that were *created in the cloud* down to your on-premises Active Directory. That means you can make Entra ID the place where users are born, and still keep the on-premises applications that depend on Active Directory working.

## The Scenario: Users Still Need Access to On-Premises Systems and Need an Account in the On-Premises Active Directory

The previous articles have been around Exchange and moving the SOA for Exchange object attributes to Exchange Online.
Microsoft just started rolling out user writeback to all tenants, and this article is about setting up the user writeback functionality and seeing a cloud user being provisioned in Active Directory.

## Why do you need this

**Make Entra ID the source of truth for new users**
- Create users once in Entra ID and let Cloud Sync handle the provisioning of the on-premises user.

**Keep dependent systems working**
- Cloud-created users get a real AD account with a SID, so on-premises Kerberos/NTLM apps and access to on-premises systems still work.

**One step further toward AD minimization**
- Combined with User SOA, Group SOA and Group Writeback, the synchronization gets switched from going from Active Directory to Entra ID to going from Entra ID to Active Directory (so kind of flipping the synchronization around from what we're used to today). Entra ID becomes the source of authority.
- The end goal is to be less dependent on Active Directory and be able to close Active Directory when the last dependent application on-premises is retired.

## Prerequisites: Getting Ready

Before you start, you'll need:

- **Microsoft Entra ID P1** licensing
- **Hybrid Identity Administrator role** (required for configuring Entra Cloud Sync)
- **Entra Cloud Sync Provisioning Agent** installed on a member server in your Active Directory domain (I installed mine in [Part III](/posts/ad-minimization-part-3-exchange-writeback/), so it's already in place)
- **Provisioning agent build 1.1.1373.0 or later**
- A **security group in Entra ID** that contains the cloud-native users you want provisioned to AD - this is your scoping filter

> **Note**: Provisioning *users* from Microsoft Entra ID to Active Directory is currently in **preview**. Provisioning *groups* is generally available. 

### Preparing a Test Group

For this walkthrough, I created a test security group named `sg_writeback_test` in Entra ID and added a single cloud-native user, Peter Svendsen, to it. Only members of this group will be written back to Active Directory.

![Test group sg_writeback_test in Entra ID](/assets/img/posts/user-writeback-walk-through-1.png)
![Cloud-native user added as a member](/assets/img/posts/user-writeback-walk-through-2.png)
_The scoping group and its single cloud-native member_

## Configuring User Writeback: Step by Step


Navigate to the [Microsoft Entra Admin Center](https://entra.microsoft.com). Go to **Identity** > **Hybrid management** > **Entra Connect** > **Cloud Sync** > **Configurations**.

Click **New configuration** and select **Microsoft Entra ID to AD sync**.

![New configuration - Microsoft Entra ID to AD sync](/assets/img/posts/user-writeback-walk-through-3.png)
_The three configuration types: AD to Entra ID, Entra ID to AD, and the EXO to AD attribute sync we used in Part III_

Select **Scoping filters** and click **Edit**. Under **User**, select **Enabled** and click **Next**.

![Scoping filters](/assets/img/posts/user-writeback-walk-through-4.png)

Select **User** and select **Enabled** and click **Next**

![Enable user provisioning](/assets/img/posts/user-writeback-walk-through-5.png)
_Enabling user objects for this configuration_

Keep the defaults on the next page and select **Next**.

Add the group that should be in target for on-premises writeback - in my case `sg_writeback_test` - and select **Next**.

![Select scoping group](/assets/img/posts/user-writeback-walk-through-6.png)

Keep the default and select **Next**
![Group added to scope](/assets/img/posts/user-writeback-walk-through-7.png)
_Only members of this group will be provisioned to Active Directory_

Select **Next**

![Keep defaults and continue](/assets/img/posts/user-writeback-walk-through-8.png)

Select **Next**

![Configure group membership](/assets/img/posts/user-writeback-walk-through-9.png)


Select **Edit attribute mapping** to review the default mappings, then select **Apply**, **Next**, and finally **Save**.

![Edit attribute mapping](/assets/img/posts/user-writeback-walk-through-10.png)
![Apply mapping](/assets/img/posts/user-writeback-walk-through-11.png)
![Next](/assets/img/posts/user-writeback-walk-through-12.png)
![Save the configuration](/assets/img/posts/user-writeback-walk-through-13.png)
_The default attribute mappings - we'll come back and tweak one of them shortly_

### Step 5: Review and Enable

Go to the **Overview** pane and select **Review and enable**.

![Review and enable](/assets/img/posts/user-writeback-walk-through-14.png)

Look through the configuration and select **Enable configuration**.

![Enable configuration](/assets/img/posts/user-writeback-walk-through-15.png)
_The configuration is now active_

## Testing: Provision on Demand

Instead of waiting for the scheduled cycle, I validated the configuration using **Provision on demand**. Go to **Provision on demand**, pick a user from the test group (Peter Svendsen) and select **Provision**.

![Provision on demand](/assets/img/posts/user-writeback-walk-through-16.png)

The result shows every attribute that was written to the new on-premises user object - and confirms that the user was **created** in Active Directory.

![Modified attributes - user created in Active Directory](/assets/img/posts/user-writeback-walk-through-17.png)
_User 'Peter@msonline.dk' was created in Active Directory. Note `msDS-ObjectSoa = Cloud` and `msDS-ExternalDirectoryObjectId` linking the AD object back to Entra ID_

### What the User Looks Like in Active Directory

In the local Active Directory, the user shows up in the **Users** container:

![User in Active Directory Users and Computers](/assets/img/posts/user-writeback-walk-through-18.png)
_Peter Svendsen_ca4b27f6f43b created in the Users container_

The account looks like this in the local AD:

![Account tab](/assets/img/posts/user-writeback-walk-through-19.png)
![General tab](/assets/img/posts/user-writeback-walk-through-20.png)
![Object tab](/assets/img/posts/user-writeback-walk-through-21.png)
![Attribute Editor](/assets/img/posts/user-writeback-walk-through-22.png)
_A fully populated AD account, including `objectSid`, `sAMAccountName`, `sn`, and `userPrincipalName`_

### Back in Entra ID: The On-Premises Security Identifier

And in the Entra admin center, the cloud user now has an **On-premises security identifier** - the SID of the AD account Cloud Sync just created. 

![On-premises security identifier populated in Entra ID](/assets/img/posts/user-writeback-walk-through-23.png)
_Peter Svendsen now has an on-premises SID, while still showing "On-premises sync enabled: No"_

## The Catch: The UPN Isn't Written Back Correctly by Default

If you looked closely at the **Account** tab in Active Directory earlier, you may have noticed that the user logon name ended up as `peter@world.local`, but it should have been `Peter@msonline.dk` to match the UPN in Entra ID.

![Wrong UPN suffix in Active Directory](/assets/img/posts/user-writeback-walk-through-24.png)
_The UPN suffix defaulted to the AD domain FQDN instead of the Entra ID UPN suffix_

The reason is the default expression on the `userPrincipalName` mapping:

```
IIF(IsPresent([onPremisesUserPrincipalName]), [onPremisesUserPrincipalName], Append(Item(Split([userPrincipalName], "@"), 1), Append("@", %DomainFQDN%)))
```

Fortunately this is easy to change in the attribute mapping.

### Fixing the UPN Mapping

Go to **Attribute mapping** in the configuration, find the `userPrincipalName` row and select the edit (pencil) button.

![Attribute mapping - edit userPrincipalName](/assets/img/posts/user-writeback-walk-through-25.png)

Here you'll see the default expression, applied **Only during object creation**.

![Default userPrincipalName expression](/assets/img/posts/user-writeback-walk-through-26.png)

Change the **Mapping type** to **Direct**.

![Change mapping type to Direct](/assets/img/posts/user-writeback-walk-through-27.png)

Set **Source attribute** to `userPrincipalName`, set **Apply this mapping** to **Always**, and select **Apply**.

![Direct mapping from userPrincipalName, applied Always](/assets/img/posts/user-writeback-walk-through-28.png)
_Direct mapping: the AD UPN will now mirror the Entra ID UPN, on creation and on every subsequent change_

Select **Save schema** and confirm with **OK**.

![Save schema](/assets/img/posts/user-writeback-walk-through-29.png)
![Confirm](/assets/img/posts/user-writeback-walk-through-30.png)

> **Note**: For this to work, `msonline.dk` must be a registered UPN suffix in your Active Directory forest. If it isn't, add it in **Active Directory Domains and Trusts** first.

### Provision Again and Verify

Run **Provision on demand** for the same user once more.

![Provision on demand again](/assets/img/posts/user-writeback-walk-through-31.png)

This time the `userPrincipalName` is written back correctly.

![userPrincipalName now correct in the provisioning result](/assets/img/posts/user-writeback-walk-through-32.png)

And in Active Directory the account now shows the correct UPN suffix:

![Correct UPN in Active Directory](/assets/img/posts/user-writeback-walk-through-33.png)
_User logon name is now Peter@msonline.dk - matching Entra ID_

## The Attributes Being Written Back

For reference, this is the full default attribute mapping for **Microsoft Entra ID to AD sync** users (with my `userPrincipalName` change applied):

![Full attribute mapping for user writeback](/assets/img/posts/user-writeback-walk-through-34.png)
_Default user attribute mappings from Entra ID to Active Directory_

| Target attribute (AD) | Source (Entra ID) | Mapping type |
|---|---|---|
| accountDisabled | `Not([accountEnabled])` | Expression |
| cn | `Append(Append(Left(Trim([displayName]), 51), "_"), Mid([objectId], 25, 12))` | Expression |
| co | `Trim([country])` | Expression |
| company | `Trim([companyName])` | Expression |
| department | `Trim([department])` | Expression |
| displayName | displayName | Direct |
| employeeID | employeeId | Direct |
| employeeType | employeeType | Direct |
| facsimileTelephoneNumber | `Trim([facsimileTelephoneNumber])` | Expression |
| givenName | `Trim([givenName])` | Expression |
| l | `Trim([city])` | Expression |
| manager | manager | Direct |
| mobile | `Trim([mobile])` | Expression |
| msDS-ObjectSoa | `Cloud` | Constant |
| parentDistinguishedName | `IIF(IsNullOrEmpty([onPremisesDistinguishedName]), "CN=Users,DC=world,DC=local", ...)` | Expression |
| postalCode | `Trim([postalCode])` | Expression |
| preferredLanguage | `Trim([preferredLanguage])` | Expression |
| sAMAccountName | `Left(Item(Split([userPrincipalName], "@"), 1), 15)` | Expression |
| sn | `Trim([surname])` | Expression |
| st | `Trim([state])` | Expression |
| streetAddress | `Trim([streetAddress])` | Expression |
| userPrincipalName | userPrincipalName | Direct (changed from Expression) |

A couple of practical notes:

- **Target OU**: `parentDistinguishedName` defaults to the `Users` container. In production you'll almost certainly want to change this expression to place cloud-provisioned users in a dedicated OU.
- **sAMAccountName** is truncated to 15 characters from the UPN prefix. 
- **Passwords are not written back.** Cloud Sync does not synchronize password hashes from Entra ID to Active Directory today. A user provisioned this way has an AD account, but no usable AD password until one is set on-premises or through another mechanism. Microsoft has indicated this is coming later, so keep an eye on the roadmap.

## What do you get from this?

By enabling User Writeback, you've:

- **Made Entra ID the birthplace of users**: Create the user once, in the cloud, and let Cloud Sync do the rest
- **Kept on-premises access working**: Cloud-native users get a real AD account with a SID for Kerberos/NTLM-based apps and file shares
- **Kept UPNs consistent**: With the direct mapping, the AD UPN mirrors Entra ID, so users see one logon name everywhere
- **Turned AD into a downstream directory**: Together with Group Writeback, on-premises Active Directory is no longer the master, it's a target

## What's Next?

The Active Directory Minimization series so far:

- **Part I**: [Exchange SOA Conversion](/posts/ad-minimization-part-1-exchange-soa-conversion/) - Move Exchange attribute management to the cloud
- **Part II**: [Group SOA Conversion](/posts/ad-minimization-part-2-group-soa-conversion/) - Move group management to Entra ID
- **Part III**: [Exchange Writeback](/posts/ad-minimization-part-3-exchange-writeback/) - Write Exchange Online attribute changes back to on-premises AD
- **Part IV**: User Writeback - Provision cloud-native users to on-premises AD

Stay tuned for more in the series!

## Let's Connect

I'm always looking to connect with others who are working on AD Minimization and related challenges. Whether you're just starting your cloud journey or deep into decommissioning on-prem infrastructure, I'd love to exchange ideas and experiences.

If you're working on Active Directory minimization, cloud-first identity, or hybrid transitions, let's talk. I learn just as much from hearing about your environment as you might from this post.

You can find me on [Twitter/X](https://x.com/dk_hcandersen) and [LinkedIn](https://www.linkedin.com/in/hanschrandersen).

## Reference

- [Microsoft: Prerequisites for Microsoft Entra Cloud Sync](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/how-to-prerequisites)
- [Microsoft: Tutorial - Govern access to an on-premises app with users and groups provisioning (Preview)](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/tutorial-users-groups-provisioning-walkthrough)
- [Microsoft: What is Entra Cloud Sync?](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)
- [AD Minimization Part I: Exchange SOA Conversion](/posts/ad-minimization-part-1-exchange-soa-conversion/)
- [AD Minimization Part II: Group SOA Conversion](/posts/ad-minimization-part-2-group-soa-conversion/)
- [AD Minimization Part III: Exchange Writeback](/posts/ad-minimization-part-3-exchange-writeback/)

---

*This is part of an ongoing series about Active Directory Minimization. I'll be creating more tools and blog posts about this subject.*
