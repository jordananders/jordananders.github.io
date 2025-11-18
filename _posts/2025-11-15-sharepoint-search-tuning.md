---
layout: default
title:  "SharePoint Search Tuning: A Practical Guide"
date:   2025-11-15 04:00:00
categories: SharePoint Search Performance
---

SharePoint Search is powerful but often underutilized. Out of the box, it works okay. With tuning, it becomes genuinely useful. I've configured search for organizations with millions of documents—here's what makes the difference.

## Understanding SharePoint Search

### How It Works

1. **Crawler** discovers content (sites, libraries, external sources)
2. **Content processor** extracts text and metadata
3. **Index** stores processed content
4. **Query processor** handles search requests
5. **Results** ranked and returned

Problems can occur at any step.

## Common Search Problems

### "I Can't Find My Document"

Usually caused by:
- Document not yet crawled
- Permissions preventing access
- Missing metadata
- Poor ranking

### "Search Is Too Slow"

Usually caused by:
- Broad queries returning too many results
- Complex refiners
- Large result sets
- Network latency

### "Wrong Results First"

Usually caused by:
- Default ranking not suited to your content
- Missing or incorrect metadata
- No customization of relevance

## Crawling and Indexing

### Full vs Incremental Crawls

**Full crawl:** Re-crawl everything
- Takes hours/days for large environments
- Required after schema changes
- Use sparingly

**Incremental crawl:** Only changed content
- Much faster
- Run frequently (every 15-60 minutes)
- Default approach

**Continuous crawl (SharePoint Online):**
- Near real-time
- Automatic
- Preferred for SharePoint Online

### Check Crawl Status

**SharePoint On-Premises:**
```
Central Administration → Application Management → Manage service applications → Search Service Application → Crawl Log
```

**SharePoint Online:**
```powershell
# PnP PowerShell
Connect-PnPOnline -Url "https://yourtenant-admin.sharepoint.com" -Interactive
Get-PnPSearchCrawlLog -ContentSource "Local SharePoint sites"
```

### Force Re-Indexing

**List/Library:**
```powershell
# Reindex a library
$list = Get-PnPList -Identity "Documents"
$list.ReIndex()
```

**Site:**
```
Site Settings → Search and offline availability → Reindex site
```

## Managed Properties

Managed properties are the key to powerful search. They let you:
- Create custom filters
- Sort by specific fields
- Build custom search experiences

### Creating Managed Properties

**Site Collection:**
```
Site Settings → Search → Schema
```

**Steps:**
1. Find the crawled property (created when crawler finds a column)
2. Create a managed property
3. Map the crawled property to managed property
4. Set properties (Searchable, Queryable, Retrievable, Refinable, Sortable)

**Example: Make "Department" refinable:**

```
Crawled property: ows_Department
Managed property: RefinableDepartment
Mapping: Map crawled to managed
Settings: Refinable = Yes, Queryable = Yes
```

### Property Settings Explained

| Setting | Purpose | When to Enable |
|---------|---------|----------------|
| Searchable | Included in full-text search | Always for text fields |
| Queryable | Can be used in queries | When you need exact matches |
| Retrievable | Returned in results | When you need to display the value |
| Refinable | Available as a refiner | When you need filtering |
| Sortable | Can sort results | When you need sorting by this field |

### Use Existing RefinableString Properties

SharePoint Online provides pre-configured properties:

```
RefinableString00 through RefinableString99
RefinableDate00 through RefinableDate19
RefinableInt00 through RefinableInt49
```

**Mapping existing column:**
1. Create site column "ProjectName"
2. Map crawled property to RefinableString00
3. Use RefinableString00 in search queries

## Search Schema Best Practices

### Use Site Columns (Not List Columns)

Site columns get indexed consistently:

```powershell
# Create site column
Add-PnPField -Type Text -InternalName "ProjectCode" -DisplayName "Project Code" -Group "Custom Columns"

# Add to content type
Add-PnPFieldToContentType -Field "ProjectCode" -ContentType "Project Document"
```

### Consistent Naming

```
Bad:  Project, ProjectName, Project_Name, project-name
Good: ProjectName (everywhere)
```

### Standard Managed Metadata

Use term sets for consistent values:

```
Department (Term Set)
├── Engineering
├── Marketing
├── Sales
└── Support
```

## Query Optimization

