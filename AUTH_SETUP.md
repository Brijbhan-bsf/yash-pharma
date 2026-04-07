# Yash Pharma - NextAuth Configuration Guide

## Overview
This document contains the NextAuth.js setup with custom OTP authentication, role-based access control, and integration patterns for the Yash Pharma application.

---

## 1. Project Dependencies

```bash
npm install next-auth@beta @prisma/client bcryptjs jsonwebtoken zod
npm install -D @types/bcryptjs @types/jsonwebtoken
```

---

## 2. Prisma Schema Extensions for Auth

```prisma
// Add to your existing schema.prisma

model User {
  id            String    @id @default(cuid())
  email         String?   @unique
  phone         String?   @unique
  phoneVerified Boolean   @default(false)
  emailVerified DateTime?
  
  password      String?
  name          String?
  role          UserRole  @default(CUSTOMER)
  image         String?
  
  addresses     Address[]
  orders        Order[]
  prescriptions Prescription[]
  reviews       ProductReview[]
  
  // OTP fields
  otp           String?
  otpExpiresAt  DateTime?
  
  // NextAuth
  accounts      Account[]
  sessions      Session[]
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}

enum UserRole {
  CUSTOMER
  ADMIN
  DOCTOR
  DELIVERY_AGENT
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at       Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  
  @@unique([identifier, token])
}
```

---

## 3. NextAuth Configuration

### 3.1 Auth Configuration File

