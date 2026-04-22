Build new listing and detail pages for a Directus-backed content type.
Requirements:
- Follow the exact patterns established in `app/[locale]/eventi/page.tsx`, `lib/directus/fetchers/server/events-fetcher.ts`, and `types/events/index.ts`.
- Do NOT use `any` or `as never`.
- Create a dedicated `types/<domain>/index.ts` for mapped output types.
- Create a dedicated `lib/utils/<domain>.ts` for mapping functions (e.g. `mapItem`, `mapItemDetail`) and filter builders (e.g. `buildFilter`).
- Create a dedicated `lib/directus/fetchers/server/<domain>-fetcher.ts` for Directus read helpers.
- Implement filtering by `testata_id = DIRECTUS_TESTATA_ID` (env var from `.env.local`).
- Implement slug-based detail lookup using `translations.slug`.
- Use the shared `Pagination`, `PreviewCard`, and `dynamicTranslations` utilities.
- Implement `loading.tsx` with the existing skeleton pattern.
- Use `paginationSearchParamsCache` from `lib/nuqs/pagination-search-params.ts` for pagination search params.
- Use `getLanguage()` from `lib/i18n/translations.ts` for locale resolution.