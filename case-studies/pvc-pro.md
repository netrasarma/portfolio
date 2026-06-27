# PVC Pro — AI-Powered Document Processing SaaS

🌐 **Live product:** [pvcpro.online](https://pvcpro.online)

## Overview

PVC Pro is a production SaaS platform that automates the detection, processing, and print-ready generation of PVC ID cards from 11+ types of Indian government documents (Aadhaar, PAN, Voter ID, Driving License, ABHA, Ayushman Bharat, E-Shram, and more). Users upload a document; the system identifies the type, removes the background, crops to exact card dimensions, and returns a print-ready front/back layout — in under 10 seconds, with no manual design work.

It is built, deployed, and actively maintained solo — covering backend, AI/ML processing, cloud infrastructure, payments, and a desktop companion app.

## The Problem

In India, millions of people need their government ID documents in a durable, wallet-sized PVC card format. The traditional route — a photocopy/CSC shop manually cropping and formatting each card in image-editing software — is slow, inconsistent, and doesn't scale. PVC Pro turns that manual, per-card design work into an automated pipeline.

## Architecture

```
User → pvcpro.online (Firebase Hosting CDN)
          ↓ (Proxy Rewrite)
     Google Cloud Run (Flask Backend, asia-south1)
          ↓                          ↓
   Firestore Database          GCP Secret Manager
   (users, credits,             (service account keys,
    transactions, sessions)      API credentials)
          ↓
   ThreadPoolExecutor    →    SSE (Server-Sent Events)
   (background processing)    real-time status to browser
```

**Processing pipeline (per document):**
1. Upload (PDF/image) → password & credit validation
2. Job queued with UUID → picked up by a thread pool (8 parallel workers)
3. Pages extracted at 300 DPI (PyMuPDF) → QR code scanned for fast classification
4. Document type auto-classified by region/script
5. AI background removal (RMBG-1.4, with `rembg` as a CPU fallback)
6. Contour detection → perspective warp → precise crop to 85.6mm × 54mm
7. Watermarking → encoded to WebP/PNG
8. Real-time progress pushed to the browser via SSE
9. Credit deducted atomically via a Firestore transaction

Average processing time: **~3–8 seconds per card** (optimized down from an initial ~15–20s).

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, JavaScript (responsive, mobile-first) |
| Backend | Python Flask (Gunicorn) |
| AI / Computer Vision | OpenCV, PyMuPDF, pyzbar, EasyOCR |
| AI Background Removal | PyTorch, RMBG-1.4, rembg |
| Database | Google Cloud Firestore |
| Auth | Firebase Authentication (Email/Password, Google OAuth, Phone OTP) |
| Payments | Cashfree Payment Gateway |
| Hosting / Infra | Google Cloud Run, Firebase Hosting (CDN) |
| CI/CD | GitHub Actions + Cloud Build → Cloud Run |
| Security | GCP Secret Manager, HMAC request signatures |

## Production Scale (real metrics, queried from the live database)

| Metric | Value |
|---|---|
| Registered users | 368 |
| Paying customers | 40 (~10.9% conversion) |
| Documents processed | 3,545 |
| Credit transactions logged | 3,933 |
| Active since | Dec 2025 |
| Average daily volume | ~18–35 cards/day |
| Peak single-day throughput | 62 cards in 24 hours |

**Processing distribution:** Aadhaar Card (48.3%), Voter ID (22.6%), Driving License (5.5%), PAN Card (5.2%), plus regional script variants (Marathi, Telugu, Kannada) and other supported document types.

## Key Engineering Challenges

- **Cookie stripping by the CDN proxy** — Firebase Hosting's rewrite layer was stripping session cookies; fixed by renaming the session cookie to the CDN-safe `__session`.
- **Front/back misclassification on Voter ID cards** — resolved by tuning Haar cascade parameters for face detection used in side classification.
- **Processing speed** — cut Aadhaar processing time by ~80% by optimizing QR-quadrant scan order and removing a redundant base64 encoding step.
- **Credential security** — all service keys and payment credentials are kept server-side in GCP Secret Manager; the client never has direct database access.

## Data & Analytics Relevance

- Designed a complete financial ledger in Firestore with atomic, auditable credit transactions (before/after balance on every event)
- Tracked and analyzed processing-time bottlenecks to prioritize optimization work
- Built referral, trial, and subscription-conversion tracking to understand user funnel behavior
- Structured error logging (e.g. distinct types for password-required vs. wrong-password failures) to support debugging and product decisions

## Also Built: Desktop App

A companion **Electron.js** desktop app wraps the same processing pipeline for fully offline use — no file-size limits and no subscription gating, packaged as a Windows installer via `electron-builder`.

---
*Live at [pvcpro.online](https://pvcpro.online) — built and maintained solo by Netra Prasad Sarma.*