```typescript
// src/lib/auth.ts
import NextAuth from "next-auth"
import { PrismaAdapter } from "@auth/prisma-adapter"
import type { NextAuthConfig, Session, User } from "next-auth"
import type { Adapter } from "next-auth/adapters"
import CredentialsProvider from "next-auth/providers/credentials"
import GoogleProvider from "next-auth/providers/google"
import { compare } from "bcryptjs"
import { z } from "zod"

import { prisma } from "@/lib/db"
import { generateOTP, sendOTP } from "@/lib/otp"
import { UserRole } from "@prisma/client"

declare module "next-auth" {
  interface Session {
    user: {
      id: string
      role: UserRole
      phone?: string | null
      email?: string | null
      name?: string | null
      image?: string | null
    }
  }
  
  interface User {
    role: UserRole
    phone?: string | null
  }
}

const credentialsSchema = z.object({
  email: z.string().email().optional(),
  phone: z.string().optional(),
  password: z.string().min(6).optional(),
  otp: z.string().length(6).optional(),
  mode: z.enum(["login", "register", "otp"]),
})

export const authConfig: NextAuthConfig = {
  adapter: PrismaAdapter(prisma) as Adapter,
  session: {
    strategy: "jwt",
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  pages: {
    signIn: "/login",
    register: "/register",
    error: "/login",
    verifyRequest: "/verify-otp",
  },
  providers: [
    // Credentials Provider (Email/Password + OTP)
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        phone: { label: "Phone", type: "tel" },
        password: { label: "Password", type: "password" },
        otp: { label: "OTP", type: "text" },
        mode: { label: "Mode", type: "text" },
      },
      async authorize(credentials) {
        const parsed = credentialsSchema.safeParse(credentials)
        
        if (!parsed.success) {
          throw new Error("Invalid credentials format")
        }
        
        const { email, phone, password, otp, mode } = parsed.data
        
        // MODE: OTP (Send OTP)
        if (mode === "otp") {
          const user = await prisma.user.findFirst({
            where: email ? { email } : { phone },
          })
          
          if (!user) {
            throw new Error("User not found. Please register first.")
          }
          
          // Generate and save OTP
          const newOtp = generateOTP()
          const otpExpiry = new Date(Date.now() + 10 * 60 * 1000) // 10 minutes
          
          await prisma.user.update({
            where: { id: user.id },
            data: {
              otp: newOtp,
              otpExpiresAt: otpExpiry,
            },
          })
          
          // Send OTP via SMS/Email
          await sendOTP(user.phone || user.email!, newOtp)
          
          return {
            id: user.id,
            email: user.email,
            phone: user.phone,
            role: user.role,
          }
        }
        
        // MODE: OTP Verification
        if (mode === "verify-otp" && otp) {
          const user = await prisma.user.findFirst({
            where: email ? { email } : { phone },
          })
          
          if (!user) {
            throw new Error("User not found")
          }
          
          if (!user.otp || !user.otpExpiresAt) {
            throw new Error("OTP not generated. Please request a new OTP.")
          }
          
          if (user.otp !== otp) {
            throw new Error("Invalid OTP")
          }
          
          if (new Date() > user.otpExpiresAt) {
            throw new Error("OTP has expired. Please request a new one.")
          }
          
          // Clear OTP after successful verification
          await prisma.user.update({
            where: { id: user.id },
            data: {
              otp: null,
              otpExpiresAt: null,
              phoneVerified: true,
            },
          })
          
          return {
            id: user.id,
            email: user.email,
            phone: user.phone,
            role: user.role,
          }
        }
        
        // MODE: Password Login
        if (mode === "login" && password) {
          const user = await prisma.user.findFirst({
            where: email ? { email } : { phone },
          })
          
          if (!user || !user.password) {
            throw new Error("Invalid credentials")
          }
          
          const isValid = await compare(password, user.password)
          
          if (!isValid) {
            throw new Error("Invalid credentials")
          }
          
          return {
            id: user.id,
            email: user.email,
            phone: user.phone,
            role: user.role,
          }
        }
        
        // MODE: Register
        if (mode === "register") {
          // Check if user exists
          const existingUser = await prisma.user.findFirst({
            where: email ? { email } : { phone },
          })
          
          if (existingUser) {
            throw new Error("User already exists with this email or phone")
          }
          
          // Create new user (password will be set later or via OTP)
          const newUser = await prisma.user.create({
            data: {
              email: email || null,
              phone: phone || null,
              name: parsed.data.name || "User",
              role: UserRole.CUSTOMER,
            },
          })
          
          // Generate OTP for verification
          const newOtp = generateOTP()
          const otpExpiry = new Date(Date.now() + 10 * 60 * 1000)
          
          await prisma.user.update({
            where: { id: newUser.id },
            data: {
              otp: newOtp,
              otpExpiresAt: otpExpiry,
            },
          })
          
          await sendOTP(newUser.phone || newUser.email!, newOtp)
          
          return {
            id: newUser.id,
            email: newUser.email,
            phone: newUser.phone,
            role: newUser.role,
          }
        }
        
        throw new Error("Invalid authentication mode")
      },
    }),
    
    // Google OAuth Provider (Optional)
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id
        token.role = user.role
        token.phone = user.phone
      }
      return token
    },
    
    async session({ session, token }): Promise<Session> {
      if (session.user) {
        session.user.id = token.id as string
        session.user.role = token.role as UserRole
        session.user.phone = token.phone as string | null
      }
      return session
    },
    
    async signIn({ user, account }) {
      if (account?.provider === "google") {
        // For Google sign-in, update user with additional info if needed
        const existingUser = await prisma.user.findUnique({
          where: { email: user.email! },
        })
        
        if (!existingUser) {
          // Create user with Google info
          await prisma.user.create({
            data: {
              email: user.email,
              name: user.name,
              image: user.image,
              emailVerified: new Date(),
              role: UserRole.CUSTOMER,
            },
          })
        }
      }
      return true
    },
  },
  events: {
    async signIn({ user }) {
      console.log(`User ${user.id} signed in`)
    },
    async signOut({ token }) {
      console.log(`User signed out`)
    },
  },
}

export const { handlers, auth, signIn, signOut } = NextAuth(authConfig)
```

---

## 4. OTP Utility Functions

