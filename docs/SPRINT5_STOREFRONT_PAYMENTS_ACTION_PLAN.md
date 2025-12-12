# Sprint 5: Storefront & Payments - Action Plan

## Overview

This document provides a comprehensive implementation guide for Sprint 5 of Phase 1, focusing on the customer-facing storefront application with multi-tenant architecture, shopping cart, checkout flow, and Stripe Connect payment integration.

### Sprint Objectives
- [ ] Setup Next.js 14 storefront application
- [ ] Implement multi-tenant routing (subdomain + custom domain)
- [ ] Build product listing and detail pages
- [ ] Implement shopping cart functionality
- [ ] Integrate Stripe Connect for checkout
- [ ] Setup custom domain handling

### Prerequisites
- Sprint 1 (Foundation) completed
- Sprint 4 (Dashboard MVP) completed
- Stripe Connect account configured
- Database schema deployed

---

## 1. Storefront Project Setup

### 1.1 Initialize Next.js Project

```bash
# From monorepo root
cd apps
pnpm create next-app@latest storefront --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

### 1.2 Package Configuration

```json
// apps/storefront/package.json
{
  "name": "@zyenta/storefront",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3001",
    "build": "next build",
    "start": "next start -p 3001",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^14.0.4",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@zyenta/database": "workspace:*",
    "@zyenta/shared-types": "workspace:*",
    "@zyenta/ui": "workspace:*",
    "@stripe/stripe-js": "^2.2.0",
    "stripe": "^14.10.0",
    "zustand": "^4.4.7",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.2.0",
    "lucide-react": "^0.302.0",
    "framer-motion": "^10.17.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.5",
    "@types/react": "^18.2.45",
    "@types/react-dom": "^18.2.18",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.3.3"
  }
}
```

### 1.3 Next.js Configuration

```javascript
// apps/storefront/next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['@zyenta/ui', '@zyenta/shared-types'],

  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.zyenta.com',
      },
      {
        protocol: 'https',
        hostname: 'cdn.zyenta.com',
      },
      {
        protocol: 'https',
        hostname: '**.amazonaws.com',
      },
    ],
  },

  // Enable experimental features for multi-tenant
  experimental: {
    serverActions: true,
  },

  // Rewrite for custom domains
  async rewrites() {
    return {
      beforeFiles: [
        // Handle custom domains
        {
          source: '/:path*',
          has: [
            {
              type: 'host',
              value: '(?<customDomain>(?!.*\\.zyenta\\.com).+)',
            },
          ],
          destination: '/_tenant/:customDomain/:path*',
        },
      ],
    };
  },
};

module.exports = nextConfig;
```

### 1.4 Tailwind Configuration

```typescript
// apps/storefront/tailwind.config.ts
import type { Config } from 'tailwindcss';
import sharedConfig from '@zyenta/ui/tailwind.config';

const config: Config = {
  presets: [sharedConfig],
  content: [
    './src/**/*.{js,ts,jsx,tsx,mdx}',
    '../../packages/ui/src/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        // Dynamic brand colors (CSS variables)
        brand: {
          primary: 'var(--brand-primary)',
          secondary: 'var(--brand-secondary)',
          accent: 'var(--brand-accent)',
          background: 'var(--brand-background)',
          foreground: 'var(--brand-foreground)',
          muted: 'var(--brand-muted)',
        },
      },
      fontFamily: {
        heading: 'var(--font-heading)',
        body: 'var(--font-body)',
      },
    },
  },
  plugins: [],
};

export default config;
```

---

## 2. Multi-Tenant Architecture

### 2.1 Tenant Resolution Middleware

```typescript
// apps/storefront/src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  const host = request.headers.get('host') || '';
  const url = request.nextUrl.clone();

  // Extract tenant from subdomain or custom domain
  let storeSlug: string | null = null;

  // Check if it's a subdomain (e.g., mystore.zyenta.com)
  const subdomainMatch = host.match(/^([^.]+)\.zyenta\.com$/);
  if (subdomainMatch) {
    storeSlug = subdomainMatch[1];
  }

  // Check if it's a custom domain
  if (!storeSlug && !host.includes('zyenta.com')) {
    // Lookup custom domain in edge config or database
    const customDomainStore = await resolveCustomDomain(host);
    if (customDomainStore) {
      storeSlug = customDomainStore;
    }
  }

  // Skip middleware for main domain
  if (!storeSlug || storeSlug === 'www' || storeSlug === 'app') {
    return NextResponse.next();
  }

  // Add store slug to headers for downstream use
  const response = NextResponse.next();
  response.headers.set('x-store-slug', storeSlug);
  response.headers.set('x-store-host', host);

  return response;
}

async function resolveCustomDomain(domain: string): Promise<string | null> {
  // In production, this would query a database or edge config
  // For now, we'll use an API route
  try {
    const res = await fetch(
      `${process.env.INTERNAL_API_URL}/api/domains/resolve?domain=${domain}`,
      { next: { revalidate: 60 } }
    );
    if (res.ok) {
      const data = await res.json();
      return data.storeSlug;
    }
  } catch {
    console.error('Failed to resolve custom domain:', domain);
  }
  return null;
}

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### 2.2 Store Context Provider

```typescript
// apps/storefront/src/lib/store-context.tsx
'use client';

import { createContext, useContext, ReactNode } from 'react';

export interface StoreData {
  id: string;
  name: string;
  slug: string;
  description: string | null;
  brandIdentity: {
    logo: string;
    colors: {
      primary: string;
      secondary: string;
      accent: string;
      background: string;
      foreground: string;
      muted: string;
    };
    fonts: {
      heading: string;
      body: string;
    };
    tone: string;
  };
  settings: {
    currency: string;
    socialLinks?: {
      instagram?: string;
      facebook?: string;
      twitter?: string;
    };
  };
  stripeAccountId: string;
}

interface StoreContextType {
  store: StoreData;
}

const StoreContext = createContext<StoreContextType | null>(null);

export function StoreProvider({
  children,
  store,
}: {
  children: ReactNode;
  store: StoreData;
}) {
  return (
    <StoreContext.Provider value={{ store }}>
      {children}
    </StoreContext.Provider>
  );
}

export function useStore() {
  const context = useContext(StoreContext);
  if (!context) {
    throw new Error('useStore must be used within a StoreProvider');
  }
  return context;
}
```

### 2.3 Store Data Fetching

```typescript
// apps/storefront/src/lib/get-store.ts
import { prisma } from '@zyenta/database';
import { headers } from 'next/headers';
import { cache } from 'react';
import type { StoreData } from './store-context';

export const getStore = cache(async (): Promise<StoreData | null> => {
  const headersList = headers();
  const storeSlug = headersList.get('x-store-slug');

  if (!storeSlug) {
    return null;
  }

  const store = await prisma.store.findUnique({
    where: { slug: storeSlug, status: 'ACTIVE' },
    include: {
      user: {
        select: {
          stripeAccountId: true,
        },
      },
    },
  });

  if (!store) {
    return null;
  }

  return {
    id: store.id,
    name: store.name,
    slug: store.slug,
    description: store.description,
    brandIdentity: store.brandIdentity as StoreData['brandIdentity'],
    settings: store.settings as StoreData['settings'],
    stripeAccountId: store.user.stripeAccountId!,
  };
});
```

---

## 3. Application Structure

### 3.1 Root Layout with Dynamic Theming

