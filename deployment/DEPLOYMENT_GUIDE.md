# VoiceRx Sync — Production Deployment Guide

This guide walks you through deploying **VoiceRx Sync** as a brand-new project from scratch using **Vercel** (Frontend) and **Render** (Backend), connected to your own **MongoDB Atlas**, **Groq AI**, and **Firebase Authentication**.

---

## Architecture Overview

```
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│        Vercel (Frontend)        │       │         Render (Backend)        │
│       Next.js 16 + React 19     │──────▶│         FastAPI + Python        │
│  https://your-app.vercel.app    │ CORS  │ https://your-backend.onrender.com
└────────────────┬────────────────┘       └────────┬───────────────┬────────┘
                 │                                 │               │
                 ▼                                 ▼               ▼
      ┌─────────────────────┐            ┌──────────────────┐ ┌───────────────┐
      │   Firebase Auth     │            │  MongoDB Atlas   │ │   Groq AI     │
      │  (Google Sign-In)   │            │   (Database M0)  │ │ (Whisper+LLaMA│
      └─────────────────────┘            └──────────────────┘ └───────────────┘
```

> **Why Render for Backend?**
> Next.js is hosted on **Vercel**. However, doctor voice consultations produce audio files that often exceed Vercel's serverless 4.5MB request limit. Deploying the FastAPI backend on **Render** (free web service) allows unlimited audio recording uploads, ReportLab PDF rendering, and long-running Whisper transcription without serverless timeout limits.

---

