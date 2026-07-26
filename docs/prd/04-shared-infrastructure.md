# Shared Infrastructure

> These are cross-cutting concerns built once and reused by every feature.
> Build these FIRST before any feature user story.

## 1. API Response Helpers

**File:** `apps/api/src/types/common.types.ts`

```ts
export interface ApiResponse<T> {
  success: boolean;
  message: string;
  data?: T;
  errors?: Record<string, string>;
}

export interface PaginatedResponse<T> {
  success: boolean;
  message: string;
  data: T[];
  meta: {
    total: number;
    page: number;
    totalPages: number;
    limit: number;
  };
}

export interface PaginationQuery {
  page?: number;
  limit?: number;
}
```

**File:** `apps/api/src/utils/response.ts`

```ts
import { Response } from "express";
import { ApiResponse, PaginatedResponse } from "../types/common.types";

export function sendSuccess<T>(res: Response, statusCode: number, message: string, data?: T): void {
  const response: ApiResponse<T> = { success: true, message, data };
  res.status(statusCode).json(response);
}

export function sendPaginated<T>(
  res: Response,
  message: string,
  data: T[],
  meta: { total: number; page: number; limit: number }
): void {
  const response: PaginatedResponse<T> = {
    success: true,
    message,
    data,
    meta: {
      ...meta,
      totalPages: Math.ceil(meta.total / meta.limit),
    },
  };
  res.status(200).json(response);
}

export function sendError(res: Response, statusCode: number, message: string, errors?: Record<string, string>): void {
  const response: ApiResponse<null> = { success: false, message, errors };
  res.status(statusCode).json(response);
}
```

---

## 2. Global Error Handler

**File:** `apps/api/src/middlewares/errorHandler.ts`

```ts
import { Request, Response, NextFunction } from "express";

// Custom error class
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public errors?: Record<string, string>
  ) {
    super(message);
    this.name = "AppError";
  }
}

// MUST be the LAST middleware registered in app.ts
export function errorHandler(err: Error, req: Request, res: Response, _next: NextFunction): void {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      message: err.message,
      errors: err.errors,
    });
    return;
  }

  // Multer errors
  if (err.name === "MulterError") {
    const multerMessages: Record<string, string> = {
      LIMIT_FILE_SIZE: "Image must be smaller than 2MB",
      LIMIT_UNEXPECTED_FILE: "Unexpected file field",
    };
    res.status(400).json({
      success: false,
      message: multerMessages[(err as any).code] || "File upload error",
    });
    return;
  }

  // Prisma known errors
  if (err.name === "PrismaClientKnownRequestError") {
    const prismaErr = err as any;
    if (prismaErr.code === "P2002") {
      res.status(409).json({
        success: false,
        message: "A record with this value already exists",
      });
      return;
    }
    if (prismaErr.code === "P2025") {
      res.status(404).json({
        success: false,
        message: "Record not found",
      });
      return;
    }
  }

  // Unknown error — log and return generic 500
  console.error("Unhandled error:", err);
  res.status(500).json({
    success: false,
    message: "Something went wrong. Please try again.",
  });
}
```

---

## 3. Authentication Middleware

**File:** `apps/api/src/middlewares/authenticate.ts`

```ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { AppError } from "./errorHandler";

export interface AuthUser {
  userId: string;
  role: "CUSTOMER" | "ORGANIZER";
}

declare global {
  namespace Express {
    interface Request {
      user?: AuthUser;
    }
  }
}

export function authenticate(req: Request, _res: Response, next: NextFunction): void {
  try {
    const token = req.cookies?.accessToken;

    if (!token) {
      throw new AppError(401, "Please log in to continue");
    }

    const decoded = jwt.verify(token, process.env.JWT_ACCESS_SECRET!) as AuthUser;
    req.user = decoded;
    next();
  } catch (error) {
    if (error instanceof AppError) {
      next(error);
      return;
    }
    next(new AppError(401, "Please log in to continue"));
  }
}
```

---

## 4. Authorization Middleware

**File:** `apps/api/src/middlewares/authorize.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { AppError } from "./errorHandler";

export function authorize(roles: ("CUSTOMER" | "ORGANIZER")[]) {
  return (req: Request, _res: Response, next: NextFunction): void => {
    if (!req.user) {
      next(new AppError(401, "Please log in to continue"));
      return;
    }

    if (!roles.includes(req.user.role)) {
      next(new AppError(403, "Only event organizers can access this resource"));
      return;
    }

    next();
  };
}
```

---

## 5. Validation Middleware

**File:** `apps/api/src/middlewares/validate.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { ZodSchema, ZodError } from "zod";
import { AppError } from "./errorHandler";

export function validate(schema: ZodSchema) {
  return (req: Request, _res: Response, next: NextFunction): void => {
    try {
      schema.parse({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        const errors: Record<string, string> = {};
        error.errors.forEach((e) => {
          const path = e.path.slice(1).join("."); // Remove "body"/"query"/"params" prefix
          errors[path] = e.message;
        });
        next(new AppError(400, "Validation failed", errors));
        return;
      }
      next(error);
    }
  };
}
```