```typescript
// apps/storefront/src/app/layout.tsx
import { Inter, Playfair_Display } from 'next/font/google';
import { getStore } from '@/lib/get-store';
import { StoreProvider } from '@/lib/store-context';
import { CartProvider } from '@/lib/cart';
import { notFound } from 'next/navigation';
import './globals.css';

const inter = Inter({ subsets: ['latin'], variable: '--font-inter' });
const playfair = Playfair_Display({ subsets: ['latin'], variable: '--font-playfair' });

export async function generateMetadata() {
  const store = await getStore();
  if (!store) return { title: 'Store Not Found' };

  return {
    title: {
      default: store.name,
      template: `%s | ${store.name}`,
    },
    description: store.description,
    openGraph: {
      title: store.name,
      description: store.description || undefined,
      siteName: store.name,
      type: 'website',
    },
  };
}

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const store = await getStore();

  if (!store) {
    notFound();
  }

  // Generate CSS variables from brand identity
  const brandStyles = `
    :root {
      --brand-primary: ${store.brandIdentity.colors.primary};
      --brand-secondary: ${store.brandIdentity.colors.secondary};
      --brand-accent: ${store.brandIdentity.colors.accent};
      --brand-background: ${store.brandIdentity.colors.background};
      --brand-foreground: ${store.brandIdentity.colors.foreground};
      --brand-muted: ${store.brandIdentity.colors.muted};
      --font-heading: ${store.brandIdentity.fonts.heading}, var(--font-playfair);
      --font-body: ${store.brandIdentity.fonts.body}, var(--font-inter);
    }
  `;

  return (
    <html lang="en" className={`${inter.variable} ${playfair.variable}`}>
      <head>
        <style dangerouslySetInnerHTML={{ __html: brandStyles }} />
      </head>
      <body className="bg-brand-background text-brand-foreground font-body">
        <StoreProvider store={store}>
          <CartProvider>
            {children}
          </CartProvider>
        </StoreProvider>
      </body>
    </html>
  );
}
```

### 3.2 Main Layout Components

```typescript
// apps/storefront/src/components/layout/header.tsx
'use client';

import Link from 'next/link';
import Image from 'next/image';
import { ShoppingCart, Menu, X, Search } from 'lucide-react';
import { useState } from 'react';
import { useStore } from '@/lib/store-context';
import { useCart } from '@/lib/cart';
import { CartDrawer } from '@/components/cart/cart-drawer';

export function Header() {
  const { store } = useStore();
  const { items } = useCart();
  const [mobileMenuOpen, setMobileMenuOpen] = useState(false);
  const [cartOpen, setCartOpen] = useState(false);

  const itemCount = items.reduce((sum, item) => sum + item.quantity, 0);

  return (
    <>
      <header className="sticky top-0 z-40 bg-brand-background border-b border-brand-muted">
        <div className="container mx-auto px-4">
          <div className="flex items-center justify-between h-16">
            {/* Logo */}
            <Link href="/" className="flex items-center gap-2">
              {store.brandIdentity.logo ? (
                <Image
                  src={store.brandIdentity.logo}
                  alt={store.name}
                  width={120}
                  height={40}
                  className="h-10 w-auto"
                />
              ) : (
                <span className="text-xl font-heading font-bold text-brand-primary">
                  {store.name}
                </span>
              )}
            </Link>

            {/* Desktop Navigation */}
            <nav className="hidden md:flex items-center gap-8">
              <Link
                href="/products"
                className="text-brand-foreground hover:text-brand-primary transition-colors"
              >
                Shop All
              </Link>
              <Link
                href="/pages/about"
                className="text-brand-foreground hover:text-brand-primary transition-colors"
              >
                About
              </Link>
              <Link
                href="/pages/contact"
                className="text-brand-foreground hover:text-brand-primary transition-colors"
              >
                Contact
              </Link>
            </nav>

            {/* Actions */}
            <div className="flex items-center gap-4">
              <button
                className="p-2 hover:bg-brand-muted rounded-full transition-colors"
                aria-label="Search"
              >
                <Search className="w-5 h-5" />
              </button>

              <button
                onClick={() => setCartOpen(true)}
                className="relative p-2 hover:bg-brand-muted rounded-full transition-colors"
                aria-label="Cart"
              >
                <ShoppingCart className="w-5 h-5" />
                {itemCount > 0 && (
                  <span className="absolute -top-1 -right-1 bg-brand-primary text-white text-xs w-5 h-5 rounded-full flex items-center justify-center">
                    {itemCount}
                  </span>
                )}
              </button>

              <button
                onClick={() => setMobileMenuOpen(!mobileMenuOpen)}
                className="md:hidden p-2 hover:bg-brand-muted rounded-full transition-colors"
                aria-label="Menu"
              >
                {mobileMenuOpen ? (
                  <X className="w-5 h-5" />
                ) : (
                  <Menu className="w-5 h-5" />
                )}
              </button>
            </div>
          </div>
        </div>

        {/* Mobile Menu */}
        {mobileMenuOpen && (
          <nav className="md:hidden border-t border-brand-muted py-4">
            <div className="container mx-auto px-4 space-y-4">
              <Link
                href="/products"
                className="block text-brand-foreground hover:text-brand-primary"
                onClick={() => setMobileMenuOpen(false)}
              >
                Shop All
              </Link>
              <Link
                href="/pages/about"
                className="block text-brand-foreground hover:text-brand-primary"
                onClick={() => setMobileMenuOpen(false)}
              >
                About
              </Link>
              <Link
                href="/pages/contact"
                className="block text-brand-foreground hover:text-brand-primary"
                onClick={() => setMobileMenuOpen(false)}
              >
                Contact
              </Link>
            </div>
          </nav>
        )}
      </header>

      <CartDrawer open={cartOpen} onClose={() => setCartOpen(false)} />
    </>
  );
}
```

```typescript
// apps/storefront/src/components/layout/footer.tsx
'use client';

import Link from 'next/link';
import { Instagram, Facebook, Twitter } from 'lucide-react';
import { useStore } from '@/lib/store-context';

export function Footer() {
  const { store } = useStore();
  const socialLinks = store.settings.socialLinks || {};

  return (
    <footer className="bg-brand-muted mt-auto">
      <div className="container mx-auto px-4 py-12">
        <div className="grid grid-cols-1 md:grid-cols-4 gap-8">
          {/* Brand */}
          <div className="space-y-4">
            <h3 className="font-heading text-xl font-bold text-brand-primary">
              {store.name}
            </h3>
            <p className="text-sm text-brand-foreground/80">
              {store.description}
            </p>
          </div>

          {/* Shop Links */}
          <div className="space-y-4">
            <h4 className="font-semibold">Shop</h4>
            <ul className="space-y-2 text-sm">
              <li>
                <Link href="/products" className="hover:text-brand-primary">
                  All Products
                </Link>
              </li>
              <li>
                <Link href="/products?sort=newest" className="hover:text-brand-primary">
                  New Arrivals
                </Link>
              </li>
              <li>
                <Link href="/products?sort=popular" className="hover:text-brand-primary">
                  Best Sellers
                </Link>
              </li>
            </ul>
          </div>

          {/* Support Links */}
          <div className="space-y-4">
            <h4 className="font-semibold">Support</h4>
            <ul className="space-y-2 text-sm">
              <li>
                <Link href="/pages/contact" className="hover:text-brand-primary">
                  Contact Us
                </Link>
              </li>
              <li>
                <Link href="/pages/faq" className="hover:text-brand-primary">
                  FAQ
                </Link>
              </li>
              <li>
                <Link href="/pages/refund-policy" className="hover:text-brand-primary">
                  Returns & Refunds
                </Link>
              </li>
            </ul>
          </div>

          {/* Legal Links */}
          <div className="space-y-4">
            <h4 className="font-semibold">Legal</h4>
            <ul className="space-y-2 text-sm">
              <li>
                <Link href="/pages/privacy-policy" className="hover:text-brand-primary">
                  Privacy Policy
                </Link>
              </li>
              <li>
                <Link href="/pages/terms-of-service" className="hover:text-brand-primary">
                  Terms of Service
                </Link>
              </li>
            </ul>
          </div>
        </div>

        {/* Bottom Bar */}
        <div className="border-t border-brand-foreground/20 mt-8 pt-8 flex flex-col md:flex-row justify-between items-center gap-4">
          <p className="text-sm text-brand-foreground/60">
            &copy; {new Date().getFullYear()} {store.name}. All rights reserved.
          </p>

          {/* Social Links */}
          <div className="flex items-center gap-4">
            {socialLinks.instagram && (
              <a
                href={socialLinks.instagram}
                target="_blank"
                rel="noopener noreferrer"
                className="hover:text-brand-primary"
              >
                <Instagram className="w-5 h-5" />
              </a>
            )}
            {socialLinks.facebook && (
              <a
                href={socialLinks.facebook}
                target="_blank"
                rel="noopener noreferrer"
                className="hover:text-brand-primary"
              >
                <Facebook className="w-5 h-5" />
              </a>
            )}
            {socialLinks.twitter && (
              <a
                href={socialLinks.twitter}
                target="_blank"
                rel="noopener noreferrer"
                className="hover:text-brand-primary"
              >
                <Twitter className="w-5 h-5" />
              </a>
            )}
          </div>
        </div>
      </div>
    </footer>
  );
}
```

