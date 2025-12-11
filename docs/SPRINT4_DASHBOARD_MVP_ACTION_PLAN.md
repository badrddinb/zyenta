# Sprint 4: Dashboard MVP - Actionable Implementation Plan

## Overview

This document provides step-by-step implementation instructions for Sprint 4 of Zyenta's Phase 1 MVP. Sprint 4 builds the **Dashboard** - the Next.js client application where users create and manage their AI-generated stores.

**Objective:** Build a fully functional dashboard with authentication, store creation wizard, and management interfaces.

**Prerequisites:** Sprints 1-3 must be completed with:
- Monorepo configured and working
- Database schema deployed with Prisma
- Genesis Engine and Media Studio services operational
- Docker infrastructure running

---

## Task 1: Next.js Project Setup

### 1.1 Initialize Project

```bash
# Navigate to apps directory
cd apps

# Create Next.js 14 project
pnpm create next-app@latest dashboard --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"

cd dashboard
```

### 1.2 Package Configuration

**File: `apps/dashboard/package.json`**
```json
{
  "name": "@zyenta/dashboard",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev --port 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "lint:fix": "next lint --fix"
  },
  "dependencies": {
    "next": "14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",

    "@zyenta/database": "workspace:*",
    "@zyenta/shared-types": "workspace:*",
    "@zyenta/ui": "workspace:*",

    "next-auth": "^4.24.0",
    "@auth/prisma-adapter": "^1.4.0",

    "@tanstack/react-query": "^5.20.0",
    "zustand": "^4.5.0",

    "@radix-ui/react-alert-dialog": "^1.0.5",
    "@radix-ui/react-avatar": "^1.0.4",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-progress": "^1.0.3",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-separator": "^1.0.3",
    "@radix-ui/react-slot": "^1.0.2",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-toast": "^1.1.5",

    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "tailwindcss-animate": "^1.0.7",

    "lucide-react": "^0.330.0",
    "date-fns": "^3.3.0",
    "zod": "^3.22.0",
    "react-hook-form": "^7.50.0",
    "@hookform/resolvers": "^3.3.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.3.0"
  }
}
```

### 1.3 TypeScript Configuration

**File: `apps/dashboard/tsconfig.json`**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "plugins": [{ "name": "next" }],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "noEmit": true,
    "incremental": true,
    "paths": {
      "@/*": ["./src/*"],
      "@zyenta/database": ["../../packages/database/src"],
      "@zyenta/shared-types": ["../../packages/shared-types/src"],
      "@zyenta/ui": ["../../packages/ui/src"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 1.4 Tailwind Configuration

**File: `apps/dashboard/tailwind.config.ts`**
```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  darkMode: ["class"],
  content: [
    "./src/**/*.{js,ts,jsx,tsx,mdx}",
    "../../packages/ui/src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
};

export default config;
```

### 1.5 Global Styles

**File: `apps/dashboard/src/styles/globals.css`**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### 1.6 Next.js Configuration

**File: `apps/dashboard/next.config.js`**
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  transpilePackages: ["@zyenta/ui", "@zyenta/database", "@zyenta/shared-types"],
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "**.zyenta.com",
      },
      {
        protocol: "http",
        hostname: "localhost",
      },
    ],
  },
  experimental: {
    serverActions: {
      bodySizeLimit: "2mb",
    },
  },
};

module.exports = nextConfig;
```

---

## Task 2: Authentication with NextAuth.js

### 2.1 NextAuth Configuration

**File: `apps/dashboard/src/lib/auth.ts`**
```typescript
import { PrismaAdapter } from "@auth/prisma-adapter";
import { type NextAuthOptions } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import GoogleProvider from "next-auth/providers/google";
import GitHubProvider from "next-auth/providers/github";
import { compare } from "bcryptjs";

import { prisma } from "@zyenta/database";

export const authOptions: NextAuthOptions = {
  adapter: PrismaAdapter(prisma),
  session: {
    strategy: "jwt",
  },
  pages: {
    signIn: "/login",
    signUp: "/register",
    error: "/login",
  },
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    GitHubProvider({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          throw new Error("Invalid credentials");
        }

        const user = await prisma.user.findUnique({
          where: { email: credentials.email },
        });

        if (!user || !user.passwordHash) {
          throw new Error("Invalid credentials");
        }

        const isValid = await compare(credentials.password, user.passwordHash);

        if (!isValid) {
          throw new Error("Invalid credentials");
        }

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          image: user.image,
        };
      },
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      if (token) {
        session.user.id = token.id as string;
        session.user.name = token.name;
        session.user.email = token.email;
        session.user.image = token.picture;
      }
      return session;
    },
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
      }

      // Fetch user from database to get latest data
      const dbUser = await prisma.user.findUnique({
        where: { email: token.email! },
      });

      if (dbUser) {
        token.id = dbUser.id;
        token.name = dbUser.name;
        token.email = dbUser.email;
        token.picture = dbUser.image;
      }

      return token;
    },
  },
};
```

### 2.2 Auth Types Extension

**File: `apps/dashboard/src/types/next-auth.d.ts`**
```typescript
import { type DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    user: {
      id: string;
    } & DefaultSession["user"];
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string;
  }
}
```

### 2.3 Auth API Route

**File: `apps/dashboard/src/app/api/auth/[...nextauth]/route.ts`**
```typescript
import NextAuth from "next-auth";
import { authOptions } from "@/lib/auth";