```typescript
// src/lib/otp.ts

// Generate 6-digit OTP
export function generateOTP(): string {
  return Math.floor(100000 + Math.random() * 900000).toString()
}

// Send OTP via SMS (using MSG91 or Twilio)
export async function sendOTP(contact: string, otp: string): Promise<boolean> {
  // For development, just log
  if (process.env.NODE_ENV === "development") {
    console.log(`[DEV] OTP for ${contact}: ${otp}`)
    return true
  }
  
  try {
    // MSG91 Implementation
    const response = await fetch(`https://api.msg91.com/api/v5/otp`, {
      method: "POST",
      headers: {
        "authkey": process.env.MSG91_AUTH_KEY!,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        template_id: process.env.MSG91_TEMPLATE_ID,
        mobile: contact.replace(/\D/g, ""), // Remove non-digits
        otp: otp,
      }),
    })
    
    return response.ok
  } catch (error) {
    console.error("Failed to send OTP:", error)
    return false
  }
}

// Send OTP via Email (using Resend or SendGrid)
export async function sendOTPEmail(email: string, otp: string): Promise<boolean> {
  if (process.env.NODE_ENV === "development") {
    console.log(`[DEV] OTP for ${email}: ${otp}`)
    return true
  }
  
  try {
    // Using Resend
    const { Resend } = await import("resend")
    const resend = new Resend(process.env.RESEND_API_KEY)
    
    await resend.emails.send({
      from: "Yash Pharma <noreply@yashpharma.com>",
      to: email,
      subject: "Your Yash Pharma OTP Code",
      html: `
        <div style="font-family: Arial, sans-serif; padding: 20px;">
          <h2>Your OTP Code</h2>
          <p>Your verification code is: <strong style="font-size: 24px; letter-spacing: 4px;">${otp}</strong></p>
          <p>This code expires in 10 minutes.</p>
          <p>If you didn't request this, please ignore this email.</p>
        </div>
      `,
    })
    
    return true
  } catch (error) {
    console.error("Failed to send OTP email:", error)
    return false
  }
}
```

---

## 5. Middleware - Protected Routes

```typescript
// src/middleware.ts
import { auth } from "@/lib/auth"
import { NextResponse } from "next/server"
import type { NextRequest } from "next/server"

// Define route patterns and their required roles
const protectedRoutes = [
  { path: "/checkout", roles: ["CUSTOMER"] },
  { path: "/orders", roles: ["CUSTOMER", "ADMIN"] },
  { path: "/account", roles: ["CUSTOMER"] },
  { path: "/appointments", roles: ["CUSTOMER", "DOCTOR"] },
]

const adminRoutes = [
  "/admin",
  "/api/admin",
]

const deliveryRoutes = [
  "/delivery",
  "/api/delivery",
]

export default auth((req) => {
  const { pathname } = req.nextUrl
  const session = req.auth
  
  // Public routes that don't need auth
  const publicRoutes = ["/", "/login", "/register", "/products", "/doctors"]
  const isPublicRoute = publicRoutes.some(route => pathname.startsWith(route))
  
  if (isPublicRoute) {
    return NextResponse.next()
  }
  
  // No session - redirect to login
  if (!session) {
    const loginUrl = new URL("/login", req.url)
    loginUrl.searchParams.set("callbackUrl", pathname)
    return NextResponse.redirect(loginUrl)
  }
  
  const userRole = session.user.role
  
  // Check admin routes
  if (adminRoutes.some(route => pathname.startsWith(route))) {
    if (userRole !== "ADMIN") {
      return NextResponse.redirect(new URL("/", req.url))
    }
  }
  
  // Check delivery routes
  if (deliveryRoutes.some(route => pathname.startsWith(route))) {
    if (userRole !== "DELIVERY_AGENT") {
      return NextResponse.redirect(new URL("/", req.url))
    }
  }
  
  // Check doctor routes
  if (pathname.startsWith("/doctor-dashboard")) {
    if (userRole !== "DOCTOR") {
      return NextResponse.redirect(new URL("/", req.url))
    }
  }
  
  // Check protected customer routes
  const isProtectedRoute = protectedRoutes.some(
    route => pathname.startsWith(route.path)
  )
  
  if (isProtectedRoute) {
    const routeConfig = protectedRoutes.find(
      route => pathname.startsWith(route.path)
    )
    
    if (routeConfig && !routeConfig.roles.includes(userRole)) {
      return NextResponse.redirect(new URL("/", req.url))
    }
  }
  
  return NextResponse.next()
})