---

## 4. Product Pages

### 4.1 Product Listing Page

```typescript
// apps/storefront/src/app/products/page.tsx
import { prisma } from '@zyenta/database';
import { getStore } from '@/lib/get-store';
import { ProductCard } from '@/components/product/product-card';
import { ProductFilters } from '@/components/product/product-filters';
import { Suspense } from 'react';

interface ProductsPageProps {
  searchParams: {
    sort?: string;
    search?: string;
    page?: string;
  };
}

export const metadata = {
  title: 'Products',
};

async function getProducts(storeId: string, searchParams: ProductsPageProps['searchParams']) {
  const { sort = 'newest', search, page = '1' } = searchParams;
  const pageSize = 12;
  const skip = (parseInt(page) - 1) * pageSize;

  const orderBy = {
    newest: { createdAt: 'desc' as const },
    oldest: { createdAt: 'asc' as const },
    'price-low': { sellingPrice: 'asc' as const },
    'price-high': { sellingPrice: 'desc' as const },
    popular: { createdAt: 'desc' as const }, // Would use order count in production
  }[sort] || { createdAt: 'desc' as const };

  const where = {
    storeId,
    status: 'ACTIVE' as const,
    ...(search && {
      OR: [
        { title: { contains: search, mode: 'insensitive' as const } },
        { description: { contains: search, mode: 'insensitive' as const } },
      ],
    }),
  };

  const [products, total] = await Promise.all([
    prisma.product.findMany({
      where,
      orderBy,
      skip,
      take: pageSize,
      include: {
        images: {
          orderBy: { position: 'asc' },
          take: 2,
        },
      },
    }),
    prisma.product.count({ where }),
  ]);

  return {
    products,
    pagination: {
      page: parseInt(page),
      pageSize,
      total,
      totalPages: Math.ceil(total / pageSize),
    },
  };
}

export default async function ProductsPage({ searchParams }: ProductsPageProps) {
  const store = await getStore();
  if (!store) return null;

  const { products, pagination } = await getProducts(store.id, searchParams);

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex flex-col md:flex-row gap-8">
        {/* Filters Sidebar */}
        <aside className="w-full md:w-64 shrink-0">
          <Suspense fallback={<div>Loading filters...</div>}>
            <ProductFilters />
          </Suspense>
        </aside>

        {/* Product Grid */}
        <main className="flex-1">
          <div className="flex justify-between items-center mb-6">
            <p className="text-sm text-brand-foreground/70">
              {pagination.total} products
            </p>
            <select
              defaultValue={searchParams.sort || 'newest'}
              className="border border-brand-muted rounded-md px-3 py-2 text-sm bg-brand-background"
              onChange={(e) => {
                const url = new URL(window.location.href);
                url.searchParams.set('sort', e.target.value);
                window.location.href = url.toString();
              }}
            >
              <option value="newest">Newest</option>
              <option value="oldest">Oldest</option>
              <option value="price-low">Price: Low to High</option>
              <option value="price-high">Price: High to Low</option>
              <option value="popular">Popular</option>
            </select>
          </div>

          {products.length > 0 ? (
            <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
              {products.map((product) => (
                <ProductCard key={product.id} product={product} />
              ))}
            </div>
          ) : (
            <div className="text-center py-12">
              <p className="text-brand-foreground/70">No products found</p>
            </div>
          )}

          {/* Pagination */}
          {pagination.totalPages > 1 && (
            <div className="flex justify-center gap-2 mt-8">
              {Array.from({ length: pagination.totalPages }, (_, i) => (
                <a
                  key={i + 1}
                  href={`?page=${i + 1}&sort=${searchParams.sort || 'newest'}`}
                  className={`px-4 py-2 rounded-md ${
                    pagination.page === i + 1
                      ? 'bg-brand-primary text-white'
                      : 'bg-brand-muted hover:bg-brand-primary/10'
                  }`}
                >
                  {i + 1}
                </a>
              ))}
            </div>
          )}
        </main>
      </div>
    </div>
  );
}
```

### 4.2 Product Card Component

```typescript
// apps/storefront/src/components/product/product-card.tsx
'use client';

import Image from 'next/image';
import Link from 'next/link';
import { useState } from 'react';
import { formatPrice } from '@/lib/utils';
import type { Product, ProductImage } from '@prisma/client';

interface ProductCardProps {
  product: Product & {
    images: ProductImage[];
  };
}

export function ProductCard({ product }: ProductCardProps) {
  const [isHovered, setIsHovered] = useState(false);
  const primaryImage = product.images[0];
  const hoverImage = product.images[1] || primaryImage;

  const hasDiscount = product.compareAtPrice &&
    product.compareAtPrice > product.sellingPrice;

  return (
    <Link
      href={`/products/${product.slug}`}
      className="group block"
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    >
      <div className="relative aspect-square overflow-hidden rounded-lg bg-brand-muted mb-3">
        {/* Primary Image */}
        <Image
          src={primaryImage?.processedUrl || primaryImage?.originalUrl || '/placeholder.jpg'}
          alt={product.title}
          fill
          className={`object-cover transition-opacity duration-300 ${
            isHovered && hoverImage !== primaryImage ? 'opacity-0' : 'opacity-100'
          }`}
          sizes="(max-width: 768px) 50vw, (max-width: 1200px) 33vw, 25vw"
        />

        {/* Hover Image */}
        {hoverImage !== primaryImage && (
          <Image
            src={hoverImage?.processedUrl || hoverImage?.originalUrl || '/placeholder.jpg'}
            alt={product.title}
            fill
            className={`object-cover transition-opacity duration-300 ${
              isHovered ? 'opacity-100' : 'opacity-0'
            }`}
            sizes="(max-width: 768px) 50vw, (max-width: 1200px) 33vw, 25vw"
          />
        )}

        {/* Discount Badge */}
        {hasDiscount && (
          <span className="absolute top-2 left-2 bg-red-500 text-white text-xs px-2 py-1 rounded">
            Sale
          </span>
        )}

        {/* Out of Stock Overlay */}
        {product.stockQuantity === 0 && (
          <div className="absolute inset-0 bg-brand-background/80 flex items-center justify-center">
            <span className="text-brand-foreground font-medium">Out of Stock</span>
          </div>
        )}
      </div>

      <h3 className="font-medium text-brand-foreground group-hover:text-brand-primary transition-colors line-clamp-2">
        {product.title}
      </h3>

      <div className="mt-1 flex items-center gap-2">
        <span className="font-semibold text-brand-primary">
          {formatPrice(Number(product.sellingPrice), product.currency)}
        </span>
        {hasDiscount && (
          <span className="text-sm text-brand-foreground/50 line-through">
            {formatPrice(Number(product.compareAtPrice), product.currency)}
          </span>
        )}
      </div>
    </Link>
  );
}
```