---

## 6. File Upload Middleware

**File:** `apps/api/src/middlewares/upload.ts`

```ts
import multer from "multer";
import { AppError } from "./errorHandler";

const ALLOWED_MIME_TYPES = ["image/jpeg", "image/png", "image/webp"];
const MAX_FILE_SIZE = 2 * 1024 * 1024; // 2MB

export const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_FILE_SIZE },
  fileFilter: (_req, file, cb) => {
    if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
      cb(new AppError(400, "Only JPG, PNG, or WEBP images are allowed") as any);
      return;
    }
    cb(null, true);
  },
});
```

---

## 7. Cloudinary Config

**File:** `apps/api/src/config/cloudinary.ts`

```ts
import { v2 as cloudinary } from "cloudinary";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

export async function uploadToCloudinary(
  buffer: Buffer,
  folder: string
): Promise<{ url: string; publicId: string }> {
  return new Promise((resolve, reject) => {
    cloudinary.uploader
      .upload_stream({ folder, resource_type: "image" }, (error, result) => {
        if (error || !result) {
          reject(error || new Error("Upload failed"));
          return;
        }
        resolve({ url: result.secure_url, publicId: result.public_id });
      })
      .end(buffer);
  });
}

export async function deleteFromCloudinary(publicId: string): Promise<void> {
  await cloudinary.uploader.destroy(publicId);
}
```

---

## 8. Mailer Service

**File:** `apps/api/src/config/mailer.ts`

```ts
import nodemailer from "nodemailer";

function createTransporter() {
  if (process.env.NODE_ENV === "production") {
    // Gmail SMTP
    return nodemailer.createTransport({
      service: "gmail",
      auth: {
        user: process.env.GMAIL_USER,
        pass: process.env.GMAIL_APP_PASSWORD,
      },
    });
  }

  // Mailtrap (development)
  return nodemailer.createTransport({
    host: process.env.MAILTRAP_HOST,
    port: Number(process.env.MAILTRAP_PORT),
    auth: {
      user: process.env.MAILTRAP_USER,
      pass: process.env.MAILTRAP_PASS,
    },
  });
}

export const transporter = createTransporter();
```

**File:** `apps/api/src/services/mailer.service.ts`

```ts
import { transporter } from "../config/mailer";

interface EmailOptions {
  to: string;
  subject: string;
  html: string;
}

export async function sendEmail(options: EmailOptions): Promise<void> {
  // Fire and forget — do NOT await in the calling service
  transporter
    .sendMail({
      from: `"EventHub" <${process.env.MAIL_FROM || "noreply@eventhub.com"}>`,
      to: options.to,
      subject: options.subject,
      html: options.html,
    })
    .catch((err) => {
      console.error("Email send failed:", err);
      // Swallow the error — email failure should not break the API flow
    });
}
```

---

## 9. Token Utilities

**File:** `apps/api/src/utils/token.ts`

```ts
import jwt from "jsonwebtoken";
import crypto from "crypto";

export function generateAccessToken(payload: { userId: string; role: string }): string {
  return jwt.sign(payload, process.env.JWT_ACCESS_SECRET!, { expiresIn: "15m" });
}

export function generateRefreshToken(payload: { userId: string; role: string }): string {
  return jwt.sign(payload, process.env.JWT_REFRESH_SECRET!, { expiresIn: "7d" });
}

export function verifyAccessToken(token: string): { userId: string; role: string } {
  return jwt.verify(token, process.env.JWT_ACCESS_SECRET!) as { userId: string; role: string };
}

export function verifyRefreshToken(token: string): { userId: string; role: string } {
  return jwt.verify(token, process.env.JWT_REFRESH_SECRET!) as { userId: string; role: string };
}

export function generateRandomToken(): string {
  return crypto.randomBytes(32).toString("hex");
}

export function hashToken(token: string): string {
  return crypto.createHash("sha256").update(token).digest("hex");
}
```

---

## 10. Password Utilities

**File:** `apps/api/src/utils/password.ts`

```ts
import bcrypt from "bcryptjs";

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function comparePassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

---

## 11. Code Generators

**File:** `apps/api/src/utils/code-generator.ts`

```ts
import crypto from "crypto";

// Characters: uppercase letters + digits, excluding ambiguous ones (0/O, 1/I/L)
const CHARSET = "ABCDEFGHJKMNPQRSTUVWXYZ23456789";

function randomCode(length: number): string {
  const bytes = crypto.randomBytes(length);
  let result = "";
  for (let i = 0; i < length; i++) {
    result += CHARSET[bytes[i] % CHARSET.length];
  }
  return result;
}

export function generateReferralCode(): string {
  return `REF-${randomCode(6)}`;
}

export function generateBookingCode(): string {
  const year = new Date().getFullYear();
  return `EVT-${year}-${randomCode(6)}`;
}

export function generateVoucherCode(): string {
  return randomCode(8);
}
```

---

## 12. Slug Generator

**File:** `apps/api/src/utils/slug.ts`

```ts
export function generateSlug(name: string): string {
  return name
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "")
    .concat("-", Date.now().toString(36)); // Append timestamp for uniqueness
}
```

---

## 13. Cookie Helpers

**File:** `apps/api/src/utils/cookie.ts`

```ts
import { Response } from "express";