// Configure which routes middleware runs on
export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - public folder
     * - api routes (except auth-related)
     */
    "/((?!_next/static|_next/image|favicon.ico|public|api/auth).*)",
  ],
}
```

---

## 6. API Routes

### 6.1 Auth Route Handler

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/lib/auth"

export const { GET, POST } = handlers
```

### 6.2 Send OTP Endpoint

```typescript
// src/app/api/auth/send-otp/route.ts
import { NextRequest, NextResponse } from "next/server"
import { z } from "zod"
import { prisma } from "@/lib/db"
import { generateOTP, sendOTP } from "@/lib/otp"

const schema = z.object({
  email: z.string().email().optional(),
  phone: z.string().optional(),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { email, phone } = schema.parse(body)
    
    const user = await prisma.user.findFirst({
      where: email ? { email } : { phone },
    })
    
    if (!user) {
      return NextResponse.json(
        { error: "User not found. Please register first." },
        { status: 404 }
      )
    }
    
    const otp = generateOTP()
    const otpExpiry = new Date(Date.now() + 10 * 60 * 1000)
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        otp,
        otpExpiresAt: otpExpiry,
      },
    })
    
    await sendOTP(user.phone || user.email!, otp)
    
    return NextResponse.json({
      message: "OTP sent successfully",
      expiresIn: 600, // 10 minutes in seconds
    })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: "Invalid input", details: error.errors },
        { status: 400 }
      )
    }
    
    console.error("Send OTP error:", error)
    return NextResponse.json(
      { error: "Failed to send OTP" },
      { status: 500 }
    )
  }
}
```

### 6.3 Resend OTP Endpoint

```typescript
// src/app/api/auth/resend-otp/route.ts
import { NextRequest, NextResponse } from "next/server"
import { z } from "zod"
import { prisma } from "@/lib/db"
import { generateOTP, sendOTP } from "@/lib/otp"

const schema = z.object({
  email: z.string().email().optional(),
  phone: z.string().optional(),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { email, phone } = schema.parse(body)
    
    const user = await prisma.user.findFirst({
      where: email ? { email } : { phone },
    })
    
    if (!user) {
      return NextResponse.json(
        { error: "User not found" },
        { status: 404 }
      )
    }
    
    // Rate limiting: Don't allow more than 3 OTP requests per 5 minutes
    const recentUpdate = await prisma.user.findFirst({
      where: {
        id: user.id,
        updatedAt: {
          gte: new Date(Date.now() - 5 * 60 * 1000),
        },
      },
    })
    
    if (recentUpdate && recentUpdate.otpExpiresAt) {
      const minutesSinceUpdate = Math.floor(
        (Date.now() - recentUpdate.updatedAt.getTime()) / 60000
      )
      const waitTime = 5 - minutesSinceUpdate
      
      if (waitTime > 0) {
        return NextResponse.json(
          { error: `Please wait ${waitTime} minutes before requesting again` },
          { status: 429 }
        )
      }
    }
    
    const otp = generateOTP()
    const otpExpiry = new Date(Date.now() + 10 * 60 * 1000)
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        otp,
        otpExpiresAt: otpExpiry,
      },
    })
    
    await sendOTP(user.phone || user.email!, otp)
    
    return NextResponse.json({
      message: "OTP resent successfully",
      expiresIn: 600,
    })
  } catch (error) {
    console.error("Resend OTP error:", error)
    return NextResponse.json(
      { error: "Failed to resend OTP" },
      { status: 500 }
    )
  }
}
```

### 6.4 Register with OTP

