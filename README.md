# esign-leadgen-shared

**Shared Components for E-Signature Functionality**

This repository contains reusable UI components and backend services used by both the **LeadGen** app (forms) and the **eSign** app (document signing).

---

## 📦 What's Inside

This monorepo contains two packages:

1. **`packages/ui`** - NPM package (`@cliqforms/esign-components`)
   - Vue 3 + Vuetify components for signature capture, consent, and OTP verification
   - Used by both leadgen-ui and esign-ui

2. **`packages/services`** - Composer package (`cliqforms/esign-services`)
   - PHP services for OTP, signature handling, audit logging, and PDF generation
   - Used by both leadgen-api and esign-api

**Note:** This repo contains **NO database migrations or models**. Each app manages its own database.

---

## 🎯 Purpose

### Legal Compliance
These components implement legally defensible electronic signatures that meet:
- **ETA** (Australia) - Electronic Transactions Act
- **eSign** (United States) - ESIGN Act
- **eIDAS** (European Union) - Simple Electronic Signature standards

### Four Pillars of Compliance

| Requirement | Implementation |
|------------|----------------|
| **Intent to sign** | Mandatory consent checkbox with legal text |
| **Attribution** | Email OTP verification (4-digit code, 5-minute expiry) |
| **Integrity** | SHA-256 hash of signed PDF |
| **Record retention** | Tamper-evident PDF with embedded audit trail |

---

## 🏗️ Architecture

### Application & Database Separation

```
┌─────────────────────────────────────────────────────────────┐
│                    esign-leadgen-shared                      │
│  (NO DATABASE - Just reusable services & UI components)     │
│                                                              │
│  ├── UI Components (Vue 3 + Vuetify)                        │
│  │   ├── SignatureCanvas, SignatureTypeInput               │
│  │   ├── SignatureUpload, SignatureField                   │
│  │   ├── ConsentCheckbox, OTPModal                         │
│  │                                                          │
│  └── Backend Services (PHP)                                 │
│      ├── OTPService, SignatureService                      │
│      ├── AuditService, PdfService                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                          ↓ Used by ↓
        ┌─────────────────────────────────────────┐
        │                                          │
┌───────▼──────┐                         ┌────────▼─────┐
│  LeadGen App │                         │  eSign App   │
├──────────────┤                         ├──────────────┤
│ leadgen-ui   │                         │ esign-ui     │
│ leadgen-api  │                         │ esign-api    │
└──────┬───────┘                         └──────┬───────┘
       │                                        │
       ▼                                        ▼
┌─────────────┐                         ┌─────────────┐
│ leadgen_db  │                         │  esign_db   │
├─────────────┤                         ├─────────────┤
│ form_leads  │                         │ documents   │
│ form_visits │                         │ signatures  │
│ users       │                         │ signers     │
│ ...etc      │                         │ ...etc      │
└─────────────┘                         └─────────────┘
```

**Key Points:**
- Each app has its **own database** (leadgen_db, esign_db)
- Each app manages its **own migrations**
- Shared repo provides **only services and UI components**
- Services connect to whatever database the parent app configures

---

## 📦 Package 1: UI Components (NPM)

### Tech Stack
- **Vue 3** (Composition API)
- **Vuetify 3** (Material Design components)
- **Vite** (Build tool)
- **HTML5 Canvas** (Signature drawing)

### Installation in LeadGen UI

```bash
cd /path/to/leadgen-ui
npm install @cliqforms/esign-components
```

### Installation in eSign UI

```bash
cd /path/to/esign-ui
npm install @cliqforms/esign-components
```

### Usage Example

```vue
<script setup>
import { SignatureField, ConsentCheckbox, OTPModal } from '@cliqforms/esign-components'
import { ref } from 'vue'

const signatureData = ref(null)
const consentGiven = ref(false)
const showOTP = ref(false)
</script>

<template>
  <v-container>
    <SignatureField v-model="signatureData" />
    <ConsentCheckbox v-model="consentGiven" />
    
    <OTPModal 
      v-if="showOTP"
      :email="userEmail"
      @verify="handleOTPVerify"
      @resend="handleOTPResend"
    />
  </v-container>
</template>
```

### Available Components

#### SignatureCanvas.vue
Draw signature with mouse or touch (uses HTML5 Canvas)
- Props: `width`, `height`, `strokeColor`, `strokeWidth`
- Emits: `update:modelValue` (base64 PNG)

#### SignatureTypeInput.vue
Type name and convert to signature image
- Props: `fontFamily`, `fontSize`
- Emits: `update:modelValue` (base64 PNG)

#### SignatureUpload.vue
Upload image file (PNG, JPG, SVG)
- Props: `maxSize`, `allowedTypes`
- Emits: `update:modelValue` (base64)

