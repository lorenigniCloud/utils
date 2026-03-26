# 🚀 Setup del Progetto basket-arcidosso

---

## 📦 Fase 1: Setup del Progetto Locale

### Azione 1.1: Inizializzazione con pnpm

```bash
pnpm create next-app .
```

> **Perché:** L'uso del punto (`.`) evita che si crei una fastidiosa "matrioska" di cartelle quando hai già clonato il repository.

### 🐛 Intoppo Risolto: Eliminazione del file `pnpm-workspace.yaml`

- **Causa:** pnpm vedeva quel file vuoto e andava in blocco credendo di dover gestire un monorepo complesso.
- **Soluzione:** Eliminazione del file.

---

### Azione 1.2: Configurazione Output Standalone

Aggiunta di `output: "standalone"` nel file `next.config.ts`.

> **Perché:** Istruisce Next.js a estrapolare solo i file strettamente necessari per la produzione.  
> **Causa:** Senza questo, l'immagine Docker si porterebbe dietro GB di dipendenze inutili (devDependencies), rallentando i caricamenti e sprecando risorse cloud.

---

## 🐳 Fase 2: Dockerizzazione

### Azione: Creazione del Dockerfile Multi-Stage e .dockerignore

> **Perché:** Il "Multi-stage" divide il processo in fasi. Usa una fase per installare tutto e compilare, ma poi copia solo il risultato finito (il server standalone) nell'immagine finale.

| Senza Multi-stage | Con Multi-stage |
|-------------------|-----------------|
| ~1.5 GB           | ~150 MB         |

---

## ☁️ Fase 3: Infrastruttura Azure

### Azione: Creazione delle risorse

- **Resource Group** — Il contenitore logico
- **App Service Plan** — Il motore/server fisico
- **Azure Container Registry (ACR)** — Il magazzino delle immagini

### 🐛 Intoppo Risolto: Errore `MissingSubscriptionRegistration`

- **Causa:** Azure tiene disattivati (non registrati) i servizi che non hai mai usato per evitare addebiti o usi accidentali.
- **Soluzione:** Registrazione manuale del provider tramite CLI.

---

### 🔐 Fase 3.5: Configurazione dei GitHub Secrets

Per far sì che la pipeline (GitHub Actions) abbia i permessi di scrivere sul Registro e di aggiornare l'App Service, è fondamentale salvare questi 3 Segreti nelle impostazioni del repository GitHub (**Settings → Secrets and variables → Actions**):

| Secret | Contenuto | A cosa serve |
|--------|-----------|--------------|
| `ACR_USERNAME` | Il nome utente dell'ACR (es. `crbasketarcidossodev`) | Identifica chi sta cercando di accedere al registro per caricare l'immagine Docker |
| `ACR_PASSWORD` | La password lunga alfanumerica generata dall'ACR | È la chiave che autorizza GitHub a fare il `docker push` dell'immagine nel registro privato su Azure |
| `AZURE_WEBAPP_PUBLISH_PROFILE` | Blocco XML scaricato tramite `az webapp deployment list-publishing-profiles` | È il "telecomando" dell'App Service. Permette a GitHub di collegarsi in modo sicuro al server di Azure e lanciare il comando di ricaricare l'applicazione |

---

### �boss 1: Errore "Failed to get app runtime OS"

**Stato:** Deploy fallito lato GitHub Actions

- **Causa:** Di recente, Azure ha disattivato per sicurezza l'autenticazione di base (Basic Auth) per la pubblicazione. Il "Publish Profile" di GitHub, che usa quel metodo, trovava la porta sbarrata.
- **Soluzione:** 
  1. Riattivata la policy scm (Basic Auth) tramite CLI
  2. Scaricato un nuovo Publish Profile fresco
  3. Aggiornato il Secret su GitHub

---

### �boss 2: La pagina web mostra "Application Error :("

**Stato:** L'app era pubblicata, ma il sito non caricava.

- **Causa:** Incomprensione sulle porte. Di default, Azure App Service bussa alla porta 80. Ma il nostro container Next.js era in ascolto sulla porta 3000. Azure credeva che il container fosse morto.
- **Soluzione:** Aggiunta la variabile d'ambiente `WEBSITES_PORT=3000` su Azure.

---

### �boss 3: Container in crash ("Image pull failed with forbidden")

**Stato:** Il log stream si chiudeva istantaneamente.

- **Causa:** La pipeline di GitHub aveva le credenziali per caricare (Push) l'immagine nel Registro (ACR). Tuttavia, la Web App di Azure non aveva le credenziali per scaricare (Pull) quell'immagine dal Registro per farla girare.
- **Soluzione:** Recuperate le credenziali (username e password) dell'ACR e fornite esplicitamente alla Web App tramite:
  ```bash
  az webapp config container set
  ```

---

## 🔄 Flusso Completo: App Service for Containers

