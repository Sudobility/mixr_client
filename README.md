# @sudobility/mixr_client

TypeScript client library for the MIXR cocktail recipe API: a type-safe HTTP client (`MixrClient`), a set of React [TanStack Query](https://tanstack.com/query) hooks, and a [Zustand](https://zustand.docs.pmnd.rs/) store. Ships as ESM.

## Installation

This package is published to **GitHub Packages** under the `@sudobility` scope. Consumers need a `.npmrc` that points the scope at the GitHub registry (and a token with `read:packages` for installs):

```
@sudobility:registry=https://npm.pkg.github.com
```

```bash
bun add @sudobility/mixr_client
```

### Peer dependencies

These must be installed by the consuming app (versions per `package.json`):

```bash
bun add @sudobility/di @sudobility/mixr_types @sudobility/types @tanstack/react-query react zustand
```

| Peer | Range |
|------|-------|
| `@sudobility/di` | ^1.5.85 |
| `@sudobility/mixr_types` | ^0.0.31 |
| `@sudobility/types` | ^1.9.78 |
| `@tanstack/react-query` | >=5.0.0 |
| `react` | >=18.0.0 |
| `zustand` | >=5.0.0 |

## Usage

### API client

`MixrClient` wraps a `NetworkClient` (from `@sudobility/types`) and builds typed requests against the MIXR REST API. It accepts either positional args or a config object:

```typescript
import { MixrClient } from '@sudobility/mixr_client';

const client = new MixrClient(baseUrl, networkClient, authToken?);
// or
const client = new MixrClient({ baseUrl, networkClient, authToken });

client.setAuthToken(token); // update / clear the bearer token later
client.getAuthToken();

// Equipment, ingredients, moods — optional subcategory filter is a plain string
const equipment = await client.getEquipment('essential');
const ingredients = await client.getIngredients();
const moods = await client.getMoods();

// Recipes
const recipe = await client.generateRecipe({ equipment_ids: [1, 2], ingredient_ids: [3, 4], mood_id: 1 });
const recipes = await client.getRecipes({ limit: 10, offset: 0 });

// Ratings
await client.submitRecipeRating(recipeId, { stars: 5, review: 'Great!' });
const aggregate = await client.getRecipeRatingAggregate(recipeId);

// User (auth token required)
const user = await client.getCurrentUser();
const favorites = await client.getUserFavorites();
```

### React hooks

All hooks take `client: MixrClient` as the first argument. Recipe hooks that read/write the cache also take the store hook returned by `createRecipeStore`. Query hooks set a per-resource `staleTime`; see [HOOKS.md](./HOOKS.md) for detailed per-hook docs and examples.

```typescript
import { useRecipes, useCreateRecipe, createRecipeStore } from '@sudobility/mixr_client';

const useRecipeStore = createRecipeStore(storage);

function RecipeList() {
  const { data, fetchNextPage, hasNextPage } = useRecipes(client, useRecipeStore, 10);
  const createRecipe = useCreateRecipe(client, useRecipeStore);
  // ...
}
```

| Hook | Type | Signature (after `client`) | Stale time |
|------|------|-----------------------------|------------|
| `useEquipments` | Query | `(subcategory?)` | 30 min |
| `useEquipmentSubcategories` | Query | `()` | 1 hr |
| `useIngredients` | Query | `(subcategory?)` | 30 min |
| `useIngredientSubcategories` | Query | `()` | 1 hr |
| `useMoods` | Query | `()` | 1 hr |
| `useRecipes` | InfiniteQuery | `(useRecipeStore, limit=10)` | 5 min |
| `useRecipe` | Query | `(useRecipeStore, recipeId)` | 10 min |
| `useCreateRecipe` | Mutation | `(useRecipeStore)` | — |
| `useCurrentUser` | Query | `(enabled=true)` | 10 min |
| `useUpdateUser` | Mutation | `()` | — |
| `useUserPreferences` | Query | `(enabled=true)` | 10 min |
| `useUpdateUserPreferences` | Mutation | `()` | — |
| `useUserRecipes` | Query | `(enabled=true)` | 5 min |
| `useUserFavorites` | Query | `(enabled=true)` | 5 min |
| `useAddFavorite` / `useRemoveFavorite` | Mutation | `()` | — |

Query keys are centralized in the exported `QUERY_KEYS` object (`import { QUERY_KEYS } from '@sudobility/mixr_client'`) for invalidation.

### Zustand store

`createRecipeStore` is a factory that injects a `PlatformStorage` (from `@sudobility/di`) so the same store works on web and React Native. It persists a `Map`-backed recipe cache.

```typescript
import { createRecipeStore } from '@sudobility/mixr_client';

// storage must implement PlatformStorage (getItem/setItem/removeItem).
// Browser localStorage satisfies this; RN apps pass an AsyncStorage adapter.
const useRecipeStore = createRecipeStore(storage, 'mixr-recipe-storage' /* optional key */);
```

Store shape: `recipes: Map<number, Recipe>`, `recipeList: Recipe[]`. Methods: `setRecipe`, `setRecipes`, `getRecipe`, `hasRecipe`, `clear`.

## API methods

| Category | Methods |
|----------|---------|
| Health | `getVersion()`, `healthCheck()` |
| Equipment | `getEquipment(subcategory?)`, `getEquipmentById(id)`, `getEquipmentSubcategories()` |
| Ingredients | `getIngredients(subcategory?)`, `getIngredientById(id)`, `getIngredientSubcategories()` |
| Moods | `getMoods()`, `getMoodById(id)` |
| Recipes | `generateRecipe(req)`, `getRecipes(params?)`, `getRecipeById(id)` |
| Ratings | `submitRecipeRating(id, req)`, `getRecipeRatings(id, params?)`, `getRecipeRatingAggregate(id)`, `deleteRecipeRating(recipeId, ratingId)` |
| User (auth) | `getCurrentUser()`, `updateCurrentUser(req)`, `getUserPreferences()`, `updateUserPreferences(req)`, `getUserRecipes(params?)`, `getUserFavorites(params?)` |
| Favorites (auth) | `addFavorite(req)`, `removeFavorite(recipeId)` |

## Development

Built and tested with [Bun](https://bun.sh). Install dependencies with `bun install` first.

```bash
bun run build        # Compile to dist/ (tsc -p tsconfig.build.json + ESM import fixup)
bun run build:watch  # tsc --watch
bun run clean        # Remove dist/
bun run test:unit    # Run Vitest once
bun run test:watch   # Vitest watch mode
bun run lint         # ESLint (src/**/*.ts,.tsx)
bun run lint:fix     # ESLint --fix
bun run typecheck    # tsc --noEmit
bun run format       # Prettier write
bun run format:check # Prettier check
```

The build runs `tsc` under `moduleResolution: "bundler"` and then `scripts/fix-esm-imports.mjs`, which rewrites relative import specifiers in the emitted `.js`/`.d.ts` to fully-qualified paths so Node's native ESM loader can resolve them. `dist/` is the only published output.

Tests live in `src/__tests__/` and run under Vitest with the `happy-dom` environment and React Testing Library.

## Related packages

- `@sudobility/mixr_types` — shared MIXR type definitions (peer dependency; all public types re-exported from `./types`)
- `@sudobility/types` — `NetworkClient` and shared primitives
- `@sudobility/di` — `PlatformStorage` and DI primitives

## License

BUSL-1.1 (Business Source License 1.1). This package is published with restricted/private access.