```typescript
// src/app/api/auth/register/route.ts
import { NextRequest, NextResponse } from "next/server"
import { z } from "zod"
import { prisma } from "@/lib/db"
import { generateOTP, sendOTP } from "@/lib/otp"
import { UserRole } from "@prisma/client"

const schema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email().optional(),
  phone: z.string().min(10).optional(),
  password: z.string().min(6).optional(),
})

export async function POST(req: NextRequest) {
  try {
    const body = await req.json()
    const { name, email, phone, password } = schema.parse(body)
    
    if (!email && !phone) {
      return NextResponse.json(
        { error: "Email or phone is required" },
        { status: 400 }
      )
    }
    
    // Check if user already exists
    const existingUser = await prisma.user.findFirst({
      where: {
        OR: [
          email ? { email } : null,
          phone ? { phone } : null,
        ].filter(Boolean) as any,
      },
    })
    
    if (existingUser) {
      return NextResponse.json(
        { error: "User already exists with this email or phone" },
        { status: 409 }
      )
    }
    
    // Create user
    const user = await prisma.user.create({
      data: {
        name,
        email: email || null,
        phone: phone || null,
        role: UserRole.CUSTOMER,
        password: password || null, // Will be set later
      },
    })
    
    // Generate OTP
    const otp = generateOTP()
    const otpExpiry = new Date(Date.now() + 10 * 60 * 1000)
    
    await prisma.user.update({
      where: { id: user.id },
      data: {
        otp,
        otpExpiresAt: otpExpiry,
      },
    })
    
    await sendOTP(user.phone || user.email!, otp)
    
    return NextResponse.json({
      message: "Registration successful. OTP sent for verification.",
      userId: user.id,
      expiresIn: 600,
    })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: "Invalid input", details: error.errors },
        { status: 400 }
      )
    }
    
    console.error("Registration error:", error)
    return NextResponse.json(
      { error: "Registration failed" },
      { status: 500 }
    )
  }
}
```

---

## 7. Client-Side Auth Hooks & Utilities

### 7.1 Auth Provider

```typescript
// src/components/providers/auth-provider.tsx
"use client"

import { SessionProvider } from "next-auth/react"
import { ReactNode } from "react"

export function AuthProvider({ children }: { children: ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>
}
```

### 7.2 Auth Hook

```typescript
// src/hooks/use-auth.ts
import { useSession, signIn, signOut } from "next-auth/react"
import { useRouter } from "next/navigation"
import { useState } from "react"

export function useAuth() {
  const { data: session, status } = useSession()
  const router = useRouter()
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  
  const login = async (email: string, password: string) => {
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await signIn("credentials", {
        email,
        password,
        mode: "login",
        redirect: false,
      })
      
      if (result?.error) {
        setError("Invalid credentials")
        return false
      }
      
      router.push("/")
      return true
    } catch (err) {
      setError("Login failed")
      return false
    } finally {
      setIsLoading(false)
    }
  }
  
  const requestOTP = async (email: string) => {
    setIsLoading(true)
    setError(null)
    
    try {
      const res = await fetch("/api/auth/send-otp", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email }),
      })
      
      const data = await res.json()
      
      if (!res.ok) {
        setError(data.error || "Failed to send OTP")
        return false
      }
      
      return true
    } catch (err) {
      setError("Network error")
      return false
    } finally {
      setIsLoading(false)
    }
  }
  
  const verifyOTP = async (email: string, otp: string) => {
    setIsLoading(true)
    setError(null)
    
    try {
      const result = await signIn("credentials", {
        email,
        otp,
        mode: "verify-otp",
        redirect: false,
      })
      
      if (result?.error) {
        setError("Invalid or expired OTP")
        return false
      }
      
      router.push("/")
      return true
    } catch (err) {
      setError("Verification failed")
      return false
    } finally {
      setIsLoading(false)
    }
  }
  
  const register = async (data: {
    name: string
    email: string
    phone?: string
    password?: string
  }) => {
    setIsLoading(true)
    setError(null)
    
    try {
      const res = await fetch("/api/auth/register", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data),
      })
      
      const response = await res.json()
      
      if (!res.ok) {
        setError(response.error || "Registration failed")
        return false
      }
      
      return true
    } catch (err) {
      setError("Network error")
      return false
    } finally {
      setIsLoading(false)
    }
  }
  
  const logout = async () => {
    await signOut({ callbackUrl: "/" })
  }
  
  return {
    session,
    status,
    user: session?.user,
    isAuthenticated: status === "authenticated",
    isLoading: status === "loading",
    isAdmin: session?.user?.role === "ADMIN",
    isDoctor: session?.user?.role === "DOCTOR",
    isDeliveryAgent: session?.user?.role === "DELIVERY_AGENT",
    login,
    requestOTP,
    verifyOTP,
    register,
    logout,
    error,
    clearError: () => setError(null),
  }
}
```

