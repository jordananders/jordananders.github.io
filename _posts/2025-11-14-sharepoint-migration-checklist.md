---
layout: default
title:  "SharePoint Migration Checklist: Lessons from the Trenches"
date:   2025-11-14 10:00:00
categories: SharePoint Migration
---

After years of working on SharePoint migrations—from 2003 to 2007, 2010 to 2013, and now on-premises to SharePoint Online—I've learned that the difference between a smooth migration and a disaster often comes down to preparation. Here's the checklist I wish I'd had from day one.

## The Reality of SharePoint Migrations

Let me be blunt: if you think a SharePoint migration is just an IT project, you're already setting yourself up for failure. The most successful migrations I've seen treat it as a business transformation project with heavy IT involvement, not the other way around.

## Pre-Migration: Do This or Regret It Later

### 1. Catalog Everything (Yes, Everything)

Before you touch a single file, you need to know what you have:
- **All sites and subsites** - Don't forget those forgotten department sites
- **Document libraries and lists** - Including custom ones
- **Workflows** - Especially InfoPath forms (more on this pain point later)
- **Custom solutions and web parts** - Many won't work in SharePoint Online
- **Permissions structure** - This is often more complex than anyone admits

I once worked on a migration where we discovered 47 site collections that IT didn't know existed. Don't be that person explaining to your boss why the marketing team can't access their files.

### 2. Clean Up Your Data (The ROT Problem)

ROT stands for Redundant, Obsolete, and Trivial data. Every SharePoint environment has it, and migrating it is like paying to move trash to your new house.

**Reality check:** In one migration, we found:
- 2,000+ documents that hadn't been accessed in 5+ years
- 150+ duplicate copies of the same company presentation
- Entire site collections for projects that ended in 2015

Clean this up *before* migration. Your future self will thank you.

### 3. Address the Custom Solution Problem

Here's a painful truth: many customizations from on-premises SharePoint won't work in SharePoint Online:
- **InfoPath forms** - Not supported in SharePoint Online
- **PerformancePoint dashboards** - Gone
- **SharePoint Designer workflows** - Being retired
- **Custom server-side code** - Doesn't exist in SharePoint Online

You need to identify these early and plan alternatives:
- InfoPath → Power Apps
- SharePoint Designer workflows → Power Automate
- Custom solutions → SharePoint Framework (SPFx)

Don't wait until migration day to discover your critical workflow won't work.

## Planning Phase: The Strategy That Separates Success from Failure

### Define Clear Objectives

Answer these questions honestly:
- **Why are we migrating?** (Cost savings? Security? Modern features?)
- **What does success look like?** (Define metrics)
- **What's our timeline?** (Be realistic—add 30% to your estimate)
- **What's our rollback plan?** (You need one)

### Choose the Right Migration Tool

This decision matters more than most people realize:

**SPMT (SharePoint Migration Tool):**
- Free from Microsoft
- Great for file server → SharePoint Online
- Limited functionality for complex scenarios
- Works well for smaller migrations

**Third-party tools (Sharegate, AvePoint, etc.):**
- More features and flexibility
- Better for tenant-to-tenant migrations
- Handle complex permission mapping
- Cost money, but may save more in time and headaches

I've used both. For simple migrations, SPMT works fine. For anything complex, the third-party tools pay for themselves.

### Plan Your Information Architecture

**Don't just lift and shift.** This is your opportunity to fix the organizational problems that have accumulated over years.

Ask:
- Do we really need 47 separate site collections?
- Should some of this content move to Teams?
- How does this fit with Microsoft 365 Groups, OneDrive, and SharePoint?
- What's our governance model going forward?

The worst migrations I've seen just recreated the old mess in a new place.

## Testing Phase: The Step Everyone Wants to Skip

### Run a Pilot Migration

Always test with a small, representative sample:
1. Pick 2-3 sites with different complexity levels
2. Include one with custom workflows
3. Include one with complex permissions
4. Migrate to a test environment
5. Have actual users test functionality

**What to validate:**
- Do all files and versions transfer correctly?
- Are permissions mapped properly?
- Do links still work?
- Is metadata preserved?
- What's the migration speed?

I once discovered during pilot testing that our chosen tool broke all internal links. Imagine finding that out after migrating 500 GB of content.

## Migration Execution: The Main Event

### Communication Plan

This cannot be overstated: **communicate early and often.**

**Before migration:**
- Email announcement 2 weeks out
- Department meetings explaining changes
- Clear timeline and expectations
- Point of contact for issues

**During migration:**
- Status updates
- Extended maintenance windows if needed
- Immediate communication if problems arise

**After migration:**
- Confirmation email
- Link to new locations
- Training schedule
- Support resources

### Migration Day Best Practices

