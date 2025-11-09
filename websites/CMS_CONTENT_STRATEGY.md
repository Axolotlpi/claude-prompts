# CMS Content Fetching Strategy

A framework-agnostic guide for implementing a scalable, maintainable content management architecture.

## Overview

This strategy emphasizes **separation of concerns** through layered abstractions, making your codebase more maintainable, testable, and adaptable to future changes in your CMS or framework.

## Architecture Layers

```
┌─────────────────────────────────────────┐
│   Pages/Routes (Data Orchestration)    │
│  - Fetch all required content          │
│  - Transform data for components        │
│  - Handle parallel requests             │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│   Helper Functions (Abstraction Layer)  │
│  - Generic content fetching functions   │
│  - Image URL generation                 │
│  - Content format conversion            │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│   Queries Module (Query Definitions)    │
│  - All CMS queries in one place         │
│  - TypeScript types for responses       │
│  - Reformat functions for data shaping  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│   CMS Client (Communication Layer)      │
│  - HTTP/API communication               │
│  - Authentication handling              │
│  - Error handling                       │
└─────────────────────────────────────────┘
```

## Layer 1: CMS Client

**Purpose**: Handle all direct communication with your CMS API.

### Implementation Guidelines

1. **Create a custom client class** (or use the official SDK with a wrapper)
   - Centralize API configuration (project ID, dataset, API version)
   - Handle authentication tokens
   - Manage CORS and headers
   - Parse responses consistently
   - Implement error handling

2. **Keep it generic and reusable**
   ```typescript
   // Example structure (pseudo-code)
   class CMSClient {
     constructor(config) {
       this.projectId = config.projectId;
       this.dataset = config.dataset;
       this.token = config.token;
     }

     async query(queryString) {
       // Construct URL
       // Make HTTP request
       // Parse response
       // Handle errors
       return data;
     }
   }
   ```

3. **Benefits**:
   - Easy to swap CMS providers later
   - Simplified testing (mock the client)
   - Single source of truth for API configuration
   - Consistent error handling

---

## Layer 2: Queries Module

**Purpose**: Define all CMS queries and their associated types in a centralized location.

### Implementation Guidelines

1. **Create a dedicated queries file** (e.g., `queries.ts`, `queries.js`)

2. **Structure each query as an object** containing:
   ```typescript
   {
     query: "RAW_QUERY_STRING",        // The actual CMS query
     reformat: (rawData) => ({...}),   // Transform function
   }
   ```

3. **Define TypeScript types** (or JSDoc for JavaScript):
   - Raw response type from CMS
   - Formatted/transformed type for your app
   ```typescript
   // Example
   export type RawMenuData = {
     _id: string;
     menuItems: Array<{...}>;
   };

   export type FormattedMenuData = {
     id: string;
     items: MenuItem[];
   };
   ```

4. **Export a queries object**:
   ```typescript
   export const queries = {
     siteSettings: {
       query: `*[_type == "settings"][0] { ... }`,
       reformat: (data: RawSettings): FormattedSettings => ({
         // Transform raw data to component-friendly format
       })
     },
     menu: {
       query: `*[_type == "menu"] { ... }`,
       reformat: (data: RawMenu): FormattedMenu => ({
         // Flatten nested structures
         // Rename fields for clarity
         // Calculate derived values
       })
     }
   };
   ```

5. **Benefits**:
   - All queries visible in one place
   - Easy to update or add queries
   - Reformat functions keep transformation logic organized
   - Type safety throughout the application
   - Framework agnostic (just functions and strings)

---

## Layer 3: Helper Functions

**Purpose**: Provide convenient, reusable functions that abstract CMS interactions.

### Implementation Guidelines

1. **Create a helpers file** (e.g., `helpers.ts`, `cms-helpers.js`)

2. **Instantiate your CMS client**:
   ```typescript
   import { CMSClient } from './cmsClient';

   const client = new CMSClient({
     projectId: 'your-project-id',
     dataset: 'production',
     token: import.meta.env.CMS_TOKEN // or process.env
   });
   ```