## Table of Contents
1. [Step 1: Create MongoDB Atlas Database](#step-1-create-mongodb-atlas-database)
2. [Step 2: Get Groq Cloud API Key](#step-2-get-groq-cloud-api-key)
3. [Step 3: Create Firebase Project & Google Auth](#step-3-create-firebase-project--google-auth)
4. [Step 4: Push Code to GitHub](#step-4-push-code-to-github)
5. [Step 5: Deploy FastAPI Backend on Render](#step-5-deploy-fastapi-backend-on-render)
6. [Step 6: Deploy Next.js Frontend on Vercel](#step-6-deploy-nextjs-frontend-on-vercel)
7. [Step 7: Authorize Vercel Domain in Firebase](#step-7-authorize-vercel-domain-in-firebase)
8. [Step 8: Verification & Testing](#step-8-verification--testing)
9. [Troubleshooting & Common Pitfalls](#troubleshooting--common-pitfalls)

---

## Step 1: Create MongoDB Atlas Database

1. Sign up or log in at **[mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)**.
2. Click **Create Deployment**:
   * Select **M0 (Free)** tier.
   * Cloud Provider: **AWS**, Region: choose whichever is closest to you (e.g. `ap-south-1` Mumbai or `eu-central-1` Frankfurt).
   * Cluster Name: `voicerx-cluster` (or default `Cluster0`).
   * Click **Create Deployment**.
3. **Create Database User**:
   * Username: e.g. `voicerx_admin`
   * Password: Click **Autogenerate Secure Password** or enter your own (e.g., `SecurePass2026`).
   * **Copy and save both the username and password**.
4. **Configure Network Access (Whitelist IP)**:
   * In the left sidebar, click **Security → Network Access**.
   * Click **Add IP Address**.
   * Click **Allow Access From Anywhere** (`0.0.0.0/0`).
   * Click **Confirm**. *(This is required so Render and Vercel cloud instances can connect to the database)*.
5. **Get Your Connection String (`MONGO_URI`)**:
   * In the left sidebar, click **Deployment → Database**.
   * Click **Connect** next to your cluster.
   * Under "Connect to your application", select **Drivers**.
   * Driver: `Python`, Version: `3.12 or later`.
   * Copy the connection string. It looks like:
     ```
     mongodb+srv://<db_username>:<db_password>@cluster0.abcde.mongodb.net/?retryWrites=true&w=majority&appName=Cluster0
     ```
   * Replace `<db_username>` and `<db_password>` with your actual credentials.
   * Set database name as `/voicerx` right before the `?`:
     ```
     mongodb+srv://voicerx_admin:SecurePass2026@cluster0.abcde.mongodb.net/voicerx?retryWrites=true&w=majority
     ```
   * **Save this as your `MONGO_URI`**.

---

## Step 2: Get Groq Cloud API Key

1. Sign up or log in at **[console.groq.com](https://console.groq.com)**.
2. In the left menu, click **API Keys**.
3. Click **Create API Key**.
4. Name: `voicerx-prod` → Click **Submit**.
5. Copy the key (starts with `gsk_...`).
6. **Save this as your `GROQ_API_KEY`**.

---

## Step 3: Create Firebase Project & Google Auth

1. Go to **[console.firebase.google.com](https://console.firebase.google.com)** and click **Add project**.
2. Project name: e.g. `voicerx-sync-app` → Click **Continue**.
3. Disable Google Analytics (optional, keeps setup quick) → Click **Create project**.
4. **Enable Google Authentication**:
   * In left sidebar, click **Build → Authentication**.
   * Click **Get started**.
   * Under the **Sign-in method** tab, click **Google**.
   * Toggle **Enable**.
   * Select your support email.
   * Click **Save**.
5. **Register Web App to get Firebase Config**:
   * Click the **Gear icon ⚙️ (Project settings)** at top left.
   * Under the **General** tab, scroll down to **Your apps**.
   * Click the **Web icon (`</>`)**.
   * App nickname: `VoiceRx Web` (leave Firebase Hosting unchecked).
   * Click **Register app**.
   * Firebase will show a configuration object like this:
     ```javascript
     const firebaseConfig = {
       apiKey: "AIzaSyDXXXXXX...",
       authDomain: "voicerx-sync-app.firebaseapp.com",
       projectId: "voicerx-sync-app",
       storageBucket: "voicerx-sync-app.firebasestorage.app",
       messagingSenderId: "987654321098",
       appId: "1:987654321098:web:abcdef123456"
     };
     ```
   * Note down all these values:
     * `NEXT_PUBLIC_FIREBASE_API_KEY` = `apiKey`
     * `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` = `authDomain`
     * `NEXT_PUBLIC_FIREBASE_PROJECT_ID` = `projectId`
     * `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` = `storageBucket`
     * `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` = `messagingSenderId`
     * `NEXT_PUBLIC_FIREBASE_APP_ID` = `appId`
     * `FIREBASE_API_KEY` (for Backend) = the same `apiKey` value.

---

## Step 4: Push Code to GitHub

Make sure all your changes are committed and pushed to your GitHub repository:

```bash
git add .
git commit -m "Configure production environment and CORS for Vercel"
git push origin main
```

---

## Step 5: Deploy FastAPI Backend on Render

1. Sign up or log in at **[render.com](https://render.com)** using your GitHub account.
2. In the Render Dashboard, click **New +** → **Web Service**.
3. Choose **Build and deploy from a Git repository** → Select your `VoiceRx-Sync` repository.
4. Fill in the service settings:
   * **Name**: `voicerx-backend`
   * **Region**: Choose the region closest to you (e.g. Singapore, Frankfurt, Oregon).
   * **Branch**: `main`
   * **Root Directory**: `backend` *(⚠️ CRITICAL: Must be `backend`, not empty!)*
   * **Runtime**: `Python 3`
   * **Build Command**: `pip install -r requirements.txt`
   * **Start Command**: `uvicorn main:app --host 0.0.0.0 --port $PORT`
   * **Instance Type**: `Free`
5. Scroll down to **Environment Variables** and add the following 5 variables:

   | Key | Value | Notes |
   |---|---|---|
   | `GROQ_API_KEY` | `gsk_...` | From Step 2 |
   | `MONGO_URI` | `mongodb+srv://...` | From Step 1 |
   | `MONGO_DB_NAME` | `voicerx` | Database name |
   | `FIREBASE_API_KEY` | `AIzaSy...` | Web API key from Step 3 |
   | `JWT_SECRET` | `1d69bd19f6fa0e9939796ff31226ec859f59ddfa5141a996157ecebd80091924` | Or click "Generate" |
   | `PATIENT_ENCRYPT_KEY` | `ZNFEY45BHSTdWDsXGHg2_Qr5Is4sQ-FODqULcJfkBiM=` | Pre-generated Fernet key |

6. Click **Deploy Web Service**.
7. Wait 2-3 minutes for the build to finish. Once it says **Live**, copy your backend service URL:
   * Example: `https://voicerx-backend.onrender.com`
8. **Verify Backend Health**:
   Open a new browser tab and navigate to:
   `https://voicerx-backend.onrender.com/api/health`
   You should see:
   ```json
   {"status":"ok","service":"VoiceRx Sync API"}
   ```

---

## Step 6: Deploy Next.js Frontend on Vercel

1. Sign up or log in at **[vercel.com](https://vercel.com)** using your GitHub account.
2. In the Vercel Dashboard, click **Add New...** → **Project**.
3. Select your GitHub repository `VoiceRx-Sync` and click **Import**.
4. In **Project Configuration**:
   * **Project Name**: `voicerx-sync` (or your preferred name).
   * **Framework Preset**: `Next.js` (automatically detected).
   * **Root Directory**: Click **Edit** → select `frontend` → click **Continue**. *(⚠️ CRITICAL: Must be `frontend`!)*
5. Expand **Environment Variables** and enter the following keys:

   | Variable Name | Value |
   |---|---|
   | `NEXT_PUBLIC_API_URL` | `https://voicerx-backend.onrender.com` *(Your Render backend URL without trailing slash)* |
   | `NEXT_PUBLIC_FIREBASE_API_KEY` | `AIzaSy...` (from Firebase config) |
   | `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | `your-project.firebaseapp.com` |
   | `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | `your-project-id` |
   | `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | `your-project.firebasestorage.app` |
   | `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | `123456789...` |
   | `NEXT_PUBLIC_FIREBASE_APP_ID` | `1:123456789:web:...` |

6. Click **Deploy**.
7. Vercel will install dependencies, build with Turbopack, and deploy. In ~1-2 minutes, you will get your live frontend URL:
   * Example: `https://voicerx-sync.vercel.app`

---

## Step 7: Authorize Vercel Domain in Firebase

Before Google Sign-In will work on your live site, Firebase must authorize your Vercel domain:

1. Go back to the **[Firebase Console](https://console.firebase.google.com)**.
2. Select your project → Click **Authentication** in the left sidebar.
3. Click the **Settings** tab at the top.
4. Scroll to **Authorized domains**.
5. Click **Add domain**.
6. Enter your Vercel deployment domain (e.g. `voicerx-sync.vercel.app`) without `https://`.
7. Click **Add**.

---

## Step 8: Verification & Testing

1. Open your live Vercel URL in your browser: `https://voicerx-sync.vercel.app`.
2. Click **Sign in with Google** → Choose your Google account.
3. You should be redirected to the Doctor Dashboard (`/dashboard`).
4. Click **New Consultation** (`/consultation/new`).
5. Click **Start Recording** (or upload a sample audio file) → dictate:
   > *"Patient John Doe, 45-year-old male. Complaining of dry cough and mild fever for 3 days. Diagnosed with acute bronchitis. Prescribe Amoxicillin 500 mg three times daily for 5 days, and Paracetamol 650 mg as needed. Advise rest and warm fluids."*
6. Click **Stop Recording & Process**.
7. Confirm that Groq extracts the patient details, medicines, dosage, and safety validation warnings.
8. Click **Save & Sync to MongoDB**.
9. Click **Download PDF** to verify prescription generation.
10. Check **Consultation History** and **Analytics** to see the stored records.

---

## Troubleshooting & Common Pitfalls

| Issue | Cause | Solution |
|---|---|---|
| **CORS error in browser console** | Backend is blocking the frontend origin | The backend `main.py` is configured with `allow_origin_regex=r"https://.*\.vercel\.app"` which automatically allows any Vercel domain. Make sure your `NEXT_PUBLIC_API_URL` on Vercel points to your HTTPS Render URL without a trailing slash. |
| **`auth/unauthorized-domain` on Google login** | Vercel domain is not added to Firebase | Follow [Step 7](#step-7-authorize-vercel-domain-in-firebase) to add your `.vercel.app` domain to Firebase Authorized Domains. |
| **Render Web Service deployment fails** | Root directory not set | In Render settings, ensure **Root Directory** is explicitly set to `backend`. |
| **Vercel deployment says `package.json not found`** | Root directory not set | In Vercel settings, ensure **Root Directory** is explicitly set to `frontend`. |
| **First request to Render backend is slow (takes 30-50s)** | Free tier spin-down | Render free instances spin down after 15 minutes of inactivity and wake up upon receiving a new HTTP request. This is normal on free tier. |
| **MongoDB connection timeout (`pymongo.errors.ServerSelectionTimeoutError`)** | IP address not whitelisted | In MongoDB Atlas → Network Access, make sure `0.0.0.0/0` (Allow from anywhere) is active. Also verify user credentials in `MONGO_URI`. |
| **Audio upload fails on Vercel** | Request payload limit | Consultations are processed directly by your Render FastAPI backend (`https://your-backend.onrender.com/api/consultations/process-audio`) which has no serverless file size restrictions. |