const handler = NextAuth(authOptions);

export { handler as GET, handler as POST };
```

### 2.4 Auth Utilities

**File: `apps/dashboard/src/lib/auth-utils.ts`**
```typescript
import { getServerSession } from "next-auth";
import { redirect } from "next/navigation";

import { authOptions } from "./auth";

export async function getCurrentUser() {
  const session = await getServerSession(authOptions);
  return session?.user;
}

export async function requireAuth() {
  const user = await getCurrentUser();
  if (!user) {
    redirect("/login");
  }
  return user;
}
```

---

## Task 3: Root Layout and Providers

### 3.1 Root Layout

**File: `apps/dashboard/src/app/layout.tsx`**
```typescript
import type { Metadata } from "next";
import { Inter } from "next/font/google";

import { Providers } from "@/components/providers";
import { Toaster } from "@/components/ui/toaster";
import "@/styles/globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Zyenta Dashboard",
  description: "Create and manage your AI-powered e-commerce stores",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className={inter.className}>
        <Providers>
          {children}
          <Toaster />
        </Providers>
      </body>
    </html>
  );
}
```

### 3.2 Providers Component

**File: `apps/dashboard/src/components/providers.tsx`**
```typescript
"use client";

import { SessionProvider } from "next-auth/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <SessionProvider>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </SessionProvider>
  );
}
```

### 3.3 Landing Page (Redirect)

**File: `apps/dashboard/src/app/page.tsx`**
```typescript
import { redirect } from "next/navigation";
import { getCurrentUser } from "@/lib/auth-utils";

export default async function HomePage() {
  const user = await getCurrentUser();

  if (user) {
    redirect("/dashboard");
  } else {
    redirect("/login");
  }
}
```

---

## Task 4: Authentication Pages

### 4.1 Auth Layout

**File: `apps/dashboard/src/app/(auth)/layout.tsx`**
```typescript
export default function AuthLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="min-h-screen flex items-center justify-center bg-muted/50">
      <div className="w-full max-w-md px-4">{children}</div>
    </div>
  );
}
```

### 4.2 Login Page

**File: `apps/dashboard/src/app/(auth)/login/page.tsx`**
```typescript
import { Metadata } from "next";
import Link from "next/link";

import { LoginForm } from "@/components/auth/login-form";

export const metadata: Metadata = {
  title: "Login | Zyenta",
  description: "Login to your Zyenta account",
};

export default function LoginPage() {
  return (
    <div className="space-y-6">
      <div className="text-center space-y-2">
        <h1 className="text-2xl font-bold">Welcome back</h1>
        <p className="text-muted-foreground">
          Enter your credentials to access your account
        </p>
      </div>

      <LoginForm />

      <p className="text-center text-sm text-muted-foreground">
        Don&apos;t have an account?{" "}
        <Link href="/register" className="text-primary hover:underline">
          Sign up
        </Link>
      </p>
    </div>
  );
}
```

**File: `apps/dashboard/src/components/auth/login-form.tsx`**
```typescript
"use client";

import { useState } from "react";
import { signIn } from "next-auth/react";
import { useRouter, useSearchParams } from "next/navigation";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Icons } from "@/components/icons";

const loginSchema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(1, "Password is required"),
});

type LoginFormData = z.infer<typeof loginSchema>;

export function LoginForm() {
  const router = useRouter();
  const searchParams = useSearchParams();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const callbackUrl = searchParams.get("callbackUrl") || "/dashboard";

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });

  async function onSubmit(data: LoginFormData) {
    setIsLoading(true);
    setError(null);

    try {
      const result = await signIn("credentials", {
        email: data.email,
        password: data.password,
        redirect: false,
      });

      if (result?.error) {
        setError("Invalid email or password");
        return;
      }

      router.push(callbackUrl);
      router.refresh();
    } catch (error) {
      setError("Something went wrong. Please try again.");
    } finally {
      setIsLoading(false);
    }
  }

  async function signInWithGoogle() {
    setIsLoading(true);
    await signIn("google", { callbackUrl });
  }

  async function signInWithGitHub() {
    setIsLoading(true);
    await signIn("github", { callbackUrl });
  }

  return (
    <div className="space-y-6">
      <div className="grid gap-4">
        <Button
          variant="outline"
          onClick={signInWithGoogle}
          disabled={isLoading}
        >
          <Icons.google className="mr-2 h-4 w-4" />
          Continue with Google
        </Button>
        <Button
          variant="outline"
          onClick={signInWithGitHub}
          disabled={isLoading}
        >
          <Icons.gitHub className="mr-2 h-4 w-4" />
          Continue with GitHub
        </Button>
      </div>

      <div className="relative">
        <div className="absolute inset-0 flex items-center">
          <span className="w-full border-t" />
        </div>
        <div className="relative flex justify-center text-xs uppercase">
          <span className="bg-background px-2 text-muted-foreground">
            Or continue with
          </span>
        </div>
      </div>

      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
        {error && (
          <div className="p-3 text-sm text-destructive bg-destructive/10 rounded-md">
            {error}
          </div>
        )}

        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <Input
            id="email"
            type="email"
            placeholder="name@example.com"
            disabled={isLoading}
            {...register("email")}
          />
          {errors.email && (
            <p className="text-sm text-destructive">{errors.email.message}</p>
          )}
        </div>

        <div className="space-y-2">
          <Label htmlFor="password">Password</Label>
          <Input
            id="password"
            type="password"
            disabled={isLoading}
            {...register("password")}
          />
          {errors.password && (
            <p className="text-sm text-destructive">
              {errors.password.message}
            </p>
          )}
        </div>

        <Button type="submit" className="w-full" disabled={isLoading}>
          {isLoading && <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />}
          Sign In
        </Button>
      </form>
    </div>
  );
}
```

### 4.3 Register Page

**File: `apps/dashboard/src/app/(auth)/register/page.tsx`**
```typescript
import { Metadata } from "next";
import Link from "next/link";