---

## 8. Environment Variables

```env
# .env.local

# NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-super-secret-key-at-least-32-chars

# Google OAuth (Optional)
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# SMS Provider (MSG91)
MSG91_AUTH_KEY=your-msg91-auth-key
MSG91_TEMPLATE_ID=your-msg91-template-id

# Email Provider (Resend)
RESEND_API_KEY=re_xxxxxxxxxxxxxxxx
```

---

## 9. Login Page Component

```typescript
// src/app/(auth)/login/page.tsx
"use client"

import { useState } from "react"
import { useAuth } from "@/hooks/use-auth"
import Link from "next/link"
import { Eye, EyeOff } from "lucide-react"

export default function LoginPage() {
  const [loginMethod, setLoginMethod] = useState<"email" | "phone">("email")
  const [email, setEmail] = useState("")
  const [phone, setPhone] = useState("")
  const [password, setPassword] = useState("")
  const [showPassword, setShowPassword] = useState(false)
  const [otpSent, setOtpSent] = useState(false)
  const [otp, setOtp] = useState("")
  
  const { login, requestOTP, verifyOTP, isLoading, error } = useAuth()
  
  const handlePasswordLogin = async (e: React.FormEvent) => {
    e.preventDefault()
    const identifier = loginMethod === "email" ? email : phone
    await login(identifier, password)
  }
  
  const handleRequestOTP = async () => {
    const identifier = loginMethod === "email" ? email : phone
    const success = await requestOTP(identifier)
    if (success) {
      setOtpSent(true)
    }
  }
  
  const handleOTPVerification = async (e: React.FormEvent) => {
    e.preventDefault()
    const identifier = loginMethod === "email" ? email : phone
    await verifyOTP(identifier, otp)
  }
  
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="w-full max-w-md p-8 bg-white rounded-xl shadow-lg">
        <h1 className="text-2xl font-bold text-center mb-2">Welcome Back!</h1>
        <p className="text-gray-600 text-center mb-6">Sign in to continue</p>
        
        {/* Login Method Toggle */}
        <div className="flex mb-6 bg-gray-100 rounded-lg p-1">
          <button
            type="button"
            onClick={() => setLoginMethod("email")}
            className={`flex-1 py-2 rounded-md text-sm font-medium transition-colors ${
              loginMethod === "email"
                ? "bg-white text-blue-600 shadow"
                : "text-gray-600 hover:text-gray-900"
            }`}
          >
            Email
          </button>
          <button
            type="button"
            onClick={() => setLoginMethod("phone")}
            className={`flex-1 py-2 rounded-md text-sm font-medium transition-colors ${
              loginMethod === "phone"
                ? "bg-white text-blue-600 shadow"
                : "text-gray-600 hover:text-gray-900"
            }`}
          >
            Phone
          </button>
        </div>
        
        {/* Password Login Form */}
        <form onSubmit={handlePasswordLogin} className="space-y-4">
          {loginMethod === "email" ? (
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Email
              </label>
              <input
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                placeholder="you@example.com"
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                required
              />
            </div>
          ) : (
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Phone
              </label>
              <input
                type="tel"
                value={phone}
                onChange={(e) => setPhone(e.target.value)}
                placeholder="+91 98765 43210"
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                required
              />
            </div>
          )}
          
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Password
            </label>
            <div className="relative">
              <input
                type={showPassword ? "text" : "password"}
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                placeholder="••••••••"
                className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 pr-10"
                required
              />
              <button
                type="button"
                onClick={() => setShowPassword(!showPassword)}
                className="absolute right-3 top-1/2 -translate-y-1/2 text-gray-400"
              >
                {showPassword ? <EyeOff size={18} /> : <Eye size={18} />}
              </button>
            </div>
          </div>
          
          <div className="flex justify-end">
            <Link href="/forgot-password" className="text-sm text-blue-600 hover:underline">
              Forgot Password?
            </Link>
          </div>
          
          <button
            type="submit"
            disabled={isLoading}
            className="w-full py-2 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700 disabled:opacity-50"
          >
            {isLoading ? "Signing in..." : "Sign In"}
          </button>
        </form>
        
        {/* Divider */}
        <div className="flex items-center my-6">
          <div className="flex-1 border-t border-gray-300" />
          <span className="px-4 text-sm text-gray-500">OR</span>
          <div className="flex-1 border-t border-gray-300" />
        </div>
        
        {/* OTP Login */}
        {!otpSent ? (
          <button
            onClick={handleRequestOTP}
            disabled={isLoading}
            className="w-full py-2 border border-blue-600 text-blue-600 rounded-lg font-medium hover:bg-blue-50 disabled:opacity-50"
          >
            Sign in with OTP
          </button>
        ) : (
          <form onSubmit={handleOTPVerification} className="space-y-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Enter OTP
              </label>
              <input
                type="text"
                value={otp}
                onChange={(e) => setOtp(e.target.value.replace(/\D/g, "").slice(0, 6))}
                placeholder="Enter 6-digit OTP"
                className="w-full px-4 py-2 border border-gray-300 rounded-lg text-center text-lg tracking-widest focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                maxLength={6}
                required
              />
              <p className="text-xs text-gray-500 mt-1 text-center">
                OTP sent to {loginMethod === "email" ? email : phone}
              </p>
            </div>
            
            <button
              type="submit"
              disabled={isLoading || otp.length !== 6}
              className="w-full py-2 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700 disabled:opacity-50"
            >
              {isLoading ? "Verifying..." : "Verify OTP"}
            </button>
            
            <button
              type="button"
              onClick={() => setOtpSent(false)}
              className="w-full text-sm text-gray-600 hover:text-gray-900"
            >
              Use password instead
            </button>
          </form>
        )}
        
        {/* Error Message */}
        {error && (
          <div className="mt-4 p-3 bg-red-50 border border-red-200 rounded-lg text-red-600 text-sm">
            {error}
          </div>
        )}
        
        {/* Register Link */}
        <p className="mt-6 text-center text-sm text-gray-600">
          Don&apos;t have an account?{" "}
          <Link href="/register" className="text-blue-600 font-medium hover:underline">
            Sign Up
          </Link>
        </p>
      </div>
    </div>
  )
}
```

