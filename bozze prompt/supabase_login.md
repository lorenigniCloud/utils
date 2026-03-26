[DOCUMENTO ARCHITETTURALE] Login via Invito, RBAC (Custom Claims) e Proxy Routing in Next.js con next-intl

=== CONTESTO ARCHITETTURALE ===

Progetto: Next.js 16 App Router, TypeScript strict, Tailwind CSS v4
Routing: next-intl con localePrefix "as-needed" (it=nessun prefisso, en=/en/)
Auth: Supabase Auth con Custom JWT Hook
DB Locale: Supabase CLI (npx supabase)
Dipendenze chiave: @supabase/ssr, @supabase/supabase-js, jose (JWT decode), zod

=== CONTESTO DB (Supabase) ===

ENUM:
app_role: 'admin' | 'editor' | 'viewer'

TABELLE:
public.users:
id UUID PK, auth_id UUID FK→auth.users (ON DELETE CASCADE),
email TEXT UNIQUE, role app_role DEFAULT 'viewer',
first_login_at TIMESTAMPTZ, created_at TIMESTAMPTZ
INDEX: users_auth_id_idx ON auth_id

public.tournament_registrations:
id UUID PK, name TEXT, email TEXT, created_at TIMESTAMPTZ
(form pubblico, solo INSERT anon)

FUNZIONI SQL:
custom_access_token_hook(event jsonb): - Lookup ruolo da public.users via auth_id = JWT user_id - Inietta in claims.app_metadata.role (fallback: 'viewer') - Eseguito ad ogni emissione/refresh del JWT - Usa operatore || (merge JSONB) — jsonb_set() non funziona con path
annidati in PostgreSQL 17 - SECURITY DEFINER + SET row_security = off (necessario per bypassare
RLS quando eseguito da supabase_auth_admin) - GRANT EXECUTE a supabase_auth_admin, service_role

sync_auth_id(): - Trigger AFTER INSERT OR UPDATE ON auth.users - Collega auth_id in public.users (match via email, solo WHERE auth_id IS NULL) - Imposta first_login_at al primo collegamento

RLS POLICIES:

- anon: INSERT su tournament_registrations (WITH CHECK true)
- authenticated SELECT su tournament_registrations (WHERE app_metadata.role IN admin,editor)
- authenticated SELECT su users (WHERE auth_id = auth.uid())
- authenticated ALL su users (WHERE app_metadata.role = admin)

CONFIG HOOK:
supabase/config.toml → [auth.hook.custom_access_token]
enabled = true
uri = "pg-functions://postgres/public/custom_access_token_hook"

SEED:
admin@locale.test (admin), editor@locale.test (editor), viewer@locale.test (viewer)

=== NOTA CRITICA: getUser() vs JWT Claims ===

Supabase.auth.getUser() chiama /auth/v1/user che ritorna raw_app_meta_data
dal DATABASE, non dal JWT. L'hook modifica i claims del JWT ma NON il campo
raw_app_meta_data in auth.users. Quindi getUser().app_metadata.role è sempre
undefined → default "viewer".

Soluzione: usare getSession() + decodificare il JWT con jose (decodeJwt)
per leggere app_metadata.role dai claims del token.

Punti che usano getRoleFromToken(): - proxy.ts (RBAC enforcement) - lib/auth/dal.ts (verifySession) - components/auth/forbidden-access-fallback.tsx (refresh retry)

=== ARCHITETTURA NEXT.JS ===

