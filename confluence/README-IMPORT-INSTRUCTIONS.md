# Confluence Import Instructions

## 📚 Documentation Files

This directory contains Confluence-compatible HTML documentation for the DSP AI Platform:

1. **Control-Tower-Documentation.html** - Complete Control Tower documentation
2. **Front-Door-Documentation.html** - Complete Front Door documentation  
3. **JWT-Service-Documentation.html** - Complete JWT Service documentation
4. **Integration-Guide.html** - End-to-end integration guide

## 🔧 Import Methods

### Method 1: Direct HTML Import (Recommended)

1. **Navigate to your Confluence space**
2. **Create a new page** or edit an existing one
3. **Click the "..." menu** (More actions)
4. **Select "Import"**
5. **Choose "HTML"** as the import format
6. **Upload the HTML file** or paste the content
7. **Click "Import"**
8. **Review and publish** the page

### Method 2: Copy-Paste with Source Editor

1. **Create a new Confluence page**
2. **Click "..." menu** and select **"View Storage Format"** or **"Source Editor"**
3. **Open the HTML file** in a text editor
4. **Copy all content** from `<body>` to `</body>` (excluding the tags themselves)
5. **Paste into Confluence** source editor
6. **Switch back to visual editor** to preview
7. **Publish the page**

### Method 3: Using Confluence REST API

```bash
# Set your Confluence details
CONFLUENCE_URL="https://your-domain.atlassian.net"
SPACE_KEY="YOUR_SPACE"
USERNAME="your-email@company.com"
API_TOKEN="your-api-token"

# Import Control Tower documentation
curl -X POST \
  "${CONFLUENCE_URL}/wiki/rest/api/content" \
  -u "${USERNAME}:${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d @- << EOF
{
  "type": "page",
  "title": "DSP AI Control Tower Documentation",
  "space": {"key": "${SPACE_KEY}"},
  "body": {
    "storage": {
      "value": "$(cat Control-Tower-Documentation.html | sed 's/"/\\"/g' | tr -d '\n')",
      "representation": "storage"
    }
  }
}
EOF
```

## 📋 Recommended Page Structure

Create a documentation space with this hierarchy:

```
DSP AI Platform
├── Overview (landing page)
├── Control Tower Documentation
├── Front Door Documentation
├── JWT Service Documentation
├── Integration Guide
├── Quick Start Guide
├── API Reference
├── Troubleshooting
└── FAQ
```

## ✨ Features Included

All documentation pages include:

- ✅ **Table of Contents** - Auto-generated navigation
- ✅ **Code Blocks** - Syntax-highlighted examples
- ✅ **Info/Warning/Tip Panels** - Highlighted callouts
- ✅ **Expandable Sections** - Collapsible content
- ✅ **Tables** - Formatted data tables
- ✅ **Cross-References** - Links between pages
- ✅ **Diagrams** - ASCII art architecture diagrams

## 🎨 Confluence Macros Used

The documentation uses these Confluence macros:

- `<ac:structured-macro ac:name="toc">` - Table of Contents
- `<ac:structured-macro ac:name="code">` - Code blocks
- `<ac:structured-macro ac:name="info">` - Info panels
- `<ac:structured-macro ac:name="tip">` - Tip panels
- `<ac:structured-macro ac:name="panel">` - Custom panels
- `<ac:structured-macro ac:name="expand">` - Expandable sections
- `<ac:link>` - Internal page links

## 🔗 Setting Up Cross-References

After importing all pages, update the cross-reference links:

1. Each page has links like: `<ac:link><ri:page ri:content-title="Page Name"/></ac:link>`
2. Confluence will automatically resolve these to actual page links
3. If links don't work, ensure:
   - Page titles match exactly
   - Pages are in the same space
   - You have view permissions

## 🎯 Post-Import Checklist

After importing each page:

- [ ] Verify all code blocks display correctly
- [ ] Check that tables are formatted properly
- [ ] Ensure info/warning panels render correctly
- [ ] Test expandable sections work
- [ ] Verify cross-reference links resolve
- [ ] Check table of contents generates
- [ ] Review on mobile/tablet views
- [ ] Set appropriate page permissions
- [ ] Add page labels/tags for organization
- [ ] Update parent page if needed