---

## 10. Protected Route Wrapper

```typescript
// src/components/auth/protected-route.tsx
"use client"

import { useAuth } from "@/hooks/use-auth"
import { useRouter } from "next/navigation"
import { useEffect } from "react"
import { Loader2 } from "lucide-react"

interface ProtectedRouteProps {
  children: React.ReactNode
  allowedRoles?: string[]
}

export function ProtectedRoute({ children, allowedRoles }: ProtectedRouteProps) {
  const { isAuthenticated, isLoading, isAdmin, isDoctor, isDeliveryAgent } = useAuth()
  const router = useRouter()
  
  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      router.push("/login")
    }
    
    if (!isLoading && allowedRoles) {
      const roleMap = { isAdmin, isDoctor, isDeliveryAgent }
      const hasAccess = allowedRoles.some(role => roleMap[role as keyof typeof roleMap])
      
      if (!hasAccess) {
        router.push("/")
      }
    }
  }, [isLoading, isAuthenticated, router, allowedRoles])
  
  if (isLoading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <Loader2 className="w-8 h-8 animate-spin text-blue-600" />
      </div>
    )
  }
  
  if (!isAuthenticated) {
    return null
  }
  
  return <>{children}</>
}
```

---

## 11. Session Types Extension

```typescript
// src/types/next-auth.d.ts
import { UserRole } from "@prisma/client"

declare module "next-auth" {
  interface Session {
    user: {
      id: string
      role: UserRole
      phone?: string | null
      email?: string | null
      name?: string | null
      image?: string | null
    }
  }
  
  interface User {
    id: string
    role: UserRole
    phone?: string | null
    email?: string | null
    name?: string | null
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string
    role: UserRole
    phone: string | null
  }
}
```

---

This completes the NextAuth configuration guide with OTP authentication and role-based access control.