### 4.3 Product Detail Page

```typescript
// apps/storefront/src/app/products/[slug]/page.tsx
import { prisma } from '@zyenta/database';
import { getStore } from '@/lib/get-store';
import { notFound } from 'next/navigation';
import { ProductGallery } from '@/components/product/product-gallery';
import { ProductInfo } from '@/components/product/product-info';
import { RelatedProducts } from '@/components/product/related-products';
import type { Metadata } from 'next';

interface ProductPageProps {
  params: { slug: string };
}

async function getProduct(storeId: string, slug: string) {
  return prisma.product.findFirst({
    where: {
      storeId,
      slug,
      status: 'ACTIVE',
    },
    include: {
      images: {
        orderBy: { position: 'asc' },
      },
      variants: true,
    },
  });
}

export async function generateMetadata({
  params,
}: ProductPageProps): Promise<Metadata> {
  const store = await getStore();
  if (!store) return {};

  const product = await getProduct(store.id, params.slug);
  if (!product) return {};

  return {
    title: product.metaTitle || product.title,
    description: product.metaDescription || product.description?.slice(0, 160),
    openGraph: {
      title: product.title,
      description: product.description || undefined,
      images: product.images[0]?.processedUrl
        ? [{ url: product.images[0].processedUrl }]
        : undefined,
    },
  };
}

export default async function ProductPage({ params }: ProductPageProps) {
  const store = await getStore();
  if (!store) return null;

  const product = await getProduct(store.id, params.slug);
  if (!product) {
    notFound();
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-12">
        {/* Product Gallery */}
        <ProductGallery images={product.images} />

        {/* Product Info */}
        <ProductInfo product={product} />
      </div>

      {/* Product Description */}
      <div className="mt-12">
        <h2 className="font-heading text-2xl font-bold mb-4">Description</h2>
        <div
          className="prose prose-brand max-w-none"
          dangerouslySetInnerHTML={{ __html: product.description }}
        />
      </div>

      {/* Related Products */}
      <RelatedProducts
        storeId={store.id}
        currentProductId={product.id}
      />
    </div>
  );
}
```

### 4.4 Product Gallery Component

```typescript
// apps/storefront/src/components/product/product-gallery.tsx
'use client';

import Image from 'next/image';
import { useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { ChevronLeft, ChevronRight, ZoomIn } from 'lucide-react';
import type { ProductImage } from '@prisma/client';

interface ProductGalleryProps {
  images: ProductImage[];
}

export function ProductGallery({ images }: ProductGalleryProps) {
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [isZoomed, setIsZoomed] = useState(false);

  const selectedImage = images[selectedIndex];

  const goToPrevious = () => {
    setSelectedIndex((prev) => (prev === 0 ? images.length - 1 : prev - 1));
  };

  const goToNext = () => {
    setSelectedIndex((prev) => (prev === images.length - 1 ? 0 : prev + 1));
  };

  return (
    <div className="space-y-4">
      {/* Main Image */}
      <div className="relative aspect-square bg-brand-muted rounded-lg overflow-hidden">
        <AnimatePresence mode="wait">
          <motion.div
            key={selectedIndex}
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            transition={{ duration: 0.2 }}
            className="relative w-full h-full"
          >
            <Image
              src={selectedImage?.processedUrl || selectedImage?.originalUrl || '/placeholder.jpg'}
              alt={selectedImage?.altText || 'Product image'}
              fill
              className={`object-cover cursor-zoom-in ${
                isZoomed ? 'scale-150' : ''
              } transition-transform duration-300`}
              onClick={() => setIsZoomed(!isZoomed)}
              sizes="(max-width: 768px) 100vw, 50vw"
              priority
            />
          </motion.div>
        </AnimatePresence>

        {/* Navigation Arrows */}
        {images.length > 1 && (
          <>
            <button
              onClick={goToPrevious}
              className="absolute left-2 top-1/2 -translate-y-1/2 p-2 bg-brand-background/80 rounded-full hover:bg-brand-background transition-colors"
              aria-label="Previous image"
            >
              <ChevronLeft className="w-5 h-5" />
            </button>
            <button
              onClick={goToNext}
              className="absolute right-2 top-1/2 -translate-y-1/2 p-2 bg-brand-background/80 rounded-full hover:bg-brand-background transition-colors"
              aria-label="Next image"
            >
              <ChevronRight className="w-5 h-5" />
            </button>
          </>
        )}

        {/* Zoom Indicator */}
        <div className="absolute bottom-2 right-2 p-2 bg-brand-background/80 rounded-full">
          <ZoomIn className="w-4 h-4" />
        </div>
      </div>

      {/* Thumbnail Grid */}
      {images.length > 1 && (
        <div className="grid grid-cols-6 gap-2">
          {images.map((image, index) => (
            <button
              key={image.id}
              onClick={() => setSelectedIndex(index)}
              className={`relative aspect-square rounded-md overflow-hidden border-2 transition-colors ${
                index === selectedIndex
                  ? 'border-brand-primary'
                  : 'border-transparent hover:border-brand-muted'
              }`}
            >
              <Image
                src={image.processedUrl || image.originalUrl}
                alt={image.altText || `Product image ${index + 1}`}
                fill
                className="object-cover"
                sizes="80px"
              />
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

### 4.5 Product Info Component

```typescript
// apps/storefront/src/components/product/product-info.tsx
'use client';

import { useState } from 'react';
import { ShoppingCart, Minus, Plus, Check } from 'lucide-react';
import { useCart } from '@/lib/cart';
import { formatPrice } from '@/lib/utils';
import type { Product, ProductVariant } from '@prisma/client';

interface ProductInfoProps {
  product: Product & {
    variants: ProductVariant[];
  };
}