const IS_PRODUCTION = process.env.NODE_ENV === "production";

export function setAuthCookies(res: Response, accessToken: string, refreshToken: string): void {
  res.cookie("accessToken", accessToken, {
    httpOnly: true,
    secure: IS_PRODUCTION,
    sameSite: IS_PRODUCTION ? "none" : "lax",
    maxAge: 15 * 60 * 1000, // 15 minutes
    path: "/",
  });

  res.cookie("refreshToken", refreshToken, {
    httpOnly: true,
    secure: IS_PRODUCTION,
    sameSite: IS_PRODUCTION ? "none" : "lax",
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    path: "/",
  });
}

export function clearAuthCookies(res: Response): void {
  res.clearCookie("accessToken", { path: "/" });
  res.clearCookie("refreshToken", { path: "/" });
}
```

---

## 14. Express App Setup

**File:** `apps/api/src/app.ts`

```ts
import express from "express";
import cors from "cors";
import cookieParser from "cookie-parser";
import { errorHandler } from "./middlewares/errorHandler";
import routes from "./routes";

const app = express();

// Middleware order matters
app.use(cors({
  origin: process.env.FRONTEND_URL || "http://localhost:3000",
  credentials: true,  // Required for httpOnly cookies
}));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

// Routes
app.use("/api/v1", routes);

// Global error handler — MUST be LAST
app.use(errorHandler);

export default app;
```

---

## 15. Environment Variables

**File:** `apps/api/.env.example`

```env
# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# JWT
JWT_ACCESS_SECRET=your-super-secret-access-key-min-32-chars
JWT_REFRESH_SECRET=your-super-secret-refresh-key-min-32-chars

# Cloudinary
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# Mailer (Development — Mailtrap)
MAILTRAP_HOST=sandbox.smtp.mailtrap.io
MAILTRAP_PORT=2525
MAILTRAP_USER=your-mailtrap-user
MAILTRAP_PASS=your-mailtrap-pass

# Mailer (Production — Gmail)
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=your-app-password

# General
MAIL_FROM=noreply@eventhub.com
FRONTEND_URL=http://localhost:3000
NODE_ENV=development
PORT=8000
```

**File:** `apps/web/.env.example`

```env
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
```

---

## 16. Frontend Axios Instance

**File:** `apps/web/lib/api/axios.ts`

```ts
import axios from "axios";

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  withCredentials: true, // Send httpOnly cookies
  headers: { "Content-Type": "application/json" },
});

// Response interceptor — auto-refresh on 401
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        await axios.post(
          `${process.env.NEXT_PUBLIC_API_URL}/auth/refresh`,
          {},
          { withCredentials: true }
        );
        return api(originalRequest);
      } catch {
        // Refresh failed — redirect to login
        if (typeof window !== "undefined") {
          window.location.href = "/login";
        }
        return Promise.reject(error);
      }
    }

    return Promise.reject(error);
  }
);

export default api;
```

## 5. CI/CD Testing Pipeline & Infrastructure Verification

Every cross-cutting infrastructure concern defined above (error handler, response envelopes, Axios interceptors, soft-deletion queries) is verified via automated verification gates before any feature code is merged to `main`, in compliance with **ADR 0002**.

### 5.1 GitHub Actions Verification Workflow (`ci.yml`)
**File:** `.github/workflows/ci.yml`
This workflow establishes 5 mandatory verification gates running in parallel or sequence against a PostgreSQL 16 service container:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
          POSTGRES_DB: event_platform_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Gate 1 — TypeScript Type Check
        run: npm run type-check --workspaces

      - name: Gate 2 — ESLint Lint Check
        run: npm run lint --workspaces

      - name: Gate 3 — Backend Integration & Unit Tests (Jest + supertest)
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/event_platform_test?schema=public
          JWT_ACCESS_SECRET: test_access_secret
          JWT_REFRESH_SECRET: test_refresh_secret
        run: |
          npx prisma migrate deploy --schema=apps/api/prisma/schema.prisma
          npm test --workspace=apps/api

      - name: Gate 4 — Frontend Component Tests (Jest + RTL)
        run: npm test --workspace=apps/web

      - name: Gate 5 — Production Bundle Build
        run: npm run build --workspaces
```

### 5.2 Infrastructure Verification Rules
- **Error Handler & Envelope Verification:** The global error handler (`errorHandler.ts`) and response helpers (`sendSuccess`, `sendPaginated`, `sendError`) are verified in `apps/api/__tests__/integration/core/error-handler.test.ts` via `supertest` asserting exact JSON envelopes for 400 Zod errors, 401 JWT errors, 403 RBAC errors, and 500 unhandled errors.
- **Axios Silent Refresh Verification:** The client-side Axios interceptor (`axios.ts`) is verified in `apps/web/__tests__/lib/axios.test.ts` using `nock` or mock service workers to simulate a 401 response followed by a successful `/refresh` call and retry.

