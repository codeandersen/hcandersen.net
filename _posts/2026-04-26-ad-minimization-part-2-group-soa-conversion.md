---
title: "AD Minimization Part II: Group SOA Conversion - From On-Premises Groups to Cloud Management"
date: 2026-04-26 10:00:00 +0200
categories: [Microsoft, Entra ID]
tags: [entraId, groups, cloudmigration, powershell, automation, activedirectory, minimization, microsoftgraph]
---

*This is Part II of the Active Directory Minimization series.*

In [Part I](/posts/ad-minimization-part-1-exchange-soa-conversion/), I walked through converting Exchange mailbox attributes to cloud management. Now it's time to tackle the next piece: **Group Source of Authority**. In this post, I'll walk through the Group SOA Conversion Tool and how to use it to move group management from on-premises Active Directory to Microsoft Entra ID.

If you're running a hybrid environment, your Mail-Enabled Security Groups and Distribution Groups are likely still managed on-premises. Every membership change, every property update has to go through your on-prem Active Directory. This can be changed.

## The Problem: Groups Locked to On-Premises Management

### From the Entra Admin Center

Let's try to edit a synced group in the Microsoft Entra Admin Center and see what happens:

Go to the [Microsoft Entra Admin Center](https://entra.microsoft.com) and navigate to **Groups** > **All groups**. Find a synced distribution group or mail-enabled security group.
Here you can see that you're not able add members directly in the Entra Admin Center.

![Entra Admin Center showing synced group](/assets/img/posts/group-walk-through-1.png)
_You can only manage this group in your on-premises environment. Use 'Active directory users & groups' or 'Exchange Admin Center' tools to edit or delete this group_

### From On-Premises

This is where you currently have to manage these groups, in your on-premises Exchange admin center or Active Directory.

![On-premises Exchange/AD showing group management](/assets/img/posts/group-walk-through-4.png)
_Managing groups on-premises_

This means changes like group membership, mail address change requires access to your on-prem infrastructure, and changes only take effect after the next Entra ID Connect/Cloud Sync sync cycle.

## Why This Matters: The Benefits of Cloud-Managed Groups

Before we dive into the solution, let's talk about why you'd want to fix this:

**Reduce infrastructure dependency**
- Stop relying on on-premises Active Directory or Exchange Server for routine group management
- One step closer to retiring your on-prem AD and Exchange Server infrastructure

**Improve security posture**
- Manage groups with modern authentication, MFA, and Conditional Access
- No more relying on on-premises access for group changes
- Reduce your on-premises attack surface

**Cleaner architecture**
- Eliminate unnecessary hybrid complexity for group management
- Simplify your environment and documentation
- Easier troubleshooting with fewer moving parts

**Enable decommissioning**
- Critical step toward removing on-premises Active Directory dependency
- Move closer to cloud-only operations

## Prerequisites: Getting Ready

Before we start, you'll need:

- **Hybrid Identity Administrator role** (least privileged role for the `onPremisesSyncBehavior` API)
- **Graph permissions**: `Group.ReadWrite.All` and `Group-OnPremisesSyncBehavior.ReadWrite.All`
- **To grant consent**: **Cloud Application Administrator** or **Application Administrator** role
- **PowerShell 5.1 or later**
- **The Group SOA Conversion Tool** - [Download from GitHub](https://github.com/codeandersen/Group-SOA-Conversion-Tool)
- **Entra Connect Sync Version 2.5.76.0 or later**, OR **Entra Cloud Sync Version 1.1.1370.0 or later**

The tool will automatically install the `Microsoft.Graph.Groups` module if you don't have it already.

## The Conversion: Flipping the Switch to Cloud Management

Let's convert a group's source of authority to cloud-managed.

### Step 1: Launch the Tool

Navigate to where you downloaded the tool and run:

```powershell
.\Group-SOA-Conversion-Tool.ps1
```

Or, to connect to a specific tenant:

```powershell
.\Group-SOA-Conversion-Tool.ps1 -TenantId "00000000-0000-0000-0000-000000000000"
```

![Group SOA Conversion Tool startup](/assets/img/posts/group-walk-through-5.png)
_The Group SOA Conversion Tool GUI_

### Step 2: Connect to Microsoft Graph

Click "Connect to Graph" and sign in with your admin credentials. The tool uses modern authentication via `Connect-MgGraph`, so you'll go through your organization's authentication flow (including MFA if enabled).

![Connect to Graph button](/assets/img/posts/group-walk-through-6.png)

The first time you run the tool, you may need to grant consent for the required permissions. Be sure to be elevated to grant consent.

After connection, the tool automatically loads all Exchange-relevant groups. Mail-Enabled Security Groups and Distribution Groups are included. The grid shows Display Name, Email, Group Type, Cloud Managed status, and Nesting Depth.

![Connected with groups loaded](/assets/img/posts/group-walk-through-9.png)
_Connected status showing Exchange-relevant groups_

### Step 3: Select and Convert

Find the group you want to convert. You can select multiple groups for batch conversion, but for this walkthrough, we'll convert a single group.

Select the group and click "Convert to Cloud Managed".

![Group selected for conversion](/assets/img/posts/group-walk-through-10.png)
_Selecting a group for cloud management conversion_

Select Yes to confirm when prompted.

![Conversion confirmation dialog](/assets/img/posts/group-walk-through-11.png)

![Conversion success message](/assets/img/posts/group-walk-through-12.png)
_Success — the group has been converted_

You can now see the group has the Cloud Managed value set to `True`.

![Grid showing Cloud Managed = True](/assets/img/posts/group-walk-through-13.png)
_Cloud Managed is now True for the converted group_

Behind the scenes, the tool is calling the Microsoft Graph `onPremisesSyncBehavior` API to set `isCloudManaged` to `true`. This tells Entra ID that it now has authority over this group's properties and membership.

### What Just Happened?

The group object is now and will now be disconnected from Entra ID Connect/Cloud Sync.

What has changed is that the group's properties and membership are now managed in the cloud. On-premises changes to the group are no longer synced to Entra ID.

**A note on nested groups**: The tool automatically detects nested group relationships and computes nesting depth. When converting multiple groups, it sorts them bottom-up (deepest nested first) following Microsoft's recommended approach. For this walkthrough, we kept it simple with a single group, but the tool handles nested scenarios automatically.

## Success: Full Control in Entra ID

Now, let's go back to the Microsoft Entra Admin Center and try editing that group again.

Navigate to **Groups** > **All groups** and find the group we just converted.

You can see that it now shows a cloud icon next to the group name, indicating it's now managed in the cloud.
![Entra Admin Center showing the converted group](/assets/img/posts/group-walk-through-14.png)

Try editing the group membership or properties.

![Successfully editing group in Entra](/assets/img/posts/group-walk-through-15.png)

![Change saved successfully](/assets/img/posts/group-walk-through-16.png)
_The group can now be managed directly in Entra ID_

It worked. Group membership and properties can now be managed directly in the Microsoft Entra Admin Center or via Exchange Online Admin Center, without touching your on-premises Active Directory.

## What Happens in Entra ID Connect Sync

Now that we've converted to cloud management, what safeguards exist if someone still makes changes on-premises? Let's test it.

After converting a group to cloud-managed, the `blockOnPremisesSync` property is set to `true` on the Entra ID object. This means on-premises changes to the group are no longer synced to the cloud.

![Entra Connect Sync showing blockOnPremisesSync](/assets/img/posts/group-walk-through-17.png)

Search for the group in this example CN=Executive-SEC, press search and double click the search object.
![Entra Connect Sync showing blockOnPremisesSync](/assets/img/posts/group-walk-through-18.png)

Click on Lineage and Metaverse Object Properties
![Entra Connect Sync showing blockOnPremisesSync](/assets/img/posts/group-walk-through-19.png)

Find the property `blockOnPremisesSync` property. Here you can see it is set to `true`. This prevents on-premises changes from being synced to the cloud.
![Entra Connect Sync showing blockOnPremisesSync](/assets/img/posts/group-walk-through-20.png)
_The sync behavior confirms on-premises changes are blocked from syncing_

You can also confirm it in event viewer under Application and filter for event ID 6956.
![Application log Event ID 6956 showing the group moved to cloud only](/assets/img/posts/group-walk-through-21.png)
_This also confirms on-premises changes are blocked from syncing_

## What You've Gained

By converting groups to cloud management, you've:

- Eliminated dependency on on-premises Active Directory or Exchange Serverfor routine group management
- Simplified your hybrid architecture
- Improved security by reducing on-premises access requirements
- Enabled yourself to eventually decommission on-premises Active Directory and Exchange Server infrastructure

This is another piece of the puzzle on the path to minimizing or eliminating your on-premises Active Directory footprint.

## What's Next?

This walkthrough focused on converting a single group, but the same approach works for:

- Batch conversions (the tool supports multi-select)
- Nested group hierarchies (the tool handles ordering automatically)

Stay tuned for more in the Active Directory Minimization series!

## Try It Yourself

The Group SOA Conversion Tool is free and available on GitHub:
[https://github.com/codeandersen/Group-SOA-Conversion-Tool](https://github.com/codeandersen/Group-SOA-Conversion-Tool)

## Let's Connect

I'm always looking to connect with others who are working on AD Minimization and related challenges. Whether you're just starting your cloud journey or deep into decommissioning on-prem infrastructure, I'd love to exchange ideas and experiences.

If you're working on Active Directory minimization, hybrid group management, or cloud-native transitions, let's talk. I learn just as much from hearing about your environment as you might from this post.

You can find me on [Twitter/X](https://x.com/dk_hcandersen) and [LinkedIn](https://www.linkedin.com/in/hanschrandersen), or open an issue on [GitHub](https://github.com/codeandersen/Group-SOA-Conversion-Tool) if you have feedback on the tool.

## Reference

- [Microsoft: Guidance for Group SOA](https://learn.microsoft.com/en-us/entra/identity/hybrid/concept-group-source-of-authority-guidance)
- [Microsoft: Configure Group SOA](https://learn.microsoft.com/en-us/entra/identity/hybrid/how-to-group-source-of-authority-configure)
- [Get onPremisesSyncBehavior API](https://learn.microsoft.com/en-us/graph/api/onpremisessyncbehavior-get)
- [Update onPremisesSyncBehavior API](https://learn.microsoft.com/en-us/graph/api/onpremisessyncbehavior-update)
- [AD Minimization Part I: Exchange SOA Conversion](/posts/ad-minimization-part-1-exchange-soa-conversion/)

---

*This is part of an ongoing series about Active Directory Minimization. I'll be creating more tools and blog posts about this subject.*