export function ProductInfo({ product }: ProductInfoProps) {
  const { addItem } = useCart();
  const [quantity, setQuantity] = useState(1);
  const [selectedVariant, setSelectedVariant] = useState<ProductVariant | null>(
    product.variants[0] || null
  );
  const [isAdded, setIsAdded] = useState(false);

  const price = selectedVariant?.price || product.sellingPrice;
  const compareAtPrice = product.compareAtPrice;
  const hasDiscount = compareAtPrice && Number(compareAtPrice) > Number(price);
  const inStock = (selectedVariant?.stock ?? product.stockQuantity) > 0;

  // Extract unique options for variant selection
  const variantOptions = product.variants.reduce((acc, variant) => {
    const options = variant.options as Record<string, string>;
    Object.entries(options).forEach(([key, value]) => {
      if (!acc[key]) acc[key] = new Set();
      acc[key].add(value);
    });
    return acc;
  }, {} as Record<string, Set<string>>);

  const handleAddToCart = () => {
    addItem({
      id: selectedVariant?.id || product.id,
      productId: product.id,
      variantId: selectedVariant?.id,
      title: product.title,
      variantTitle: selectedVariant?.title,
      price: Number(price),
      image: product.images?.[0]?.processedUrl || product.images?.[0]?.originalUrl,
      quantity,
    });

    setIsAdded(true);
    setTimeout(() => setIsAdded(false), 2000);
  };

  return (
    <div className="space-y-6">
      {/* Title */}
      <h1 className="font-heading text-3xl font-bold text-brand-foreground">
        {product.title}
      </h1>

      {/* Price */}
      <div className="flex items-center gap-3">
        <span className="text-2xl font-semibold text-brand-primary">
          {formatPrice(Number(price), product.currency)}
        </span>
        {hasDiscount && (
          <>
            <span className="text-lg text-brand-foreground/50 line-through">
              {formatPrice(Number(compareAtPrice), product.currency)}
            </span>
            <span className="bg-red-500 text-white text-sm px-2 py-1 rounded">
              {Math.round((1 - Number(price) / Number(compareAtPrice)) * 100)}% OFF
            </span>
          </>
        )}
      </div>

      {/* Variant Options */}
      {Object.entries(variantOptions).map(([optionName, values]) => (
        <div key={optionName} className="space-y-2">
          <label className="font-medium">{optionName}</label>
          <div className="flex flex-wrap gap-2">
            {Array.from(values).map((value) => {
              const isSelected = selectedVariant?.options?.[optionName] === value;
              return (
                <button
                  key={value}
                  onClick={() => {
                    const variant = product.variants.find(
                      (v) => (v.options as Record<string, string>)[optionName] === value
                    );
                    if (variant) setSelectedVariant(variant);
                  }}
                  className={`px-4 py-2 rounded-md border transition-colors ${
                    isSelected
                      ? 'border-brand-primary bg-brand-primary text-white'
                      : 'border-brand-muted hover:border-brand-primary'
                  }`}
                >
                  {value}
                </button>
              );
            })}
          </div>
        </div>
      ))}

      {/* Quantity */}
      <div className="space-y-2">
        <label className="font-medium">Quantity</label>
        <div className="flex items-center gap-2">
          <button
            onClick={() => setQuantity((q) => Math.max(1, q - 1))}
            className="p-2 border border-brand-muted rounded-md hover:bg-brand-muted transition-colors"
            disabled={quantity <= 1}
          >
            <Minus className="w-4 h-4" />
          </button>
          <input
            type="number"
            value={quantity}
            onChange={(e) => setQuantity(Math.max(1, parseInt(e.target.value) || 1))}
            className="w-16 text-center border border-brand-muted rounded-md py-2"
            min="1"
          />
          <button
            onClick={() => setQuantity((q) => q + 1)}
            className="p-2 border border-brand-muted rounded-md hover:bg-brand-muted transition-colors"
          >
            <Plus className="w-4 h-4" />
          </button>
        </div>
      </div>

      {/* Add to Cart Button */}
      <button
        onClick={handleAddToCart}
        disabled={!inStock || isAdded}
        className={`w-full py-4 rounded-md font-semibold flex items-center justify-center gap-2 transition-colors ${
          isAdded
            ? 'bg-green-500 text-white'
            : inStock
            ? 'bg-brand-primary text-white hover:bg-brand-primary/90'
            : 'bg-brand-muted text-brand-foreground/50 cursor-not-allowed'
        }`}
      >
        {isAdded ? (
          <>
            <Check className="w-5 h-5" />
            Added to Cart
          </>
        ) : inStock ? (
          <>
            <ShoppingCart className="w-5 h-5" />
            Add to Cart
          </>
        ) : (
          'Out of Stock'
        )}
      </button>

      {/* Stock Status */}
      {inStock && (selectedVariant?.stock ?? product.stockQuantity) < 10 && (
        <p className="text-sm text-orange-500">
          Only {selectedVariant?.stock ?? product.stockQuantity} left in stock
        </p>
      )}
    </div>
  );
}
```

---

## 5. Shopping Cart

### 5.1 Cart State Management

```typescript
// apps/storefront/src/lib/cart.tsx
'use client';

import { createContext, useContext, useEffect, ReactNode } from 'react';
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export interface CartItem {
  id: string;
  productId: string;
  variantId?: string;
  title: string;
  variantTitle?: string;
  price: number;
  image?: string;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  getTotal: () => number;
  getItemCount: () => number;
}

const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (item) => {
        set((state) => {
          const existingIndex = state.items.findIndex((i) => i.id === item.id);

          if (existingIndex > -1) {
            const newItems = [...state.items];
            newItems[existingIndex] = {
              ...newItems[existingIndex],
              quantity: newItems[existingIndex].quantity + item.quantity,
            };
            return { items: newItems };
          }

          return { items: [...state.items, item] };
        });
      },

      removeItem: (id) => {
        set((state) => ({
          items: state.items.filter((item) => item.id !== id),
        }));
      },

      updateQuantity: (id, quantity) => {
        set((state) => ({
          items: state.items.map((item) =>
            item.id === id ? { ...item, quantity: Math.max(0, quantity) } : item
          ).filter((item) => item.quantity > 0),
        }));
      },

      clearCart: () => set({ items: [] }),

      getTotal: () => {
        return get().items.reduce(
          (total, item) => total + item.price * item.quantity,
          0
        );
      },

      getItemCount: () => {
        return get().items.reduce((count, item) => count + item.quantity, 0);
      },
    }),
    {
      name: 'cart-storage',
    }
  )
);

interface CartContextType extends CartStore {}

const CartContext = createContext<CartContextType | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const store = useCartStore();

  return (
    <CartContext.Provider value={store}>
      {children}
    </CartContext.Provider>
  );
}

export function useCart() {
  const context = useContext(CartContext);
  if (!context) {
    throw new Error('useCart must be used within a CartProvider');
  }
  return context;
}
```

### 5.2 Cart Drawer Component

```typescript
// apps/storefront/src/components/cart/cart-drawer.tsx
'use client';

import { Fragment } from 'react';
import Image from 'next/image';
import Link from 'next/link';
import { X, Minus, Plus, ShoppingBag } from 'lucide-react';
import { motion, AnimatePresence } from 'framer-motion';
import { useCart } from '@/lib/cart';
import { formatPrice } from '@/lib/utils';

interface CartDrawerProps {
  open: boolean;
  onClose: () => void;
}

