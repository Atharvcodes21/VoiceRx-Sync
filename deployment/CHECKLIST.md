# Deployment Quick Checklist

Print or keep this checklist open while setting up your project.

### 1. MongoDB Atlas
- [ ] Created M0 (Free) cluster on MongoDB Atlas
- [ ] Created database user and saved username & password
- [ ] Whitelisted IP `0.0.0.0/0` in Network Access
- [ ] Copied connection string (`MONGO_URI`) and appended `/voicerx`

### 2. Groq Cloud
- [ ] Created free account on `console.groq.com`
- [ ] Generated new API Key (`GROQ_API_KEY`)

### 3. Firebase Console
- [ ] Created project on `console.firebase.google.com`
- [ ] Enabled Authentication → Sign-in method → Google
- [ ] Registered Web App (`</>`) in Project Settings
- [ ] Copied SDK configuration keys:
  - [ ] `API_KEY`
  - [ ] `AUTH_DOMAIN`
  - [ ] `PROJECT_ID`
  - [ ] `STORAGE_BUCKET`
  - [ ] `MESSAGING_SENDER_ID`
  - [ ] `APP_ID`

### 4. Git & GitHub
- [ ] Committed all updated files: `git add .`
- [ ] Committed: `git commit -m "Configure production environment"`
- [ ] Pushed to GitHub: `git push origin main`

### 5. Render Backend Deployment
- [ ] Created new Web Service on `render.com` connected to GitHub repo
- [ ] Root Directory set to: `backend`
- [ ] Build Command: `pip install -r requirements.txt`
- [ ] Start Command: `uvicorn main:app --host 0.0.0.0 --port $PORT`
- [ ] Added Environment Variables:
  - [ ] `GROQ_API_KEY`
  - [ ] `MONGO_URI`
  - [ ] `MONGO_DB_NAME` = `voicerx`
  - [ ] `FIREBASE_API_KEY`
  - [ ] `JWT_SECRET`
  - [ ] `PATIENT_ENCRYPT_KEY`
- [ ] Tested live health check endpoint: `https://<your-backend>.onrender.com/api/health`

### 6. Vercel Frontend Deployment
- [ ] Imported repository on `vercel.com`
- [ ] Root Directory set to: `frontend`
- [ ] Added Environment Variables:
  - [ ] `NEXT_PUBLIC_API_URL` = `https://<your-backend>.onrender.com`
  - [ ] `NEXT_PUBLIC_FIREBASE_API_KEY`
  - [ ] `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`
  - [ ] `NEXT_PUBLIC_FIREBASE_PROJECT_ID`
  - [ ] `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`
  - [ ] `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
  - [ ] `NEXT_PUBLIC_FIREBASE_APP_ID`
- [ ] Deployed and copied Vercel URL: `https://<your-app>.vercel.app`

### 7. Post-Deployment Linking
- [ ] Added `<your-app>.vercel.app` to Firebase Authorized Domains
- [ ] Tested Google Sign-In on live Vercel URL
- [ ] Recorded test consultation, generated PDF, and verified record in MongoDB
