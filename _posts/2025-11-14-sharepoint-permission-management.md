---
layout: default
title:  "SharePoint Permission Management: Patterns That Actually Work"
date:   2025-11-14 11:00:00
categories: SharePoint Security
---

I've seen SharePoint permission structures that would make you weep. Site collections with 500+ unique permission configurations. Folders where not even the site owner knows who has access. Documents shared so many times that tracking down all the permissions requires a flowchart.

Here's what I've learned about managing SharePoint permissions in a way that actually scales.

## The Fundamental Truth About SharePoint Permissions

SharePoint permissions are like compound interest—small decisions made early have massive consequences later. Break inheritance on one folder today, and three years from now you'll have a permission nightmare that nobody can untangle.

Let me share the patterns that work and the ones that will haunt you.

## Understanding Permission Inheritance

Before we dive into patterns, you need to understand how SharePoint permissions actually work.

### The Inheritance Hierarchy

Permissions flow down in this order:
1. **Tenant** (entire Microsoft 365 organization)
2. **Site Collection** (top-level site)
3. **Site** (subsites)
4. **List/Library**
5. **Folder**
6. **Item** (individual files)

By default, everything inherits from its parent. When you break this inheritance, you're creating what's called **unique permissions** or **broken inheritance**.

### The 50,000 Problem

Here's a technical limit that will bite you: SharePoint supports up to 50,000 unique permissions per list or library, but the **recommended limit is 5,000**.

I've seen libraries hit this limit. Performance degrades, management becomes impossible, and you're essentially forced to restructure everything. Don't let this happen to you.

## Pattern 1: Group-Based Access (The Foundation)

**Never assign permissions to individual users unless absolutely necessary.**

I'll say it again: **Never assign permissions to individual users.**

### Why Groups Matter

When you assign permissions to individual users:
- You create unique permission entries that count toward your 5,000 limit
- Auditing becomes a nightmare ("Who has access to this?" requires checking potentially hundreds of users)
- Removing access when someone changes roles requires hunting down every individual assignment
- You have no clear access patterns

When you use groups:
- One permission entry covers multiple users
- Auditing is simple ("These three groups have access")
- Role changes mean updating group membership, not permissions
- Clear patterns emerge ("Editors group has edit rights")

### The Group Strategy

**Create purpose-specific groups:**

```
Site: Marketing
├── Marketing_Owners (Full Control)
├── Marketing_Members (Edit)
├── Marketing_Visitors (Read)
└── Marketing_Approvers (Contribute + Approve)
```

**For Microsoft 365 team sites:**
- Use the associated Microsoft 365 Group
- Manage membership through the Group, not SharePoint directly
- Permissions automatically sync

**For Communication Sites:**
- Communication sites don't have Microsoft 365 Groups
- Create SharePoint groups explicitly
- Follow a consistent naming convention across all sites

### Real-World Example

**Bad approach:**
```
/sites/marketing/documents/2025-campaign
├── Individual: JohnD (Edit)
├── Individual: SarahM (Edit)
├── Individual: MikeR (Read)
├── Individual: JennyL (Edit)
└── Individual: TomK (Read)
```

**Good approach:**
```
/sites/marketing/documents/2025-campaign
├── Group: Marketing_2025_Campaign_Team (Edit)
└── Group: Marketing_Leadership (Read)
```

When Sarah leaves the team, you remove her from the group. When you need to audit access, you review two groups instead of five (or 50) individuals.

## Pattern 2: Minimize Broken Inheritance

**Breaking inheritance should be a last resort, not a first option.**

### The Broken Inheritance Problem

Every time you break inheritance:
- You create a unique permission entry
- You add management overhead
- You make auditing more complex
- You increase the risk of permission errors

I've seen site collections where 80% of folders have broken inheritance. It's unmanageable.

### When Breaking Inheritance is Justified

There are legitimate cases:
1. **Confidential HR documents** in an otherwise open HR site
2. **Executive-only content** within a department site
3. **Project-specific sensitive data** that needs restricted access

But these should be **rare exceptions**, not the norm.

### The Alternative: Separate Libraries or Sites

Instead of breaking inheritance on folders, consider:

**Option 1: Separate Document Library**
```
Site: HR
├── General HR Documents (inherited permissions)
├── Policies and Procedures (inherited permissions)
└── Confidential Employee Files (unique permissions)
```

**Option 2: Separate Site**
```
/sites/hr (general HR team)
/sites/hr-confidential (restricted access)
```