export function CartDrawer({ open, onClose }: CartDrawerProps) {
  const { items, updateQuantity, removeItem, getTotal } = useCart();
  const total = getTotal();

  return (
    <AnimatePresence>
      {open && (
        <>
          {/* Backdrop */}
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
            className="fixed inset-0 bg-black/50 z-50"
          />

          {/* Drawer */}
          <motion.div
            initial={{ x: '100%' }}
            animate={{ x: 0 }}
            exit={{ x: '100%' }}
            transition={{ type: 'spring', damping: 30, stiffness: 300 }}
            className="fixed right-0 top-0 h-full w-full max-w-md bg-brand-background shadow-xl z-50 flex flex-col"
          >
            {/* Header */}
            <div className="flex items-center justify-between p-4 border-b border-brand-muted">
              <h2 className="font-heading text-xl font-bold">Your Cart</h2>
              <button
                onClick={onClose}
                className="p-2 hover:bg-brand-muted rounded-full transition-colors"
              >
                <X className="w-5 h-5" />
              </button>
            </div>

            {/* Cart Items */}
            {items.length > 0 ? (
              <>
                <div className="flex-1 overflow-y-auto p-4 space-y-4">
                  {items.map((item) => (
                    <div
                      key={item.id}
                      className="flex gap-4 p-4 bg-brand-muted/50 rounded-lg"
                    >
                      {/* Image */}
                      <div className="relative w-20 h-20 rounded-md overflow-hidden bg-brand-muted shrink-0">
                        {item.image ? (
                          <Image
                            src={item.image}
                            alt={item.title}
                            fill
                            className="object-cover"
                          />
                        ) : (
                          <div className="w-full h-full flex items-center justify-center">
                            <ShoppingBag className="w-8 h-8 text-brand-foreground/30" />
                          </div>
                        )}
                      </div>

                      {/* Details */}
                      <div className="flex-1 min-w-0">
                        <h3 className="font-medium line-clamp-2">{item.title}</h3>
                        {item.variantTitle && (
                          <p className="text-sm text-brand-foreground/70">
                            {item.variantTitle}
                          </p>
                        )}
                        <p className="font-semibold text-brand-primary mt-1">
                          {formatPrice(item.price)}
                        </p>

                        {/* Quantity Controls */}
                        <div className="flex items-center gap-2 mt-2">
                          <button
                            onClick={() => updateQuantity(item.id, item.quantity - 1)}
                            className="p-1 border border-brand-muted rounded hover:bg-brand-muted transition-colors"
                          >
                            <Minus className="w-3 h-3" />
                          </button>
                          <span className="w-8 text-center">{item.quantity}</span>
                          <button
                            onClick={() => updateQuantity(item.id, item.quantity + 1)}
                            className="p-1 border border-brand-muted rounded hover:bg-brand-muted transition-colors"
                          >
                            <Plus className="w-3 h-3" />
                          </button>
                          <button
                            onClick={() => removeItem(item.id)}
                            className="ml-auto text-sm text-red-500 hover:text-red-600"
                          >
                            Remove
                          </button>
                        </div>
                      </div>
                    </div>
                  ))}
                </div>

                {/* Footer */}
                <div className="border-t border-brand-muted p-4 space-y-4">
                  <div className="flex items-center justify-between text-lg">
                    <span>Subtotal</span>
                    <span className="font-semibold">{formatPrice(total)}</span>
                  </div>
                  <p className="text-sm text-brand-foreground/70">
                    Shipping and taxes calculated at checkout
                  </p>
                  <Link
                    href="/checkout"
                    onClick={onClose}
                    className="block w-full bg-brand-primary text-white text-center py-4 rounded-md font-semibold hover:bg-brand-primary/90 transition-colors"
                  >
                    Checkout
                  </Link>
                  <button
                    onClick={onClose}
                    className="block w-full text-center py-2 text-brand-foreground/70 hover:text-brand-foreground"
                  >
                    Continue Shopping
                  </button>
                </div>
              </>
            ) : (
              <div className="flex-1 flex flex-col items-center justify-center p-8 text-center">
                <ShoppingBag className="w-16 h-16 text-brand-foreground/30 mb-4" />
                <h3 className="font-heading text-xl font-bold mb-2">
                  Your cart is empty
                </h3>
                <p className="text-brand-foreground/70 mb-6">
                  Looks like you haven't added any items yet
                </p>
                <Link
                  href="/products"
                  onClick={onClose}
                  className="bg-brand-primary text-white px-6 py-3 rounded-md font-semibold hover:bg-brand-primary/90 transition-colors"
                >
                  Start Shopping
                </Link>
              </div>
            )}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
}
```

---

## 6. Stripe Connect Checkout

### 6.1 Checkout Page

```typescript
// apps/storefront/src/app/checkout/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import Image from 'next/image';
import { useCart } from '@/lib/cart';
import { useStore } from '@/lib/store-context';
import { formatPrice } from '@/lib/utils';
import { Loader2, ShoppingBag, Lock } from 'lucide-react';

export default function CheckoutPage() {
  const router = useRouter();
  const { store } = useStore();
  const { items, getTotal, clearCart } = useCart();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const subtotal = getTotal();
  const shipping = 0; // Free shipping or calculate based on rules
  const total = subtotal + shipping;

  const handleCheckout = async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          items: items.map((item) => ({
            productId: item.productId,
            variantId: item.variantId,
            quantity: item.quantity,
          })),
        }),
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || 'Checkout failed');
      }

      // Redirect to Stripe Checkout
      window.location.href = data.url;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Checkout failed');
      setLoading(false);
    }
  };

  if (items.length === 0) {
    return (
      <div className="container mx-auto px-4 py-16 text-center">
        <ShoppingBag className="w-16 h-16 mx-auto text-brand-foreground/30 mb-4" />
        <h1 className="font-heading text-2xl font-bold mb-2">Your cart is empty</h1>
        <p className="text-brand-foreground/70 mb-6">
          Add some items to your cart to checkout
        </p>
        <a
          href="/products"
          className="inline-block bg-brand-primary text-white px-6 py-3 rounded-md font-semibold hover:bg-brand-primary/90 transition-colors"
        >
          Continue Shopping
        </a>
      </div>
    );
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="font-heading text-3xl font-bold mb-8">Checkout</h1>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-12">
        {/* Order Summary */}
        <div className="lg:order-2">
          <div className="bg-brand-muted/50 rounded-lg p-6 sticky top-24">
            <h2 className="font-heading text-xl font-bold mb-6">Order Summary</h2>

            {/* Items */}
            <div className="space-y-4 mb-6">
              {items.map((item) => (
                <div key={item.id} className="flex gap-4">
                  <div className="relative w-16 h-16 rounded-md overflow-hidden bg-brand-muted shrink-0">
                    {item.image ? (
                      <Image
                        src={item.image}
                        alt={item.title}
                        fill
                        className="object-cover"
                      />
                    ) : (
                      <div className="w-full h-full flex items-center justify-center">
                        <ShoppingBag className="w-6 h-6 text-brand-foreground/30" />
                      </div>
                    )}
                    <span className="absolute -top-2 -right-2 w-5 h-5 bg-brand-primary text-white text-xs rounded-full flex items-center justify-center">
                      {item.quantity}
                    </span>
                  </div>
                  <div className="flex-1 min-w-0">
                    <h3 className="font-medium text-sm line-clamp-2">{item.title}</h3>
                    {item.variantTitle && (
                      <p className="text-xs text-brand-foreground/70">
                        {item.variantTitle}
                      </p>
                    )}
                  </div>
                  <span className="font-medium">
                    {formatPrice(item.price * item.quantity)}
                  </span>
                </div>
              ))}
            </div>

            {/* Totals */}
            <div className="border-t border-brand-foreground/20 pt-4 space-y-2">
              <div className="flex justify-between">
                <span>Subtotal</span>
                <span>{formatPrice(subtotal)}</span>
              </div>
              <div className="flex justify-between">
                <span>Shipping</span>
                <span>{shipping === 0 ? 'Free' : formatPrice(shipping)}</span>
              </div>
              <div className="flex justify-between text-lg font-semibold pt-2 border-t border-brand-foreground/20">
                <span>Total</span>
                <span>{formatPrice(total)}</span>
              </div>
            </div>
          </div>
        </div>

        {/* Checkout Form */}
        <div className="lg:order-1">
          <div className="space-y-6">
            {/* Info Notice */}
            <div className="bg-blue-50 border border-blue-200 rounded-lg p-4 flex gap-3">
              <Lock className="w-5 h-5 text-blue-500 shrink-0" />
              <div className="text-sm text-blue-700">
                <p className="font-medium">Secure Checkout</p>
                <p>You'll be redirected to Stripe for secure payment processing.</p>
              </div>
            </div>

            {/* Error Message */}
            {error && (
              <div className="bg-red-50 border border-red-200 rounded-lg p-4 text-red-700">
                {error}
              </div>
            )}

            {/* Checkout Button */}
            <button
              onClick={handleCheckout}
              disabled={loading}
              className="w-full bg-brand-primary text-white py-4 rounded-md font-semibold hover:bg-brand-primary/90 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center gap-2"
            >
              {loading ? (
                <>
                  <Loader2 className="w-5 h-5 animate-spin" />
                  Processing...
                </>
              ) : (
                <>
                  <Lock className="w-5 h-5" />
                  Proceed to Payment
                </>
              )}
            </button>

            {/* Trust Badges */}
            <div className="flex items-center justify-center gap-4 text-sm text-brand-foreground/70">
              <span className="flex items-center gap-1">
                <Lock className="w-4 h-4" />
                SSL Secured
              </span>
              <span>|</span>
              <span>Powered by Stripe</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 6.2 Checkout API Route

```typescript
// apps/storefront/src/app/api/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { prisma } from '@zyenta/database';
import { headers } from 'next/headers';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

interface CheckoutItem {
  productId: string;
  variantId?: string;
  quantity: number;
}

export async function POST(request: NextRequest) {
  try {
    const headersList = headers();
    const storeSlug = headersList.get('x-store-slug');
    const host = headersList.get('x-store-host') || headersList.get('host');

    if (!storeSlug) {
      return NextResponse.json(
        { error: 'Store not found' },
        { status: 404 }
      );
    }

    // Get store with Stripe account
    const store = await prisma.store.findUnique({
      where: { slug: storeSlug },
      include: {
        user: {
          select: {
            stripeAccountId: true,
            stripeOnboarded: true,
          },
        },
      },
    });

    if (!store || !store.user.stripeAccountId || !store.user.stripeOnboarded) {
      return NextResponse.json(
        { error: 'Store not configured for payments' },
        { status: 400 }
      );
    }

    const { items } = (await request.json()) as { items: CheckoutItem[] };

    if (!items || items.length === 0) {
      return NextResponse.json(
        { error: 'Cart is empty' },
        { status: 400 }
      );
    }

    // Fetch products from database
    const productIds = items.map((item) => item.productId);
    const products = await prisma.product.findMany({
      where: {
        id: { in: productIds },
        storeId: store.id,
        status: 'ACTIVE',
      },
      include: {
        variants: true,
        images: {
          take: 1,
          orderBy: { position: 'asc' },
        },
      },
    });

    // Build line items for Stripe
    const lineItems: Stripe.Checkout.SessionCreateParams.LineItem[] = [];

    for (const item of items) {
      const product = products.find((p) => p.id === item.productId);
      if (!product) continue;

      const variant = item.variantId
        ? product.variants.find((v) => v.id === item.variantId)
        : null;

      const price = variant ? Number(variant.price) : Number(product.sellingPrice);
      const name = variant
        ? `${product.title} - ${variant.title}`
        : product.title;

      lineItems.push({
        price_data: {
          currency: product.currency.toLowerCase(),
          product_data: {
            name,
            images: product.images[0]?.processedUrl
              ? [product.images[0].processedUrl]
              : undefined,
          },
          unit_amount: Math.round(price * 100), // Convert to cents
        },
        quantity: item.quantity,
      });
    }

    if (lineItems.length === 0) {
      return NextResponse.json(
        { error: 'No valid items in cart' },
        { status: 400 }
      );
    }

    // Calculate platform fee (e.g., 5%)
    const platformFeePercent = 5;

    // Create Stripe Checkout Session with Connect
    const session = await stripe.checkout.sessions.create(
      {
        payment_method_types: ['card'],
        line_items: lineItems,
        mode: 'payment',
        success_url: `https://${host}/checkout/success?session_id={CHECKOUT_SESSION_ID}`,
        cancel_url: `https://${host}/checkout`,
        payment_intent_data: {
          application_fee_amount: Math.round(
            lineItems.reduce(
              (total, item) =>
                total + (item.price_data?.unit_amount || 0) * (item.quantity || 1),
              0
            ) *
              (platformFeePercent / 100)
          ),
        },
        metadata: {
          storeId: store.id,
          storeSlug: store.slug,
        },
        shipping_address_collection: {
          allowed_countries: ['US', 'CA', 'GB', 'AU'], // Configure as needed
        },
        billing_address_collection: 'required',
      },
      {
        stripeAccount: store.user.stripeAccountId,
      }
    );

    return NextResponse.json({ url: session.url });
  } catch (error) {
    console.error('Checkout error:', error);
    return NextResponse.json(
      { error: 'Checkout failed' },
      { status: 500 }
    );
  }
}
```

### 6.3 Checkout Success Page

```typescript
// apps/storefront/src/app/checkout/success/page.tsx
import { prisma } from '@zyenta/database';
import { CheckCircle, Package, ArrowRight } from 'lucide-react';
import Link from 'next/link';

