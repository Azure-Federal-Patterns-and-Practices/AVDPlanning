---
title: Planning for Azure Virtual Desktop
description: Considerations and topics for planning out an Azure Virtual Desktop Solution.
author: 
ms.topic: conceptual
ms.date: 10/24/2022
ms.author: 
manager: 
---

TOPICS
profile migration, licensing, clients, SLA

# Planning for Azure Virtual Desktop Deployment

In order to ensure your Azure Virtual Desktop solution meets your needs, it is recommended to consider the user requirements and needs in conjunction with the available features and components of Azure. This guide should provide you with some of the initial planning steps as you plan out and design your Azure Virtual Desktop deployment and solution.  

## Prerequisites

Azure AD Tenant  
Subscription  
Virtual Network Infrastructure  

## Identity  

**Same Cloud and Tenant is Ideal**  
Azure Active Directory is where your Azure based user accounts, groups and possibly devices reside. In some cases this is referred to you as a Azure AD Tenant. When it comes to deploying resources in Azure like Azure Virtual Desktop, it's important that the Subscription and Azure Infrastructure reside in the same Azure Cloud. If your cloud journey began with Microsoft 365 or Office 365, you likely already have an Azure AD Tenant and user accounts within Azure. With Azure Virtual Desktop or any Azure based resource there is often requirements for assigning specific rights and roles to Users, Groups and in some cases Devices which can only be done when the infrastructure (Subscription) is within the same Tenant.  

While there are solutions that allow for cross-cloud, they often come with complexities and limitations. It is recommended to avoid these scenarios if at all possible. There are a few options that are documented which you may be aware of but as noted come with some challenges and limitations.  