Both approaches are clearer and more maintainable than dozens of folders with broken inheritance.

### How to Audit Broken Inheritance

You need visibility into where permissions are broken. Use PowerShell:

```powershell
# Get all items with unique permissions
$web = Get-SPWeb "https://yourtenant.sharepoint.com/sites/yoursite"
$list = $web.Lists["Your Library"]

$list.Items | Where-Object { $_.HasUniqueRoleAssignments } |
    Select-Object Name, Url
```

Run this quarterly. If the list is growing, you have a problem.

## Pattern 3: Clear Permission Levels

SharePoint has default permission levels:
- **Full Control** - Everything
- **Design** - Create lists, libraries, pages
- **Edit** - Add, edit, delete items
- **Contribute** - Add, edit, delete items in existing lists
- **Read** - View only

### Use Standard Levels When Possible

Don't create custom permission levels unless you have a specific need. Standard levels are:
- Well-understood
- Documented
- Predictable

### When Custom Levels Make Sense

I've created custom levels for:
1. **Approver** - Contribute + Approve permissions
2. **Viewer with Download** - Read + open items in Office apps
3. **Restricted Contributor** - Add items only, can't edit others' items

But these were for specific business requirements, not because we felt creative.

### Document Everything

If you create a custom permission level, document:
- Its purpose
- Who should receive it
- Specific permissions granted
- Why standard levels weren't sufficient

## Pattern 4: Site Collection Admin Limits

**Limit Site Collection Administrators to 4 or fewer.**

Every Site Collection Admin has Full Control over everything. The more admins you have:
- The higher the risk of accidental changes
- The harder it is to track who did what
- The more likely someone will "just quickly fix something"

### Who Should Be a Site Collection Admin?

Typically:
1. **Primary site owner** (the business owner)
2. **Secondary site owner** (backup)
3. **IT administrator** (for technical support)
4. **Maybe one more** (for large, critical sites)

Everyone else should have appropriate permission levels through groups, not admin rights.

## Pattern 5: Regular Auditing

**You must audit permissions regularly.** Set a schedule and stick to it.

### What to Audit