3. **Create abstraction functions**:
   ```typescript
   // Generic content fetcher
   export async function getContent<T>(
     queryKey: keyof typeof queries
   ): Promise<T> {
     const { query, reformat } = queries[queryKey];
     const rawData = await client.query(query);
     return reformat(rawData);
   }

   // Image URL generator
   export function getImageUrl(imageRef, options = {}) {
     // Use CMS image builder or construct URL
     return imageUrl;
   }

   // Content format converter (e.g., rich text to markdown)
   export function convertContent(blockContent) {
     // Transform CMS content format to usable format
     return convertedContent;
   }

   // Specific fetchers (optional, for common patterns)
   export async function getIsOpen(): Promise<boolean> {
     const settings = await getContent('liveSettings');
     return settings.isOpen;
   }
   ```

4. **Benefits**:
   - Components never directly touch the CMS client
   - Easy to mock in tests
   - Can add caching, retry logic, or logging here
   - Single place to update if CMS client changes

---

## Layer 4: Page-Level Data Orchestration

**Purpose**: Fetch and prepare all content needed for a page/route.

### Implementation Guidelines

1. **Fetch at the appropriate level for your framework**:
   - **SSR Frameworks** (Next.js, SvelteKit, Nuxt): Use server-side loaders
   - **Client-side Frameworks**: Use component lifecycle methods or hooks
   - **Static Generation**: Use build-time data fetching

2. **Use parallel fetching** for independent queries:
   ```typescript
   // SvelteKit example (load function)
   export async function load() {
     const [menu, settings, info] = await Promise.all([
       getContent('menu'),
       getContent('siteSettings'),
       getContent('businessInfo')
     ]);

     return { menu, settings, info };
   }

   // React example (useEffect or loader)
   useEffect(() => {
     Promise.all([
       getContent('menu'),
       getContent('siteSettings'),
       getContent('businessInfo')
     ]).then(([menu, settings, info]) => {
       setData({ menu, settings, info });
     });
   }, []);
   ```

3. **Transform data for component consumption**:
   - Combine related data
   - Extract nested values
   - Apply business logic
   - Structure data to match component prop interfaces
   ```typescript
   // Example transformation
   export async function load() {
     const [menu, settings] = await Promise.all([
       getContent('menu'),
       getContent('siteSettings')
     ]);

     // Combine and shape for components
     return {
       menuWithSettings: {
         items: menu.items,
         displayMode: settings.menuDisplayMode,
         showPrices: settings.showPrices
       }
     };
   }
   ```

4. **Handle errors gracefully**:
   ```typescript
   try {
     const data = await getContent('menu');
     return { data };
   } catch (error) {
     console.error('Failed to fetch menu:', error);
     return { data: null, error: error.message };
   }
   ```

5. **Benefits**:
   - All page data fetched in one place
   - Clear data dependencies
   - Optimal performance with parallel requests
   - Components remain pure (just props in, UI out)

---

## Framework-Specific Adaptations

### Server-Side Rendering (Next.js, SvelteKit, Nuxt)

- Use framework's data loading functions
- Fetch during server-side rendering
- Pass data as props to components
- **Example locations**:
  - Next.js: `getServerSideProps`, `getStaticProps`, or App Router loaders
  - SvelteKit: `+page.ts` or `+page.server.ts` load functions
  - Nuxt: `asyncData` or `useFetch`

### Client-Side Rendering (React, Vue, Svelte with CSR)

- Fetch in component lifecycle (useEffect, onMount, etc.)
- Consider using data fetching libraries (React Query, SWR, etc.)
- Implement loading and error states
- **Example**: React with hooks
  ```typescript
  function MenuPage() {
    const [data, setData] = useState(null);

    useEffect(() => {
      getContent('menu').then(setData);
    }, []);

    if (!data) return <Loading />;
    return <Menu data={data} />;
  }
  ```

### Static Site Generation (Astro, Eleventy, Gatsby)

- Fetch during build time
- Generate static HTML with embedded data
- Consider incremental regeneration for updates
- **Example**: Astro
  ```typescript
  ---
  const menu = await getContent('menu');
  const settings = await getContent('siteSettings');
  ---

  <Menu data={menu} settings={settings} />
  ```