1. [Re-registration of the AVD VMs post deployment to AVD in the Tenant where your Identities reside.](https://techcommunity.microsoft.com/t5/azure-architecture-blog/gov-topics-avd-cross-cloud-cross-tenant-deployment/ba-p/3543939)
    - Machine Accounts will not reside in cloud with Users thus Intune management will not be possible for AVD VMs.
    - Special consideration required for authentication with SMB Shares for Profiles and MSIX App attach.  
2. [Dual Sync of Identities using AD Connect to 2 different Tenants.](https://learn.microsoft.com/azure/active-directory/hybrid/plan-connect-topologies#multiple-azure-ad-tenants)
    - Custom Domain Names will be unique for each Tenant and may affect user logon names/ experience.  

>[!IMPORTANT] While [**Cross Cloud B2B**](https://learn.microsoft.com/azure/active-directory/external-identities/cross-cloud-settings) provides a means to create an account in your other Tenant, it is still a "Guest" account which is **not supported by AVD.**  

### Hybrid or Cloud Only

Azure Virtual Desktop does support Cloud Only or Azure AD only accounts without the need for a native Domain Services or Active Directory. This would assume that users would access resources that only support modern authentication. If you have resources the users will access that require Kerberos authentication like a File Server, an application with a SQL backend that is domain joined, etc., then you'll need to consider an Azure infrastructure that supports Active Directory.  

**Resource access in both Azure and On-Prem:** Deploy a Domain Controller within Azure that has connectivity and ability to replicate with On-Prem via a Site to Site VPN or Express Route. If desired, utilize Intune or configure the AAD Connect and GPOs to also Azure AD Join the VMs post deployment.  
**Resource access NOT Active Directory Domain joined. (Kerberos not Required):** Azure AD Join Only during deployment.

>[!Note] If you plan to utilize MSIX App attach, you will need to consider the first option with a Domain Controller **OR** Azure AD Domain Services to support machine/device permissions required to the file share for your MSIX app packages.

## Users

When it comes to users, you'll need to consider not only the number of users you intend to support, but also the various roles and applications they may need.  

**Number:** The actual total user count that will have access and the concurrent user count is important to help you decide on the host pool size and how you might scale VMs and Storage for profiles.  Additionally there may be some Azure Limitations when it comes to quotas and availability of VM types and SKUs.  

**Type:** Consider breaking out your users into roles based on applications or daily work activities. In most cases the 'general office' type user needs a basic Windows Desktop with Office related apps. However you may have groups that require a different subset of applications and possibly a more specific hardware requirement. (i.e. Memory, GPU, etc.)  

**Location:** Azure has regional data centers all over the globe. However you may find that there is a specific one or subset are the closest to your user base. Additionally various regions have different hardware and availability and you may need to work with Microsoft to ensure soft or hard quotas and specific VM SKU availability are not going to pause or halt your deployment. Keep in mind you may have a subset of users that travel frequently which may play into your planning.  

### Virtual Machine Type and SKU

With an idea of scenarios for each user type, you can then start planning your Virtual Machine requirements. Things to consider are the hardware requirements, Operating System version and whether apps can only run on a Server platform. As well as any potential GPU specific applications or developer type users.  Some of the common roles or groupings are:  

**General Office Users:**  In most cases a multi-session Windows Client will perform well and optimize cost.  
**Developers:**  Typically this group requires a different subset of applications and possibly a heavier duty VM SKU or even personal desktop.  
**Computer Aided Drafting (CAD):**  Most importantly, this type of user will likely need a VM with a dedicated Graphics Processor or GPU. These are not available in all Azure Regions and there are options for AMD and NVIDIA that need to be considered.  

#### Host Pool Types

**Pooled:** This option is what we refer to as non-persistent or multi-session. Multi-session requires using Windows 10 or 11 specific client images that are only in Azure but customizable. It's often the most cost effective and widely used for most all scenarios and used in conjunction with FSLogix for user profiles.  
**Personal:** With Personal or Persistent desktops, users will always logon to the same VM. This option is a 1:1 ratio between users and VMs, thus higher in cost. However there are times in which users need to be able to install their own applications and have local administrative privileges in which this is the ideal option.  

## Network Infrastructure

Most importantly, your network configuration and infrastructure is vital to a successful AVD deployment as it is the foundation to any Virtual Machine deployment. Typically you'll have some sort of isolation of the Virtual Machines for AVD and possibly even have some security requirements that dictate the flow of traffic to the Internet and/or On-Prem. You'll need to additionally consider if you need connectivity to On-Prem and the throughput limitations of the network services (i.e. VPN SKU, Express Route Option, Network Virtual Appliance, etc.).  
[Example Infrastructure](https://learn.microsoft.com/azure/architecture/example-scenario/wvd/windows-virtual-desktop)  
[Required URLs for AVD](https://learn.microsoft.com/azure/virtual-desktop/safe-url-list?tabs=azure)  
[Firewall Considerations](https://learn.microsoft.com/azure/firewall/protect-azure-virtual-desktop?tabs=azure)  
[VPN Options](https://azure.microsoft.com/pricing/details/vpn-gateway/)  
[Express Route](https://azure.microsoft.com/en-us/pricing/details/expressroute/)  

## Applications

**Group by User Roles**  
Try to consider the user types and role they may have to group applications. This will help you define what might be a Host Pool for AVD.  

**Updates**  

- Disable Automatic Updates where possible on Pooled or multi-session for applications and Windows Update.
- Apply within Image or Post Deployment? Depending on the number of updates and applications, this may drive whether or not you have a need to manage an image or simply incorporate a post configuration script on top of the Microsoft provided image.

**MSIX App Attach**  
This option provides a way to virtualize applications and separate them from the Desktop or Operating System.

- Provides mechanism for keeping the image free of installed software and possibly not need to maintain a custom image.
- It does require a code signing certificate and separate storage is recommended.
- Permissions require Computer Account assignment which may not be possible with Azure AD Domain Services. (Azure NetApp Files may be workaround)
- Application management requires different tools and strategy but more centralized.

**FSLogix App Masking**  
Provides mechanism to hide applications for specific users based on group membership.  Ideally for when you have only a small subset that you don't want to have access to an application that is installed.

**Remote Apps or Full Desktop**  

Full Desktop: Best for when the users will need to use multiple applications that may interact with each other. For example the Office Suite of applications.  
Remote Apps: Typically used in scenarios where users just need a single application or when applications need more robust VMs that you want to limit used to minimize cost but provide the best user experience only when those apps are needed.  

## Management



### Policy Settings

### RDP Properties

### Profiles

### Operating System Images and Updates

## Availability

SLA, RTO

## Security

RBAC
Best Practice Doc