**Monthly:**
- Site Collection Admins (who's added/removed)
- Guest user access (external sharing)

**Quarterly:**
- Items with broken inheritance (is the list growing?)
- Group memberships (do they still make sense?)
- Inactive users with access (people who've left the company)

**Annually:**
- Full permission review (does access still align with business needs?)
- Permission structure (can we simplify?)

### Tools for Auditing

**Built-in:**
- SharePoint Admin Center → Active Sites → Permissions
- Site Settings → Site Permissions → Check Permissions
- Microsoft Purview compliance portal

**PowerShell:**
```powershell
# Check permissions for a specific user
$web = Get-SPWeb "https://yourtenant.sharepoint.com/sites/site"
$user = $web.SiteUsers["user@domain.com"]
$web.GetUserEffectivePermissions($user)
```

**Third-party tools:**
- Sharegate (permission reporting)
- Syskit (comprehensive governance)
- AvePoint (compliance and management)

I've used all of these. For small environments, built-in tools work. For complex environments, third-party tools pay for themselves.

## Pattern 6: External Sharing Controls

External sharing is where many permission strategies fall apart.

### The Problem

User shares document → External recipient shares with others → Now you have unknown external users with access → Compliance issue

### The Solution

**Set clear policies:**

1. **Tenant-level:** Define what can be shared externally
2. **Site-level:** Further restrict based on content sensitivity
3. **Expiration dates:** External links expire automatically
4. **Audit logs:** Monitor all external sharing

**In SharePoint Admin Center:**
```
Sharing settings:
- Most permissive: Anyone (anonymous links)
- Moderate: New and existing guests
- Most restrictive: Only people in your organization
```

**Best practice:** Default to "Existing guests" and require justification for "Anyone" links.

### Track External Sharing

```powershell
# Get all externally shared items
Get-SPOSite | Get-SPOList | Get-SPOListItem |
    Where-Object { $_.SharingInfo.SharedWith -ne $null }
```

Review this monthly.

## Pattern 7: Governance Documentation

This is the pattern people skip and later regret.

### Document Your Permission Model

Create a wiki or SharePoint page that explains:

**Permission Strategy:**
- How permissions are structured
- Which groups exist and their purpose
- When to break inheritance (and when not to)
- How to request access

**Roles and Responsibilities:**
- Who are Site Collection Admins
- Who are Site Owners
- Who can approve access requests
- Who to contact for issues

**Processes:**
- How to request access
- How to request a new site
- How access is reviewed
- What happens when employees leave

### Real-World Example Document

```markdown
# Marketing Site Permissions

## Access Groups
- Marketing_Owners: Full control (Leadership team)
- Marketing_Members: Edit access (Marketing team)
- Marketing_Visitors: Read-only (Rest of company)
- Marketing_Vendors: Read-only (External partners)

## Requesting Access
1. Submit ticket to helpdesk
2. Include business justification
3. Manager approval required for Edit access
4. Access granted within 24 hours

## Quarterly Review
First week of each quarter, Site Owners review:
- Group memberships
- External sharing
- Broken inheritance

Contact: marketing-sharepoint-owners@company.com
```

Simple, clear, actionable.

## Common Mistakes I've Fixed

### Mistake 1: Everyone Gets Edit Rights
**Problem:** "Just give everyone Edit, it's easier"
**Result:** Accidental deletions, version chaos, no accountability
**Fix:** Use Contribute or custom levels that limit what users can break

### Mistake 2: Folders for Everything
**Problem:** Breaking inheritance on dozens of folders
**Result:** Impossible to audit, performance issues, permission confusion
**Fix:** Use separate libraries or sites for different permission needs

### Mistake 3: No Permission Governance
**Problem:** "We'll figure it out as we go"
**Result:** Three years later, nobody knows who should have access to what
**Fix:** Document governance model *before* widespread adoption

### Mistake 4: Individual User Assignments
**Problem:** Assigning permissions to users directly
**Result:** Can't scale, can't audit, can't manage
**Fix:** Always use groups, even for "temporary" access

### Mistake 5: Ignoring Guest Access
**Problem:** Users sharing externally without oversight
**Result:** Compliance violations, data leaks, unknown access
**Fix:** Enable audit logging, regular reviews, expiration policies

## My Permission Checklist

When I set up a new SharePoint site, I follow this checklist:

**Planning:**
- [ ] Define who needs access and at what level
- [ ] Create necessary groups (Owners, Members, Visitors)
- [ ] Document permission strategy
- [ ] Set external sharing policy

**Implementation:**
- [ ] Assign permissions to groups, not individuals
- [ ] Use inherited permissions wherever possible
- [ ] Configure access request settings
- [ ] Set up sensitivity labels (if applicable)

**Governance:**
- [ ] Document the permission model
- [ ] Set calendar reminders for quarterly reviews
- [ ] Enable audit logging
- [ ] Train site owners on permission best practices

**Monitoring:**
- [ ] Review Site Collection Admins
- [ ] Audit external sharing monthly
- [ ] Check for broken inheritance quarterly
- [ ] Full permission review annually

## Final Thoughts

SharePoint permissions aren't complicated in theory—they're just prone to degradation over time. Start with good patterns, enforce them consistently, and audit regularly.

The permission structure you create today will either make your life easier or haunt you for years. Choose wisely.

## Sources and Further Reading

- [8 Best Practices When Using SharePoint Permissions - NENS](https://www.nens.com/sharepoint-permissions-best-practices/)
- [SharePoint Permission Levels and Best Practices in Microsoft 365](https://o365reports.com/2023/04/11/must-know-sharepoint-permission-levels-and-best-practices-in-microsoft-365/)
- [SharePoint Best Practice Guidance: Permissions Management – NHSmail Support](https://support.nhs.net/knowledge-base/sharepoint-best-practice-guidance-permissions-management/)
- [Top SharePoint permission management practices](https://sharegate.com/blog/sharepoint-permissions-best-practices-2-ways-to-manage)
- [SharePoint Permissions Best Practices: 10 Actionable Insights](https://www.cloudfuze.com/sharepoint-permissions-best-practices-10-actionable-insights/)
- [Understanding SharePoint Permissions Inheritance](https://www.codesigned.com/blog/understanding-sharepoint-permissions-inheritance)
- [What is permissions inheritance? - Microsoft Learn](https://support.office.com/en-us/article/What-is-permissions-inheritance-06bb1ed1-d150-42f4-9600-fb261d4b590c)
- [SharePoint Permissions & Inheritance - Explained! - Acuity Training](https://www.acuitytraining.co.uk/news-tips/sharepoint-permissions-inheritance/)

---

*Questions about SharePoint permissions? [Let me know](mailto:jordan@jordananderson.us).*