```
┌─────────────────────────────────────────────────────────────────┐
│                     GitHub Actions                              │
│  1. Legge il Dockerfile                                         │
│  2. Crea immagine Docker (Alpine Linux + Node 20 + App)         │
│  3. Push immagine → Azure Container Registry (ACR)              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Azure Container Registry                    │
│                    (Magazzino immagini)                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Azure App Service                           │
│  • Configurato per Container                                    │
│  • Pull immagine da ACR                                         │
│  • Esegue il container                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Riepilogo: Come Creare l'Infrastruttura su Azure

1. **Crea il Resource Group**
2. **Crea il Container Registry (ACR)**
3. **Crea l'App Service Plan**
4. **Crea la Web App for Containers**


## ☁️ Fase 4: Passaggio da Supabase Locale a Remoto

### Azione 4.1: Recupero chiavi remote e setup GitHub Secrets

Per far comunicare l'app su Azure con il Supabase in cloud, è necessario recuperare Project URL e anon/public key dalle impostazioni API di Supabase e salvarle nei Segreti di GitHub (Settings → Secrets and variables → Actions).

| Secret | Contenuto | A cosa serve |
|--------|-----------|--------------|
| `NEXT_PUBLIC_SUPABASE_URL` | L'URL del progetto Supabase cloud | Endpoint remoto per le chiamate DB/Auth |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | La chiave pubblica di Supabase | Autorizzazione per il client frontend |

---

### Azione 4.2: Modifica Dockerfile per le variabili d'ambiente (Build-time)

Next.js "inforna" (hardcode) le variabili che iniziano con `NEXT_PUBLIC_` nei file statici durante la compilazione. È fondamentale passare queste chiavi allo stage di build del Dockerfile.

**Modifica allo Stage 2 (Builder):**

```dockerfile
# ... [Stage 1: Base] ...

FROM node:20-alpine AS builder
WORKDIR /app
RUN corepack enable pnpm
COPY --from=base /app/node_modules ./node_modules
COPY . .

# 1. Dichiariamo che Docker riceverà questi argomenti
ARG NEXT_PUBLIC_SUPABASE_URL
ARG NEXT_PUBLIC_SUPABASE_ANON_KEY

# 2. Li convertiamo in variabili d'ambiente per il processo di build
ENV NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL
ENV NEXT_PUBLIC_SUPABASE_ANON_KEY=$NEXT_PUBLIC_SUPABASE_ANON_KEY

# 3. Lanciamo la build che ora può leggere le variabili
RUN pnpm build

# ... [Stage 3: Runner] ...
```

---

### Azione 4.3: Aggiornamento Pipeline GitHub Actions (deploy.yml)

Istruiamo la pipeline a pescare i Segreti appena salvati e a passarli al comando Docker tramite i flag `--build-arg`.

```yaml
      - name: Build e Push dell'immagine Docker
        run: |
          docker build \
            --build-arg NEXT_PUBLIC_SUPABASE_URL=${{ secrets.NEXT_PUBLIC_SUPABASE_URL }} \
            --build-arg NEXT_PUBLIC_SUPABASE_ANON_KEY=${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }} \
            -t ${{ env.REGISTRY_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          
          docker push ${{ env.REGISTRY_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

### Azione 4.4: Configurazione Variabili su Azure (Run-time)

Il codice che gira lato server (API Routes, Server Actions) non legge le variabili infornate durante la build, ma cerca le variabili del sistema operativo in tempo reale.

Le stesse chiavi vanno quindi inserite nel Portale di Azure:
- **App Service → Environment variables**
- Aggiungere `NEXT_PUBLIC_SUPABASE_URL` e `NEXT_PUBLIC_SUPABASE_ANON_KEY` (senza virgolette).
- Riavviare l'App Service dall'**Overview** per applicare le modifiche al container.

---

### 🐛 Intoppo Risolto: Autenticazione rotta e reindirizzamenti errati

**Stato:** Dopo aver inviato un'email di invito/reset password, il clic sul link portava a una pagina JSON di errore con il messaggio: `{"message":"No API key found in request", "hint":"No 'apikey' request header or url param was found."}`.

- **Causa:** Errore nella configurazione del Site URL sulla dashboard di Supabase. Se l'URL dell'applicazione viene inserito senza il prefisso esplicito del protocollo (es. `app-basket...` invece di `https://app-basket...`), Supabase concatena in modo errato l'URL del backend con quello del frontend. Questo fa sì che il link nell'email punti direttamente alle API nude e crude di Supabase, saltando completamente l'applicazione su Azure e perdendo gli header di autenticazione.

- **Soluzione:** 
  1. Andare su Supabase → Authentication → URL Configuration.
  2. Assicurarsi che il Site URL sia esattamente l'URL pubblico di Azure, comprensivo di `https://` (es. `https://app-basket-arcidosso-fe-dev.azurewebsites.net`).
  3. Aggiungere lo stesso URL seguito da `/**` nei Redirect URLs.