## 🛠️ Customization

### Changing Colors

Info panels use these background colors:
- Info (blue): `#DEEBFF`
- Success (green): `#E3FCEF`
- Warning (yellow): `#FFF0B3`
- Error (red): `#FFEBE6`

To change, edit the `bgColor` parameter:
```html
<ac:parameter ac:name="bgColor">#YOUR_COLOR</ac:parameter>
```

### Adding Your Logo

Replace the placeholder text with your logo:
```html
<ac:image>
  <ri:attachment ri:filename="your-logo.png"/>
</ac:image>
```

### Customizing Code Themes

Code blocks support these languages:
- `bash`, `python`, `json`, `yaml`, `javascript`, `sql`, `text`

Change the language parameter:
```html
<ac:parameter ac:name="language">python</ac:parameter>
```

## 📱 Mobile Optimization

The documentation is optimized for mobile viewing:
- Responsive tables
- Collapsible sections for long content
- Readable code blocks with horizontal scroll
- Touch-friendly expandable panels

## 🔍 Search Optimization

To improve searchability:

1. **Add labels** to each page:
   - `dsp-platform`
   - `control-tower` / `front-door` / `jwt-service`
   - `documentation`
   - `api-gateway`
   - `authentication`

2. **Add page descriptions** in page properties

3. **Use consistent terminology** across pages

## 🚀 Bulk Import Script

For importing all files at once:

```bash
#!/bin/bash
# bulk-import.sh

CONFLUENCE_URL="https://your-domain.atlassian.net"
SPACE_KEY="YOUR_SPACE"
USERNAME="your-email@company.com"
API_TOKEN="your-api-token"

FILES=(
  "Control-Tower-Documentation.html:DSP AI Control Tower Documentation"
  "Front-Door-Documentation.html:DSP Front Door Documentation"
  "JWT-Service-Documentation.html:DSP AI JWT Service Documentation"
  "Integration-Guide.html:DSP AI Platform Integration Guide"
)

for file_info in "${FILES[@]}"; do
  IFS=':' read -r file title <<< "$file_info"
  
  echo "Importing $title..."
  
  content=$(cat "$file" | sed 's/"/\\"/g' | tr -d '\n')
  
  curl -X POST \
    "${CONFLUENCE_URL}/wiki/rest/api/content" \
    -u "${USERNAME}:${API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{
      \"type\": \"page\",
      \"title\": \"$title\",
      \"space\": {\"key\": \"${SPACE_KEY}\"},
      \"body\": {
        \"storage\": {
          \"value\": \"$content\",
          \"representation\": \"storage\"
        }
      }
    }"
  
  echo "✓ Imported $title"
  sleep 2
done

echo "All pages imported successfully!"
```

## 📞 Support

If you encounter issues:

1. **Check Confluence version** - These files work with Confluence Cloud and Server 7.0+
2. **Verify permissions** - Ensure you have page creation rights
3. **Test with one page** - Import Control Tower doc first as a test
4. **Check macro compatibility** - Some macros may differ between Cloud/Server
5. **Contact your Confluence admin** - For space-specific issues

## 📝 Version History

- **v1.0** (November 2024) - Initial documentation release
  - Control Tower documentation
  - Front Door documentation
  - JWT Service documentation
  - Integration guide

## 🔄 Updating Documentation

To update existing pages:

1. Export current page to HTML
2. Make changes to the HTML file
3. Re-import using "Import" feature
4. Select "Replace existing content"
5. Review changes before publishing

## 💡 Tips

- **Use page templates** - Create a template from one of these pages for consistency
- **Set up page hierarchy** - Organize with parent-child relationships
- **Enable comments** - Allow team feedback on documentation
- **Set up notifications** - Get alerts when pages are updated
- **Create shortcuts** - Add frequently accessed pages to favorites
- **Use page restrictions** - Control who can view/edit sensitive docs

---

**Ready to import?** Start with the Integration Guide for a complete overview, then import the component-specific documentation as needed.

For questions or issues, contact the DSP AI Platform documentation team.