#### SignatureField.vue
Main wrapper with tabs (Draw | Type | Upload) - uses Vuetify tabs
- Props: `required`, `disabled`
- Emits: `update:modelValue`, `validate`

#### ConsentCheckbox.vue
Legal consent checkbox (Vuetify checkbox)
- Props: `text` (customizable legal text)
- Emits: `update:modelValue` (boolean)

#### OTPModal.vue
Email verification popup (Vuetify dialog)
- Props: `email`, `expiresIn`
- Emits: `verify`, `resend`

---

## 📦 Package 2: Backend Services (Composer)

### Tech Stack
- **PHP 8.3+**
- **Laravel** (peer dependency for Mail, Storage)
- **dompdf** (PDF generation)
- **intervention/image** (Image processing)

### Installation in LeadGen API

```bash
cd /path/to/leadgen-api
composer require cliqforms/esign-services
```

**Note:** leadgen-api manages its own database (leadgen_db) with migrations.

### Installation in eSign API

```bash
cd /path/to/esign-api
composer require cliqforms/esign-services
```

**Note:** esign-api manages its own database (esign_db) with migrations.

### Usage Example

```php
use Cliqforms\EsignServices\Services\{
    OTPService,
    SignatureService,
    AuditService,
    PdfService
};

class SignatureController extends Controller
{
    public function __construct(
        protected OTPService $otpService,
        protected SignatureService $signatureService,
        protected AuditService $auditService,
        protected PdfService $pdfService
    ) {}
    
    public function sendOTP(Request $request)
    {
        $otp = $this->otpService->generate(); // "1234"
        $this->otpService->store($request->email, $otp, $submissionId);
        $this->otpService->send($request->email, $otp);
        
        $this->auditService->log($submissionId, 'otp_sent', [
            'email' => $request->email,
            'ip' => $request->ip()
        ]);
        
        return response()->json(['message' => 'OTP sent']);
    }
}
```

### Available Services

#### OTPService
```php
generate(): string                                    // Generate 4-digit OTP
store(string $email, string $otp, int $id): void     // Store in database
verify(string $email, string $otp): bool             // Verify OTP code
send(string $email, string $otp): void               // Send via email
isExpired(string $email): bool                       // Check 5-minute expiry
```

**Database:** Uses whatever database connection the parent app configures.

#### SignatureService
```php
validate(string $base64Data): bool                   // Validate signature data
uploadToStorage(string $base64, int $id): string     // Upload to Digital Ocean, return URL
getImageType(string $base64): string                 // Detect image type
```

#### AuditService
```php
log(int $submissionId, string $eventType, array $metadata): void  // Log event
getTrail(int $submissionId): array                                 // Get complete audit trail
```

Event types: `'otp_sent'`, `'otp_verified'`, `'signature_captured'`, `'pdf_generated'`

#### PdfService
```php
generate(array $formData, string $signatureUrl, array $audit): string  // Generate PDF, return path
computeHash(string $pdfPath): string                                   // SHA-256 hash
uploadToStorage(string $pdfContent, int $id): string                   // Upload PDF, return URL
embedHashInPdf(string $pdfPath, string $hash): void                    // Add hash to footer
```

---

## 🗄️ Database Management (NOT in this repo)

### Each App Manages Its Own Database

**leadgen-api** (Phase 1):
```
leadgen-api/database/migrations/
├── xxx_create_signature_submissions_table.php
├── xxx_create_otp_verifications_table.php
└── xxx_create_audit_logs_table.php
```
Connects to: **leadgen_db**

**esign-api** (Phase 2):
```
esign-api/database/migrations/
├── xxx_create_documents_table.php
├── xxx_create_signature_submissions_table.php
├── xxx_create_otp_verifications_table.php
└── xxx_create_audit_logs_table.php
```
Connects to: **esign_db**

### Recommended Table Schema

The shared services expect these tables to exist in the parent app's database:

**For LeadGen (in leadgen_db):**
```sql
-- Signature submissions
CREATE TABLE signature_submissions (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  form_lead_id BIGINT,                    -- Link to form_leads
  signer_email VARCHAR(255),
  signature_image_url VARCHAR(500),
  signature_type VARCHAR(20),             -- 'draw', 'type', 'upload'
  consent_given BOOLEAN,
  consent_timestamp TIMESTAMP,
  email_verified BOOLEAN,
  signed_pdf_url VARCHAR(500),
  pdf_hash VARCHAR(64),
  status VARCHAR(50),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

-- OTP verifications
CREATE TABLE otp_verifications (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  submission_id BIGINT,
  email VARCHAR(255),
  otp_code VARCHAR(6),
  verified BOOLEAN,
  expires_at TIMESTAMP,
  created_at TIMESTAMP
);

-- Audit logs
CREATE TABLE audit_logs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  submission_id BIGINT,
  event_type VARCHAR(50),
  ip_address VARCHAR(45),
  user_agent TEXT,
  metadata JSON,
  created_at TIMESTAMP
);
```

