# Sidekick Skills Gallery — Setup Guide

Step-by-step instructions to recreate the Sidekick Skills Gallery from scratch on a Shopify store.

## Prerequisites

- A Shopify store (dev store works)
- The Horizon theme with the Skills Gallery code deployed (via GitHub integration or `shopify theme push`)
- Access to the [Shopify Admin GraphQL API](https://shopify.dev/docs/api/admin-graphql) — use the [GraphiQL app](https://shopify-graphiql-app.shopifycloud.com/), Shopify CLI, or `curl` with an Admin API token

## Overview

The Skills Gallery consists of:

| File | Purpose |
|------|---------|
| `snippets/skill-card.liquid` | Reusable card component for displaying a skill |
| `sections/skills-gallery.liquid` | Gallery section with category filter tabs and card grid |
| `templates/page.skills.json` | Page template for the gallery listing |
| `sections/skill-detail.liquid` | Detail view for a single skill |
| `templates/metaobject/skill.json` | Metaobject template routing `/skills/{handle}` to the detail section |

Two metaobject definitions power the data:

- **`skill_category`** — Categories like "Analytics", "Marketing", etc.
- **`skill`** — Individual skills with prompts, commands, and metadata

---

## Step 1: Create the `skill_category` Metaobject Definition

```graphql
mutation {
  metaobjectDefinitionCreate(definition: {
    type: "skill_category"
    name: "Skill Category"
    displayNameKey: "name"
    access: {
      storefront: PUBLIC_READ
    }
    fieldDefinitions: [
      { key: "name", name: "Name", type: "single_line_text_field", required: true }
      { key: "emoji", name: "Emoji", type: "single_line_text_field" }
      { key: "sort_order", name: "Sort Order", type: "number_integer" }
    ]
  }) {
    metaobjectDefinition { id type }
    userErrors { field message }
  }
}
```

Save the returned `id` — you'll need it in Step 2b.

---

## Step 2a: Create the `skill` Metaobject Definition

> **Note:** The `related_skills` field (self-referencing) cannot be added during creation. We add it in Step 2b.

> **Note:** `file_type_options` values must be title-case (`"Image"` not `"IMAGE"`).

```graphql
mutation {
  metaobjectDefinitionCreate(definition: {
    type: "skill"
    name: "Skill"
    displayNameKey: "name"
    access: {
      storefront: PUBLIC_READ
    }
    capabilities: {
      publishable: { enabled: true }
      onlineStore: { enabled: true, data: { urlHandle: "skills" } }
    }
    fieldDefinitions: [
      { key: "name", name: "Name", type: "single_line_text_field", required: true }
      { key: "command", name: "Command", type: "single_line_text_field" }
      { key: "tagline", name: "Tagline", type: "single_line_text_field" }
      { key: "steps", name: "Prompt Steps", type: "multi_line_text_field", required: true }
      { key: "creator", name: "Creator", type: "single_line_text_field" }
      { key: "creator_url", name: "Creator URL", type: "url" }
      { key: "categories", name: "Categories", description: "Comma-separated category names", type: "single_line_text_field" }
      { key: "featured", name: "Featured", type: "boolean" }
      {
        key: "image"
        name: "Image"
        type: "file_reference"
        validations: [{ name: "file_type_options", value: "[\"Image\"]" }]
      }
    ]
  }) {
    metaobjectDefinition { id type }
    userErrors { field message }
  }
}
```

Save the returned `id` (e.g. `gid://shopify/MetaobjectDefinition/XXXXX`) — you'll need it for Step 2b.

---

## Step 2b: Add the `related_skills` Field

Use the `id` returned from Step 2a for both `SKILL_DEFINITION_ID` values:

```graphql
mutation {
  metaobjectDefinitionUpdate(
    id: "SKILL_DEFINITION_ID"
    definition: {
      fieldDefinitions: [
        {
          create: {
            key: "related_skills"
            name: "Related Skills"
            type: "list.metaobject_reference"
            validations: [{
              name: "metaobject_definition_type"
              value: "SKILL_DEFINITION_ID"
            }]
          }
        }
      ]
    }
  ) {
    metaobjectDefinition { id }
    userErrors { field message }
  }
}
```

---

## Step 3: Verify `onlineStore` Capability

Confirm the `skill` definition has `onlineStore` enabled with the correct `urlHandle`:

```graphql
{
  metaobjectDefinitionByType(type: "skill") {
    id
    capabilities {
      publishable { enabled }
      onlineStore { enabled data { urlHandle } }
    }
  }
}
```

Expected: `onlineStore.enabled: true`, `urlHandle: "skills"`.

If `onlineStore` shows `false`, enable it:

```graphql
mutation {
  metaobjectDefinitionUpdate(
    id: "SKILL_DEFINITION_ID"
    definition: {
      capabilities: {
        onlineStore: {
          enabled: true
          data: { urlHandle: "skills" }
        }
      }
    }
  ) {
    metaobjectDefinition {
      capabilities {
        onlineStore { enabled data { urlHandle } }
      }
    }
    userErrors { field message }
  }
}
```

---

## Step 4: Create Category Entries

Create as many categories as needed. Batch multiple in one request:

```graphql
mutation {
  analytics: metaobjectCreate(metaobject: {
    type: "skill_category"
    handle: "analytics"
    fields: [
      { key: "name", value: "Analytics" }
      { key: "emoji", value: "\ud83d\udcca" }
      { key: "sort_order", value: "1" }
    ]
  }) {
    metaobject { id handle }
    userErrors { field message }
  }

  marketing: metaobjectCreate(metaobject: {
    type: "skill_category"
    handle: "marketing"
    fields: [
      { key: "name", value: "Marketing" }
      { key: "emoji", value: "\ud83d\udce3" }
      { key: "sort_order", value: "2" }
    ]
  }) {
    metaobject { id handle }
    userErrors { field message }
  }

  inventory: metaobjectCreate(metaobject: {
    type: "skill_category"
    handle: "inventory"
    fields: [
      { key: "name", value: "Inventory" }
      { key: "emoji", value: "\ud83d\udce6" }
      { key: "sort_order", value: "3" }
    ]
  }) {
    metaobject { id handle }
    userErrors { field message }
  }
}
```

---

## Step 5: Create Skill Entries

Each skill must be published (`ACTIVE`) and have `onlineStore.templateSuffix` set for its detail page to work:

```graphql
mutation {
  metaobjectCreate(metaobject: {
    type: "skill"
    handle: "weekly-sales"
    capabilities: {
      publishable: { status: ACTIVE }
      onlineStore: { templateSuffix: "" }
    }
    fields: [
      { key: "name", value: "Weekly Sales Report" }
      { key: "command", value: "/weekly-sales" }
      { key: "tagline", value: "Generate a comprehensive weekly sales summary with trend analysis" }
      { key: "steps", value: "1. Pull sales data from the last 7 days\n2. Break down revenue by product category\n3. Compare with the previous week and highlight changes > 10%\n4. List the top 5 best-selling products\n5. Summarize key takeaways in 3 bullet points" }
      { key: "creator", value: "Shopify" }
      { key: "creator_url", value: "https://shopify.com" }
      { key: "categories", value: "Analytics" }
      { key: "featured", value: "true" }
    ]
  }) {
    metaobject { id handle }
    userErrors { field message }
  }
}
```

Another example:

```graphql
mutation {
  metaobjectCreate(metaobject: {
    type: "skill"
    handle: "restock-alert"
    capabilities: {
      publishable: { status: ACTIVE }
      onlineStore: { templateSuffix: "" }
    }
    fields: [
      { key: "name", value: "Restock Alert" }
      { key: "command", value: "/restock-alert" }
      { key: "tagline", value: "Find products running low and draft a restock plan" }
      { key: "steps", value: "1. Check inventory levels for all active products\n2. Flag any variants with fewer than 10 units in stock\n3. Sort flagged items by sales velocity (units sold per week)\n4. Suggest reorder quantities based on 30-day demand\n5. Draft a purchase order summary" }
      { key: "creator", value: "Shopify" }
      { key: "categories", value: "Inventory" }
      { key: "featured", value: "false" }
    ]
  }) {
    metaobject { id handle }
    userErrors { field message }
  }
}
```

### Updating an existing skill entry

If you need to re-publish or update a skill (e.g. after enabling `onlineStore` on the definition):

```graphql
mutation {
  metaobjectUpdate(
    id: "SKILL_ENTRY_GID"
    metaobject: {
      capabilities: {
        publishable: { status: ACTIVE }
        onlineStore: { templateSuffix: "" }
      }
    }
  ) {
    metaobject { handle }
    userErrors { field message }
  }
}
```

---

## Step 6: Create the "Skills" Gallery Page

```graphql
mutation {
  pageCreate(page: {
    title: "Skills"
    handle: "skills"
    templateSuffix: "skills"
    body: ""
    isPublished: true
  }) {
    page { id handle title }
    userErrors { field message }
  }
}
```

The gallery is now at `/pages/skills`.

---

## Step 7: Connect Data in the Theme Editor

This step must be done in the Shopify Admin UI:

1. Go to **Online Store > Themes > Customize**
2. Use the page selector dropdown at the top to navigate to the **Skills** page
3. Click the **Skills Gallery** section in the sidebar
4. Under **"Skills"**, click **Connect** and select all your `skill` metaobject entries
5. Under **"Categories"**, click **Connect** and select all your `skill_category` entries
6. Adjust grid columns, spacing, and color scheme as desired
7. Click **Save**

---

## Step 8: Verify

| URL | Expected |
|-----|----------|
| `/pages/skills` | Gallery with filter tabs and skill cards |
| `/skills/{handle}` | Skill detail page (if `onlineStore` is working) |
| Click "Add to Sidekick" | Opens `admin.shopify.com/sidekick?sk_s=...&sk_p=...` |
| Category filter tabs | Show/hide cards by category |
| "Copy prompt" / "Copy install link" / "Copy page link" | Copies to clipboard |

---

## Useful Queries

### List all skills

```graphql
{
  metaobjects(type: "skill", first: 50) {
    nodes {
      id
      handle
      displayName
      fields {
        key
        value
      }
    }
  }
}
```

### List all categories

```graphql
{
  metaobjects(type: "skill_category", first: 20) {
    nodes {
      id
      handle
      displayName
      fields {
        key
        value
      }
    }
  }
}
```

### Check a definition's capabilities

```graphql
{
  metaobjectDefinitionByType(type: "skill") {
    id
    type
    fieldDefinitions {
      key
      name
      type { name }
      required
    }
    capabilities {
      publishable { enabled }
      onlineStore { enabled data { urlHandle } }
    }
  }
}
```

### Delete a skill entry

```graphql
mutation {
  metaobjectDelete(id: "SKILL_ENTRY_GID") {
    deletedId
    userErrors { field message }
  }
}
```

---

## Metaobject Field Reference

### `skill` fields

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `single_line_text_field` | Yes | Skill display name |
| `command` | `single_line_text_field` | No | Slash command (e.g. `/weekly-sales`) |
| `tagline` | `single_line_text_field` | No | Short description |
| `steps` | `multi_line_text_field` | Yes | The full prompt text |
| `creator` | `single_line_text_field` | No | Creator name |
| `creator_url` | `url` | No | Link to creator's site |
| `categories` | `single_line_text_field` | No | Comma-separated category names |
| `featured` | `boolean` | No | Featured skills appear first in the gallery |
| `image` | `file_reference` (Image) | No | Card/detail hero image |
| `related_skills` | `list.metaobject_reference` | No | Links to related skill entries |

### `skill_category` fields

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | `single_line_text_field` | Yes | Category display name |
| `emoji` | `single_line_text_field` | No | Emoji shown on filter tab |
| `sort_order` | `number_integer` | No | Controls tab ordering |

---

## Gotchas

- **Self-referencing metaobjects**: You cannot add a `list.metaobject_reference` field that references its own type during `metaobjectDefinitionCreate`. Create the definition first, then add the field via `metaobjectDefinitionUpdate`.
- **Validation values**: The `metaobject_definition_type` validation requires the full GID (`gid://shopify/MetaobjectDefinition/XXXXX`), not just the type string.
- **File type validations**: Use title-case values: `"Image"`, `"Video"`, `"GenericFile"` — not `"IMAGE"`.
- **onlineStore capability**: Must include `data: { urlHandle: "..." }` or you'll get a "URL handle cannot be blank" error.
- **Publishing entries**: Each metaobject entry needs `capabilities.publishable.status: ACTIVE` and `capabilities.onlineStore.templateSuffix: ""` to appear on the storefront.
- **Iterating metaobject lists in Liquid**: Use `for item in section.settings.my_list` — do NOT append `.values` (that's for `shop.metaobjects.type.values`, not section settings).