FILE MAP (auth/RBAC):
proxy.ts → Middleware: i18n + RBAC enforcement
lib/auth/roles.ts → AppRole type, roleRank, hasRequiredRole(), getRequiredRoleForPath()
lib/auth/jwt.ts → getRoleFromToken() [decode JWT via jose]
lib/auth/dal.ts → verifySession() [cache + server-only], requireRole()
lib/i18n/path.ts → splitLocaleFromPath(), toLocalizedPath()
lib/i18n/routing.ts → defineRouting (locales: [it, en], defaultLocale: it)
lib/supabase/server-client.ts → getServerSupabaseClient() [read-only], getWritableServerSupabaseClient() [write cookies]
lib/supabase/browser-client.ts → getBrowserSupabaseClient() [singleton, autoRefreshToken]
lib/validation/set-password.schema.ts → Zod schema per password (8+ char, lettera+numero+speciale)
lib/validation/login.schema.ts → Zod schema per login (email+password) e forgot password (email)
app/actions/auth.ts → setPassword() [revalidatePath + redirect], login() [signInWithPassword], forgotPassword() [resetPasswordForEmail], logout() [signOut + redirect]
components/auth/login-form.tsx → Form client-side login con useActionState + link "Password dimenticata?"
components/auth/forgot-password-form.tsx → Form client-side forgot password + stato successo
components/auth/auth-callback-client.tsx → Token exchange + redirect (3 flussi)
components/auth/forbidden-access-fallback.tsx → refreshSession() + decode JWT retry + UX states
components/auth/set-password-form.tsx → Form con useActionState
app/[locale]/auth/login/page.tsx → Pagina login (redirect /dashboard se autenticato)
app/[locale]/auth/forgot-password/page.tsx → Pagina forgot password
app/[locale]/auth/callback/page.tsx → Server wrapper → AuthCallbackClient
app/[locale]/auth/set-password/page.tsx → Pagina set-password con form
app/[locale]/auth/forbidden/page.tsx → Pagina 403 + ForbiddenAccessFallback
app/[locale]/auth/error.tsx → Error boundary client component
app/[locale]/dashboard/page.tsx → Dashboard utente (verifySession + role-based UI)

=== FLUSSI AUTH ===

A. FLUSSO LOGIN (utente già registrato)

1.  Utente naviga a /auth/login
    - Proxy: route in authBypassRoutes → solo handleI18n (skip RBAC)

2.  LOGIN PAGE (login/page.tsx) — Server Component
    - getServerSupabaseClient() → getSession()
    - Se sessione presente → redirect /dashboard
    - Altrimenti → renderizza LoginForm
    - Usa getTranslations("LoginPage") per i18n

3.  LOGIN FORM (login-form.tsx) — Client Component
    - useActionState(login, undefined)
    - Campi: email (type=email, required, autocomplete), password (type=password, required, autocomplete)
    - Link "Password dimenticata?" → /auth/forgot-password
    - Stati UI: errore email[], errore password[], messaggio errore

4.  LOGIN SERVER ACTION (app/actions/auth.ts)
    - Validazione Zod: email obbligatoria/valida, password obbligatoria
    - getWritableServerSupabaseClient()
    - signInWithPassword({ email, password })
    - On success: revalidatePath('/', 'layout') → redirect /dashboard
    - On error "Invalid login credentials" → "Email o password non validi."
    - Altri errori → mostra messaggio errore Supabase

5.  DASHBOARD (/dashboard)
    - verifySession() → getSession() + getRoleFromToken()
    - Mostra profilo utente (ID, ruolo badge)
    - Sezioni role-based: admin → "Amministrazione", admin|editor → "Contenuti"
    - Bottone "Esci" → Server Action logout() → signOut() + redirect /

B. FLUSSO FORGOT PASSWORD (password dimenticata)

1.  Da /auth/login → link "Password dimenticata?" → /auth/forgot-password
    - Proxy: route in authBypassRoutes → solo handleI18n

2.  FORGOT PASSWORD PAGE (forgot-password/page.tsx) — Server Component
    - Renderizza ForgotPasswordForm
    - Usa getTranslations("ForgotPasswordPage") per i18n

3.  FORGOT PASSWORD FORM (forgot-password-form.tsx) — Client Component
    - useActionState(forgotPassword, undefined)
    - Campo: email (type=email, required, autocomplete)
    - Stati UI: errore email[], messaggio errore, successo
    - On success: mostra messaggio "Controlla la tua email..." + link "Torna al login"
    - Link "Torna al login" → /auth/login