**For eSign (in esign_db):**
Similar tables with document-specific columns.

---

## 🔄 Phase 1 vs Phase 2

### Phase 1: LeadGen Forms (Current)
**App:** leadgen-ui + leadgen-api  
**Database:** leadgen_db  
**Use Case:** Add signature field to forms

**What's built:**
- Signature capture (draw/type/upload)
- Consent checkbox
- Email OTP verification
- PDF generation with audit trail

### Phase 2: eSign Standalone App (Future)
**App:** esign-ui + esign-api  
**Database:** esign_db  
**Use Case:** Upload documents, send for signature

**What's added:**
- Document upload (PDF, Word, etc.)
- Signature placement on documents
- Multi-signer workflows
- Template library

**Reuse from Phase 1:**
- ✅ All UI components from this repo (no changes needed)
- ✅ All backend services from this repo (no changes needed)
- ✅ Same signature capture flow
- ✅ Same OTP verification
- ✅ Same PDF generation

**Effort savings:** ~50% of Phase 2 work already complete!

---

## 🛠️ Development

### Building UI Components

```bash
cd packages/ui
npm install
npm run build        # Build for production
npm run dev          # Development mode with hot reload
```

### Testing Services Locally

```bash
cd packages/services
composer install
composer test        # Run PHPUnit tests (when added)
```

### Publishing Packages

#### NPM Package
```bash
cd packages/ui
npm version patch    # or minor, major
npm publish
```

#### Composer Package
```bash
cd packages/services
# Tag version in git
git tag v1.0.1
git push origin v1.0.1
# Packagist auto-updates from GitHub tags
```

---

## 📋 Implementation Checklist

### For LeadGen (Phase 1)
- [ ] Install `@cliqforms/esign-components` in leadgen-ui
- [ ] Install `cliqforms/esign-services` in leadgen-api
- [ ] Create migrations in leadgen-api for signature tables
- [ ] Run migrations against leadgen_db
- [ ] Add SignatureController to leadgen-api
- [ ] Modify LeadService for signature handling
- [ ] Add SIGNATURE field to form builder
- [ ] Add OTP flow to form submission
- [ ] Test complete flow: form → signature → OTP → PDF

### For eSign (Phase 2)
- [ ] Install `@cliqforms/esign-components` in esign-ui
- [ ] Install `cliqforms/esign-services` in esign-api
- [ ] Create migrations in esign-api for signature + document tables
- [ ] Run migrations against esign_db
- [ ] Build document upload UI
- [ ] Build signature placement UI
- [ ] Implement multi-signer workflows
- [ ] Reuse OTP, PDF, audit services (no changes needed)

---

## 🔒 Security & Compliance

### Data Storage
- **Signature images:** Digital Ocean Spaces (configured by parent app)
- **PDFs:** Digital Ocean Spaces (configured by parent app)
- **Database records:** Managed by parent app (leadgen_db or esign_db)

### Email Verification
- **OTP:** 4-digit numeric code
- **Expiry:** 5 minutes
- **Delivery:** Via parent app's mail configuration

### Audit Trail
Every signature action logged with:
- Timestamp (UTC)
- IP address
- User agent (browser/device)
- Event type
- Additional metadata

### PDF Integrity
- **Hash algorithm:** SHA-256
- **Hash location:** Embedded in PDF footer
- **Verification:** Anyone can recompute hash to verify document unchanged

---

## 📞 Support & Questions

**For LeadGen integration issues:**
- Check leadgen-api logs for backend errors
- Check leadgen-ui console for frontend errors
- Verify leadgen_db connection in `.env`

**For eSign integration issues:**
- Check esign-api logs
- Check esign-ui console
- Verify esign_db connection in `.env`

**For package bugs:**
- Open issue in this repository
- Include: steps to reproduce, expected behavior, actual behavior
- Tag with `bug` label

---

## 📄 License

Proprietary - CliqForms

---

## 🚀 Version History

### v1.0.0 (Phase 1 - Jan 2025)
- Initial release
- Vue 3 + Vuetify components
- Signature capture (draw, type, upload)
- Consent checkbox
- Email OTP verification
- PDF generation with audit trail
- SHA-256 hashing
- Support for LeadGen forms

### v2.0.0 (Phase 2 - Future)
- Document upload support
- Multi-signer workflows
- Signature placement on PDFs
- Template library
- Support for eSign standalone app

---

**Ready to build legally compliant e-signatures! 🚀**
