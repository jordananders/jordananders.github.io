---
layout: default
title:  "Custom SharePoint Workflows"
date:   2015-09-29 11:30:00
categories: SharePoint Workflows
---

Custom workflows in SharePoint provide powerful automation capabilities that go beyond the out-of-the-box workflow actions. This post covers the different types of custom workflows available in SharePoint 2010, 2013, and SharePoint Online, based on a presentation I gave at DEG.

## Workflow Basics

Before diving into custom workflows, let's review the fundamental building blocks:

### Workflow Types

SharePoint supports several workflow types:

- **List Workflows** - Associated with a specific list and triggered by list item events
- **Site Workflows** - Not associated with any list, manually triggered or scheduled
- **Reusable Workflows** - Can be associated with multiple lists or content types
- **Content Type Workflows** - Available in SharePoint 2010, associated with content types

### Workflow Elements

Modern SharePoint workflows are composed of several key elements:

- **Actions** - The operations that workflows perform (send email, update item, etc.)
- **Conditions** - Logic that determines workflow behavior
- **Steps** - Containers for actions and conditions, can be:
  - Sequential (one after another)
  - Parallel (execute simultaneously)
  - App/Impersonation steps (elevated permissions)
- **Loops** (2013 only) - Repeat actions:
  - N times (fixed iteration count)
  - Conditional (while a condition is true)
- **Stages** (2013 only) - State machine workflow components

## SharePoint 2010 vs 2013: Key Differences

### Impersonation Step vs. App Step

One of the most significant changes between SharePoint 2010 and 2013 is how workflows handle elevated permissions:

**SharePoint 2010 - Impersonation Step:**
- Runs as the workflow creator
- Limited to the creator's permissions
- Useful when users need to perform actions they don't normally have rights to

**SharePoint 2013 - App Step:**
- Elevates workflow to Full Read/Write on all lists
- Much more powerful than impersonation
- Requires careful consideration of security implications

### New Capabilities in SharePoint 2013

SharePoint 2013 introduced several game-changing features:

- **Call HTTP Web Service Action (HTTPSend)** - Make REST API calls directly from Designer workflows
- **OAuth Support** - Authenticate with external services
- **Start "2010" Workflows** - Trigger legacy 2010 workflows from 2013 workflows for backward compatibility

## Advanced Custom Workflows

When out-of-the-box actions aren't enough, SharePoint provides several options for custom workflows:

### 1. Designer Workflows with HTTPSend (2013/Online)

The Call HTTP Web Service action in SharePoint 2013 and Online is incredibly powerful:

- Make REST API calls to SharePoint or external services
- Support for OAuth authentication
- No code deployment required
- Works in SharePoint Online

This is often the best choice for cloud-based solutions since it requires no server-side code deployment.

### 2. Custom Workflow Actions

**SharePoint 2010:**
- Custom workflow activities deployed as Farm Solutions
- Requires server access for deployment
- Can perform complex operations not available in Designer

**SharePoint 2013 On-Premises:**
- Custom code activities installed into Workflow Manager
- Still requires Farm Solution deployment
- More complex architecture than 2010

**SharePoint 2013/Online:**
- Custom declarative activities
- Deployed as Sandbox Solutions
- Cloud-compatible option for custom actions

### 3. Custom Code Workflows

For maximum flexibility and control:

**SharePoint 2010:**
- Deployed as Farm Solutions
- Full access to SharePoint object model
- Requires Visual Studio development

**SharePoint 2013 On-Premises:**
- Still deployed as Farm Solutions
- Can leverage new 2013 APIs
- Not available in SharePoint Online

### 4. Custom Workflow Apps (2013 Only)

SharePoint 2013 introduced the App model for workflows:

- Cloud-compatible
- Isolated from the SharePoint farm
- Can be distributed through the App Store
- Works in SharePoint Online

## Choosing the Right Approach

Here's a quick summary of what's available in each platform:

**SharePoint 2010:**
- Custom Code Activity (Farm Solution)
- Custom Code Workflow (Farm Solution)

**SharePoint 2013 On-Premises:**
- Call HTTP Web Service Action (Designer)
- Custom Code Activity (Workflow Manager)
- Custom Declarative Activity (Sandbox)
- Custom Code Workflow (Farm Solution)
- Workflow App/Add-In

**SharePoint Online:**
- Call HTTP Web Service Action (Designer)
- Custom Declarative Activity (Sandbox)
- Workflow App/Add-In

For SharePoint Online, focus on HTTPSend actions and workflow apps since server-side code deployment isn't available.

## Resources

For more information on SharePoint workflows, check out these resources:

- [What's changed in SharePoint Designer 2013](https://msdn.microsoft.com/en-us/library/jj728659.aspx)
- [What's new in workflows for SharePoint 2013](https://msdn.microsoft.com/en-us/library/office/jj163177.aspx)
- [How to trigger a SharePoint 2010 workflow from a SharePoint 2013 workflow](http://blogs.msdn.com/b/sharepointdesigner/archive/2012/08/18/how-to-trigger-a-sharepoint-2010-workflow-from-a-sharepoint-2013-workflow.aspx)
- [Use workflow interop for SharePoint 2013](https://msdn.microsoft.com/en-us/library/office/jj670125.aspx)
- [SharePoint 2010: Create a workflow activity using Visual Studio 2010](http://social.technet.microsoft.com/wiki/contents/articles/13604.sharepoint-2010-create-a-workflow-activity-using-visual-studio-2010.aspx)

---

*The PowerPoint presentation that this blog post is based on can be found [here]({{ site.url }}/files/2015-09-29-CustomWorkflows.pptx).*