4.  FORGOT PASSWORD SERVER ACTION (app/actions/auth.ts)
    - Validazione Zod: email obbligatoria/valida
    - getWritableServerSupabaseClient()
    - resetPasswordForEmail(email, {
      redirectTo: '${NEXT_PUBLIC_SITE_URL}/auth/callback?next=/auth/set-password'
      })
    - On success: { success: true, message: "Controlla la tua email..." }
    - On error: { message: errore Supabase }

5.  SUPABASE INVIA EMAIL RECOVERY
    - Link con formato: /auth/callback#access_token=xxx&refresh_token=yyy&type=recovery
    - Oppure: /auth/callback?code=xxx&type=recovery

6.  AUTH CALLBACK CLIENT (auth-callback-client.tsx)
    - Legge hash tokens o ?code= dall'URL
    - exchangeCodeForSession(code) o setSession(tokens)
    - Default nextPath: /auth/set-password
    - Validazione nextPath: isValidInternalPath() previene open redirect
    - On success: window.location.replace(toLocalizedPath(locale, nextPath))

7.  CUSTOM ACCESS TOKEN HOOK (scatta durante exchange)
    - L'hook legge il ruolo da public.users
    - Cuce app_metadata.role nel JWT

8.  SET-PASSWORD PAGE (set-password/page.tsx)
    - Form client (SetPasswordForm) con useActionState
    - Schema Zod: min 8 char, lettera, numero, carattere speciale, conferma match
    - Server Action setPassword(): updateUser({ password })
    - On success: revalidatePath('/', 'layout') → redirect /dashboard

C. FLUSSO INVITO (primo accesso tramite admin)

1.  INVITO MANUALE (Dashboard Supabase Studio)
    - Sezione Auth → "Invite user" → inserisci email (es. viewer@locale.test)

2.  TRIGGER sync_auth_id (scatta subito)
    - Supabase crea riga in auth.users
    - Il trigger cerca email corrispondente in public.users
    - Collega auth_id e registra first_login_at

3.  EMAIL IN INBUCKET (porta 54324 locale)
    - Apri Inbucket, leggi email invito, clicca link di conferma

4.  VALIDAZIONE E REDIRECT A CALLBACK
    - Supabase verifica validità invito
    - Redirect a: http://127.0.0.1:3000/auth/callback?code=xxxxxx

5.  PROXY (proxy.ts) — BYPASS AUTH ROUTES
    - cleanPathname in authBypassRoutes → solo handleI18n (skip RBAC)
    - Gestisce correttamente sia /auth/callback che /en/auth/callback