1. **Migrate during off-hours** - Minimize disruption
2. **Disable access to source during migration** - Prevent changes mid-migration
3. **Monitor in real-time** - Have your team watching the process
4. **Document everything** - Errors, warnings, issues
5. **Have rollback plan ready** - Just in case

## Post-Migration: You're Not Done Yet

### User Training

The fanciest SharePoint environment is worthless if nobody knows how to use it.

**Provide:**
- Live training sessions
- Recorded tutorials
- Quick reference guides
- Office hours for questions
- Champions program (power users who can help others)

### Validation

Don't assume it worked. Verify:
- Random sample of files (open them, check versions)
- Permission testing (can the right people access the right content?)
- Workflow functionality
- Search functionality
- Custom solutions

### Leverage Modern Features

You didn't migrate just to replicate old functionality. Take advantage of:
- **Microsoft Syntex** - AI-powered content management
- **Microsoft Copilot** - AI assistance
- **Power Platform** - Modern workflows and apps
- **Teams integration** - Collaboration features

## The Biggest Mistakes I've Seen

### Mistake #1: IT-Only Project
**Problem:** Business users left out of planning
**Result:** Migrated structure doesn't match how people work
**Fix:** Include stakeholders from every department

### Mistake #2: No Data Cleanup
**Problem:** Migrating everything "just in case"
**Result:** Slow migration, high storage costs, cluttered environment
**Fix:** Ruthlessly archive or delete old content

### Mistake #3: Ignoring Permissions
**Problem:** Assuming permissions will "just work"
**Result:** Security breaches or access problems
**Fix:** Map permissions explicitly, test thoroughly

### Mistake #4: No Testing
**Problem:** Going straight to production migration
**Result:** Discovering problems when users can't work
**Fix:** Always run pilot migrations

### Mistake #5: Viewing SharePoint in Isolation
**Problem:** Not considering Teams, OneDrive, Microsoft 365
**Result:** Suboptimal architecture
**Fix:** Plan holistically across the entire Microsoft 365 ecosystem

## My Personal Checklist

Here's the actual checklist I use:

**8 Weeks Before:**
- [ ] Complete inventory of all sites and content
- [ ] Identify stakeholders and form migration team
- [ ] Choose migration tool
- [ ] Define success criteria

**6 Weeks Before:**
- [ ] Begin data cleanup
- [ ] Document custom solutions and plan replacements
- [ ] Design new information architecture
- [ ] Set up test environment

**4 Weeks Before:**
- [ ] Run pilot migration
- [ ] Test and validate pilot results
- [ ] Adjust plan based on pilot learnings
- [ ] Finalize migration schedule

**2 Weeks Before:**
- [ ] Communicate migration plan to all users
- [ ] Complete data cleanup
- [ ] Prepare training materials
- [ ] Confirm rollback procedures

**Migration Week:**
- [ ] Final communication to users
- [ ] Execute migration during off-hours
- [ ] Monitor and document progress
- [ ] Validate migrated content

**Post-Migration:**
- [ ] User validation and testing
- [ ] Training sessions
- [ ] Monitor for issues
- [ ] Document lessons learned

## Final Thoughts

SharePoint migrations are complex, but they don't have to be chaotic. The key is respecting the complexity, planning thoroughly, testing extensively, and communicating constantly.

Every migration teaches you something. I've learned most of these lessons the hard way so you don't have to.

## Sources and Further Reading

- [SharePoint Migration Checklist: Move Without Data Loss](https://reality-tech.com/blog/sharepoint-migration-checklist/)
- [Your complete SharePoint migration strategy guide | Sharegate](https://sharegate.com/resource/ebook-the-ultimate-sharepoint-migration-checklist)
- [SharePoint Migration Checklist: Your Ultimate Guide | Multishoring](https://multishoring.com/blog/sharepoint-migration-checklist/)
- [The ultimate SharePoint Online migration checklist | RecordPoint](https://www.recordpoint.com/blog/the-ultimate-sharepoint-online-migration-checklist)
- [SharePoint Data Migration: How to Avoid Common Mistakes](https://intranet.ai/articles/microsoft-365/sharepoint-data-migration/)
- [Migrating data to SharePoint? Avoid these 6 challenges](https://antares.solutions/migrating-data-to-sharepoint-avoid-these-6-challenges/)
- [5 Common SharePoint Migration Mistakes - Bridgeall](https://www.bridgeall.com/2023/04/13/5-common-sharepoint-migration-mistakes/)
- [Common SharePoint Migration Challenges & Solutions](https://awesometechinc.com/sharepoint-migration-challenges/)

---

*Have your own migration war stories or questions? I'd love to hear them - [reach out](mailto:jordan@jordananderson.us).*
