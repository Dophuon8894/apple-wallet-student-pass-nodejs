# 🍎 Apple Wallet Student Pass — Complete Guide (Node.js)

> Generate, sign, and distribute `.pkpass` student ID cards for Apple Wallet using Node.js + AWS S3.

---

## 📑 Table of Contents

- [What is a PKPass?](#what-is-a-pkpass)
- [How It Looks on iPhone](#how-it-looks-on-iphone)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Step 1 — Apple Developer Setup](#step-1--apple-developer-setup)
- [Step 2 — Generate Certificates](#step-2--generate-certificates)
- [Step 3 — Install Dependencies](#step-3--install-dependencies)
- [Step 4 — Create the Pass Model](#step-4--create-the-pass-model)
- [Step 5 — Main Code (generatePass.js)](#step-5--main-code-generatepassjs)
- [Step 6 — S3 Upload Configuration](#step-6--s3-upload-configuration)
- [Pass Design Rules & Limitations](#pass-design-rules--limitations)
- [Pass Field Layout Reference](#pass-field-layout-reference)
- [Environment Variables](#environment-variables)
- [How It Works — Full Flow](#how-it-works--full-flow)
- [Usage Example](#usage-example)
- [Testing on iPhone](#testing-on-iphone)
- [Common Errors & Fixes](#common-errors--fixes)
- [Important Notes](#important-notes)

---

## What is a PKPass?

A `.pkpass` file is a digitally signed ZIP archive that Apple Wallet can read. It contains:

| File | Purpose |
|------|---------|
| `pass.json` | Pass data, fields, colors, barcode |
| `logo.png` / `logo@2x.png` | Logo shown on the pass |
| `icon.png` / `icon@2x.png` | App icon for notifications |
| `manifest.json` | SHA1 hash of every file |
| `signature` | Apple-signed cryptographic signature |

> ⚠️ The signature is created using your Apple Developer certificates. Without valid certificates, the pass **will not open** on any iPhone.

---

## How It Looks on iPhone

```
┌─────────────────────────────────┐
│  [Logo]        UNIVERSITY NAME  │  ← Header Field
├─────────────────────────────────┤
│  STUDENT                        │
│  John Doe                       │  ← Primary Field
├─────────────────────────────────┤
│  STUDENT ID      PASS CODE      │
│  STU-001         ABC123         │  ← Secondary Fields
├─────────────────────────────────┤
│  EMAIL           SESSION        │
│  john@uni.com    2026-27        │  ← Auxiliary Fields
├─────────────────────────────────┤
│         [QR CODE]               │  ← Barcode
└─────────────────────────────────┘
```

Back of the pass (tap ⓘ):
- ISSUED ON: 31/03/2026

---

## Requirements

### Software
- Node.js v16 or later
- macOS (preferred for certificate generation via Keychain)
- OpenSSL (comes with macOS; install via `brew install openssl` if needed)

### Accounts & Services
- ✅ **Apple Developer Account** — paid ($99/year) at [developer.apple.com](https://developer.apple.com)
- ✅ **AWS Account** — for S3 file storage

### Files You Must Have
- `wwdr.pem` — Apple's WWDR (Worldwide Developer Relations) root certificate
- `signerCert.pem` — Your Pass Type certificate
- `signerKey.pem` — Your private key

---

## Project Structure

```
project-root/
│
├── certificates/
│   ├── wwdr.pem                  ← Apple root cert (download from Apple)
│   ├── signerCert.pem            ← Your pass certificate
│   └── signerKey.pem             ← Your private key
│
├── models/
│   └── passModel.pass/           ← Pass template folder (must end in .pass)
│       ├── pass.json             ← Pass structure & default values
│       ├── logo.png              ← Logo (160×50 px recommended)
│       ├── logo@2x.png           ← Retina logo (320×100 px)
│       ├── icon.png              ← Icon (29×29 px)
│       └── icon@2x.png           ← Retina icon (58×58 px)
│
├── temp/                         ← Temporary .pkpass files before S3 upload
│
├── services/
│   └── generatePass.js           ← Main pass generation logic
│
└── common/
    └── controllers/
        └── fileuploadController.js  ← Your existing S3 upload helper
```

---

## Step 1 — Apple Developer Setup

### 1.1 Create a Pass Type ID

1. Log in to [developer.apple.com](https://developer.apple.com)
2. Go to **Certificates, Identifiers & Profiles**
3. Click **Identifiers** → **+** (Add new)
4. Select **Pass Type IDs** → Continue
5. Enter a description (e.g., `Student Pass`) and an identifier in the format:
   ```
   pass.com.yourcompany.studentpass
   ```
6. Click **Register**

> 📝 Save your Pass Type ID — you will need it as `PASS_TYPE_IDENTIFIER`.

### 1.2 Find Your Team ID

1. Go to **Membership** in your Apple Developer account
2. Copy your **Team ID** (10-character alphanumeric code, e.g., `ABC1234DEF`)

> 📝 Save your Team ID — you will need it as `TEAM_IDENTIFIER`.

---

## Step 2 — Generate Certificates

### 2.1 Create a Certificate Signing Request (CSR) — macOS

1. Open **Keychain Access** (Applications → Utilities)
2. Menu: **Keychain Access → Certificate Assistant → Request a Certificate from a Certificate Authority**
3. Fill in:
   - **User Email Address**: your Apple Developer email
   - **Common Name**: any name (e.g., `StudentPassKey`)
   - **CA Email Address**: leave blank
   - Select **Saved to disk**
4. Click **Continue** and save the `.certSigningRequest` file

### 2.2 Create the Pass Certificate on Apple Developer

1. Go to **Certificates, Identifiers & Profiles → Certificates**
2. Click **+** to add a new certificate
3. Under **Services**, select **Pass Type ID Certificate**
4. Select your Pass Type ID → Continue
5. Upload the `.certSigningRequest` file you just created
6. Download the generated certificate (`pass.cer`)

### 2.3 Install and Export as .p12

1. Double-click `pass.cer` to install it into Keychain Access
2. In Keychain Access, find the certificate under **My Certificates**
3. Right-click → **Export** → choose `.p12` format
4. Set a password (save this — you'll need it for OpenSSL)
5. Save as `certificate.p12`

### 2.4 Convert to PEM Format

Run these commands in Terminal:

```bash
# Extract certificate (signerCert.pem)
openssl pkcs12 -in certificate.p12 -out signerCert.pem -clcerts -nokeys

# Extract private key (signerKey.pem)
openssl pkcs12 -in certificate.p12 -out signerKey.pem -nocerts -nodes

# You'll be prompted for the .p12 password you set above
```

### 2.5 Download Apple WWDR Certificate

Download the latest G4 WWDR certificate from Apple:

```bash
curl -o wwdr.pem https://www.apple.com/certificateauthority/AppleWWDRCAG4.cer

# If it downloaded as DER format, convert it:
openssl x509 -inform DER -in wwdr.pem -out wwdr.pem
```

Or download manually from: [Apple PKI](https://www.apple.com/certificateauthority/)

> ⚠️ Use **WWDR G4** certificate. Older G1 certificates expired in 2023.

### 2.6 Place Certificates

Copy all three `.pem` files into your `certificates/` folder:

```bash
cp wwdr.pem ./certificates/
cp signerCert.pem ./certificates/
cp signerKey.pem ./certificates/
```

---

## Step 3 — Install Dependencies

```bash
npm install passkit-generator
```

The `passkit-generator` package handles:
- Reading your pass model folder
- Injecting dynamic field values
- Auto-generating `manifest.json` (file hashes)
- Signing the pass with your certificates
- Producing the final `.pkpass` buffer

---

## Step 4 — Create the Pass Model

### 4.1 Create the model folder

```bash
mkdir -p models/passModel.pass
```

> ⚠️ The folder **must** end with `.pass` — this is required by `passkit-generator`.

### 4.2 pass.json — The Pass Template

Create `models/passModel.pass/pass.json`:

```json
{
  "passTypeIdentifier": "pass.com.yourcompany.studentpass",
  "formatVersion": 1,
  "organizationName": "UNIVERSITY",
  "teamIdentifier": "YOUR_TEAM_ID",
  "serialNumber": "STUDENT_PASS_CODE",
  "description": "Student Pass",
  "logoText": "Student Pass",
  "backgroundColor": "rgb(139,30,63)",
  "foregroundColor": "rgb(255,255,255)",
  "labelColor": "rgb(255,255,255)",
  "generic": {
    "headerFields": [
      {
        "key": "university",
        "value": "University Name"
      }
    ],
    "primaryFields": [
      {
        "key": "student",
        "label": "STUDENT",
        "value": "STUDENT NAME"
      }
    ],
    "secondaryFields": [
      {
        "key": "studentId",
        "label": "Student ID",
        "value": "STUDENT_ID"
      },
      {
        "key": "passCode",
        "label": "Pass Code",
        "value": "PASS_CODE"
      }
    ],
    "auxiliaryFields": [
      {
        "key": "email",
        "label": "Email",
        "value": "student@email.com"
      },
      {
        "key": "session",
        "label": "Session",
        "value": "2026-27"
      }
    ],
    "backFields": [
      {
        "key": "issued",
        "label": "Issued On",
        "value": "Issued Date"
      }
    ]
  },
  "barcodes": [
    {
      "format": "PKBarcodeFormatQR",
      "message": "PASS_CODE",
      "altText": "PASS_CODE",
      "messageEncoding": "iso-8859-1"
    }
  ]
}
```

### 4.3 Add Images to the Model

Place these image files in `models/passModel.pass/`:

| File | Size | Purpose |
|------|------|---------|
| `logo.png` | 160 × 50 px | Logo on the pass front |
| `logo@2x.png` | 320 × 100 px | Retina logo |
| `icon.png` | 29 × 29 px | App icon for push notifications |
| `icon@2x.png` | 58 × 58 px | Retina app icon |

> ℹ️ Images in the model are **fallback defaults**. Your code can override them dynamically (see `generatePass.js`).

---

## Step 5 — Main Code (generatePass.js)

Create `services/generatePass.js`:

```javascript
const fs = require('fs');
const path = require('path');
const { PKPass } = require('passkit-generator');
const { uploadToS3 } = require('../common/controllers/fileuploadController');

async function generateAppleWalletPass(student, user) {
  try {
    // ── 1. Certificate paths ──────────────────────────────────────────────
    const certPath       = path.join(__dirname, '../certificates/wwdr.pem');
    const signerCertPath = path.join(__dirname, '../certificates/signerCert.pem');
    const signerKeyPath  = path.join(__dirname, '../certificates/signerKey.pem');

    // Return null if any certificate file is missing
    if (
      !fs.existsSync(certPath) ||
      !fs.existsSync(signerCertPath) ||
      !fs.existsSync(signerKeyPath)
    ) {
      console.error('Missing certificate files');
      return null;
    }

    const certificates = {
      wwdr:       fs.readFileSync(certPath),
      signerCert: fs.readFileSync(signerCertPath),
      signerKey:  fs.readFileSync(signerKeyPath)
    };

    // ── 2. Create pass from model folder ─────────────────────────────────
    const pass = await PKPass.from({
      model: path.join(__dirname, '../models/passModel.pass'),
      certificates
    });

    // ── 3. Pass identity ─────────────────────────────────────────────────
    pass.passTypeIdentifier = process.env.PASS_TYPE_IDENTIFIER || 'pass.com.yourcompany.studentpass';
    pass.teamIdentifier     = process.env.TEAM_IDENTIFIER     || 'YOUR_TEAM_ID';
    pass.serialNumber       = student.pass_code;
    pass.organizationName   = `${user?.first_name || ''} ${user?.last_name || ''}`.trim() || 'UNIVERSITY';
    pass.description        = `${student.student_name}'s Student Pass`;
    pass.logoText           = 'Student Pass';

    // ── 4. Dynamic logo / icon from user's hero image ────────────────────
    if (user?.hero_image) {
      try {
        const logoResponse = await fetch(user.hero_image);
        if (logoResponse.ok) {
          const logoBuffer  = Buffer.from(await logoResponse.arrayBuffer());
          pass.addBuffer('logo.png',    logoBuffer);
          pass.addBuffer('icon.png',    logoBuffer);
          pass.addBuffer('logo@2x.png', logoBuffer);
          pass.addBuffer('icon@2x.png', logoBuffer);
        } else {
          await loadDefaultIcons(pass);
        }
      } catch {
        await loadDefaultIcons(pass);
      }
    } else {
      await loadDefaultIcons(pass);
    }

    // ── 5. Colours ────────────────────────────────────────────────────────
    pass.backgroundColor = 'rgb(139,30,63)';  // Dark red
    pass.foregroundColor = 'rgb(255,255,255)'; // White text
    pass.labelColor      = 'rgb(255,255,255)'; // White labels

    // ── 6. Barcode (QR Code) ──────────────────────────────────────────────
    pass.setBarcodes({
      message:         student.pass_code,
      format:          'PKBarcodeFormatQR',
      messageEncoding: 'iso-8859-1'
    });

    // ── 7. Pass fields ────────────────────────────────────────────────────

    // Header (top-right corner)
    pass.headerFields[0] = {
      key:   'university',
      value: `${user?.first_name || ''} ${user?.last_name || ''}`.trim() || 'UNIVERSITY'
    };

    // Primary (large, prominent)
    pass.primaryFields[0] = {
      key:   'student',
      label: 'STUDENT',
      value: student.student_name
    };

    // Secondary (row below primary)
    pass.secondaryFields[0] = {
      key:   'studentId',
      label: 'STUDENT ID',
      value: student.student_id
    };
    pass.secondaryFields[1] = {
      key:   'passCode',
      label: 'PASS CODE',
      value: student.pass_code
    };

    // Auxiliary (row below secondary)
    pass.auxiliaryFields[0] = {
      key:   'email',
      label: 'EMAIL',
      value: student.student_email
    };
    pass.auxiliaryFields[1] = {
      key:     'session',
      label:   'SESSION',
      value:   '2026-27'
    };

    // Back of the pass (visible when user taps ⓘ)
    pass.backFields[0] = {
      key:   'issued',
      label: 'ISSUED ON',
      value: new Date().toLocaleDateString()
    };

    // ── 8. Generate buffer & validate ─────────────────────────────────────
    const buffer = pass.getAsBuffer();

    // Validate: must be a non-empty ZIP (starts with PK = 0x50 0x4B)
    if (!buffer || buffer.length === 0 || buffer[0] !== 0x50 || buffer[1] !== 0x4B) {
      console.error('Generated buffer is not a valid ZIP/pkpass file');
      return null;
    }

    // ── 9. Save temp file ─────────────────────────────────────────────────
    const fileName    = `student-pass-${student.id}-${Date.now()}.pkpass`;
    const tempDir     = path.join(__dirname, '../temp');
    const tempFilePath = path.join(tempDir, fileName);

    if (!fs.existsSync(tempDir)) {
      fs.mkdirSync(tempDir, { recursive: true });
    }

    fs.writeFileSync(tempFilePath, buffer);

    // ── 10. Upload to S3 & clean up ───────────────────────────────────────
    const s3Result = await uploadToS3(fileName, tempFilePath);
    fs.unlinkSync(tempFilePath); // Remove temp file after upload

    return s3Result.Location; // Public S3 URL of the .pkpass file

  } catch (error) {
    console.error('generateAppleWalletPass error:', error);
    return null;
  }
}

// ── Helper: load default icons from model folder ──────────────────────────
async function loadDefaultIcons(passInstance) {
  const logoPath = path.join(__dirname, '../models/passModel.pass/logo.png');
  if (fs.existsSync(logoPath)) {
    const logoBuffer = fs.readFileSync(logoPath);
    passInstance.addBuffer('logo.png', logoBuffer);
    passInstance.addBuffer('icon.png', logoBuffer);
  }
}

module.exports = { generateAppleWalletPass };
```

---

## Step 6 — S3 Upload Configuration

When uploading `.pkpass` files to S3, set the correct MIME type and headers so browsers prompt users to open in Wallet — not download as a ZIP.

```javascript
// Inside your fileuploadController.js (S3 upload helper)

const ext = path.extname(fileName).toLowerCase();

if (ext === '.pkpass') {
  params.ContentType        = 'application/vnd.apple.pkpass'; // ← Critical MIME type
  params.ContentDisposition = `attachment; filename="${fileName}"`;
  params.CacheControl       = 'no-cache, no-store, must-revalidate';
}

// Example full S3 params object:
const params = {
  Bucket:             process.env.AWS_S3_BUCKET,
  Key:                `passes/${fileName}`,
  Body:               fs.createReadStream(tempFilePath),
  ContentType:        'application/vnd.apple.pkpass',
  ContentDisposition: `attachment; filename="${fileName}"`,
  CacheControl:       'no-cache, no-store, must-revalidate',
  ACL:                'public-read' // or use pre-signed URLs
};
```

> ⚠️ Without `ContentType: 'application/vnd.apple.pkpass'`, iOS will not recognize the file as a Wallet pass.

---

## Pass Design Rules & Limitations

Apple imposes strict design restrictions on Wallet passes. You **cannot** customize freely.

### ✅ What You CAN Customize

| Property | Options |
|----------|---------|
| `backgroundColor` | Any `rgb(r,g,b)` value |
| `foregroundColor` | Any `rgb(r,g,b)` value (affects field values) |
| `labelColor` | Any `rgb(r,g,b)` value (affects field labels) |
| `logo.png` | Your brand/institution logo |
| Field values | Dynamic text for any field |
| Barcode type | QR, PDF417, Aztec, Code 128 |
| Pass type | `generic`, `boardingPass`, `coupon`, `eventTicket`, `storeCard` |

### ❌ What You CANNOT Customize

| Restriction | Explanation |
|-------------|-------------|
| Font | Always San Francisco (Apple's system font) — no custom fonts |
| Font size | Controlled entirely by pass type and field position |
| Layout / positions | Field positions (header, primary, secondary, etc.) are fixed by Apple |
| Border/shadow/shape | The pass card shape is fixed (rounded rectangle) |
| Background image | Supported in `boardingPass` and `eventTicket` types only, not `generic` |
| Strip image | Only for `coupon` and `storeCard` types |
| Thumbnail image | Only for `generic` and `eventTicket` types |
| Number of fields | Apple enforces limits: 1 primary, up to 3 secondary, up to 4 auxiliary |
| Animation / interactivity | Passes are static — no animation, no buttons (except system "Add to Wallet") |
| Custom barcode design | Barcode style is system-rendered |

### 📐 Image Size Reference

| Image | 1x Size | 2x Size | Notes |
|-------|---------|---------|-------|
| `logo.png` | 160 × 50 | 320 × 100 | Shown top-left on pass |
| `icon.png` | 29 × 29 | 58 × 58 | Lock screen / notifications |
| `thumbnail.png` | 90 × 90 | 180 × 180 | Right side (generic/eventTicket only) |
| `strip.png` | 375 × 123 | 750 × 246 | Behind primary field (coupon/storeCard only) |
| `background.png` | 180 × 220 | 360 × 440 | Behind entire pass (boardingPass/eventTicket only) |
| `footer.png` | 286 × 15 | 572 × 30 | Above barcode (boardingPass only) |

> ✅ Always provide both `1x` and `@2x` versions for all images.

---

## Pass Field Layout Reference

```
GENERIC pass type layout:
┌──────────────────────────────────────────┐
│ [logo.png]    logoText    [headerField 1] │
│                                          │
│  primaryField label                      │
│  primaryField value (large)              │
│                                          │
│  secondaryField 1    secondaryField 2    │
│  label               label              │
│  value               value              │
│                                          │
│  auxiliaryField 1    auxiliaryField 2   │
│  label               label             │
│  value               value             │
│                                          │
│            [BARCODE]                     │
└──────────────────────────────────────────┘
```

### Field Count Limits

| Field type | Max count |
|------------|-----------|
| `headerFields` | 1 (sometimes 2, but 1 recommended) |
| `primaryFields` | 1 |
| `secondaryFields` | Up to 3 |
| `auxiliaryFields` | Up to 4 |
| `backFields` | Unlimited |

---

## Environment Variables

Add these to your `.env` file:

```env
# Apple Wallet
PASS_TYPE_IDENTIFIER=pass.com.yourcompany.studentpass
TEAM_IDENTIFIER=ABC1234DEF

# AWS S3
AWS_S3_BUCKET=your-s3-bucket-name
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-east-1
```

---

## How It Works — Full Flow

```
Student record + User record
         │
         ▼
  Load certificates (wwdr, signerCert, signerKey)
         │
         ▼
  PKPass.from(model folder + certificates)
         │
         ▼
  Inject dynamic data (name, ID, email, pass code)
         │
         ▼
  Add logo image (from user's hero_image URL or default)
         │
         ▼
  Set barcode (QR code with pass_code)
         │
         ▼
  pass.getAsBuffer() → signed .pkpass ZIP buffer
         │
         ▼
  Validate buffer (must start with PK bytes)
         │
         ▼
  Save to /temp/student-pass-{id}-{timestamp}.pkpass
         │
         ▼
  Upload to AWS S3 with correct MIME type
         │
         ▼
  Delete temp file
         │
         ▼
  Return public S3 URL
```

---

## Usage Example

```javascript
const { generateAppleWalletPass } = require('./services/generatePass');

// Example student object (from your DB)
const student = {
  id:            42,
  student_name:  'John Doe',
  student_id:    'STU-2024-001',
  student_email: 'john.doe@university.edu',
  pass_code:     'ABC123XYZ'
};

// Example user/institution object
const user = {
  first_name: 'State',
  last_name:  'University',
  hero_image: 'https://cdn.example.com/university-logo.png'
};

// Generate pass and get S3 URL
const passUrl = await generateAppleWalletPass(student, user);

if (passUrl) {
  console.log('Pass URL:', passUrl);
  // Send this URL to the student via email or in-app link
  // When they tap it on iPhone → iOS prompts "Add to Wallet"
} else {
  console.log('Pass generation failed');
}
```

---

## Testing on iPhone

1. Deploy your backend and generate a pass URL
2. **Send the URL** to an iPhone via email, SMS, or in-app link
3. Tap the link on iPhone — iOS should display the "Add to Wallet" dialog
4. Tap **Add** — the pass appears in Wallet

### Simulator Testing (Xcode)

```bash
# Drag and drop .pkpass file into iOS Simulator
# Or use simctl:
xcrun simctl openurl booted "file:///path/to/student-pass.pkpass"
```

> ⚠️ Passes **only work on real iOS devices or Xcode Simulator**. They do not work on Android or in web browsers.

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Certificate not found` | Missing `.pem` files | Check `certificates/` folder exists with all 3 files |
| `Pass not opening on iPhone` | Wrong MIME type on S3 | Set `ContentType: 'application/vnd.apple.pkpass'` |
| `Invalid signature` | Expired or wrong certificate | Re-download certificate from Apple Developer portal |
| `Buffer starts with wrong bytes` | `getAsBuffer()` failed | Check certificate validity; catch and log the error |
| `WWDR certificate error` | Using old G1 WWDR cert | Download latest **WWDR G4** from Apple PKI |
| `Pass Type ID mismatch` | `.env` value ≠ Apple Developer | Ensure `PASS_TYPE_IDENTIFIER` exactly matches your registered Pass Type ID |
| `Team ID mismatch` | Wrong team ID in env | Copy Team ID from **Membership** section in Apple Developer |
| `Logo not showing` | Image fetch failed | Falls back to default `logo.png` in model folder — ensure that exists |
| `Pass fields blank` | Wrong field keys | Keys must match exactly between `pass.json` and your JS assignments |

---

## Important Notes

- ✅ Passes work **only on iOS** (iPhone, iPad, Apple Watch)
- ✅ Android and web browsers **cannot** open `.pkpass` files natively
- ✅ Certificates **expire** — renew them annually on Apple Developer
- ✅ Always provide `@2x` images for crisp display on Retina screens
- ✅ The `serialNumber` should be **unique per student** — use `pass_code`
- ✅ Back fields (`backFields`) are shown when the user flips the pass — good for T&Cs or extra info
- ✅ Use `PKBarcodeFormatQR` for QR codes, `PKBarcodeFormatPDF417` for PDF417
- ✅ Keep the temp directory clean — always delete after S3 upload
- ✅ Test with real student data before deploying to production
- ⚠️ Never commit your `.pem` certificate files to Git — add to `.gitignore`

```gitignore
# .gitignore
certificates/
*.pem
*.p12
*.pkpass
temp/
```

---

## Quick Checklist Before Going Live

- [ ] Pass Type ID registered on Apple Developer
- [ ] `signerCert.pem`, `signerKey.pem`, `wwdr.pem` in `certificates/`
- [ ] `pass.json` in `models/passModel.pass/` with correct identifiers
- [ ] `logo.png` and `icon.png` in `models/passModel.pass/`
- [ ] `.env` has `PASS_TYPE_IDENTIFIER` and `TEAM_IDENTIFIER`
- [ ] S3 upload sets `ContentType: 'application/vnd.apple.pkpass'`
- [ ] Tested on a real iPhone

---