interface SuccessPageProps {
  searchParams: {
    session_id?: string;
  };
}

export default async function CheckoutSuccessPage({
  searchParams,
}: SuccessPageProps) {
  // In production, verify the session with Stripe
  // and display order details

  return (
    <div className="container mx-auto px-4 py-16 text-center">
      <div className="max-w-md mx-auto">
        <div className="w-20 h-20 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-6">
          <CheckCircle className="w-10 h-10 text-green-500" />
        </div>

        <h1 className="font-heading text-3xl font-bold mb-4">
          Thank You for Your Order!
        </h1>

        <p className="text-brand-foreground/70 mb-8">
          We've received your order and will process it shortly. You'll receive
          a confirmation email with your order details and tracking information.
        </p>

        <div className="bg-brand-muted/50 rounded-lg p-6 mb-8">
          <div className="flex items-center gap-3 text-left">
            <Package className="w-6 h-6 text-brand-primary" />
            <div>
              <p className="font-medium">Estimated Delivery</p>
              <p className="text-sm text-brand-foreground/70">
                7-14 business days
              </p>
            </div>
          </div>
        </div>

        <div className="space-y-4">
          <Link
            href="/products"
            className="flex items-center justify-center gap-2 bg-brand-primary text-white py-3 px-6 rounded-md font-semibold hover:bg-brand-primary/90 transition-colors"
          >
            Continue Shopping
            <ArrowRight className="w-4 h-4" />
          </Link>
        </div>
      </div>
    </div>
  );
}
```

### 6.4 Stripe Webhook Handler

```typescript
// apps/storefront/src/app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { prisma } from '@zyenta/database';
import { headers } from 'next/headers';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2023-10-16',
});

const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(request: NextRequest) {
  const body = await request.text();
  const headersList = headers();
  const signature = headersList.get('stripe-signature')!;

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(body, signature, webhookSecret);
  } catch (err) {
    console.error('Webhook signature verification failed:', err);
    return NextResponse.json(
      { error: 'Invalid signature' },
      { status: 400 }
    );
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;

      // Create order record
      await handleCheckoutComplete(session);
      break;
    }

    case 'payment_intent.succeeded': {
      const paymentIntent = event.data.object as Stripe.PaymentIntent;
      console.log('Payment succeeded:', paymentIntent.id);
      break;
    }

    case 'payment_intent.payment_failed': {
      const paymentIntent = event.data.object as Stripe.PaymentIntent;
      console.log('Payment failed:', paymentIntent.id);
      break;
    }

    default:
      console.log(`Unhandled event type: ${event.type}`);
  }

  return NextResponse.json({ received: true });
}

