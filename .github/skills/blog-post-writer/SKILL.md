---
name: blog-post-writer
description: Expert guidance for developing and contributing to Azure Quick Review (azqr) - A Go-based CLI tool for Azure resource compliance analysis
---

# Blog Post Writer Skill

## Description
This skill helps create new technical blog posts for a Hugo-based blog with proper structure, documentation references, and metadata. It leverages the Microsoft Learn MCP server to ensure accurate technical content and proper citations.

## Trigger Phrases
- "write a blog post about..."
- "create a new post about..."
- "generate a blog article on..."
- "help me write a post about..."
- "new blog post on..."

## Workspace Context
- **Blog Engine**: Hugo static site generator
- **Posts Location**: `content/posts/YYYY/` (where YYYY is the current year)
- **File Naming**: `YYYY-MM-DD-title-slug.md`
- **Author**: Carlos Mendible

## Post Structure Requirements

### Front Matter (YAML)
Every post must include the following front matter:

```yaml
---
author: Carlos Mendible
categories:
- azure  # or other relevant categories
date: "YYYY-MM-DD"
description: 'Brief description of the post (1-2 sentences)'
draft: false  # or true if work in progress
tags: ["tag1", "tag2", "tag3"]
title: 'Post Title'
---
```

### Content Structure
1. **Introduction** (1-2 paragraphs)
   - Set the context
   - Explain what will be covered
   - Use blockquotes (`>`) for important notes or highlights

2. **Prerequisites** (if applicable)
   - List required tools, accounts, or knowledge
   - Use bullet points for clarity

3. **Main Content**
   - Break into logical sections with H2 headers (`##`)
   - Use H3 headers (`###`) for subsections
   - Include code blocks with appropriate language identifiers
   - Add explanatory text between code sections
   - Use inline code for commands, file names, and technical terms

4. **Conclusion** (optional)
   - Brief summary or closing thoughts
   - Often ends with "Hope it helps!"

5. **References** (REQUIRED)
   - Must include at least 2-3 relevant Microsoft Learn or official documentation links
   - Format:
     ```markdown
     **References:**
     
     * [Link Title](https://learn.microsoft.com/...)
     * [Another Reference](https://github.com/...)
     ```

## Instructions for Generating Posts

### Step 1: Understand the Topic
1. Ask the user for:
   - Topic/technology to write about
   - Target audience level (beginner/intermediate/advanced)
   - Specific scenario or use case
   - Preferred categories and tags

### Step 2: Research with Microsoft Learn MCP
1. **ALWAYS use `mcp_microsoft-lea_microsoft_docs_search`** to find relevant documentation:
   ```
   Example query: "Azure Kubernetes Service deployment best practices"
   ```

2. **For code examples, use `mcp_microsoft-lea_microsoft_code_sample_search`**:
   ```
   Example: Search for relevant code snippets in the target language
   ```

3. **If detailed information needed, use `mcp_microsoft-lea_microsoft_docs_fetch`**:
   ```
   Fetch complete documentation pages for in-depth content
   ```

### Step 3: Generate Metadata
1. Create appropriate file name: `YYYY-MM-DD-topic-slug.md`
2. Generate front matter with:
   - Current date
   - Descriptive title (capitalize major words)
   - Brief description (1-2 sentences)
   - Relevant categories (typically 'azure' for Azure content)
   - 3-5 specific tags (lowercase, hyphenated if needed)
   - Set `draft: false` unless user requests draft mode

### Step 4: Structure the Content
1. Write an engaging introduction (1-2 paragraphs)
2. Add Prerequisites section if needed
3. Create main content sections:
   - Use clear section headers
   - Include step-by-step instructions
   - Add code blocks with proper syntax highlighting
   - Explain each major step
   - Use `bash`, `yaml`, `python`, `terraform`, `bicep`, etc. for code blocks

### Step 5: Add References (CRITICAL)
1. **Extract URLs from MCP server responses** used during research
2. **Format as markdown bullet list** under `**References:**` header
3. **Include at least 2-3 official Microsoft Learn links**
4. Additional references can include:
   - GitHub repositories
   - Official product documentation
   - Related tools or resources

### Step 6: Review and Validate
Before presenting to user, verify:
- [ ] Front matter is complete and properly formatted
- [ ] Date format is correct (YYYY-MM-DD)
- [ ] Code blocks have language identifiers
- [ ] At least 2-3 references from Microsoft Learn included
- [ ] All URLs are valid and properly formatted
- [ ] Content follows the blog's style (technical but accessible)
- [ ] File name follows convention: YYYY-MM-DD-topic-slug.md

## Example Workflow

**User**: "Write a blog post about deploying AKS with Terraform"

**Agent Actions**:
1. Use `mcp_microsoft-lea_microsoft_docs_search` with query "Azure Kubernetes Service AKS terraform deployment"
2. Use `mcp_microsoft-lea_microsoft_code_sample_search` with query "terraform azurerm kubernetes cluster" and language "terraform"
3. Generate file name: `2026-02-01-deploying-aks-with-terraform.md`
4. Create front matter with:
   - categories: azure
   - tags: ["aks", "terraform", "kubernetes", "infrastructure-as-code"]
   - Current date
5. Write introduction explaining AKS and Terraform benefits
6. Add Prerequisites: Azure subscription, Terraform installed, Azure CLI
7. Structure main content:
   - Setting up the Terraform configuration
   - Defining the AKS cluster resource
   - Configuring node pools
   - Deploying the infrastructure
   - Verifying the deployment
8. Add References section with URLs from MCP server responses:
   - Microsoft Learn AKS documentation
   - Terraform AzureRM provider docs
   - AKS best practices guide

## Best Practices

### Content Style
- **Technical but accessible**: Assume competent readers but explain concepts clearly
- **Action-oriented**: Use imperative mood for instructions
- **Code-heavy**: Include practical, working code examples
- **Well-structured**: Use headers, lists, and formatting for readability

### Documentation Integration
- **Leverage MCP server**: Always search Microsoft Learn for authoritative information
- **Verify accuracy**: Use official documentation for technical details
- **Cite sources**: Every technical claim should have a reference
- **Stay current**: Prefer latest documentation and API versions

### Common Categories
- `azure` - For Azure-related posts
- `kubernetes` - For K8s content
- `devops` - For automation and CI/CD
- `dotnet` - For .NET development
- `security` - For security-focused content

### Common Tags Format
- Lowercase
- Hyphenated for multi-word tags (e.g., `github-actions`, `azure-functions`)
- Product names: `aks`, `terraform`, `bicep`, `azqr`, `aifoundry`
- Technologies: `kubernetes`, `docker`, `helm`
- Concepts: `infrastructure-as-code`, `gitops`, `security`

## Error Prevention

1. **Never skip the references section** - This is a requirement for every post
2. **Always use MCP server** - Don't rely solely on training data
3. **Validate file naming** - Must follow YYYY-MM-DD-slug.md format
4. **Check front matter** - YAML must be valid and complete
5. **Verify code blocks** - Must have proper language identifiers
6. **Confirm output path** - Posts go in `content/posts/YYYY/` directory

## Post-Generation Actions

After generating the post:
1. Create the file in `content/posts/YYYY/` where YYYY is current year
2. Show the user the generated content
3. Offer to:
   - Adjust the content
   - Add more sections
   - Search for additional references
   - Change categories or tags
   - Set as draft or publish-ready

## Notes
- The blog uses Hugo with content organized by year
- Posts from 2009-2025 exist, showing long-running blog
- Technical focus is primarily Azure, Kubernetes, and cloud-native technologies
- Author has consistent writing style: practical, code-focused, ends with "Hope it helps!"