6.  AUTH CALLBACK CLIENT (auth-callback-client.tsx)
    - Legge ?code= o hash tokens (#access_token=...&refresh_token=...)
    - exchangeCodeForSession(code) o setSession(tokens)
    - Validazione nextPath: isValidInternalPath() previene open redirect
    - Stati: loading → redirecting | error
    - On error: mostra messaggio + link home
    - On success: window.location.replace(toLocalizedPath(locale, nextPath))
    - Default nextPath: /auth/set-password

7.  CUSTOM ACCESS TOKEN HOOK (scatta ora)
    - Mentre Supabase fabbrica il JWT, l'hook legge il ruolo da public.users
    - Cuce app_metadata.role nel JWT (non nel DB) prima di consegnarlo
    - Scatterà di nuovo ad ogni refresh del token

8.  SET-PASSWORD PAGE (set-password/page.tsx)
    - Form client (SetPasswordForm) con useActionState
    - Schema Zod: min 8 char, lettera, numero, carattere speciale, conferma match
    - Server Action setPassword(): supabase.auth.updateUser({ password })
    - On success: revalidatePath('/', 'layout') → redirect /dashboard

9.  DASHBOARD (/dashboard)
    - Server Component chiama verifySession() → getSession() + getRoleFromToken()
    - Mostra profilo utente (ID, ruolo badge)
    - Sezioni role-based: admin → "Amministrazione", admin|editor → "Contenuti"
    - Bottone "Esci" → Server Action logout() → signOut + redirect /

D. RBAC ENFORCEMENT (successive navigazioni)

cleanPathname non è in authBypassRoutes
→ handleI18n (gestisce locale redirect)
→ getRequiredRoleForPath(cleanPathname):
null (route pubblica) → pass through
ruolo richiesto → continue
→ supabase.auth.getSession():
session null → redirect /auth/login?next=cleanPathname
session presente → getRoleFromToken(session.access_token)
→ hasRequiredRole(currentRole, requiredRole):
true → pass through
false → redirect /auth/forbidden?next=cleanPathname

=== ROTTE PROTETTE (con e senza prefisso /en) ===

/admin/_ o /en/admin/_ → richiede admin (rank 3)
/editor/_ o /en/editor/_ → richiede editor (rank 2, admin ha accesso)
/viewer/_ o /en/viewer/_ → richiede viewer (rank 1, tutti i ruoli)

Route non mappate (es. /dashboard, /profile) → pubbliche (pass through,
autenticazione gestita dal DAL nella page stessa)

=== AUTH BYPASS ROUTES (proxy.ts) ===

/auth/callback → solo i18n
/auth/set-password → solo i18n
/auth/forbidden → solo i18n (evita loop redirect)
/auth/login → solo i18n (evita loop redirect)
/auth/forgot-password → solo i18n (utente non autenticato)

=== PAGINA 403 (Forbidden) — GRACEFUL FALLBACK ===

1. Proxy redirect a /auth/forbidden?next=/admin
2. ForbiddenAccessFallback client component:
   - useRef previene re-execution dell'useEffect
   - refreshSession() → nuovo JWT emesso con hook
   - getRoleFromToken(session.access_token) per leggere il ruolo
   - Verifica: hasRequiredRole(userRole, requiredRole) post-refresh?
     true → redirect alla pagina originale
     false + sessione null → redirect /auth/login
     false + sessione presente → mostra "Non hai i permessi" + link home
   - UI states: loading ("Verifica in corso...") → redirecting | denied

=== DATA ACCESS LAYER (lib/auth/dal.ts) ===

verifySession() [React cache memoizzato]: - getServerSupabaseClient() → supabase.auth.getSession() - session null → redirect /auth/login - getRoleFromToken(session.access_token) → legge ruolo dal JWT - Ritorna { isAuth: true, userId, role }

requireRole(requiredRole): - Chiama verifySession() - hasRequiredRole() fallisce → forbidden() di Next.js - Ritorna SessionData se autorizzato

Usage in Server Components:
const session = await verifySession();
if (session.role === 'admin') { ... }

Usage in Server Actions:
const session = await requireRole('admin');
// operazione protetta

=== JWT DECODE (lib/auth/jwt.ts) ===

getRoleFromToken(token: string) → AppRole | null - Usa jose.decodeJwt() per decodificare il JWT senza verificarne la firma - Legge payload.app_metadata.role - Ritorna null se il token è malformato o non ha il ruolo - Usato in: proxy.ts, dal.ts, forbidden-access-fallback.tsx

Perché non getUser().app_metadata?
getUser() chiama /auth/v1/user → ritorna raw_app_meta_data dal DB
L'hook modifica solo i JWT claims, non il DB
Quindi getUser().app_metadata non contiene il ruolo

=== SUPABASE CLIENTS ===

Browser (browser-client.ts): - createBrowserClient singleton - autoRefreshToken, detectSessionInUrl, persistSession - Usato da: auth-callback-client, forbidden-access-fallback, set-password-form

Server read-only (server-client.ts → getServerSupabaseClient): - setAll: () => {} (no write cookies) - Usato da: Server Components, dal.ts

Server writable (server-client.ts → getWritableServerSupabaseClient): - setAll scrive cookies su cookieStore - Usato da: Server Actions che necessitano refresh (setPassword)

Proxy (proxy.ts → getSupabaseClient): - Scrive cookies su NextResponse - Usato solo nel proxy

=== ERROR BOUNDARIES ===

app/[locale]/auth/error.tsx: - Cattura errori runtime nei route /auth/\* - Mostra "Si è verificato un errore" + bottone Riprova + link home - Log su console.error per debug

app/[locale]/auth/forbidden/page.tsx: - ForbiddenAccessFallback client component - refreshSession() + decode JWT retry per verificare se il ruolo è aggiornato - Se post-refresh il ruolo è sufficiente → redirect alla pagina originale - Altrimenti mostra "Non hai i permessi" + link home

=== TEST PLAN LOCALE ===

Pre-requisiti:

1. npx supabase start
2. pnpm dev (porta 3000)

Test A: Viewer accede a /admin → bloccato

1. Invite viewer@locale.test da Supabase Studio
2. Apri Inbucket (porta 54324), clicca link invito
3. Completa set-password
4. Vai a /admin → proxy redirect a /auth/forbidden?next=/admin
5. ForbiddenAccessFallback: "Non hai i permessi necessari" + link home

Test B: Viewer accede a /en/admin → bloccato (stesso risultato, prefisso en)

Test C: Admin accede a /admin → permesso

1. Login come admin@locale.test
2. Vai a /admin → accesso consentito

Test D: Non autenticato accede a /viewer/\* → redirect a /auth/login

1. Logout (cancella cookie)
2. Vai a /viewer → redirect a /auth/login?next=/viewer

Test E: Token stale (ruolo aggiornato dopo refresh)

1. Login come viewer, naviga a /admin → bloccato su /auth/forbidden
2. Cambia ruolo a 'admin' in public.users via SQL
3. ForbiddenAccessFallback fa refreshSession() → nuovo JWT con admin
4. Redirect automatico a /admin

Test F: Dashboard mostra ruolo corretto

1. Login come editor@locale.test
2. Vai a /dashboard → badge mostra "editor", sezione "Contenuti" visibile
3. Login come admin@locale.test → badge "admin", sezioni "Amministrazione" + "Contenuti"
4. Login come viewer@locale.test → badge "viewer", nessuna sezione extra

Test G: Login utente esistente

1. Vai a /auth/login
2. Inserisci email/password di viewer@locale.test
3. Click "Accedi" → redirect a /dashboard
4. Dashboard mostra badge "viewer"

Test H: Login con credenziali errate

1. Vai a /auth/login
2. Inserisci email esistente + password sbagliata
3. Click "Accedi" → mostra errore "Email o password non validi."

Test I: Forgot Password

1. Vai a /auth/login
2. Click "Password dimenticata?" → /auth/forgot-password
3. Inserisci email viewer@locale.test
4. Click "Invia link" → mostra "Controlla la tua email..."
5. Apri Inbucket (porta 54324), leggi email recovery
6. Clicca link → /auth/callback?code=xxxxx
7. AuthCallbackClient → redirect a /auth/set-password
8. Inserisci nuova password → redirect a /dashboard

Test J: Login utente già autenticato

1. Se già loggato, vai a /auth/login
2. LoginPage → redirect automatico a /dashboard

=== STRUMENTI DEBUG SUPABASE ===

- Schema: mcp_supabase_list_tables
- Migration: mcp_supabase_list_migrations
- Extension: mcp_supabase_list_extensions
- Tipi TS: mcp_supabase_generate_typescript_types
- Sicurezza: mcp_supabase_get_advisors
- Log: mcp_supabase_get_logs

=== PITFALLS NOTI ===

1. PostgreSQL 17: jsonb_set() non funziona con path annidati (es. '{app_metadata,role}')
   → Usare operatore || con jsonb_build_object() per merge

2. Hook RLS: la funzione hook ha SECURITY DEFINER ma se manca SET row_security = off,
   le RLS policies di public.users bloccano la SELECT quando eseguita da supabase_auth_admin
   (ruolo non presente nelle policies)

3. getUser() vs JWT: getUser().app_metadata NON contiene i claims custom dall'hook.
   Serve getSession() + decodeJwt() per leggere il ruolo dal JWT.