### KQL (Keyword Query Language)

**Basic search:**
```
sharepoint migration guide
```

**Field-specific:**
```
Title:migration FileType:pdf
```

**Boolean operators:**
```
sharepoint AND migration NOT 2010
```

**Wildcards:**
```
share* migrat*
```

**Property filters:**
```
Author:"John Doe" Created>2025-01-01
```

### Query Performance Tips

**Avoid leading wildcards:**
```
Bad:  *sharepoint
Good: sharepoint*
```

**Be specific:**
```
Bad:  *
Good: project plan 2025
```

**Limit scope:**
```
Path:https://tenant.sharepoint.com/sites/projects AND budget
```

**Use refiners instead of query filters:**
```
Instead of adding FileType:pdf to every query,
let users click PDF in refiners
```

## Custom Search Experiences

### Promoted Results (Best Bets)

Show specific results for specific queries:

**SharePoint Admin Center:**
```
Search → Query Rules
```

**Create rule:**
- Condition: Query contains "HR policy"
- Action: Promote result to top
- URL: https://tenant.sharepoint.com/sites/hr/policies/handbook.pdf

### Result Sources

Create scopes that limit where searches look:

```
Name: Project Documents
Query: Path:https://tenant.sharepoint.com/sites/projects AND ContentType:Document
```

**Usage:**
- Default for project site search
- Specific search web parts
- API queries

### Display Templates (Classic)

Customize how results appear:

```javascript
// Item_ProjectDocument.html
<div class="ms-srch-item">
    <h3><a href="_#= ctx.CurrentItem.Path =#_">_#= ctx.CurrentItem.Title =#_</a></h3>
    <p>Project: _#= ctx.CurrentItem.ProjectNameOWSText =#_</p>
    <p>Department: _#= ctx.CurrentItem.DepartmentOWSText =#_</p>
</div>
```

### PnP Modern Search (Modern)

Use PnP Modern Search web parts for customization in modern pages.

## Troubleshooting

### Document Not Appearing in Search

**Checklist:**
1. Wait for crawl (can take 15+ minutes)
2. Check permissions (can you access the document?)
3. Check crawl log for errors
4. Force re-index if needed

**PowerShell check:**
```powershell
$results = Submit-PnPSearchQuery -Query "filename.pdf" -All
if ($results.RowCount -eq 0) {
    Write-Host "Document not indexed"
}
```

### Wrong Results Ranking

**Solutions:**
- Create Query Rules to promote correct results
- Adjust result weights in Search Service Application
- Improve metadata (better titles, descriptions)
- Use authoritative pages

### Slow Search

**Check:**
- Query complexity (simplify)
- Number of results (add refiners)
- Network latency (geographic issues)
- Index size (consider archiving)

## Performance Monitoring

### Search Analytics

**SharePoint Admin Center:**
```
Reports → Usage → Search queries
```

**What to look for:**
- Top queries (are they finding results?)
- Abandoned queries (no clicks = bad results)
- Query latency

### Query Logging

**PowerShell:**
```powershell
# Get recent queries
$searchApp = Get-SPEnterpriseSearchServiceApplication
$analytics = Get-SPEnterpriseSearchQueryScope -SearchApplication $searchApp
```

## My Search Tuning Checklist

### Initial Setup:
- [ ] Define key content types
- [ ] Create site columns for important metadata
- [ ] Map managed properties
- [ ] Configure crawl schedule

### Optimization:
- [ ] Analyze top queries
- [ ] Create promoted results for common searches
- [ ] Set up result sources for different scopes
- [ ] Configure refiners for key metadata

### Ongoing:
- [ ] Review search analytics monthly
- [ ] Update promoted results
- [ ] Check for crawl errors
- [ ] Gather user feedback

## Resources

- [SharePoint Search Administration - Microsoft Docs](https://docs.microsoft.com/en-us/sharepoint/search/search-administration)
- [Manage the search schema - SharePoint Online](https://docs.microsoft.com/en-us/sharepoint/manage-search-schema)
- [PnP Modern Search](https://microsoft-search.github.io/pnp-modern-search/)
- [KQL Syntax Reference](https://docs.microsoft.com/en-us/sharepoint/dev/general-development/keyword-query-language-kql-syntax-reference)

---

*Questions about SharePoint search? [Let me know](mailto:jordan@jordananderson.us).*