async function handleCheckoutComplete(session: Stripe.Checkout.Session) {
  const storeId = session.metadata?.storeId;

  if (!storeId) {
    console.error('No store ID in session metadata');
    return;
  }

  // Here you would:
  // 1. Create an Order record in the database
  // 2. Update inventory
  // 3. Send confirmation email
  // 4. Notify fulfillment system

  console.log('Order completed for store:', storeId);
}
```

---

## 7. Custom Domain Setup

### 7.1 Domain Verification API

```typescript
// apps/storefront/src/app/api/domains/verify/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@zyenta/database';
import dns from 'dns/promises';

export async function POST(request: NextRequest) {
  try {
    const { domain, storeId } = await request.json();

    // Verify ownership via CNAME or TXT record
    const verificationToken = `zyenta-verify=${storeId}`;

    try {
      // Check for TXT record
      const txtRecords = await dns.resolveTxt(domain);
      const hasValidRecord = txtRecords.some((record) =>
        record.join('').includes(verificationToken)
      );

      if (!hasValidRecord) {
        // Check for CNAME pointing to our domain
        const cnameRecords = await dns.resolveCname(domain);
        const hasValidCname = cnameRecords.some(
          (record) => record.endsWith('.zyenta.com')
        );

        if (!hasValidCname) {
          return NextResponse.json(
            { verified: false, error: 'Domain not verified' },
            { status: 400 }
          );
        }
      }

      // Update store with custom domain
      await prisma.store.update({
        where: { id: storeId },
        data: { customDomain: domain },
      });

      return NextResponse.json({ verified: true });
    } catch (dnsError) {
      return NextResponse.json(
        { verified: false, error: 'DNS lookup failed' },
        { status: 400 }
      );
    }
  } catch (error) {
    console.error('Domain verification error:', error);
    return NextResponse.json(
      { error: 'Verification failed' },
      { status: 500 }
    );
  }
}
```

### 7.2 Domain Resolution API

```typescript
// apps/storefront/src/app/api/domains/resolve/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@zyenta/database';

// Cache for domain lookups
const domainCache = new Map<string, { slug: string; expires: number }>();
const CACHE_TTL = 60 * 1000; // 1 minute

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const domain = searchParams.get('domain');

  if (!domain) {
    return NextResponse.json(
      { error: 'Domain required' },
      { status: 400 }
    );
  }

  // Check cache
  const cached = domainCache.get(domain);
  if (cached && cached.expires > Date.now()) {
    return NextResponse.json({ storeSlug: cached.slug });
  }

  // Lookup in database
  const store = await prisma.store.findFirst({
    where: {
      customDomain: domain,
      status: 'ACTIVE',
    },
    select: { slug: true },
  });

  if (!store) {
    return NextResponse.json(
      { error: 'Domain not found' },
      { status: 404 }
    );
  }

  // Update cache
  domainCache.set(domain, {
    slug: store.slug,
    expires: Date.now() + CACHE_TTL,
  });

  return NextResponse.json({ storeSlug: store.slug });
}
```

---

## 8. Utility Functions

### 8.1 Common Utilities

```typescript
// apps/storefront/src/lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatPrice(
  amount: number,
  currency = 'USD',
  locale = 'en-US'
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount);
}

export function formatDate(date: Date | string, locale = 'en-US'): string {
  return new Intl.DateTimeFormat(locale, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(new Date(date));
}

export function truncate(str: string, maxLength: number): string {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength - 3) + '...';
}

export function slugify(str: string): string {
  return str
    .toLowerCase()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
}

export function getImageUrl(url: string | null | undefined): string {
  if (!url) return '/placeholder.jpg';
  if (url.startsWith('http')) return url;
  return `${process.env.NEXT_PUBLIC_CDN_URL}/${url}`;
}
```

---

## 9. Testing

### 9.1 Test Configuration

```typescript
// apps/storefront/jest.config.js
const nextJest = require('next/jest');

const createJestConfig = nextJest({
  dir: './',
});

const customJestConfig = {
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  testEnvironment: 'jest-environment-jsdom',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
};

module.exports = createJestConfig(customJestConfig);
```

### 9.2 Component Tests

```typescript
// apps/storefront/src/components/cart/__tests__/cart-drawer.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { CartDrawer } from '../cart-drawer';
import { CartProvider } from '@/lib/cart';

const mockStore = {
  id: 'store-1',
  name: 'Test Store',
  slug: 'test-store',
  // ... other required fields
};

const wrapper = ({ children }) => (
  <CartProvider>{children}</CartProvider>
);

describe('CartDrawer', () => {
  it('renders empty cart message when cart is empty', () => {
    render(<CartDrawer open={true} onClose={jest.fn()} />, { wrapper });

    expect(screen.getByText('Your cart is empty')).toBeInTheDocument();
  });

  it('calls onClose when close button is clicked', () => {
    const onClose = jest.fn();
    render(<CartDrawer open={true} onClose={onClose} />, { wrapper });

    fireEvent.click(screen.getByLabelText('Close'));
    expect(onClose).toHaveBeenCalled();
  });
});
```

---

## 10. Deployment

### 10.1 Vercel Configuration

```json
// apps/storefront/vercel.json
{
  "buildCommand": "pnpm turbo build --filter=@zyenta/storefront",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "regions": ["iad1"],
  "env": {
    "DATABASE_URL": "@database-url",
    "STRIPE_SECRET_KEY": "@stripe-secret-key",
    "STRIPE_WEBHOOK_SECRET": "@stripe-webhook-secret"
  }
}
```

### 10.2 Environment Variables

```bash
# apps/storefront/.env.example

# Database
DATABASE_URL="postgresql://..."

# Stripe
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_test_..."

# CDN
NEXT_PUBLIC_CDN_URL="https://cdn.zyenta.com"

# Internal API (for domain resolution)
INTERNAL_API_URL="https://api.zyenta.com"

# App URL
NEXT_PUBLIC_APP_URL="https://zyenta.com"
```

---

## Verification Checklist

### Core Functionality
- [ ] Multi-tenant routing works with subdomains
- [ ] Custom domain resolution works
- [ ] Dynamic theming applies correctly
- [ ] Product listing displays correctly
- [ ] Product detail page shows all information
- [ ] Shopping cart persists across sessions
- [ ] Checkout creates Stripe session
- [ ] Payment processing completes
- [ ] Order success page displays
- [ ] Webhooks update order status

### Performance
- [ ] Page load time < 2 seconds
- [ ] Images optimized with Next.js Image
- [ ] Static pages cached properly
- [ ] Cart state managed efficiently

### Security
- [ ] Stripe webhook signatures verified
- [ ] No sensitive data exposed to client
- [ ] CORS configured correctly
- [ ] Input validation on all forms

---

## Next Steps

After completing Sprint 5, proceed to:

1. **Sprint 6: Integration & Polish**
   - End-to-end testing
   - Error handling improvements
   - Performance optimization
   - Documentation
   - Security audit
   - Beta launch preparation

---

*Sprint 5 establishes the customer-facing storefront with full e-commerce capabilities including multi-tenant architecture, product browsing, shopping cart, and Stripe Connect payment processing.*