import { RegisterForm } from "@/components/auth/register-form";

export const metadata: Metadata = {
  title: "Register | Zyenta",
  description: "Create a new Zyenta account",
};

export default function RegisterPage() {
  return (
    <div className="space-y-6">
      <div className="text-center space-y-2">
        <h1 className="text-2xl font-bold">Create an account</h1>
        <p className="text-muted-foreground">
          Enter your details to get started with Zyenta
        </p>
      </div>

      <RegisterForm />

      <p className="text-center text-sm text-muted-foreground">
        Already have an account?{" "}
        <Link href="/login" className="text-primary hover:underline">
          Sign in
        </Link>
      </p>
    </div>
  );
}
```

**File: `apps/dashboard/src/components/auth/register-form.tsx`**
```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Icons } from "@/components/icons";

const registerSchema = z
  .object({
    name: z.string().min(2, "Name must be at least 2 characters"),
    email: z.string().email("Invalid email address"),
    password: z.string().min(8, "Password must be at least 8 characters"),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  });

type RegisterFormData = z.infer<typeof registerSchema>;

export function RegisterForm() {
  const router = useRouter();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
  });

  async function onSubmit(data: RegisterFormData) {
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch("/api/auth/register", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          name: data.name,
          email: data.email,
          password: data.password,
        }),
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message || "Registration failed");
      }

      router.push("/login?registered=true");
    } catch (error) {
      setError(
        error instanceof Error ? error.message : "Something went wrong"
      );
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      {error && (
        <div className="p-3 text-sm text-destructive bg-destructive/10 rounded-md">
          {error}
        </div>
      )}

      <div className="space-y-2">
        <Label htmlFor="name">Name</Label>
        <Input
          id="name"
          placeholder="John Doe"
          disabled={isLoading}
          {...register("name")}
        />
        {errors.name && (
          <p className="text-sm text-destructive">{errors.name.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input
          id="email"
          type="email"
          placeholder="name@example.com"
          disabled={isLoading}
          {...register("email")}
        />
        {errors.email && (
          <p className="text-sm text-destructive">{errors.email.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="password">Password</Label>
        <Input
          id="password"
          type="password"
          disabled={isLoading}
          {...register("password")}
        />
        {errors.password && (
          <p className="text-sm text-destructive">{errors.password.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label htmlFor="confirmPassword">Confirm Password</Label>
        <Input
          id="confirmPassword"
          type="password"
          disabled={isLoading}
          {...register("confirmPassword")}
        />
        {errors.confirmPassword && (
          <p className="text-sm text-destructive">
            {errors.confirmPassword.message}
          </p>
        )}
      </div>

      <Button type="submit" className="w-full" disabled={isLoading}>
        {isLoading && <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />}
        Create Account
      </Button>
    </form>
  );
}
```

### 4.4 Registration API Route

**File: `apps/dashboard/src/app/api/auth/register/route.ts`**
```typescript
import { NextResponse } from "next/server";
import { hash } from "bcryptjs";
import { z } from "zod";

import { prisma } from "@zyenta/database";

const registerSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  password: z.string().min(8),
});

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const { name, email, password } = registerSchema.parse(body);

    // Check if user exists
    const existingUser = await prisma.user.findUnique({
      where: { email },
    });

    if (existingUser) {
      return NextResponse.json(
        { message: "User with this email already exists" },
        { status: 400 }
      );
    }

    // Hash password
    const passwordHash = await hash(password, 12);

    // Create user
    const user = await prisma.user.create({
      data: {
        name,
        email,
        passwordHash,
      },
    });

    return NextResponse.json(
      { message: "User created successfully", userId: user.id },
      { status: 201 }
    );
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { message: "Invalid input", errors: error.errors },
        { status: 400 }
      );
    }

    console.error("Registration error:", error);
    return NextResponse.json(
      { message: "Something went wrong" },
      { status: 500 }
    );
  }
}
```

---

## Task 5: Dashboard Layout

### 5.1 Dashboard Layout Component

**File: `apps/dashboard/src/app/(dashboard)/layout.tsx`**
```typescript
import { redirect } from "next/navigation";

import { getCurrentUser } from "@/lib/auth-utils";
import { DashboardNav } from "@/components/dashboard/nav";
import { DashboardHeader } from "@/components/dashboard/header";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getCurrentUser();

  if (!user) {
    redirect("/login");
  }

  return (
    <div className="min-h-screen flex">
      {/* Sidebar */}
      <DashboardNav />

      {/* Main content */}
      <div className="flex-1 flex flex-col">
        <DashboardHeader user={user} />
        <main className="flex-1 p-6 bg-muted/30">{children}</main>
      </div>
    </div>
  );
}
```

### 5.2 Navigation Component

**File: `apps/dashboard/src/components/dashboard/nav.tsx`**
```typescript
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import {
  LayoutDashboard,
  Store,
  Settings,
  Activity,
  CreditCard,
  Package,
} from "lucide-react";

import { cn } from "@/lib/utils";

const navItems = [
  {
    title: "Overview",
    href: "/dashboard",
    icon: LayoutDashboard,
  },
  {
    title: "Stores",
    href: "/dashboard/stores",
    icon: Store,
  },
  {
    title: "Jobs",
    href: "/dashboard/jobs",
    icon: Activity,
  },
  {
    title: "Billing",
    href: "/dashboard/settings/billing",
    icon: CreditCard,
  },
  {
    title: "Settings",
    href: "/dashboard/settings",
    icon: Settings,
  },
];

export function DashboardNav() {
  const pathname = usePathname();

  return (
    <nav className="w-64 border-r bg-card p-4 space-y-4">
      {/* Logo */}
      <div className="px-3 py-2">
        <Link href="/dashboard" className="flex items-center space-x-2">
          <Package className="h-6 w-6" />
          <span className="font-bold text-xl">Zyenta</span>
        </Link>
      </div>

      {/* Nav items */}
      <div className="space-y-1">
        {navItems.map((item) => {
          const isActive = pathname === item.href;
          return (
            <Link
              key={item.href}
              href={item.href}
              className={cn(
                "flex items-center space-x-3 px-3 py-2 rounded-md text-sm font-medium transition-colors",
                isActive
                  ? "bg-primary text-primary-foreground"
                  : "text-muted-foreground hover:bg-muted hover:text-foreground"
              )}
            >
              <item.icon className="h-4 w-4" />
              <span>{item.title}</span>
            </Link>
          );
        })}
      </div>
    </nav>
  );
}
```

### 5.3 Header Component

**File: `apps/dashboard/src/components/dashboard/header.tsx`**
```typescript
"use client";

import { signOut } from "next-auth/react";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";

interface DashboardHeaderProps {
  user: {
    name?: string | null;
    email?: string | null;
    image?: string | null;
  };
}

export function DashboardHeader({ user }: DashboardHeaderProps) {
  const initials = user.name
    ?.split(" ")
    .map((n) => n[0])
    .join("")
    .toUpperCase() || "U";

  return (
    <header className="h-16 border-b bg-card px-6 flex items-center justify-between">
      <div>
        {/* Breadcrumbs or search could go here */}
      </div>

      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button variant="ghost" className="relative h-10 w-10 rounded-full">
            <Avatar>
              <AvatarImage src={user.image || ""} alt={user.name || ""} />
              <AvatarFallback>{initials}</AvatarFallback>
            </Avatar>
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end">
          <DropdownMenuLabel>
            <div className="flex flex-col space-y-1">
              <p className="text-sm font-medium">{user.name}</p>
              <p className="text-xs text-muted-foreground">{user.email}</p>
            </div>
          </DropdownMenuLabel>
          <DropdownMenuSeparator />
          <DropdownMenuItem asChild>
            <a href="/dashboard/settings">Settings</a>
          </DropdownMenuItem>
          <DropdownMenuItem
            className="text-destructive"
            onClick={() => signOut({ callbackUrl: "/login" })}
          >
            Sign out
          </DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>
    </header>
  );
}
```

### 5.4 Dashboard Overview Page

**File: `apps/dashboard/src/app/(dashboard)/dashboard/page.tsx`**
```typescript
import { Metadata } from "next";
import Link from "next/link";
import { Plus, Store, Activity, TrendingUp } from "lucide-react";

import { requireAuth } from "@/lib/auth-utils";
import { Button } from "@/components/ui/button";
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";

export const metadata: Metadata = {
  title: "Dashboard | Zyenta",
};

export default async function DashboardPage() {
  const user = await requireAuth();

  // TODO: Fetch actual stats from database
  const stats = {
    totalStores: 0,
    activeStores: 0,
    pendingJobs: 0,
    totalProducts: 0,
  };

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-3xl font-bold">Welcome back, {user.name}</h1>
          <p className="text-muted-foreground">
            Here&apos;s what&apos;s happening with your stores
          </p>
        </div>
        <Button asChild>
          <Link href="/dashboard/stores/new">
            <Plus className="mr-2 h-4 w-4" />
            Create Store
          </Link>
        </Button>
      </div>

      {/* Stats Grid */}
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
        <Card>
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="text-sm font-medium">Total Stores</CardTitle>
            <Store className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">{stats.totalStores}</div>
            <p className="text-xs text-muted-foreground">
              {stats.activeStores} active
            </p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="text-sm font-medium">Pending Jobs</CardTitle>
            <Activity className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">{stats.pendingJobs}</div>
            <p className="text-xs text-muted-foreground">Processing</p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="text-sm font-medium">Total Products</CardTitle>
            <TrendingUp className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">{stats.totalProducts}</div>
            <p className="text-xs text-muted-foreground">Across all stores</p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="text-sm font-medium">This Month</CardTitle>
            <TrendingUp className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">$0</div>
            <p className="text-xs text-muted-foreground">Total revenue</p>
          </CardContent>
        </Card>
      </div>

      {/* Quick Actions */}
      <Card>
        <CardHeader>
          <CardTitle>Quick Actions</CardTitle>
          <CardDescription>
            Get started with your first AI-powered store
          </CardDescription>
        </CardHeader>
        <CardContent className="grid gap-4 md:grid-cols-3">
          <Button variant="outline" className="h-24" asChild>
            <Link href="/dashboard/stores/new">
              <div className="text-center">
                <Plus className="h-6 w-6 mx-auto mb-2" />
                <span>Create New Store</span>
              </div>
            </Link>
          </Button>
          <Button variant="outline" className="h-24" asChild>
            <Link href="/dashboard/settings/integrations/stripe">
              <div className="text-center">
                <Store className="h-6 w-6 mx-auto mb-2" />
                <span>Connect Stripe</span>
              </div>
            </Link>
          </Button>
          <Button variant="outline" className="h-24" asChild>
            <Link href="/dashboard/settings/integrations/suppliers">
              <div className="text-center">
                <Activity className="h-6 w-6 mx-auto mb-2" />
                <span>Connect Suppliers</span>
              </div>
            </Link>
          </Button>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## Task 6: Store Creation Wizard

### 6.1 Wizard State Management

**File: `apps/dashboard/src/stores/wizard-store.ts`**
```typescript
import { create } from "zustand";

export interface WizardState {
  step: number;
  niche: string;
  preferences: {
    style?: string;
    tone?: string;
  };
  suppliers: string[];
  stripeConnected: boolean;

  setStep: (step: number) => void;
  setNiche: (niche: string) => void;
  setPreferences: (preferences: WizardState["preferences"]) => void;
  setSuppliers: (suppliers: string[]) => void;
  setStripeConnected: (connected: boolean) => void;
  reset: () => void;
}

const initialState = {
  step: 1,
  niche: "",
  preferences: {},
  suppliers: [],
  stripeConnected: false,
};

export const useWizardStore = create<WizardState>((set) => ({
  ...initialState,

  setStep: (step) => set({ step }),
  setNiche: (niche) => set({ niche }),
  setPreferences: (preferences) => set({ preferences }),
  setSuppliers: (suppliers) => set({ suppliers }),
  setStripeConnected: (stripeConnected) => set({ stripeConnected }),
  reset: () => set(initialState),
}));
```

### 6.2 Store Creation Page

**File: `apps/dashboard/src/app/(dashboard)/dashboard/stores/new/page.tsx`**
```typescript
import { Metadata } from "next";

import { requireAuth } from "@/lib/auth-utils";
import { StoreWizard } from "@/components/stores/wizard";

export const metadata: Metadata = {
  title: "Create Store | Zyenta",
};

export default async function NewStorePage() {
  await requireAuth();

  return (
    <div className="max-w-3xl mx-auto">
      <div className="mb-8">
        <h1 className="text-3xl font-bold">Create Your Store</h1>
        <p className="text-muted-foreground">
          Tell us what you want to sell and we&apos;ll build your store
        </p>
      </div>

      <StoreWizard />
    </div>
  );
}
```

### 6.3 Wizard Component

**File: `apps/dashboard/src/components/stores/wizard/index.tsx`**
```typescript
"use client";

import { useWizardStore } from "@/stores/wizard-store";
import { StepIndicator } from "./step-indicator";
import { NicheStep } from "./steps/niche-step";
import { SupplierStep } from "./steps/supplier-step";
import { PaymentStep } from "./steps/payment-step";
import { ReviewStep } from "./steps/review-step";
import { ProgressStep } from "./steps/progress-step";

const steps = [
  { id: 1, title: "Niche" },
  { id: 2, title: "Suppliers" },
  { id: 3, title: "Payment" },
  { id: 4, title: "Review" },
  { id: 5, title: "Creating" },
];

export function StoreWizard() {
  const { step } = useWizardStore();

  return (
    <div className="space-y-8">
      <StepIndicator steps={steps} currentStep={step} />

      <div className="bg-card rounded-lg border p-6">
        {step === 1 && <NicheStep />}
        {step === 2 && <SupplierStep />}
        {step === 3 && <PaymentStep />}
        {step === 4 && <ReviewStep />}
        {step === 5 && <ProgressStep />}
      </div>
    </div>
  );
}
```

### 6.4 Step Indicator

**File: `apps/dashboard/src/components/stores/wizard/step-indicator.tsx`**
```typescript
import { Check } from "lucide-react";
import { cn } from "@/lib/utils";

interface Step {
  id: number;
  title: string;
}

interface StepIndicatorProps {
  steps: Step[];
  currentStep: number;
}

export function StepIndicator({ steps, currentStep }: StepIndicatorProps) {
  return (
    <div className="flex items-center justify-between">
      {steps.map((step, index) => (
        <div key={step.id} className="flex items-center">
          <div className="flex flex-col items-center">
            <div
              className={cn(
                "w-10 h-10 rounded-full flex items-center justify-center border-2 transition-colors",
                step.id < currentStep
                  ? "bg-primary border-primary text-primary-foreground"
                  : step.id === currentStep
                  ? "border-primary text-primary"
                  : "border-muted text-muted-foreground"
              )}
            >
              {step.id < currentStep ? (
                <Check className="h-5 w-5" />
              ) : (
                <span>{step.id}</span>
              )}
            </div>
            <span
              className={cn(
                "mt-2 text-sm",
                step.id === currentStep
                  ? "text-foreground font-medium"
                  : "text-muted-foreground"
              )}
            >
              {step.title}
            </span>
          </div>
          {index < steps.length - 1 && (
            <div
              className={cn(
                "h-0.5 w-16 mx-2 transition-colors",
                step.id < currentStep ? "bg-primary" : "bg-muted"
              )}
            />
          )}
        </div>
      ))}
    </div>
  );
}
```

### 6.5 Niche Selection Step

**File: `apps/dashboard/src/components/stores/wizard/steps/niche-step.tsx`**
```typescript
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

import { useWizardStore } from "@/stores/wizard-store";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";

const nicheSchema = z.object({
  niche: z.string().min(3, "Please describe your niche in at least 3 characters"),
  style: z.string().optional(),
  tone: z.string().optional(),
});

type NicheFormData = z.infer<typeof nicheSchema>;

const nicheSuggestions = [
  "Cyberpunk home decor",
  "Minimalist Japanese office supplies",
  "Vintage-inspired kitchen gadgets",
  "Eco-friendly pet accessories",
  "Luxury skincare tools",
];

export function NicheStep() {
  const { niche, preferences, setNiche, setPreferences, setStep } = useWizardStore();

  const {
    register,
    handleSubmit,
    setValue,
    formState: { errors },
  } = useForm<NicheFormData>({
    resolver: zodResolver(nicheSchema),
    defaultValues: {
      niche,
      style: preferences.style,
      tone: preferences.tone,
    },
  });

  function onSubmit(data: NicheFormData) {
    setNiche(data.niche);
    setPreferences({
      style: data.style,
      tone: data.tone,
    });
    setStep(2);
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      <div className="space-y-2">
        <Label htmlFor="niche">What do you want to sell?</Label>
        <Textarea
          id="niche"
          placeholder="Describe your niche, e.g., 'Cyberpunk home decor'"
          className="min-h-[100px]"
          {...register("niche")}
        />
        {errors.niche && (
          <p className="text-sm text-destructive">{errors.niche.message}</p>
        )}
      </div>

      <div className="space-y-2">
        <Label className="text-muted-foreground">Or try one of these:</Label>
        <div className="flex flex-wrap gap-2">
          {nicheSuggestions.map((suggestion) => (
            <Button
              key={suggestion}
              type="button"
              variant="outline"
              size="sm"
              onClick={() => setValue("niche", suggestion)}
            >
              {suggestion}
            </Button>
          ))}
        </div>
      </div>

      <div className="grid gap-4 md:grid-cols-2">
        <div className="space-y-2">
          <Label htmlFor="style">Visual Style (optional)</Label>
          <Input
            id="style"
            placeholder="e.g., Modern, Vintage, Minimalist"
            {...register("style")}
          />
        </div>
        <div className="space-y-2">
          <Label htmlFor="tone">Brand Tone (optional)</Label>
          <Input
            id="tone"
            placeholder="e.g., Professional, Playful, Luxurious"
            {...register("tone")}
          />
        </div>
      </div>

      <div className="flex justify-end">
        <Button type="submit">Continue</Button>
      </div>
    </form>
  );
}
```

### 6.6 Review Step

**File: `apps/dashboard/src/components/stores/wizard/steps/review-step.tsx`**
```typescript
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";

import { useWizardStore } from "@/stores/wizard-store";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Icons } from "@/components/icons";

export function ReviewStep() {
  const router = useRouter();
  const { niche, preferences, suppliers, stripeConnected, setStep } = useWizardStore();
  const [isCreating, setIsCreating] = useState(false);

  async function handleCreate() {
    setIsCreating(true);

    try {
      const response = await fetch("/api/stores", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          niche,
          preferences,
          suppliers,
        }),
      });

      if (!response.ok) {
        throw new Error("Failed to create store");
      }

      const data = await response.json();

      // Move to progress step
      setStep(5);
    } catch (error) {
      console.error("Failed to create store:", error);
      setIsCreating(false);
    }
  }

  return (
    <div className="space-y-6">
      <div>
        <h2 className="text-xl font-semibold">Review Your Store</h2>
        <p className="text-muted-foreground">
          Make sure everything looks good before we start building
        </p>
      </div>

      <div className="grid gap-4">
        <Card>
          <CardHeader>
            <CardTitle className="text-base">Niche</CardTitle>
          </CardHeader>
          <CardContent>
            <p>{niche}</p>
            {preferences.style && (
              <p className="text-sm text-muted-foreground">
                Style: {preferences.style}
              </p>
            )}
            {preferences.tone && (
              <p className="text-sm text-muted-foreground">
                Tone: {preferences.tone}
              </p>
            )}
          </CardContent>
        </Card>

        <Card>
          <CardHeader>
            <CardTitle className="text-base">Suppliers</CardTitle>
          </CardHeader>
          <CardContent>
            {suppliers.length > 0 ? (
              <ul className="list-disc list-inside">
                {suppliers.map((s) => (
                  <li key={s}>{s}</li>
                ))}
              </ul>
            ) : (
              <p className="text-muted-foreground">
                No suppliers connected (using demo products)
              </p>
            )}
          </CardContent>
        </Card>

        <Card>
          <CardHeader>
            <CardTitle className="text-base">Payment</CardTitle>
          </CardHeader>
          <CardContent>
            {stripeConnected ? (
              <p className="text-green-600">Stripe connected</p>
            ) : (
              <p className="text-muted-foreground">
                Stripe not connected (can be set up later)
              </p>
            )}
          </CardContent>
        </Card>
      </div>

      <div className="bg-muted/50 rounded-lg p-4">
        <h3 className="font-medium mb-2">What happens next?</h3>
        <ul className="text-sm text-muted-foreground space-y-1">
          <li>1. AI generates your brand identity (~1 min)</li>
          <li>2. Products are scouted from suppliers (~2 min)</li>
          <li>3. Product descriptions are written (~3 min)</li>
          <li>4. Images are processed (~5 min)</li>
          <li>5. Your store goes live!</li>
        </ul>
        <p className="text-sm mt-2">
          <strong>Estimated time:</strong> ~10 minutes
        </p>
      </div>

      <div className="flex justify-between">
        <Button variant="outline" onClick={() => setStep(3)}>
          Back
        </Button>
        <Button onClick={handleCreate} disabled={isCreating}>
          {isCreating && (
            <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />
          )}
          Create My Store
        </Button>
      </div>
    </div>
  );
}
```

---

## Task 7: Job Progress Tracking

### 7.1 Progress Step Component

**File: `apps/dashboard/src/components/stores/wizard/steps/progress-step.tsx`**
```typescript
"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { Check, Loader2 } from "lucide-react";

import { Progress } from "@/components/ui/progress";
import { useWizardStore } from "@/stores/wizard-store";

interface JobStep {
  id: string;
  title: string;
  status: "pending" | "processing" | "completed" | "failed";
}

export function ProgressStep() {
  const router = useRouter();
  const { reset } = useWizardStore();
  const [progress, setProgress] = useState(0);
  const [steps, setSteps] = useState<JobStep[]>([
    { id: "brand", title: "Generating brand identity", status: "processing" },
    { id: "scout", title: "Scouting products", status: "pending" },
    { id: "copy", title: "Writing product descriptions", status: "pending" },
    { id: "images", title: "Processing images", status: "pending" },
    { id: "deploy", title: "Deploying store", status: "pending" },
  ]);

  // Simulate progress (in real app, poll job status from API)
  useEffect(() => {
    const interval = setInterval(() => {
      setProgress((prev) => {
        if (prev >= 100) {
          clearInterval(interval);
          return 100;
        }
        return prev + 2;
      });
    }, 500);

    return () => clearInterval(interval);
  }, []);

  // Update steps based on progress
  useEffect(() => {
    const thresholds = [20, 40, 60, 80, 100];

    setSteps((prevSteps) =>
      prevSteps.map((step, index) => {
        if (progress >= thresholds[index]!) {
          return { ...step, status: "completed" };
        } else if (progress >= (thresholds[index - 1] || 0)) {
          return { ...step, status: "processing" };
        }
        return { ...step, status: "pending" };
      })
    );

    if (progress >= 100) {
      setTimeout(() => {
        reset();
        router.push("/dashboard/stores");
      }, 1500);
    }
  }, [progress, router, reset]);

  return (
    <div className="space-y-8">
      <div className="text-center">
        <h2 className="text-xl font-semibold mb-2">Building Your Store</h2>
        <p className="text-muted-foreground">
          This usually takes about 10 minutes. Feel free to grab a coffee!
        </p>
      </div>

      <div className="space-y-2">
        <div className="flex justify-between text-sm">
          <span>Progress</span>
          <span>{progress}%</span>
        </div>
        <Progress value={progress} />
      </div>

      <div className="space-y-4">
        {steps.map((step) => (
          <div
            key={step.id}
            className="flex items-center space-x-3 p-3 rounded-lg bg-muted/50"
          >
            <div className="flex-shrink-0">
              {step.status === "completed" ? (
                <div className="w-6 h-6 rounded-full bg-green-500 flex items-center justify-center">
                  <Check className="h-4 w-4 text-white" />
                </div>
              ) : step.status === "processing" ? (
                <Loader2 className="h-6 w-6 animate-spin text-primary" />
              ) : (
                <div className="w-6 h-6 rounded-full border-2 border-muted-foreground/30" />
              )}
            </div>
            <span
              className={
                step.status === "completed"
                  ? "text-muted-foreground line-through"
                  : step.status === "processing"
                  ? "font-medium"
                  : "text-muted-foreground"
              }
            >
              {step.title}
            </span>
          </div>
        ))}
      </div>

      {progress >= 100 && (
        <div className="text-center text-green-600 font-medium">
          Store created successfully! Redirecting...
        </div>
      )}
    </div>
  );
}
```

---

## Task 8: UI Components

### 8.1 Icons Component

**File: `apps/dashboard/src/components/icons.tsx`**
```typescript
import {
  Loader2,
  LucideIcon,
  Moon,
  Sun,
  AlertCircle,
} from "lucide-react";

export type Icon = LucideIcon;

export const Icons = {
  spinner: Loader2,
  sun: Sun,
  moon: Moon,
  alert: AlertCircle,
  google: ({ ...props }) => (
    <svg role="img" viewBox="0 0 24 24" {...props}>
      <path
        fill="currentColor"
        d="M12.48 10.92v3.28h7.84c-.24 1.84-.853 3.187-1.787 4.133-1.147 1.147-2.933 2.4-6.053 2.4-4.827 0-8.6-3.893-8.6-8.72s3.773-8.72 8.6-8.72c2.6 0 4.507 1.027 5.907 2.347l2.307-2.307C18.747 1.44 16.133 0 12.48 0 5.867 0 .307 5.387.307 12s5.56 12 12.173 12c3.573 0 6.267-1.173 8.373-3.36 2.16-2.16 2.84-5.213 2.84-7.667 0-.76-.053-1.467-.173-2.053H12.48z"
      />
    </svg>
  ),
  gitHub: ({ ...props }) => (
    <svg viewBox="0 0 438.549 438.549" {...props}>
      <path
        fill="currentColor"
        d="M409.132 114.573c-19.608-33.596-46.205-60.194-79.798-79.8-33.598-19.607-70.277-29.408-110.063-29.408-39.781 0-76.472 9.804-110.063 29.408-33.596 19.605-60.192 46.204-79.8 79.8C9.803 148.168 0 184.854 0 224.63c0 47.78 13.94 90.745 41.827 128.906 27.884 38.164 63.906 64.572 108.063 79.227 5.14.954 8.945.283 11.419-1.996 2.475-2.282 3.711-5.14 3.711-8.562 0-.571-.049-5.708-.144-15.417a2549.81 2549.81 0 01-.144-25.406l-6.567 1.136c-4.187.767-9.469 1.092-15.846 1-6.374-.089-12.991-.757-19.842-1.999-6.854-1.231-13.229-4.086-19.13-8.559-5.898-4.473-10.085-10.328-12.56-17.556l-2.855-6.57c-1.903-4.374-4.899-9.233-8.992-14.559-4.093-5.331-8.232-8.945-12.419-10.848l-1.999-1.431c-1.332-.951-2.568-2.098-3.711-3.429-1.142-1.331-1.997-2.663-2.568-3.997-.572-1.335-.098-2.43 1.427-3.289 1.525-.859 4.281-1.276 8.28-1.276l5.708.853c3.807.763 8.516 3.042 14.133 6.851 5.614 3.806 10.229 8.754 13.846 14.842 4.38 7.806 9.657 13.754 15.846 17.847 6.184 4.093 12.419 6.136 18.699 6.136 6.28 0 11.704-.476 16.274-1.423 4.565-.952 8.848-2.383 12.847-4.285 1.713-12.758 6.377-22.559 13.988-29.41-10.848-1.14-20.601-2.857-29.264-5.14-8.658-2.286-17.605-5.996-26.835-11.14-9.235-5.137-16.896-11.516-22.985-19.126-6.09-7.614-11.088-17.61-14.987-29.979-3.901-12.374-5.852-26.648-5.852-42.826 0-23.035 7.52-42.637 22.557-58.817-7.044-17.318-6.379-36.732 1.997-58.24 5.52-1.715 13.706-.428 24.554 3.853 10.85 4.283 18.794 7.952 23.84 10.994 5.046 3.041 9.089 5.618 12.135 7.708 17.705-4.947 35.976-7.421 54.818-7.421s37.117 2.474 54.823 7.421l10.849-6.849c7.419-4.57 16.18-8.758 26.262-12.565 10.088-3.805 17.802-4.853 23.134-3.138 8.562 21.509 9.325 40.922 2.279 58.24 15.036 16.18 22.559 35.787 22.559 58.817 0 16.178-1.958 30.497-5.853 42.966-3.9 12.471-8.941 22.457-15.125 29.979-6.191 7.521-13.901 13.85-23.131 18.986-9.232 5.14-18.182 8.85-26.84 11.136-8.662 2.286-18.415 4.004-29.263 5.146 9.894 8.562 14.842 22.077 14.842 40.539v60.237c0 3.422 1.19 6.279 3.572 8.562 2.379 2.279 6.136 2.95 11.276 1.995 44.163-14.653 80.185-41.062 108.068-79.226 27.88-38.161 41.825-81.126 41.825-128.906-.01-39.771-9.818-76.454-29.414-110.049z"
      />
    </svg>
  ),
};
```

### 8.2 Basic UI Components

Create the following Shadcn UI components by running:

```bash
cd apps/dashboard
npx shadcn-ui@latest init
npx shadcn-ui@latest add button input label card avatar dropdown-menu dialog progress tabs toast separator select alert-dialog
```

Or create them manually following the Shadcn UI documentation.

---

## Verification Checklist

### Setup
- [ ] Next.js app starts: `pnpm dev`
- [ ] Tailwind styles working
- [ ] TypeScript compiling without errors

### Authentication
- [ ] Login page renders
- [ ] Registration works
- [ ] OAuth providers work (if configured)
- [ ] Protected routes redirect to login

### Dashboard
- [ ] Layout renders with sidebar and header
- [ ] Navigation works
- [ ] User dropdown shows profile info
- [ ] Sign out works

### Store Wizard
- [ ] All steps render correctly
- [ ] Form validation works
- [ ] Progress tracking simulates correctly
- [ ] Redirects to stores list on completion

---

## Next Steps

After completing Sprint 4, proceed to:

**Sprint 5: Storefront & Payments**
- Storefront Next.js application
- Multi-tenant routing
- Product pages and cart
- Stripe Connect integration

Refer to `PHASE1_EXECUTION_PLAN.md` for detailed specifications.