---

## Component Guidelines

### Keep Components Pure

Components should:
- Accept data via props
- Not directly fetch CMS content
- Not know about the CMS at all
- Be framework-agnostic (just UI logic)

```typescript
// Good: Pure component
function MenuCard({ title, items, showPrices }) {
  return (
    <div>
      <h2>{title}</h2>
      {items.map(item => (
        <MenuItem key={item.id} {...item} showPrice={showPrices} />
      ))}
    </div>
  );
}

// Avoid: Component that fetches
function MenuCard() {
  const [data, setData] = useState(null);
  useEffect(() => {
    getContent('menu').then(setData); // Don't do this
  }, []);
  // ...
}
```

### Exception: Container Components

For complex features, you may create "container" components that:
- Fetch their own data
- Poll for updates (e.g., live status)
- Are explicitly marked as data-aware

```typescript
// Acceptable: Container component with polling
function LiveStatusBanner() {
  const [isOpen, setIsOpen] = useState(false);

  useEffect(() => {
    const poll = setInterval(async () => {
      const status = await getIsOpen();
      setIsOpen(status);
    }, 60000); // Poll every 60s

    return () => clearInterval(poll);
  }, []);

  return <StatusBanner isOpen={isOpen} />;
}
```

---

## File Organization

Recommended structure:

```
src/
├── lib/
│   ├── cms/
│   │   ├── client.ts       # CMS client implementation
│   │   ├── queries.ts      # All query definitions + types
│   │   └── helpers.ts      # Abstraction functions
│   ├── components/         # Pure UI components
│   ├── compositions/       # Complex feature assemblies
│   └── config.ts           # App configuration
└── routes/ or pages/       # Framework-specific routing
```

---

## Benefits of This Strategy

1. **Maintainability**: Clear separation makes code easier to understand and modify
2. **Testability**: Each layer can be tested independently
3. **Flexibility**: Easy to swap CMS providers or frameworks
4. **Type Safety**: Strong typing throughout the data flow
5. **Performance**: Parallel fetching and optimized queries
6. **Developer Experience**: Intuitive patterns, autocomplete support
7. **Scalability**: Adding new content types follows the same pattern

---

## Migration Example: Same Strategy, Different Stack

### Original: SvelteKit + Sanity
```typescript
// queries.ts
export const queries = {
  menu: {
    query: `*[_type == "menu"]{ ... }`,
    reformat: (data) => ({ items: data.menuItems })
  }
};

// +page.ts (SvelteKit loader)
export async function load() {
  const menu = await getContent('menu');
  return { menu };
}
```

### Migration: Next.js + Contentful
```typescript
// queries.ts
export const queries = {
  menu: {
    query: `{ menuCollection { items { ... } } }`, // GraphQL
    reformat: (data) => ({ items: data.menuCollection.items })
  }
};

// page.tsx (Next.js App Router)
export default async function MenuPage() {
  const menu = await getContent('menu');
  return <Menu data={menu} />;
}
```

**Only the CMS client and query syntax changed. The overall pattern remains the same.**

---

## Additional Considerations

### Caching
- Implement caching in the helper layer
- Use framework-specific cache (Next.js cache, SvelteKit cache)
- Consider Redis for shared cache across instances

### Real-Time Updates
- For live data (e.g., "open/closed" status), implement polling or webhooks
- Poll from container components, not page loaders
- Consider WebSocket connections for instant updates

### Error Boundaries
- Implement error boundaries at the page/route level
- Provide fallback UI when content fails to load
- Log errors for monitoring

### Content Preview
- Add draft/preview mode support in the CMS client
- Use query parameters or cookies to enable preview
- Fetch draft content instead of published

---

## Conclusion

This strategy creates a **robust, scalable architecture** for CMS-driven websites. By maintaining clear boundaries between layers, you ensure that your codebase remains:
- Easy to understand for new developers
- Simple to test and debug
- Adaptable to changing requirements
- Framework and CMS agnostic

Follow these patterns consistently, and you'll build a maintainable content-driven application that can evolve with your needs.
