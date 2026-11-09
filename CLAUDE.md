# MIXR Client

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it. Stay on the current branch; do not create
> feature branches for code changes.

TypeScript client library for the MIXR cocktail recipe API: `MixrClient` (HTTP), TanStack Query hooks, and a Zustand recipe store. Pure library — no app entrypoint. Ships ESM only; `dist/` is the sole published artifact.

**npm**: `@sudobility/mixr_client` — published to **GitHub Packages** (restricted), license **BUSL-1.1**.

## Tech Stack

- **Language**: TypeScript (strict; `moduleResolution: "bundler"`, `module: ESNext`, target ES2020)
- **Runtime / package manager**: Bun — do NOT use npm/yarn/pnpm to install
- **Build**: `tsc` + post-build `scripts/fix-esm-imports.mjs`
- **Test**: Vitest + React Testing Library + happy-dom

## Commands

```bash
bun install          # restore deps (node_modules is gitignored and may be absent)
bun run build        # tsc -p tsconfig.build.json && node scripts/fix-esm-imports.mjs
bun run build:watch  # tsc --watch
bun run clean        # remove dist/
bun run test:unit    # Vitest run (NOTE: no `test` script; use test:unit / test:watch)
bun run test:watch   # Vitest watch
bun run lint         # ESLint src --ext .ts,.tsx
bun run typecheck    # tsc --noEmit
bun run format       # Prettier write
```

## Project Structure

```
src/
├── index.ts                  # Public barrel: re-exports network/utils/hooks/stores + all types from ./types
├── types.ts                  # Type re-exports from @sudobility/mixr_types
├── network/
│   ├── index.ts
│   └── mixr-client.ts        # MixrClient class — all ~30 API methods
├── hooks/
│   ├── index.ts              # Hook barrel + QUERY_KEYS + UseRecipeStore type
│   ├── query-keys.ts         # QUERY_KEYS: single source of truth for TanStack query keys
│   ├── use-recipe-store-type.ts
│   ├── use-equipments.ts / use-equipment-subcategories.ts
│   ├── use-ingredients.ts / use-ingredient-subcategories.ts
│   ├── use-moods.ts
│   ├── use-recipes.ts / use-recipe.ts / use-create-recipe.ts
│   └── use-current-user.ts, use-update-user.ts, use-user-preferences.ts,
│       use-update-user-preferences.ts, use-user-recipes.ts, use-user-favorites.ts,
│       use-toggle-favorite.ts  (useAddFavorite + useRemoveFavorite)
├── stores/{index.ts, recipe-store.ts}   # createRecipeStore (Zustand + persist)
├── utils/{index.ts, helpers.ts}         # buildUrl, createHeaders, handleApiError
└── __tests__/                           # Vitest suites mirroring the layout (setup.ts, network/, hooks/, stores/, utils/)
```

See `HOOKS.md` for per-hook usage docs.

## MixrClient

```typescript
new MixrClient(baseUrl, networkClient, authToken?)
new MixrClient({ baseUrl, networkClient, authToken? })  // both constructors supported
client.setAuthToken(token) / client.getAuthToken()
```

Wraps a `NetworkClient` (`@sudobility/types`). A private `request<T>()` helper centralizes method/URL/header building and error handling: on `!response.ok || !response.data` it throws `handleApiError(response, name)`.

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

Subcategory filters are passed as a plain string (e.g. `getEquipment('essential')`), NOT an object.

## Hooks

Every hook takes `client: MixrClient` first. Recipe hooks that touch the cache also take `useRecipeStore` (the store hook from `createRecipeStore`).

| Hook | Kind | Args after client | staleTime |
|------|------|-------------------|-----------|
| `useEquipments` / `useIngredients` | Query | `(subcategory?)` | 30 min |
| `useEquipmentSubcategories` / `useIngredientSubcategories` / `useMoods` | Query | `()` | 1 hr |
| `useRecipes` | InfiniteQuery | `(useRecipeStore, limit=10)` | 5 min |
| `useRecipe` | Query | `(useRecipeStore, recipeId)` | 10 min |
| `useCreateRecipe` | Mutation | `(useRecipeStore)` | — |
| `useCurrentUser` / `useUserPreferences` | Query | `(enabled=true)` | 10 min |
| `useUserRecipes` / `useUserFavorites` | Query | `(enabled=true)` | 5 min |
| `useUpdateUser` / `useUpdateUserPreferences` / `useAddFavorite` / `useRemoveFavorite` | Mutation | `()` | — |

Query keys live in `QUERY_KEYS` (`query-keys.ts`): `equipment.{all,list(sub?),subcategories}`, `ingredients.{…}`, `moods.{all,list}`, `recipes.{all,list(limit),detail(id)}`, `user.{all,profile,preferences,recipes,favorites}`. Mutations invalidate via these keys. Use them for invalidation rather than inline arrays.

## Store

```typescript
const useRecipeStore = createRecipeStore(storage /* PlatformStorage */, storageName?);
```

Factory injects a `PlatformStorage` (`@sudobility/di`) so web (localStorage) and RN (AsyncStorage adapter) share one store. Persists via `zustand/middleware` with a custom replacer/reviver that serializes the `recipes: Map<number, Recipe>` as entries. Default key `mixr-recipe-storage`. Shape: `recipes` (Map), `recipeList` (array); methods `setRecipe`, `setRecipes`, `getRecipe`, `hasRecipe`, `clear`.

## Dependency / Architecture Notes

- Peer deps: `@sudobility/di ^1.5.85`, `@sudobility/mixr_types ^0.0.31`, `@sudobility/types ^1.9.78`, `@tanstack/react-query >=5`, `react >=18`, `zustand >=5`.
- All public API types come from `@sudobility/mixr_types`, re-exported through `src/types.ts` and `src/index.ts` — add new type re-exports there, don't redefine locally.
- `RecipeListParams` in `mixr-client.ts` is a deprecated alias for `PaginationQueryParams`.
- Consumers (and this repo) need `.npmrc` with `@sudobility:registry=https://npm.pkg.github.com` to resolve `@sudobility/*` from GitHub Packages.

## Gotchas

- **No `test` script** — it's `test:unit` / `test:watch`.
- **ESM import fixup is mandatory**: `tsc` emits extensionless relative imports (`bundler` resolution); `scripts/fix-esm-imports.mjs` rewrites them to `.js`/`index.js` so Node's ESM loader works. The script is idempotent and never fails the build (unresolvable specifiers → warnings). Running bare `tsc` without it yields output Node can't import.
- Build uses `tsconfig.build.json` (extends `tsconfig.json`, excludes tests) — don't point the build at `tsconfig.json`.
- tsconfig is very strict (`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noUnusedLocals/Parameters`); typecheck accordingly.
