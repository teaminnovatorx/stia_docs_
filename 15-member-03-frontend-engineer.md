# UDARA AI — Frontend Engineer Deep Dive

> **Document ID:** UDARA-ENG-FE-001  
> **Version:** 1.0.0  
> **Last Updated:** 2026-05-27  
> **Owner:** Frontend Engineering Team  
> **Classification:** Internal — Engineering

---

## Table of Contents

1. [Role Overview](#1-role-overview)
2. [Key Skills & Competencies](#2-key-skills--competencies)
3. [Complete File Structure](#3-complete-file-structure)
4. [UDARA Design System](#4-udara-design-system)
5. [Core Component Code](#5-core-component-code)
6. [API Client & Data Layer](#6-api-client--data-layer)
7. [Data Visualization](#7-data-visualization)
8. [Week-by-Week Plan](#8-week-by-week-plan)
9. [Deliverables Checklist](#9-deliverables-checklist)
10. [Investor Demo Framing Tips](#10-investor-demo-framing-tips)

---

## 1. Role Overview

The Frontend Engineer owns everything the user sees and interacts with across **three surfaces**:

| Surface | Tech Stack | Primary Users |
|---------|-----------|---------------|
| **Dashboard Web App** | Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS, TanStack Table/Query, ECharts, MapLibre GL | National Surveillance Officers, Lab Scientists, Policymakers |
| **Admin Panel** | Same stack, role-gated routes | System Administrators, MoH Officials |
| **Public Health Maps** | MapLibre GL JS, deck.gl | Public health researchers, partner orgs |

### Core Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND ENGINEER                            │
├──────────────────┬──────────────────┬───────────────────────────┤
│  Design System   │   Dashboard      │    Data Visualization     │
│  & Components    │   Architecture   │    & Map Interactions     │
├──────────────────┼──────────────────┼───────────────────────────┤
│  • Tailwind cfg  │  • App Router    │  • Choropleth maps        │
│  • Component lib │  • Layout system │  • ECharts time series    │
│  • Storybook     │  • Auth flows    │  • Real-time updates      │
│  • Animations    │  • Role gating   │  • Offline-first         │
├──────────────────┼──────────────────┼───────────────────────────┤
│  Performance     │  Accessibility   │  Testing                  │
│  Optimization    │  (WCAG 2.1 AA)   │                           │
├──────────────────┼──────────────────┼───────────────────────────┤
│  • Lighthouse 90+│  • Keyboard nav  │  • Vitest + RTL           │
│  • Bundle < 250K │  • Screen reader │  • Playwright E2E         │
│  • Code splitting│  • High contrast │  • Chromatic visual       │
└──────────────────┴──────────────────┴───────────────────────────┘
```

---

## 2. Key Skills & Competencies

| Skill | Proficiency | Tools / Libraries |
|-------|-------------|-------------------|
| React + TypeScript | Expert | React 18, TypeScript 5.x, Zod |
| Next.js App Router | Expert | Server Components, Streaming, ISR |
| State Management | Advanced | Zustand, TanStack Query v5 |
| CSS / Styling | Expert | Tailwind CSS 3.4, CSS Modules, Radix UI |
| Data Visualization | Advanced | ECharts 5, MapLibre GL JS 4, D3.js |
| Table / Data Grid | Advanced | TanStack Table v8, virtualization |
| Testing | Advanced | Vitest, React Testing Library, Playwright |
| Performance | Advanced | Lighthouse, Web Vitals, bundle analysis |
| CI/CD | Intermediate | GitHub Actions, Vercel / self-hosted |
| i18n | Intermediate | next-intl, ICU message format |
| Offline / PWA | Intermediate | Service Workers, IndexedDB, Workbox |
| Maps / GIS | Intermediate | GeoJSON, TopoJSON, MapLibre expressions |

---

## 3. Complete File Structure

```
web/
├── .env.local                          # API_URL, MAP_TOKEN, SENTRY_DSN
├── .env.production
├── next.config.ts                      # i18n, image domains, headers, CSP
├── tailwind.config.ts                  # UDARA design tokens
├── tsconfig.json                       # strict mode, path aliases
├── postcss.config.js
├── vitest.config.ts
├── playwright.config.ts
│
├── public/
│   ├── favicon.ico
│   ├── icons/
│   │   ├── icon-192.png
│   │   └── icon-512.png
│   ├── fonts/
│   │   ├── Inter-Variable.woff2
│   │   └── JetBrainsMono-Variable.woff2
│   ├── manifest.json                   # PWA manifest
│   └── sw.js                           # Service worker (generated)
│
├── app/                                # Next.js App Router
│   ├── layout.tsx                      # Root layout: fonts, providers, sidebar
│   ├── page.tsx                        # Dashboard home (redirect to /dashboard)
│   ├── globals.css                     # Tailwind directives + CSS variables
│   ├── loading.tsx                     # Global loading skeleton
│   ├── error.tsx                       # Error boundary
│   ├── not-found.tsx
│   │
│   ├── (auth)/                         # Auth route group
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   ├── forgot-password/
│   │   │   └── page.tsx
│   │   └── layout.tsx                  # Centered layout, no sidebar
│   │
│   ├── (dashboard)/                    # Dashboard route group (sidebar layout)
│   │   ├── layout.tsx                  # Sidebar + main content layout
│   │   ├── dashboard/
│   │   │   └── page.tsx                # Overview: stats cards, map, chart
│   │   ├── cases/
│   │   │   ├── page.tsx                # Cases table (TanStack Table)
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx            # Case detail view
│   │   │   └── new/
│   │   │       └── page.tsx            # Manual case entry form
│   │   ├── resistance/
│   │   │   ├── page.tsx                # Resistance map (MapLibre)
│   │   │   └── trends/
│   │   │       └── page.tsx            # Trend analysis (ECharts)
│   │   ├── alerts/
│   │   │   └── page.tsx                # Alert feed + rule management
│   │   ├── reports/
│   │   │   ├── page.tsx                # GLASS report generator
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   ├── labs/
│   │   │   └── page.tsx                # Laboratory data management
│   │   ├── chws/
│   │   │   ├── page.tsx                # CHW registry
│   │   │   └── leaderboard/
│   │   │       └── page.tsx
│   │   ├── settings/
│   │   │   └── page.tsx                # User/org settings
│   │   └── admin/
│   │       ├── users/
│   │       │   └── page.tsx
│   │       ├── organizations/
│   │       │   └── page.tsx
│   │       └── system/
│   │           └── page.tsx
│   │
│   └── api/                            # Next.js API routes (BFF)
│       ├── health/
│       │   └── route.ts
│       ├── proxy/
│       │   └── [...path]/
│       │       └── route.ts            # Reverse proxy to backend
│       └── export/
│           └── [...format]/
│               └── route.ts            # CSV/PDF export
│
├── components/
│   ├── ui/                             # Base UI components (Radix primitives)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── select.tsx
│   │   ├── dialog.tsx
│   │   ├── sheet.tsx
│   │   ├── dropdown-menu.tsx
│   │   ├── tooltip.tsx
│   │   ├── badge.tsx
│   │   ├── avatar.tsx
│   │   ├── separator.tsx
│   │   ├── skeleton.tsx
│   │   ├── tabs.tsx
│   │   ├── card.tsx
│   │   ├── table.tsx
│   │   ├── toast.tsx
│   │   ├── progress.tsx
│   │   ├── switch.tsx
│   │   └── command.tsx                 # cmd+k search
│   │
│   ├── layout/
│   │   ├── app-sidebar.tsx             # Main sidebar navigation
│   │   ├── header.tsx                  # Top bar: breadcrumbs, search, avatar
│   │   ├── breadcrumbs.tsx
│   │   ├── user-nav.tsx
│   │   └── theme-toggle.tsx
│   │
│   ├── dashboard/
│   │   ├── stats-cards.tsx             # KPI stat cards grid
│   │   ├── resistance-map.tsx          # MapLibre GL choropleth
│   │   ├── trend-chart.tsx             # ECharts time series
│   │   ├── recent-cases.tsx            # Latest 5 cases list
│   │   ├── alert-feed.tsx              # Scrolling alert ticker
│   │   ├── drug-resistance-bar.tsx     # Horizontal bar per drug
│   │   └── loading-skeleton.tsx        # Dashboard skeleton loaders
│   │
│   ├── cases/
│   │   ├── cases-table.tsx             # TanStack Table with all features
│   │   ├── case-detail-dialog.tsx      # Slide-over detail panel
│   │   ├── case-form.tsx               # New case entry
│   │   ├── case-source-badge.tsx
│   │   ├── case-outcome-badge.tsx
│   │   └── case-filters.tsx
│   │
│   ├── resistance/
│   │   ├── resistance-map.tsx
│   │   ├── resistance-legend.tsx
│   │   ├── resistance-popup.tsx
│   │   └── district-layer.tsx
│   │
│   ├── charts/
│   │   ├── trend-chart.tsx
│   │   ├── drug-comparison-chart.tsx
│   │   ├── age-distribution-chart.tsx
│   │   └── source-breakdown-chart.tsx
│   │
│   └── shared/
│       ├── data-table.tsx              # Reusable TanStack Table wrapper
│       ├── empty-state.tsx
│       ├── error-boundary.tsx
│       ├── loading-spinner.tsx
│       ├── confirmation-dialog.tsx
│       ├── date-range-picker.tsx
│       └── export-button.tsx
│
├── hooks/
│   ├── use-cases.ts                    # TanStack Query hook: getCases
│   ├── use-resistance.ts               # getResistanceData
│   ├── use-trends.ts                   # getTrends
│   ├── use-alerts.ts                   # getAlerts
│   ├── use-stats.ts                    # getStats
│   ├── use-chw.ts                      # getCHWProfile, getCHWStats
│   ├── use-map-style.ts                # Dynamic MapLibre style
│   ├── use-debounce.ts
│   ├── use-local-storage.ts
│   ├── use-media-query.ts
│   └── use-intersection-observer.ts
│
├── lib/
│   ├── api-client.ts                   # Typed HTTP client
│   ├── auth.ts                         # NextAuth configuration
│   ├── utils.ts                        # cn(), formatDate(), formatNumber()
│   ├── constants.ts                    # NAV_ITEMS, DRUGS, COUNTRIES
│   ├── validators.ts                   # Zod schemas matching backend Pydantic
│   └── map-styles.ts                   # Light/dark/satellite MapLibre styles
│
├── stores/
│   ├── sidebar-store.ts                # Zustand: collapsed, active item
│   ├── filter-store.ts                 # Zustand: date range, district, drug
│   └── user-store.ts                   # Zustand: user profile, role, org
│
├── types/
│   ├── case.ts                         # Case, Symptom, Medication interfaces
│   ├── resistance.ts                   # ResistanceData, DistrictResistance
│   ├── alert.ts                        # Alert, AlertRule
│   ├── chw.ts                          # CHW, CHWStats
│   ├── api.ts                          # ApiResponse<T>, PaginatedResponse<T>
│   └── auth.ts                         # User, Role, Session
│
└── styles/
    └── map-popups.css                  # MapLibre popup styles
```

---

## 4. UDARA Design System

### 4.1 Tailwind Configuration — `tailwind.config.ts`

```typescript
import type { Config } from "tailwindcss";
import plugin from "tailwindcss/plugin";
import tailwindAnimate from "tailwindcss-animate";

const config: Config = {
  darkMode: "class",
  content: [
    "./app/**/*.{ts,tsx}",
    "./components/**/*.{ts,tsx}",
    "./lib/**/*.{ts,tsx}",
  ],
  theme: {
    container: {
      center: true,
      padding: "1rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        /* ── UDARA Primary Palette ─────────────────────── */
        udara: {
          // Primary teal — represents trust, health, clean data
          50:  "#effefb",
          100: "#c7fff3",
          200: "#90ffe8",
          300: "#51f7d9",
          400: "#1ee4c6",
          500: "#06c9af",
          600: "#01a28e",
          700: "#078172",
          800: "#0c665c",
          900: "#0f554d",
          950: "#003331",

          /* ── Resistance Color Scale ───────────────────── */
          resistance: {
            critical: {
              DEFAULT: "#dc2626",
              light:   "#fecaca",
              dark:    "#991b1b",
              bg:      "#fef2f2",
            },
            warning: {
              DEFAULT: "#f59e0b",
              light:   "#fde68a",
              dark:    "#d97706",
              bg:      "#fffbeb",
            },
            moderate: {
              DEFAULT: "#3b82f6",
              light:   "#bfdbfe",
              dark:    "#2563eb",
              bg:      "#eff6ff",
            },
            safe: {
              DEFAULT: "#10b981",
              light:   "#a7f3d0",
              dark:    "#059669",
              bg:      "#ecfdf5",
            },
            unknown: {
              DEFAULT: "#6b7280",
              light:   "#e5e7eb",
              dark:    "#4b5563",
              bg:      "#f9fafb",
            },
          },

          /* ── Data Source Colors ───────────────────────── */
          source: {
            telegram:   "#229ED9",
            whatsapp:   "#25D366",
            ussd:       "#8B5CF6",
            manual:     "#6366F1",
            api:        "#EC4899",
          },
        },

        /* ── Semantic Aliases ───────────────────────────── */
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

      fontFamily: {
        sans: ["Inter", "system-ui", "sans-serif"],
        mono: ["JetBrains Mono", "monospace"],
      },

      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },

      /* ── Custom Keyframe Animations ──────────────────── */
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
        // UDARA-specific animations
        "pulse-resistance": {
          "0%, 100%": { opacity: "1", transform: "scale(1)" },
          "50%": { opacity: "0.7", transform: "scale(1.05)" },
        },
        "fade-in-up": {
          from: { opacity: "0", transform: "translateY(10px)" },
          to: { opacity: "1", transform: "translateY(0)" },
        },
        "slide-in-right": {
          from: { transform: "translateX(100%)" },
          to: { transform: "translateX(0)" },
        },
        "count-up": {
          from: { opacity: "0", transform: "translateY(4px)" },
          to: { opacity: "1", transform: "translateY(0)" },
        },
        "alert-flash": {
          "0%, 100%": { backgroundColor: "hsl(var(--destructive) / 0.1)" },
          "50%": { backgroundColor: "hsl(var(--destructive) / 0.25)" },
        },
        "map-pin-bounce": {
          "0%, 100%": { transform: "translateY(0)" },
          "50%": { transform: "translateY(-6px)" },
        },
        "shimmer": {
          from: { backgroundPosition: "-200% 0" },
          to: { backgroundPosition: "200% 0" },
        },
      },

      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
        "pulse-resistance": "pulse-resistance 2s ease-in-out infinite",
        "fade-in-up": "fade-in-up 0.3s ease-out",
        "slide-in-right": "slide-in-right 0.3s ease-out",
        "count-up": "count-up 0.4s ease-out",
        "alert-flash": "alert-flash 2s ease-in-out 3",
        "map-pin-bounce": "map-pin-bounce 1s ease-in-out infinite",
        "shimmer": "shimmer 2s linear infinite",
      },
    },
  },
  plugins: [
    tailwindAnimate,
    // Custom UDARA plugin for resistance utilities
    plugin(function ({ addUtilities, matchUtilities, theme }) {
      // Resistance background utilities
      matchUtilities(
        {
          "bg-resistance": (value: string) => ({
            backgroundColor: value,
          }),
        },
        {
          values: {
            critical: theme("colors.udara.resistance.critical.bg"),
            warning: theme("colors.udara.resistance.warning.bg"),
            moderate: theme("colors.udara.resistance.moderate.bg"),
            safe: theme("colors.udara.resistance.safe.bg"),
          },
        }
      );
    }),
  ],
};

export default config;
```

### 4.2 CSS Variables — `app/globals.css`

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

    --primary: 170 98% 39%;  /* udara.primary teal */
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
    --ring: 170 98% 39%;

    --radius: 0.625rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;

    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;

    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;

    --primary: 170 98% 45%;
    --primary-foreground: 222.2 84% 4.9%;

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
    --ring: 170 98% 45%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
    font-feature-settings: "rlig" 1, "calt" 1;
  }
}

/* Shimmer skeleton effect */
.skeleton-shimmer {
  background: linear-gradient(
    90deg,
    hsl(var(--muted)) 25%,
    hsl(var(--muted-foreground) / 0.1) 50%,
    hsl(var(--muted)) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 2s linear infinite;
}

/* MapLibre popup overrides */
.maplibregl-popup-content {
  @apply rounded-lg shadow-lg border border-border bg-card text-card-foreground;
  padding: 0 !important;
  min-width: 220px;
}
```

---

## 5. Core Component Code

### 5.1 App Sidebar — `components/layout/app-sidebar.tsx`

```tsx
"use client";

import React, { useCallback } from "react";
import Link from "next/link";
import { usePathname } from "next/navigation";
import {
  LayoutDashboard,
  FileText,
  ShieldAlert,
  Bell,
  Microscope,
  Users,
  Settings,
  ChevronLeft,
  ChevronRight,
  Activity,
} from "lucide-react";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Tooltip, TooltipContent, TooltipTrigger } from "@/components/ui/tooltip";
import { Badge } from "@/components/ui/badge";
import { useSidebarStore } from "@/stores/sidebar-store";
import { useAlerts } from "@/hooks/use-alerts";

interface NavItem {
  title: string;
  href: string;
  icon: React.ElementType;
  badge?: "alerts" | null;
  badgeCount?: number;
}

const NAV_ITEMS: NavItem[] = [
  {
    title: "Dashboard",
    href: "/dashboard",
    icon: LayoutDashboard,
  },
  {
    title: "Cases",
    href: "/cases",
    icon: FileText,
  },
  {
    title: "Resistance Map",
    href: "/resistance",
    icon: ShieldAlert,
  },
  {
    title: "Alerts",
    href: "/alerts",
    icon: Bell,
    badge: "alerts",
  },
  {
    title: "Laboratory",
    href: "/labs",
    icon: Microscope,
  },
  {
    title: "CHW Network",
    href: "/chws",
    icon: Users,
  },
  {
    title: "Live Feed",
    href: "/dashboard?tab=feed",
    icon: Activity,
  },
  {
    title: "Settings",
    href: "/settings",
    icon: Settings,
  },
];

function SidebarNavLink({ item, collapsed }: { item: NavItem; collapsed: boolean }) {
  const pathname = usePathname();
  const isActive = pathname.startsWith(item.href);
  const { data: alertsData } = useAlerts({ enabled: item.badge === "alerts" });
  const alertCount = item.badge === "alerts" ? alertsData?.unread_count ?? 0 : 0;

  const linkContent = (
    <Link
      href={item.href}
      className={cn(
        "flex items-center gap-3 rounded-lg px-3 py-2.5 text-sm font-medium",
        "transition-all duration-200 hover:bg-accent hover:text-accent-foreground",
        "group relative",
        isActive
          ? "bg-primary/10 text-primary dark:bg-primary/20"
          : "text-muted-foreground",
        collapsed && "justify-center px-2"
      )}
    >
      {/* Active indicator bar */}
      {isActive && (
        <div className="absolute left-0 top-1/2 -translate-y-1/2 h-6 w-1 rounded-r-full bg-primary" />
      )}

      <item.icon
        className={cn(
          "h-5 w-5 shrink-0 transition-colors",
          isActive ? "text-primary" : "text-muted-foreground group-hover:text-foreground"
        )}
      />

      {!collapsed && (
        <>
          <span className="flex-1 truncate">{item.title}</span>
          {alertCount > 0 && (
            <Badge
              variant="destructive"
              className="h-5 min-w-[20px] justify-center rounded-full px-1.5 text-[10px] font-bold animate-pulse-resistance"
            >
              {alertCount > 99 ? "99+" : alertCount}
            </Badge>
          )}
        </>
      )}

      {/* Collapsed: show badge as dot */}
      {collapsed && alertCount > 0 && (
        <span className="absolute -right-0.5 -top-0.5 h-2.5 w-2.5 rounded-full bg-destructive ring-2 ring-background" />
      )}
    </Link>
  );

  if (collapsed) {
    return (
      <Tooltip delayDuration={0}>
        <TooltipTrigger asChild>{linkContent}</TooltipTrigger>
        <TooltipContent side="right" className="flex items-center gap-2">
          {item.title}
          {alertCount > 0 && (
            <Badge variant="destructive" className="text-[10px]">
              {alertCount}
            </Badge>
          )}
        </TooltipContent>
      </Tooltip>
    );
  }

  return linkContent;
}

export function AppSidebar() {
  const { collapsed, toggle } = useSidebarStore();

  return (
    <aside
      className={cn(
        "flex h-screen flex-col border-r bg-card transition-all duration-300 ease-in-out",
        collapsed ? "w-[68px]" : "w-[260px]"
      )}
    >
      {/* Logo */}
      <div className={cn(
        "flex h-16 items-center border-b px-4",
        collapsed ? "justify-center" : "gap-3"
      )}>
        <div className="flex h-9 w-9 shrink-0 items-center justify-center rounded-lg bg-primary text-primary-foreground font-bold text-lg">
          U
        </div>
        {!collapsed && (
          <div className="flex flex-col">
            <span className="text-sm font-bold tracking-tight text-foreground">
              UDARA AI
            </span>
            <span className="text-[10px] text-muted-foreground uppercase tracking-widest">
              AMR Surveillance
            </span>
          </div>
        )}
      </div>

      {/* Navigation */}
      <nav className="flex-1 overflow-y-auto px-3 py-4 space-y-1 scrollbar-thin">
        {NAV_ITEMS.map((item) => (
          <SidebarNavLink key={item.href} item={item} collapsed={collapsed} />
        ))}
      </nav>

      {/* Collapse toggle */}
      <div className="border-t p-3">
        <Button
          variant="ghost"
          size="sm"
          onClick={toggle}
          className={cn("w-full", collapsed ? "justify-center" : "justify-start")}
        >
          {collapsed ? (
            <ChevronRight className="h-4 w-4" />
          ) : (
            <>
              <ChevronLeft className="h-4 w-4 mr-2" />
              <span className="text-xs">Collapse</span>
            </>
          )}
        </Button>
      </div>
    </aside>
  );
}
```

### 5.2 Color Scales — `lib/color-scales.ts`

```typescript
/**
 * UDARA Resistance Color Scale
 *
 * Maps resistance percentages to colors, labels, and emojis.
 * Used across the entire application: maps, charts, badges, tables.
 *
 * Scale:
 *   0-30%    → Safe     (green)
 *   30-50%   → Moderate (blue)
 *   50-70%   → Warning  (amber)
 *   70-100%  → Critical (red)
 */

export interface ResistanceLevel {
  value: number;
  color: string;
  colorLight: string;
  colorDark: string;
  bgClass: string;
  textClass: string;
  label: string;
  emoji: string;
  hex: string;
}

export const RESISTANCE_COLOR_RANGE: ResistanceLevel[] = [
  { value: 0,   color: "#10b981", colorLight: "#a7f3d0", colorDark: "#059669", bgClass: "bg-resistance-safe",      textClass: "text-udara-resistance-safe",    label: "Safe",     emoji: "🟢", hex: "#10b981" },
  { value: 25,  color: "#10b981", colorLight: "#a7f3d0", colorDark: "#059669", bgClass: "bg-resistance-safe",      textClass: "text-udara-resistance-safe",    label: "Safe",     emoji: "🟢", hex: "#10b981" },
  { value: 30,  color: "#3b82f6", colorLight: "#bfdbfe", colorDark: "#2563eb", bgClass: "bg-resistance-moderate",   textClass: "text-udara-resistance-moderate", label: "Moderate", emoji: "🔵", hex: "#3b82f6" },
  { value: 50,  color: "#f59e0b", colorLight: "#fde68a", colorDark: "#d97706", bgClass: "bg-resistance-warning",    textClass: "text-udara-resistance-warning",  label: "Warning",  emoji: "🟡", hex: "#f59e0b" },
  { value: 70,  color: "#ef4444", colorLight: "#fecaca", colorDark: "#dc2626", bgClass: "bg-resistance-critical",   textClass: "text-udara-resistance-critical", label: "Critical", emoji: "🔴", hex: "#ef4444" },
  { value: 100, color: "#dc2626", colorLight: "#fecaca", colorDark: "#991b1b", bgClass: "bg-resistance-critical",   textClass: "text-udara-resistance-critical", label: "Critical", emoji: "🔴", hex: "#dc2626" },
];

/**
 * Returns the hex color for a resistance percentage value.
 * Uses linear interpolation between defined stops.
 */
export function getResistanceColor(percentage: number): string {
  const clamped = Math.max(0, Math.min(100, percentage));

  // Find the two stops to interpolate between
  let lower = RESISTANCE_COLOR_RANGE[0];
  let upper = RESISTANCE_COLOR_RANGE[RESISTANCE_COLOR_RANGE.length - 1];

  for (let i = 0; i < RESISTANCE_COLOR_RANGE.length - 1; i++) {
    if (
      clamped >= RESISTANCE_COLOR_RANGE[i].value &&
      clamped <= RESISTANCE_COLOR_RANGE[i + 1].value
    ) {
      lower = RESISTANCE_COLOR_RANGE[i];
      upper = RESISTANCE_COLOR_RANGE[i + 1];
      break;
    }
  }

  // If at exact stop, return directly
  if (clamped === lower.value) return lower.color;
  if (clamped === upper.value) return upper.color;

  // Linear interpolation
  const range = upper.value - lower.value;
  const factor = range === 0 ? 0 : (clamped - lower.value) / range;

  return interpolateColor(lower.color, upper.color, factor);
}

/**
 * Returns the semantic label for a resistance percentage.
 */
export function getResistanceLabel(percentage: number): string {
  if (percentage < 30)  return "Safe";
  if (percentage < 50)  return "Moderate";
  if (percentage < 70)  return "Warning";
  return "Critical";
}

/**
 * Returns the emoji indicator for a resistance percentage.
 */
export function getResistanceEmoji(percentage: number): string {
  if (percentage < 30)  return "🟢";
  if (percentage < 50)  return "🔵";
  if (percentage < 70)  return "🟡";
  return "🔴";
}

/**
 * Returns the Tailwind background class for resistance level.
 */
export function getResistanceBgClass(percentage: number): string {
  if (percentage < 30)  return "bg-udara-resistance-safe-bg";
  if (percentage < 50)  return "bg-udara-resistance-moderate-bg";
  if (percentage < 70)  return "bg-udara-resistance-warning-bg";
  return "bg-udara-resistance-critical-bg";
}

/**
 * Returns the full ResistanceLevel object for a percentage.
 */
export function getResistanceLevel(percentage: number): ResistanceLevel {
  const clamped = Math.max(0, Math.min(100, percentage));

  if (clamped < 30)  return RESISTANCE_COLOR_RANGE[0];
  if (clamped < 50)  return RESISTANCE_COLOR_RANGE[2];
  if (clamped < 70)  return RESISTANCE_COLOR_RANGE[3];
  return RESISTANCE_COLOR_RANGE[5];
}

/**
 * Generates a MapLibre GL step expression for choropleth fills.
 * Usage: map.setPaintProperty('districts-fill', 'fill-color', getResistanceExpression())
 */
export function getResistanceExpression(): object[] {
  return [
    "interpolate",
    ["linear"],
    ["get", "resistance_pct"],
    0,   "#10b981",  // green
    30,  "#3b82f6",  // blue
    50,  "#f59e0b",  // amber
    70,  "#ef4444",  // red
    100, "#dc2626",  // dark red
  ];
}

// ── Helpers ────────────────────────────────────────────────────────

function interpolateColor(a: string, b: string, factor: number): string {
  const ah = parseInt(a.replace("#", ""), 16);
  const bh = parseInt(b.replace("#", ""), 16);

  const ar = (ah >> 16) & 0xff, ag = (ah >> 8) & 0xff, ab = ah & 0xff;
  const br = (bh >> 16) & 0xff, bg = (bh >> 8) & 0xff, bb = bh & 0xff;

  const rr = Math.round(ar + (br - ar) * factor);
  const rg = Math.round(ag + (bg - ag) * factor);
  const rb = Math.round(ab + (bb - ab) * factor);

  return `#${((rr << 16) | (rg << 8) | rb).toString(16).padStart(6, "0")}`;
}

/**
 * Formats a percentage with color for use in markdown/HTML.
 */
export function formatResistancePct(percentage: number): string {
  const color = getResistanceColor(percentage);
  const emoji = getResistanceEmoji(percentage);
  const label = getResistanceLabel(percentage);
  return `${emoji} **${percentage.toFixed(1)}%** — ${label}`;
}
```

### 5.3 Resistance Map — `components/dashboard/resistance-map.tsx`

```tsx
"use client";

import React, { useEffect, useRef, useCallback, useState } from "react";
import maplibregl from "maplibre-gl";
import "maplibre-gl/dist/maplibre-gl.css";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { getResistanceExpression, getResistanceLabel, getResistanceEmoji } from "@/lib/color-scales";
import { useResistance } from "@/hooks/use-resistance";
import { Layers, Maximize2, RotateCcw } from "lucide-react";

interface DistrictFeature {
  type: "Feature";
  properties: {
    district_id: string;
    name: string;
    state: string;
    country: string;
    resistance_pct: number;
    total_cases: number;
    population: number;
  };
  geometry: {
    type: "Polygon" | "MultiPolygon";
    coordinates: number[][][];
  };
}

interface ResistancePopupData {
  district: string;
  state: string;
  resistance: number;
  cases: number;
  drugs: { name: string; resistance: number }[];
}

export function ResistanceMap() {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<maplibregl.Map | null>(null);
  const popup = useRef<maplibregl.Popup | null>(null);
  const [selectedDrug, setSelectedDrug] = useState<string>("all");
  const [isFullscreen, setIsFullscreen] = useState(false);

  const { data: resistanceData, isLoading, error } = useResistance({
    drug: selectedDrug,
    granularity: "district",
  });

  // Initialize map
  useEffect(() => {
    if (map.current || !mapContainer.current) return;

    map.current = new maplibregl.Map({
      container: mapContainer.current,
      style: {
        version: 8,
        name: "UDARA Light",
        sources: {
          osm: {
            type: "raster",
            tiles: [
              "https://a.tile.openstreetmap.org/{z}/{x}/{y}.png",
              "https://b.tile.openstreetmap.org/{z}/{x}/{y}.png",
              "https://c.tile.openstreetmap.org/{z}/{x}/{y}.png",
            ],
            tileSize: 256,
            attribution:
              '&copy; <a href="https://openstreetmap.org">OpenStreetMap</a> contributors',
          },
        },
        layers: [
          {
            id: "osm-tiles",
            type: "raster",
            source: "osm",
            minzoom: 0,
            maxzoom: 19,
          },
        ],
      },
      center: [7.4914, 9.0765], // Nigeria center
      zoom: 6,
      minZoom: 3,
      maxZoom: 14,
      attributionControl: false,
    });

    // Add attribution in bottom-left
    map.current.addControl(
      new maplibregl.AttributionControl({ compact: true }),
      "bottom-left"
    );

    // Add navigation controls
    map.current.addControl(
      new maplibregl.NavigationControl({ showCompass: false }),
      "top-right"
    );

    // Add scale control
    map.current.addControl(
      new maplibregl.ScaleControl({ maxWidth: 200, unit: "metric" }),
      "bottom-right"
    );

    // Change cursor on hover over districts
    map.current.on("mouseenter", "districts-fill", () => {
      map.current?.getCanvas().style.setProperty("cursor", "pointer");
    });
    map.current.on("mouseleave", "districts-fill", () => {
      map.current?.getCanvas().style.setProperty("cursor", "");
    });

    return () => {
      map.current?.remove();
      map.current = null;
    };
  }, []);

  // Add/update district layer when resistance data loads
  useEffect(() => {
    if (!map.current || !resistanceData?.geojson) return;

    const mapInstance = map.current;
    const sourceId = "districts";
    const fillLayerId = "districts-fill";
    const lineLayerId = "districts-line";

    // Remove existing layers/sources
    if (mapInstance.getLayer(fillLayerId)) mapInstance.removeLayer(fillLayerId);
    if (mapInstance.getLayer(lineLayerId)) mapInstance.removeLayer(lineLayerId);
    if (mapInstance.getSource(sourceId)) mapInstance.removeSource(sourceId);

    // Add GeoJSON source
    mapInstance.addSource(sourceId, {
      type: "geojson",
      data: resistanceData.geojson,
      generateId: true,
    });

    // Fill layer with resistance color expression
    mapInstance.addLayer({
      id: fillLayerId,
      type: "fill",
      source: sourceId,
      paint: {
        "fill-color": getResistanceExpression(),
        "fill-opacity": [
          "case",
          ["boolean", ["feature-state", "hover"], false],
          0.8,
          0.6,
        ],
      },
    });

    // Border line
    mapInstance.addLayer({
      id: lineLayerId,
      type: "line",
      source: sourceId,
      paint: {
        "line-color": "#ffffff",
        "line-width": [
          "case",
          ["boolean", ["feature-state", "hover"], false],
          2.5,
          1,
        ],
        "line-opacity": 0.8,
      },
    });

    // Click handler — show popup
    mapInstance.on("click", fillLayerId, (e) => {
      if (!e.features?.[0]?.properties) return;
      const props = e.features[0].properties as ResistancePopupData & {
        district: string;
        state: string;
        resistance_pct: number;
        total_cases: number;
      };

      const coords = e.lngLat;

      const popupHtml = `
        <div class="p-4 min-w-[240px]">
          <h3 class="font-semibold text-sm text-foreground">${props.district}</h3>
          <p class="text-xs text-muted-foreground mb-3">${props.state}</p>

          <div class="space-y-2">
            <div class="flex justify-between items-center">
              <span class="text-xs text-muted-foreground">Resistance</span>
              <span class="text-sm font-bold" style="color: ${getResistanceColor(props.resistance_pct)}">
                ${getResistanceEmoji(props.resistance_pct)} ${props.resistance_pct.toFixed(1)}%
              </span>
            </div>
            <div class="flex justify-between items-center">
              <span class="text-xs text-muted-foreground">Level</span>
              <span class="text-xs font-medium">${getResistanceLabel(props.resistance_pct)}</span>
            </div>
            <div class="flex justify-between items-center">
              <span class="text-xs text-muted-foreground">Cases</span>
              <span class="text-sm font-medium">${props.total_cases}</span>
            </div>
          </div>

          <div class="mt-3 pt-3 border-t">
            <button class="text-xs text-primary hover:underline font-medium"
                    onclick="window.location.href='/cases?district=${encodeURIComponent(props.district)}'">
              View Cases →
            </button>
          </div>
        </div>
      `;

      if (popup.current) popup.current.remove();

      popup.current = new maplibregl.Popup({
        closeButton: true,
        closeOnClick: false,
        maxWidth: "300px",
        offset: 12,
        className: "udara-map-popup",
      })
        .setLngLat(coords)
        .setHTML(popupHtml)
        .addTo(mapInstance);
    });

    // Hover state for feature highlighting
    let hoveredFeatureId: string | number | null = null;

    mapInstance.on("mousemove", fillLayerId, (e) => {
      if (e.features?.[0]?.id) {
        if (hoveredFeatureId !== null) {
          mapInstance.setFeatureState(
            { source: sourceId, id: hoveredFeatureId },
            { hover: false }
          );
        }
        hoveredFeatureId = e.features[0].id;
        mapInstance.setFeatureState(
          { source: sourceId, id: hoveredFeatureId },
          { hover: true }
        );
      }
    });

    mapInstance.on("mouseleave", fillLayerId, () => {
      if (hoveredFeatureId !== null) {
        mapInstance.setFeatureState(
          { source: sourceId, id: hoveredFeatureId },
          { hover: false }
        );
      }
      hoveredFeatureId = null;
    });

    // Fit bounds to data
    const bounds = new maplibregl.LngLatBounds();
    resistanceData.geojson.features.forEach((f: DistrictFeature) => {
      // Simplified: fit to a reasonable view
    });
    if (resistanceData.geojson.features.length > 0) {
      mapInstance.fitBounds(resistanceData.bounds, { padding: 50 });
    }
  }, [resistanceData]);

  const handleFullscreen = useCallback(() => {
    if (!mapContainer.current) return;
    if (!isFullscreen) {
      mapContainer.current.requestFullscreen?.();
    } else {
      document.exitFullscreen?.();
    }
    setIsFullscreen(!isFullscreen);
  }, [isFullscreen]);

  const handleReset = useCallback(() => {
    map.current?.flyTo({ center: [7.4914, 9.0765], zoom: 6, duration: 1000 });
  }, []);

  return (
    <Card className={isFullscreen ? "fixed inset-0 z-50 rounded-none" : ""}>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <div className="flex items-center gap-2">
          <CardTitle className="text-base font-semibold">
            Resistance Surveillance Map
          </CardTitle>
          {selectedDrug !== "all" && (
            <Badge variant="secondary" className="text-xs">
              {selectedDrug}
            </Badge>
          )}
        </div>
        <div className="flex items-center gap-1">
          <Button variant="ghost" size="icon" className="h-8 w-8" onClick={handleFullscreen}>
            <Maximize2 className="h-4 w-4" />
          </Button>
          <Button variant="ghost" size="icon" className="h-8 w-8" onClick={handleReset}>
            <RotateCcw className="h-4 w-4" />
          </Button>
        </div>
      </CardHeader>
      <CardContent className="p-0 relative">
        {/* Drug filter pills */}
        <div className="absolute top-2 left-2 z-10 flex flex-wrap gap-1 bg-background/80 backdrop-blur-sm rounded-lg p-2">
          {["all", "Amoxicillin", "Ciprofloxacin", "Ceftriaxone", "Co-trimoxazole"].map(
            (drug) => (
              <button
                key={drug}
                onClick={() => setSelectedDrug(drug)}
                className={`px-2 py-1 rounded-md text-xs font-medium transition-colors ${
                  selectedDrug === drug
                    ? "bg-primary text-primary-foreground"
                    : "bg-muted text-muted-foreground hover:bg-muted/80"
                }`}
              >
                {drug === "all" ? "All Drugs" : drug}
              </button>
            )
          )}
        </div>

        {/* Map container */}
        <div
          ref={mapContainer}
          className="h-[500px] w-full"
          style={{ display: isLoading ? "none" : "block" }}
        />

        {/* Legend */}
        <div className="absolute bottom-4 left-4 z-10 bg-background/90 backdrop-blur-sm rounded-lg p-3 shadow-sm">
          <div className="text-[10px] uppercase tracking-wider text-muted-foreground font-medium mb-2">
            Resistance Level
          </div>
          <div className="flex items-center gap-3 text-xs">
            <div className="flex items-center gap-1.5">
              <div className="h-3 w-3 rounded-sm bg-[#10b981]" />
              <span>Safe (&lt;30%)</span>
            </div>
            <div className="flex items-center gap-1.5">
              <div className="h-3 w-3 rounded-sm bg-[#3b82f6]" />
              <span>Mod (30-50%)</span>
            </div>
            <div className="flex items-center gap-1.5">
              <div className="h-3 w-3 rounded-sm bg-[#f59e0b]" />
              <span>Warn (50-70%)</span>
            </div>
            <div className="flex items-center gap-1.5">
              <div className="h-3 w-3 rounded-sm bg-[#ef4444]" />
              <span>Crit (&gt;70%)</span>
            </div>
          </div>
        </div>

        {/* Loading state */}
        {isLoading && (
          <div className="h-[500px] w-full flex items-center justify-center">
            <div className="text-center">
              <div className="skeleton-shimmer h-4 w-32 rounded mx-auto mb-2" />
              <p className="text-xs text-muted-foreground">Loading resistance data...</p>
            </div>
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

### 5.4 Stats Cards — `components/dashboard/stats-cards.tsx`

```tsx
"use client";

import React from "react";
import {
  Activity,
  AlertTriangle,
  TrendingUp,
  TrendingDown,
  ShieldAlert,
  Users,
  Microscope,
  ArrowUpRight,
  ArrowDownRight,
} from "lucide-react";
import { Card, CardContent } from "@/components/ui/card";
import { cn } from "@/lib/utils";
import { useStats } from "@/hooks/use-stats";

interface KpiData {
  title: string;
  value: string | number;
  change: number;
  changeLabel: string;
  icon: React.ElementType;
  iconBgClass: string;
  iconColorClass: string;
  format?: "number" | "percentage" | "plain";
}

interface StatsResponse {
  total_cases: number;
  total_cases_change: number;
  active_alerts: number;
  active_alerts_change: number;
  avg_resistance_pct: number;
  avg_resistance_change: number;
  active_chws: number;
  active_chws_change: number;
  cases_this_week: number;
  period: string;
}

function StatCard({
  title,
  value,
  change,
  changeLabel,
  icon: Icon,
  iconBgClass,
  iconColorClass,
  format = "plain",
}: KpiData) {
  const isPositive = change >= 0;
  // For alerts, increase is bad (show red); for others, increase is good (show green)
  const isAlertCard = title.toLowerCase().includes("alert");

  return (
    <Card className="relative overflow-hidden group hover:shadow-md transition-shadow">
      {/* Subtle accent bar */}
      <div className={cn("absolute top-0 left-0 right-0 h-1", iconBgClass)} />

      <CardContent className="p-5">
        <div className="flex items-start justify-between">
          <div className="space-y-1 flex-1">
            <p className="text-xs font-medium text-muted-foreground uppercase tracking-wider">
              {title}
            </p>
            <div className="flex items-baseline gap-2">
              <span className="text-2xl font-bold tracking-tight text-foreground">
                {format === "number"
                  ? Number(value).toLocaleString()
                  : format === "percentage"
                  ? `${Number(value).toFixed(1)}%`
                  : value}
              </span>
            </div>
            {/* Trend indicator */}
            <div className="flex items-center gap-1.5 pt-1">
              {isPositive ? (
                <ArrowUpRight
                  className={cn(
                    "h-3.5 w-3.5",
                    isAlertCard ? "text-udara-resistance-critical" : "text-emerald-500"
                  )}
                />
              ) : (
                <ArrowDownRight
                  className={cn(
                    "h-3.5 w-3.5",
                    isAlertCard ? "text-emerald-500" : "text-udara-resistance-critical"
                  )}
                />
              )}
              <span
                className={cn(
                  "text-xs font-medium",
                  isAlertCard
                    ? isPositive
                      ? "text-udara-resistance-critical"
                      : "text-emerald-600"
                    : isPositive
                    ? "text-emerald-600"
                    : "text-udara-resistance-critical"
                )}
              >
                {isPositive ? "+" : ""}
                {change.toFixed(1)}%
              </span>
              <span className="text-[11px] text-muted-foreground">{changeLabel}</span>
            </div>
          </div>

          {/* Icon */}
          <div
            className={cn(
              "flex h-10 w-10 shrink-0 items-center justify-center rounded-lg",
              iconBgClass
            )}
          >
            <Icon className={cn("h-5 w-5", iconColorClass)} />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}

export function DashboardStats() {
  const { data: stats, isLoading, error } = useStats();

  if (isLoading) {
    return (
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        {[...Array(4)].map((_, i) => (
          <Card key={i} className="overflow-hidden">
            <div className="h-1 bg-muted" />
            <CardContent className="p-5 space-y-3">
              <div className="skeleton-shimmer h-3 w-24 rounded" />
              <div className="skeleton-shimmer h-7 w-20 rounded" />
              <div className="skeleton-shimmer h-3 w-32 rounded" />
            </CardContent>
          </Card>
        ))}
      </div>
    );
  }

  if (error || !stats) {
    return (
      <div className="rounded-lg border border-destructive/20 bg-destructive/5 p-4 text-sm text-destructive">
        Failed to load dashboard statistics. Retrying...
      </div>
    );
  }

  const kpis: KpiData[] = [
    {
      title: "Total Cases",
      value: stats.total_cases,
      change: stats.total_cases_change,
      changeLabel: "vs last period",
      icon: Activity,
      iconBgClass: "bg-udara.primary/10",
      iconColorClass: "text-udara.primary",
      format: "number",
    },
    {
      title: "Active Alerts",
      value: stats.active_alerts,
      change: stats.active_alerts_change,
      changeLabel: "vs last period",
      icon: AlertTriangle,
      iconBgClass: "bg-udara.resistance.critical/10",
      iconColorClass: "text-udara.resistance.critical",
      format: "number",
    },
    {
      title: "Avg Resistance",
      value: stats.avg_resistance_pct,
      change: stats.avg_resistance_change,
      changeLabel: "vs last period",
      icon: ShieldAlert,
      iconBgClass: "bg-udara.resistance.warning/10",
      iconColorClass: "text-udara.resistance.warning",
      format: "percentage",
    },
    {
      title: "Active CHWs",
      value: stats.active_chws,
      change: stats.active_chws_change,
      changeLabel: "vs last period",
      icon: Users,
      iconBgClass: "bg-indigo-500/10",
      iconColorClass: "text-indigo-500",
      format: "number",
    },
  ];

  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
      {kpis.map((kpi) => (
        <StatCard key={kpi.title} {...kpi} />
      ))}
    </div>
  );
}
```

### 5.5 Trend Chart — `components/dashboard/trend-chart.tsx`

```tsx
"use client";

import React, { useEffect, useRef, useState, useCallback } from "react";
import * as echarts from "echarts/core";
import { LineChart } from "echarts/charts";
import {
  GridComponent,
  TooltipComponent,
  LegendComponent,
  MarkLineComponent,
  DataZoomComponent,
  ToolboxComponent,
} from "echarts/components";
import { CanvasRenderer } from "echarts/renderers";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { cn } from "@/lib/utils";
import { useTrends } from "@/hooks/use-trends";

echarts.use([
  LineChart,
  GridComponent,
  TooltipComponent,
  LegendComponent,
  MarkLineComponent,
  DataZoomComponent,
  ToolboxComponent,
  CanvasRenderer,
]);

type TimeRange = "7d" | "30d" | "90d" | "12m" | "all";

const TIME_RANGES: { label: string; value: TimeRange }[] = [
  { label: "7D", value: "7d" },
  { label: "30D", value: "30d" },
  { label: "90D", value: "90d" },
  { label: "12M", value: "12m" },
  { label: "All", value: "all" },
];

const DRUG_COLORS: Record<string, string> = {
  Amoxicillin:    "#06c9af",
  Ciprofloxacin:  "#3b82f6",
  Ceftriaxone:    "#f59e0b",
  "Co-trimoxazole": "#ef4444",
  Gentamicin:     "#8b5cf6",
  Metronidazole:  "#ec4899",
};

interface TrendDataPoint {
  date: string;
  drugs: Record<string, number>;
}

export function TrendChart() {
  const chartRef = useRef<HTMLDivElement>(null);
  const chartInstance = useRef<echarts.ECharts | null>(null);
  const [timeRange, setTimeRange] = useState<TimeRange>("30d");

  const { data: trendsData, isLoading } = useTrends({
    timeRange,
    granularity: "week",
  });

  const initChart = useCallback(() => {
    if (!chartRef.current) return;

    chartInstance.current = echarts.init(chartRef.current, undefined, {
      renderer: "canvas",
    });

    const resizeObserver = new ResizeObserver(() => {
      chartInstance.current?.resize();
    });
    resizeObserver.observe(chartRef.current);

    return () => {
      resizeObserver.disconnect();
      chartInstance.current?.dispose();
    };
  }, []);

  useEffect(() => {
    const cleanup = initChart();
    return cleanup;
  }, [initChart]);

  useEffect(() => {
    if (!chartInstance.current || !trendsData?.series) return;

    const drugNames = Object.keys(trendsData.series);
    const dates = trendsData.dates;

    const series = drugNames.map((drug) => ({
      name: drug,
      type: "line" as const,
      smooth: true,
      symbol: "circle",
      symbolSize: 6,
      lineStyle: {
        width: 2.5,
        color: DRUG_COLORS[drug] || "#6b7280",
      },
      itemStyle: {
        color: DRUG_COLORS[drug] || "#6b7280",
      },
      areaStyle: {
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: `${DRUG_COLORS[drug] || "#6b7280"}40` },
          { offset: 1, color: `${DRUG_COLORS[drug] || "#6b7280"}05` },
        ]),
      },
      data: trendsData.series[drug],
      markLine: drug === drugNames[0]
        ? {
            silent: true,
            symbol: "none",
            lineStyle: { type: "dashed", width: 1.5 },
            label: { fontSize: 10, position: "insideEndTop" },
            data: [
              {
                yAxis: 30,
                name: "30% Threshold",
                lineStyle: { color: "#10b981" },
                label: { formatter: "Safe (30%)" },
              },
              {
                yAxis: 60,
                name: "60% Threshold",
                lineStyle: { color: "#ef4444" },
                label: { formatter: "Critical (60%)" },
              },
            ],
          }
        : undefined,
    }));

    const option: echarts.EChartsOption = {
      backgroundColor: "transparent",
      tooltip: {
        trigger: "axis",
        backgroundColor: "hsl(var(--card))",
        borderColor: "hsl(var(--border))",
        textStyle: {
          color: "hsl(var(--card-foreground))",
          fontSize: 12,
          fontFamily: "Inter, sans-serif",
        },
        formatter: (params: unknown) => {
          const p = params as { name: string; marker: string; seriesName: string; value: number }[];
          let html = `<div style="font-weight:600;margin-bottom:8px">${p[0].name}</div>`;
          p.forEach((item) => {
            const level =
              item.value < 30
                ? "🟢"
                : item.value < 50
                ? "🔵"
                : item.value < 70
                ? "🟡"
                : "🔴";
            html += `<div style="display:flex;align-items:center;justify-content:space-between;gap:12px;margin:4px 0">
              <span>${item.marker} ${item.seriesName}</span>
              <span style="font-weight:600">${level} ${item.value.toFixed(1)}%</span>
            </div>`;
          });
          return html;
        },
      },
      legend: {
        top: 0,
        right: 0,
        itemWidth: 12,
        itemHeight: 3,
        textStyle: {
          fontSize: 11,
          color: "hsl(var(--muted-foreground))",
          fontFamily: "Inter, sans-serif",
        },
        icon: "roundRect",
      },
      grid: {
        left: 10,
        right: 16,
        top: 48,
        bottom: 8,
        containLabel: true,
      },
      xAxis: {
        type: "category",
        data: dates,
        boundaryGap: false,
        axisLine: { lineStyle: { color: "hsl(var(--border))" } },
        axisLabel: {
          fontSize: 11,
          color: "hsl(var(--muted-foreground))",
          fontFamily: "Inter, sans-serif",
        },
        axisTick: { show: false },
      },
      yAxis: {
        type: "value",
        min: 0,
        max: 100,
        axisLabel: {
          fontSize: 11,
          color: "hsl(var(--muted-foreground))",
          formatter: "{value}%",
          fontFamily: "Inter, sans-serif",
        },
        splitLine: {
          lineStyle: { color: "hsl(var(--border))", type: "dashed" },
        },
      },
      series,
      dataZoom: [
        {
          type: "inside",
          start: 0,
          end: 100,
          zoomOnMouseWheel: true,
          moveOnMouseMove: true,
        },
      ],
    };

    chartInstance.current.setOption(option, true);
  }, [trendsData]);

  // Dark mode sync
  useEffect(() => {
    const observer = new MutationObserver(() => {
      chartInstance.current?.dispose();
      initChart();
      if (trendsData) {
        // Re-render after theme change (triggers the data useEffect)
        chartInstance.current?.setOption({}, true);
      }
    });

    observer.observe(document.documentElement, {
      attributes: true,
      attributeFilter: ["class"],
    });

    return () => observer.disconnect();
  }, [initChart, trendsData]);

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <CardTitle className="text-base font-semibold">
          Resistance Trend Analysis
        </CardTitle>
        <div className="flex items-center gap-1 bg-muted rounded-lg p-0.5">
          {TIME_RANGES.map((range) => (
            <Button
              key={range.value}
              variant="ghost"
              size="sm"
              onClick={() => setTimeRange(range.value)}
              className={cn(
                "h-7 px-2.5 text-xs font-medium rounded-md transition-all",
                timeRange === range.value
                  ? "bg-background text-foreground shadow-sm"
                  : "text-muted-foreground hover:text-foreground"
              )}
            >
              {range.label}
            </Button>
          ))}
        </div>
      </CardHeader>
      <CardContent className="p-4 pt-0">
        {isLoading ? (
          <div className="h-[350px] flex items-center justify-center">
            <div className="text-center">
              <div className="skeleton-shimmer h-4 w-32 rounded mx-auto mb-2" />
              <p className="text-xs text-muted-foreground">Loading trend data...</p>
            </div>
          </div>
        ) : (
          <div ref={chartRef} className="h-[350px] w-full" />
        )}
      </CardContent>
    </Card>
  );
}
```

### 5.6 TypeScript Types — `types/case.ts`

```typescript
/**
 * UDARA Case Types
 *
 * These interfaces mirror the backend Pydantic models exactly.
 * Any change here MUST be synced with backend schemas.
 * Generated from: backend/models/case.py
 */

// ── Enums ──────────────────────────────────────────────────────────

export type CaseSource =
  | "telegram"
  | "whatsapp"
  | "ussd"
  | "manual"
  | "api"
  | "fhir";

export type CaseOutcome =
  | "resolved"
  | "ongoing"
  | "referred"
  | "lost_to_followup"
  | "deceased";

export type CaseStatus =
  | "draft"
  | "pending_review"
  | "verified"
  | "rejected"
  | "flagged";

export type SeverityLevel =
  | "mild"
  | "moderate"
  | "severe"
  | "critical";

export type SpecimenType =
  | "blood"
  | "urine"
  | "sputum"
  | "stool"
  | "csf"
  | "wound_swab"
  | "throat_swab"
  | "other";

export type ResistanceCategory =
  | "susceptible"
  | "intermediate"
  | "resistant"
  | "not_tested";

// ── Core Models ────────────────────────────────────────────────────

export interface Symptom {
  id?: string;
  name: string;
  severity: SeverityLevel;
  onset_date?: string;
  notes?: string;
}

export interface Medication {
  id?: string;
  drug_name: string;
  dose?: string;
  frequency?: string;
  duration_days?: number;
  resistance_category: ResistanceCategory;
  resistance_pct?: number;
  prescribed_by?: string;
  start_date?: string;
}

export interface LabResult {
  id?: string;
  specimen_type: SpecimenType;
  organism?: string;
  ast_results: ASTResult[];
  lab_name?: string;
  collected_date: string;
  result_date?: string;
}

export interface ASTResult {
  antibiotic: string;
  category: ResistanceCategory;
  mic_value?: string;
  zone_diameter?: number;
}

export interface Case {
  id: string;
  external_id?: string;

  // Patient demographics
  patient_id?: string;
  patient_age_years?: number;
  patient_age_months?: number;
  patient_sex: "male" | "female" | "unknown";
  patient_location?: PatientLocation;

  // Clinical data
  chief_complaint: string;
  symptoms: Symptom[];
  medications: Medication[];
  lab_results: LabResult[];
  clinical_notes?: string;

  // Resistance analysis (computed by backend)
  resistance_summary?: {
    overall_risk: "low" | "medium" | "high" | "critical";
    resistant_drugs: string[];
    susceptible_alternatives: string[];
    confidence_score: number;
    reasoning?: string;
  };

  // Metadata
  source: CaseSource;
  status: CaseStatus;
  outcome?: CaseOutcome;
  severity: SeverityLevel;

  // Location
  location: GeoLocation;

  // Reporting
  reported_by?: {
    id: string;
    name: string;
    role: "chw" | "clinician" | "lab_tech" | "nurse";
    facility?: string;
  };

  // Timestamps
  reported_at: string;
  updated_at: string;
  reviewed_at?: string;
  reviewed_by?: string;

  // Tags & flags
  tags?: string[];
  flagged?: boolean;
  flag_reason?: string;
}

export interface PatientLocation {
  country: string;
  state: string;
  district?: string;
  facility?: string;
  village?: string;
  gps?: { latitude: number; longitude: number };
}

export interface GeoLocation {
  type: "Point";
  coordinates: [number, number]; // [longitude, latitude] per GeoJSON spec
}

// ── API Request/Response Types ────────────────────────────────────

export interface CasesFilters {
  page?: number;
  per_page?: number;
  search?: string;
  source?: CaseSource;
  status?: CaseStatus;
  outcome?: CaseOutcome;
  severity?: SeverityLevel;
  drug_name?: string;
  country?: string;
  state?: string;
  district?: string;
  date_from?: string;
  date_to?: string;
  reported_by_role?: string;
  sort_by?: string;
  sort_order?: "asc" | "desc";
}

export interface CasesResponse {
  cases: Case[];
  total: number;
  page: number;
  per_page: number;
  total_pages: number;
  aggregations?: {
    by_source: Record<CaseSource, number>;
    by_status: Record<CaseStatus, number>;
    by_severity: Record<SeverityLevel, number>;
    by_state: Record<string, number>;
  };
}

export interface CreateCaseRequest {
  patient_age_years?: number;
  patient_age_months?: number;
  patient_sex: Case["patient_sex"];
  chief_complaint: string;
  symptoms: Omit<Symptom, "id">[];
  medications?: Omit<Medication, "id">[];
  source: CaseSource;
  location?: GeoLocation;
  patient_location?: Partial<PatientLocation>;
  clinical_notes?: string;
}

export interface CaseDetailResponse {
  case: Case;
  related_cases?: Case[];
  resistance_data?: {
    district_avg: number;
    national_avg: number;
    trend_12w: number[];
  };
}
```

### 5.7 API Client — `lib/api-client.ts`

```typescript
/**
 * UDARA API Client
 *
 * Typed HTTP client with TanStack Query integration.
 * Handles auth, refresh, error mapping, and request deduplication.
 */

import type {
  CasesResponse,
  CasesFilters,
  CaseDetailResponse,
  CreateCaseRequest,
} from "@/types/case";
import type { ResistanceData } from "@/types/resistance";
import type { TrendData } from "@/types/resistance";
import type { Alert, AlertsResponse, AlertRule } from "@/types/alert";
import type { StatsResponse } from "@/types/api";

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || "https://api.udara.ai/v1";

// ── Core HTTP Client ──────────────────────────────────────────────

class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public details?: Record<string, unknown>
  ) {
    super(message);
    this.name = "ApiError";
  }
}

async function request<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const url = `${API_BASE_URL}${path}`;

  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    ...((options.headers as Record<string, string>) || {}),
  };

  // Attach auth token if available (from NextAuth or localStorage)
  if (typeof window !== "undefined") {
    const token = localStorage.getItem("udara_token");
    if (token) {
      headers["Authorization"] = `Bearer ${token}`;
    }
  }

  const response = await fetch(url, {
    ...options,
    headers,
  });

  if (!response.ok) {
    const errorBody = await response.json().catch(() => ({}));
    throw new ApiError(
      response.status,
      errorBody.code || "UNKNOWN_ERROR",
      errorBody.detail || `Request failed: ${response.status}`,
      errorBody.details
    );
  }

  // Handle 204 No Content
  if (response.status === 204) {
    return undefined as T;
  }

  return response.json();
}

// ── API Methods ───────────────────────────────────────────────────

export const api = {
  // ── Cases ────────────────────────────────────────────
  cases: {
    list: (filters?: CasesFilters): Promise<CasesResponse> =>
      request("/cases", {
        method: "GET",
        // Serialize filters as query params
      }),

    get: (id: string): Promise<CaseDetailResponse> =>
      request(`/cases/${id}`),

    create: (data: CreateCaseRequest): Promise<{ case: { id: string } }> =>
      request("/cases", {
        method: "POST",
        body: JSON.stringify(data),
      }),

    update: (id: string, data: Partial<CreateCaseRequest>): Promise<void> =>
      request(`/cases/${id}`, {
        method: "PATCH",
        body: JSON.stringify(data),
      }),

    delete: (id: string): Promise<void> =>
      request(`/cases/${id}`, { method: "DELETE" }),

    export: (filters: CasesFilters, format: "csv" | "pdf"): Promise<Blob> =>
      request(`/cases/export?format=${format}`, {
        method: "POST",
        body: JSON.stringify(filters),
      }).then((res) => res as unknown as Blob),
  },

  // ── Resistance ───────────────────────────────────────
  resistance: {
    getMap: (params: {
      drug?: string;
      country?: string;
      state?: string;
      granularity?: "country" | "state" | "district";
    }): Promise<ResistanceData> =>
      request(
        `/resistance/map?${new URLSearchParams(
          params as Record<string, string>
        ).toString()}`
      ),

    getDrugs: (): Promise<{ drugs: string[] }> =>
      request("/resistance/drugs"),

    getDistrict: (
      districtId: string,
      drug?: string
    ): Promise<ResistanceData> =>
      request(
        `/resistance/districts/${districtId}${drug ? `?drug=${drug}` : ""}`
      ),
  },

  // ── Trends ───────────────────────────────────────────
  trends: {
    get: (params: {
      timeRange: string;
      granularity: "day" | "week" | "month";
      drugs?: string[];
      country?: string;
      state?: string;
    }): Promise<TrendData> =>
      request(
        `/trends?${new URLSearchParams(
          params as Record<string, string>
        ).toString()}`
      ),
  },

  // ── Alerts ───────────────────────────────────────────
  alerts: {
    list: (params?: {
      page?: number;
      per_page?: number;
      severity?: string;
      status?: "active" | "acknowledged" | "resolved";
    }): Promise<AlertsResponse> =>
      request(`/alerts?${new URLSearchParams(params as Record<string, string>).toString()}`),

    get: (id: string): Promise<Alert> =>
      request(`/alerts/${id}`),

    acknowledge: (id: string): Promise<void> =>
      request(`/alerts/${id}/acknowledge`, { method: "POST" }),

    resolve: (id: string, notes?: string): Promise<void> =>
      request(`/alerts/${id}/resolve`, {
        method: "POST",
        body: JSON.stringify({ notes }),
      }),

    getRules: (): Promise<AlertRule[]> =>
      request("/alerts/rules"),

    createRule: (rule: Omit<AlertRule, "id">): Promise<AlertRule> =>
      request("/alerts/rules", {
        method: "POST",
        body: JSON.stringify(rule),
      }),
  },

  // ── Stats ────────────────────────────────────────────
  stats: {
    getDashboard: (params?: {
      country?: string;
      state?: string;
    }): Promise<StatsResponse> =>
      request(
        `/stats/dashboard?${new URLSearchParams(
          (params as Record<string, string>) || {}
        ).toString()}`
      ),
  },

  // ── Health ───────────────────────────────────────────
  health: (): Promise<{ status: string; version: string; uptime: number }> =>
    request("/health"),
};

// ── Query Key Factory ──────────────────────────────────────────────

export const queryKeys = {
  cases: {
    all: ["cases"] as const,
    list: (filters?: CasesFilters) =>
      ["cases", "list", filters] as const,
    detail: (id: string) => ["cases", "detail", id] as const,
  },
  resistance: {
    map: (params: object) => ["resistance", "map", params] as const,
    drugs: () => ["resistance", "drugs"] as const,
    district: (id: string, drug?: string) =>
      ["resistance", "district", id, drug] as const,
  },
  trends: {
    all: (params: object) => ["trends", params] as const,
  },
  alerts: {
    all: (params?: object) => ["alerts", params] as const,
    detail: (id: string) => ["alerts", id] as const,
    rules: () => ["alerts", "rules"] as const,
  },
  stats: {
    dashboard: (params?: object) => ["stats", "dashboard", params] as const,
  },
} as const;

export { ApiError };
export default api;
```

### 5.8 Cases Table — `components/cases/cases-table.tsx`

```tsx
"use client";

import React, { useMemo, useState } from "react";
import {
  ColumnDef,
  flexRender,
  getCoreRowModel,
  getSortedRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  SortingState,
  ColumnFiltersState,
  useReactTable,
} from "@tanstack/react-table";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Badge } from "@/components/ui/badge";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import {
  ArrowUpDown,
  ChevronLeft,
  ChevronRight,
  ChevronsLeft,
  ChevronsRight,
  Search,
  Filter,
  MoreHorizontal,
  Eye,
  ChevronDown,
} from "lucide-react";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { cn } from "@/lib/utils";
import { useCases } from "@/hooks/use-cases";
import type { Case, CaseSource, CaseOutcome, CaseStatus, SeverityLevel } from "@/types/case";
import { CaseDetailDialog } from "./case-detail-dialog";

// ── Badge Components ───────────────────────────────────────────────

const SOURCE_CONFIG: Record<
  CaseSource,
  { label: string; className: string }
> = {
  telegram: {
    label: "Telegram",
    className: "bg-[#229ED9]/10 text-[#229ED9] border-[#229ED9]/20",
  },
  whatsapp: {
    label: "WhatsApp",
    className: "bg-[#25D366]/10 text-[#25D366] border-[#25D366]/20",
  },
  ussd: {
    label: "USSD",
    className: "bg-[#8B5CF6]/10 text-[#8B5CF6] border-[#8B5CF6]/20",
  },
  manual: {
    label: "Manual",
    className: "bg-[#6366F1]/10 text-[#6366F1] border-[#6366F1]/20",
  },
  api: {
    label: "API",
    className: "bg-[#EC4899]/10 text-[#EC4899] border-[#EC4899]/20",
  },
  fhir: {
    label: "FHIR",
    className: "bg-[#F59E0B]/10 text-[#F59E0B] border-[#F59E0B]/20",
  },
};

const OUTCOME_CONFIG: Record<
  CaseOutcome,
  { label: string; className: string }
> = {
  resolved: {
    label: "Resolved",
    className: "bg-emerald-100 text-emerald-700 border-emerald-200",
  },
  ongoing: {
    label: "Ongoing",
    className: "bg-blue-100 text-blue-700 border-blue-200",
  },
  referred: {
    label: "Referred",
    className: "bg-amber-100 text-amber-700 border-amber-200",
  },
  lost_to_followup: {
    label: "Lost",
    className: "bg-gray-100 text-gray-700 border-gray-200",
  },
  deceased: {
    label: "Deceased",
    className: "bg-red-100 text-red-700 border-red-200",
  },
};

const SEVERITY_CONFIG: Record<SeverityLevel, { icon: string; className: string }> = {
  mild:     { icon: "🟢", className: "text-emerald-600" },
  moderate: { icon: "🟡", className: "text-amber-600" },
  severe:   { icon: "🟠", className: "text-orange-600" },
  critical: { icon: "🔴", className: "text-red-600" },
};

// ── Column Definitions ─────────────────────────────────────────────

const columns: ColumnDef<Case>[] = [
  {
    accessorKey: "reported_at",
    header: ({ column }) => (
      <Button
        variant="ghost"
        size="sm"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
        className="-ml-3"
      >
        Date
        <ArrowUpDown className="ml-1 h-3.5 w-3.5" />
      </Button>
    ),
    cell: ({ getValue }) => {
      const date = new Date(getValue<string>());
      return (
        <span className="text-sm text-muted-foreground">
          {date.toLocaleDateString("en-US", {
            month: "short",
            day: "numeric",
            year: "numeric",
          })}
          <br />
          <span className="text-xs">
            {date.toLocaleTimeString("en-US", {
              hour: "2-digit",
              minute: "2-digit",
            })}
          </span>
        </span>
      );
    },
    size: 120,
  },
  {
    accessorKey: "id",
    header: "Case ID",
    cell: ({ getValue }) => (
      <span className="font-mono text-xs text-muted-foreground">
        {getValue<string>().slice(0, 8)}...
      </span>
    ),
    size: 100,
  },
  {
    accessorKey: "patient_sex",
    header: "Sex",
    cell: ({ getValue }) => (
      <span className="text-sm capitalize">{getValue<string>()}</span>
    ),
    size: 60,
  },
  {
    accessorKey: "patient_age_years",
    header: "Age",
    cell: ({ row }) => {
      const years = row.original.patient_age_years;
      const months = row.original.patient_age_months;
      if (years) return <span className="text-sm">{years}y</span>;
      if (months) return <span className="text-sm">{months}m</span>;
      return <span className="text-sm text-muted-foreground">—</span>;
    },
    size: 60,
  },
  {
    accessorKey: "chief_complaint",
    header: "Complaint",
    cell: ({ getValue }) => (
      <span className="text-sm line-clamp-2 max-w-[200px]">
        {getValue<string>()}
      </span>
    ),
    size: 200,
  },
  {
    accessorKey: "severity",
    header: "Severity",
    cell: ({ getValue }) => {
      const sev = getValue<SeverityLevel>();
      const config = SEVERITY_CONFIG[sev];
      return (
        <span className={cn("text-sm font-medium", config.className)}>
          {config.icon} {sev.charAt(0).toUpperCase() + sev.slice(1)}
        </span>
      );
    },
    size: 100,
    filterFn: (row, id, value) => value.includes(row.getValue(id)),
  },
  {
    accessorKey: "source",
    header: "Source",
    cell: ({ getValue }) => {
      const src = getValue<CaseSource>();
      const config = SOURCE_CONFIG[src];
      return (
        <Badge variant="outline" className={cn("text-[10px] font-medium", config.className)}>
          {config.label}
        </Badge>
      );
    },
    size: 100,
    filterFn: (row, id, value) => value.includes(row.getValue(id)),
  },
  {
    accessorKey: "outcome",
    header: "Outcome",
    cell: ({ getValue }) => {
      const outcome = getValue<CaseOutcome | undefined>();
      if (!outcome) return <span className="text-sm text-muted-foreground">—</span>;
      const config = OUTCOME_CONFIG[outcome];
      return (
        <Badge variant="outline" className={cn("text-[10px] font-medium", config.className)}>
          {config.label}
        </Badge>
      );
    },
    size: 100,
  },
  {
    accessorKey: "resistance_summary",
    header: "Risk",
    cell: ({ getValue }) => {
      const summary = getValue<Case["resistance_summary"]>();
      if (!summary) return <span className="text-sm text-muted-foreground">—</span>;
      const colors: Record<string, string> = {
        low: "text-emerald-600 bg-emerald-50",
        medium: "text-blue-600 bg-blue-50",
        high: "text-amber-600 bg-amber-50",
        critical: "text-red-600 bg-red-50",
      };
      return (
        <span className={cn("text-xs font-semibold px-2 py-0.5 rounded-full", colors[summary.overall_risk])}>
          {summary.overall_risk.toUpperCase()}
          <span className="ml-1 text-[10px] font-normal text-muted-foreground">
            {Math.round(summary.confidence_score * 100)}%
          </span>
        </span>
      );
    },
    size: 90,
  },
  {
    id: "actions",
    cell: ({ row }) => {
      // Actions handled by parent
      return null;
    },
    size: 50,
  },
];

// ── Table Component ────────────────────────────────────────────────

export function CasesTable() {
  const [sorting, setSorting] = useState<SortingState>([
    { id: "reported_at", desc: true },
  ]);
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([]);
  const [globalFilter, setGlobalFilter] = useState("");
  const [selectedCase, setSelectedCase] = useState<Case | null>(null);
  const [pagination, setPagination] = useState({
    pageIndex: 0,
    pageSize: 20,
  });

  const { data, isLoading, error } = useCases({
    page: pagination.pageIndex + 1,
    per_page: pagination.pageSize,
    search: globalFilter || undefined,
    sort_by: sorting[0]?.id,
    sort_order: sorting[0]?.desc ? "desc" : "asc",
  });

  const tableData = useMemo(() => data?.cases ?? [], [data]);
  const totalCount = data?.total ?? 0;

  const table = useReactTable({
    data: tableData,
    columns,
    state: {
      sorting,
      columnFilters,
      globalFilter,
      pagination,
    },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onGlobalFilterChange: setGlobalFilter,
    onPaginationChange: setPagination,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    manualPagination: true,
    pageCount: Math.ceil(totalCount / pagination.pageSize),
  });

  return (
    <div className="space-y-4">
      {/* Toolbar */}
      <div className="flex items-center gap-3">
        <div className="relative flex-1 max-w-sm">
          <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
          <Input
            placeholder="Search cases..."
            value={globalFilter}
            onChange={(e) => setGlobalFilter(e.target.value)}
            className="pl-9"
          />
        </div>

        <Select onValueChange={(value) => setColumnFilters([{ id: "source", value }])}>
          <SelectTrigger className="w-[130px]">
            <SelectValue placeholder="Source" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Sources</SelectItem>
            {Object.entries(SOURCE_CONFIG).map(([key, config]) => (
              <SelectItem key={key} value={key}>
                {config.label}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>

        <Select onValueChange={(value) => setColumnFilters([{ id: "severity", value }])}>
          <SelectTrigger className="w-[130px]">
            <SelectValue placeholder="Severity" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All</SelectItem>
            {Object.keys(SEVERITY_CONFIG).map((sev) => (
              <SelectItem key={sev} value={sev}>
                {sev.charAt(0).toUpperCase() + sev.slice(1)}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>

        <div className="ml-auto text-sm text-muted-foreground">
          {totalCount.toLocaleString()} cases
        </div>
      </div>

      {/* Table */}
      <div className="rounded-lg border overflow-hidden">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <TableHead
                    key={header.id}
                    style={{ width: header.getSize() }}
                  >
                    {header.isPlaceholder
                      ? null
                      : flexRender(
                          header.column.columnDef.header,
                          header.getContext()
                        )}
                  </TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {isLoading ? (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-48 text-center">
                  <div className="flex items-center justify-center gap-2">
                    <div className="h-4 w-4 animate-spin rounded-full border-2 border-primary border-t-transparent" />
                    <span className="text-sm text-muted-foreground">
                      Loading cases...
                    </span>
                  </div>
                </TableCell>
              </TableRow>
            ) : table.getRowModel().rows.length === 0 ? (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-48 text-center">
                  <div className="text-muted-foreground">
                    <p className="font-medium">No cases found</p>
                    <p className="text-sm mt-1">
                      Try adjusting your search or filters
                    </p>
                  </div>
                </TableCell>
              </TableRow>
            ) : (
              table.getRowModel().rows.map((row) => (
                <TableRow
                  key={row.original.id}
                  className="cursor-pointer hover:bg-muted/50 transition-colors"
                  onClick={() => setSelectedCase(row.original)}
                >
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>
                      {flexRender(
                        cell.column.columnDef.cell,
                        cell.getContext()
                      )}
                    </TableCell>
                  ))}
                </TableRow>
              ))
            )}
          </TableBody>
        </Table>
      </div>

      {/* Pagination */}
      <div className="flex items-center justify-between">
        <div className="text-sm text-muted-foreground">
          Page {pagination.pageIndex + 1} of {table.getPageCount() || 1}
        </div>
        <div className="flex items-center gap-1">
          <Button
            variant="outline"
            size="icon"
            className="h-8 w-8"
            onClick={() => table.setPageIndex(0)}
            disabled={!table.getCanPreviousPage()}
          >
            <ChevronsLeft className="h-4 w-4" />
          </Button>
          <Button
            variant="outline"
            size="icon"
            className="h-8 w-8"
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
          >
            <ChevronLeft className="h-4 w-4" />
          </Button>
          <Button
            variant="outline"
            size="icon"
            className="h-8 w-8"
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
          >
            <ChevronRight className="h-4 w-4" />
          </Button>
          <Button
            variant="outline"
            size="icon"
            className="h-8 w-8"
            onClick={() => table.setPageIndex(table.getPageCount() - 1)}
            disabled={!table.getCanNextPage()}
          >
            <ChevronsRight className="h-4 w-4" />
          </Button>
        </div>
      </div>

      {/* Case Detail Dialog */}
      {selectedCase && (
        <CaseDetailDialog
          caseData={selectedCase}
          open={!!selectedCase}
          onClose={() => setSelectedCase(null)}
        />
      )}
    </div>
  );
}
```

### 5.9 Loading Skeleton — `components/dashboard/loading-skeleton.tsx`

```tsx
"use client";

import React from "react";
import { Card, CardContent, CardHeader } from "@/components/ui/card";

export function DashboardSkeleton() {
  return (
    <div className="space-y-6 animate-fade-in-up">
      {/* Stats cards skeleton */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        {[...Array(4)].map((_, i) => (
          <Card key={i} className="overflow-hidden">
            <div className="h-1 skeleton-shimmer" />
            <CardContent className="p-5 space-y-3">
              <div className="skeleton-shimmer h-3 w-28 rounded" />
              <div className="skeleton-shimmer h-8 w-24 rounded" />
              <div className="skeleton-shimmer h-3 w-36 rounded" />
            </CardContent>
          </Card>
        ))}
      </div>

      {/* Map + Chart row skeleton */}
      <div className="grid grid-cols-1 lg:grid-cols-5 gap-6">
        {/* Map skeleton */}
        <Card className="lg:col-span-3">
          <CardHeader className="pb-2">
            <div className="skeleton-shimmer h-5 w-48 rounded" />
          </CardHeader>
          <CardContent className="p-4 pt-0">
            <div className="skeleton-shimmer h-[500px] w-full rounded-lg" />
          </CardContent>
        </Card>

        {/* Chart skeleton */}
        <Card className="lg:col-span-2">
          <CardHeader className="pb-2">
            <div className="skeleton-shimmer h-5 w-40 rounded" />
          </CardHeader>
          <CardContent className="p-4 pt-0">
            <div className="skeleton-shimmer h-[500px] w-full rounded-lg" />
          </CardContent>
        </Card>
      </div>

      {/* Recent cases skeleton */}
      <Card>
        <CardHeader className="pb-2">
          <div className="skeleton-shimmer h-5 w-36 rounded" />
        </CardHeader>
        <CardContent className="p-4 pt-0">
          <div className="space-y-3">
            {[...Array(5)].map((_, i) => (
              <div
                key={i}
                className="flex items-center gap-4 p-3 rounded-lg bg-muted/30"
              >
                <div className="skeleton-shimmer h-10 w-10 rounded-full" />
                <div className="flex-1 space-y-2">
                  <div className="skeleton-shimmer h-4 w-3/4 rounded" />
                  <div className="skeleton-shimmer h-3 w-1/2 rounded" />
                </div>
                <div className="skeleton-shimmer h-6 w-16 rounded-full" />
              </div>
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

export function MapSkeleton() {
  return (
    <Card>
      <CardHeader className="pb-2">
        <div className="flex items-center gap-2">
          <div className="skeleton-shimmer h-5 w-48 rounded" />
          <div className="skeleton-shimmer h-6 w-20 rounded-full" />
        </div>
      </CardHeader>
      <CardContent className="p-4 pt-0">
        <div className="relative">
          <div className="skeleton-shimmer h-[500px] w-full rounded-lg" />
          {/* Simulated map controls */}
          <div className="absolute top-3 right-3 space-y-1">
            <div className="skeleton-shimmer h-8 w-8 rounded" />
            <div className="skeleton-shimmer h-8 w-8 rounded" />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}

export function ChartSkeleton() {
  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between pb-2">
        <div className="skeleton-shimmer h-5 w-40 rounded" />
        <div className="flex gap-1">
          {[...Array(5)].map((_, i) => (
            <div key={i} className="skeleton-shimmer h-7 w-12 rounded-md" />
          ))}
        </div>
      </CardHeader>
      <CardContent className="p-4 pt-0">
        <div className="skeleton-shimmer h-[350px] w-full rounded-lg" />
      </CardContent>
    </Card>
  );
}

export function TableSkeleton({ rows = 8, cols = 7 }: { rows?: number; cols?: number }) {
  return (
    <div className="rounded-lg border overflow-hidden">
      {/* Header */}
      <div className="bg-muted/30 px-4 py-3 flex gap-4 border-b">
        {[...Array(cols)].map((_, i) => (
          <div
            key={i}
            className="skeleton-shimmer h-4 rounded"
            style={{ width: `${60 + Math.random() * 40}%`, flex: 1 }}
          />
        ))}
      </div>
      {/* Rows */}
      <div className="divide-y">
        {[...Array(rows)].map((_, i) => (
          <div key={i} className="px-4 py-3 flex gap-4 items-center">
            {[...Array(cols)].map((_, j) => (
              <div
                key={j}
                className="skeleton-shimmer h-4 rounded"
                style={{ width: `${40 + Math.random() * 60}%`, flex: 1 }}
              />
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 6. API Client & Data Layer

### 6.1 TanStack Query Hooks — `hooks/use-cases.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { api, queryKeys } from "@/lib/api-client";
import type { CasesFilters, CaseDetailResponse, CreateCaseRequest } from "@/types/case";

export function useCases(filters?: CasesFilters) {
  return useQuery({
    queryKey: queryKeys.cases.list(filters),
    queryFn: () => api.cases.list(filters),
    staleTime: 30_000,        // 30 seconds
    gcTime: 5 * 60_000,       // 5 minutes (formerly cacheTime)
    refetchOnWindowFocus: true,
    refetchInterval: 60_000,  // Auto-refresh every minute
  });
}

export function useCase(id: string) {
  return useQuery({
    queryKey: queryKeys.cases.detail(id),
    queryFn: () => api.cases.get(id),
    staleTime: 60_000,
    enabled: !!id,
  });
}

export function useCreateCase() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateCaseRequest) => api.cases.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.cases.all });
      queryClient.invalidateQueries({ queryKey: queryKeys.stats.dashboard() });
    },
  });
}
```

### 6.2 Example — `hooks/use-stats.ts`

```typescript
import { useQuery } from "@tanstack/react-query";
import { api, queryKeys } from "@/lib/api-client";

interface StatsResponse {
  total_cases: number;
  total_cases_change: number;
  active_alerts: number;
  active_alerts_change: number;
  avg_resistance_pct: number;
  avg_resistance_change: number;
  active_chws: number;
  active_chws_change: number;
  cases_this_week: number;
  period: string;
}

export function useStats(params?: { country?: string; state?: string }) {
  return useQuery({
    queryKey: queryKeys.stats.dashboard(params),
    queryFn: () => api.stats.getDashboard(params),
    staleTime: 15_000,        // 15 seconds — stats should be fresh
    gcTime: 2 * 60_000,
    refetchOnWindowFocus: true,
    refetchInterval: 30_000,  // Refresh every 30s
  });
}
```

---

## 7. Data Visualization

### MapLibre GL Choropleth — Color Interpolation

```
  Resistance %
  ┌──────────────────────────────────────────┐
100 │ ████████████████████ Dark Red (#dc2626) │
 80 │ ████████████████████ Red (#ef4444)      │
 70 │ ─ ─ ─ ─ ─ CRITICAL THRESHOLD ─ ─ ─ ─ │
 60 │ ████████████████████ Red (#ef4444)      │
 50 │ ─ ─ ─ ─ ─ WARNING THRESHOLD ─ ─ ─ ─  │
 40 │ ████████████████████ Amber (#f59e0b)    │
 30 │ ─ ─ ─ ─ ─ MODERATE THRESHOLD ─ ─ ─ ─ │
 20 │ ████████████████████ Blue (#3b82f6)     │
 10 │ ████████████████████ Green (#10b981)    │
  0 │ ████████████████████ Green (#10b981)    │
  └──────────────────────────────────────────┘
```

### ECharts Series Configuration (Drug Resistance Trends)

```
  Resistance %
  100│                                        ╱
   80│                              ╱─────────╱  ← Ceftriaxone
   60│ - - - - CRITICAL - - - - -╱
   50│                   ╱──────╱               ← Ciprofloxacin
   40│           ╱─────╱
   30│ - - - - SAFE - - -╱
   20│     ╱───╱
   10│ ╱─╱     ← Amoxicillin
    0│╱
    └───┬───┬───┬───┬───┬───┬───┬───┬───┬───→
      W1  W2  W3  W4  W5  W6  W7  W8  W9 W10

  Legend: ● Amoxicillin  ● Ciprofloxacin  ● Ceftriaxone
```

---

## 8. Week-by-Week Plan

| Week | Focus | Deliverables |
|------|-------|-------------|
| **W1** | Project setup & design system | Next.js 14 scaffold, Tailwind config with UDARA tokens, CSS variables, Radix UI primitives, Storybook setup, Figma tokens sync |
| **W2** | Layout & navigation | AppSidebar, Header, Breadcrumbs, Auth layout (login/register), Dashboard layout, responsive breakpoints |
| **W3** | Dashboard core | StatsCards, ResistanceMap (MapLibre), TrendChart (ECharts), loading skeletons, API client + TanStack Query hooks |
| **W4** | Cases table | TanStack Table v8 with sorting/filtering/pagination, CaseDetailDialog, case form, source/outcome badges |
| **W5** | Resistance deep dive | District drill-down, drug comparison chart, resistance legend, popup enhancements, time range selectors |
| **W6** | Alerts & notifications | Alert feed, alert rule management, real-time alert ticker, alert acknowledgment UI |
| **W7** | CHW features | CHW registry table, leaderboard, gamification badges display, CHW profile view |
| **W8** | Settings & admin | User/org settings, admin user management, system health page, audit log viewer |
| **W9** | Offline & PWA | Service worker, IndexedDB case cache, offline indicator, background sync, manifest.json |
| **W10** | Performance & polish | Lighthouse audit (target 90+), bundle analysis, code splitting, image optimization, accessibility audit |
| **W11** | E2E testing | Playwright tests for critical paths (login, case entry, map interaction, export), visual regression |
| **W12** | Final QA & deploy | Cross-browser testing, dark mode polish, mobile responsive QA, Vercel deployment, monitoring setup |

---

## 9. Deliverables Checklist

```
┌─────────────────────────────────────────────────────────┐
│  FRONTEND ENGINEER — DELIVERABLES CHECKLIST             │
├────────────────────────────────────┬───────┬────────────┤
│  Deliverable                      │ Status│ Priority   │
├────────────────────────────────────┼───────┼────────────┤
│  □ Next.js 14 App Router scaffold │  ○    │ P0         │
│  □ UDARA Tailwind design system   │  ○    │ P0         │
│  □ App sidebar (8 nav links)      │  ○    │ P0         │
│  □ Stats cards (4 KPIs)           │  ○    │ P0         │
│  □ MapLibre resistance choropleth │  ○    │ P0         │
│  □ ECharts trend chart            │  ○    │ P0         │
│  □ TanStack Table (cases)         │  ○    │ P0         │
│  □ API client + Query hooks       │  ○    │ P0         │
│  □ Auth flows (login/register)    │  ○    │ P0         │
│  □ TypeScript types (match Pydantic)│ ○   │ P0         │
├────────────────────────────────────┼───────┼────────────┤
│  □ Case detail dialog             │  ○    │ P1         │
│  □ Case creation form             │  ○    │ P1         │
│  □ Alert feed + management        │  ○    │ P1         │
│  □ CHW leaderboard                │  ○    │ P1         │
│  □ Drug comparison chart          │  ○    │ P1         │
│  □ GLASS report viewer            │  ○    │ P1         │
├────────────────────────────────────┼───────┼────────────┤
│  □ Dark mode                      │  ○    │ P2         │
│  □ PWA + offline support          │  ○    │ P2         │
│  □ i18n (EN, FR, SW initially)    │  ○    │ P2         │
│  □ Admin panel                    │  ○    │ P2         │
│  □ CSV/PDF export                 │  ○    │ P2         │
├────────────────────────────────────┼───────┼────────────┤
│  □ Storybook component docs       │  ○    │ P3         │
│  □ Playwright E2E tests (20+)     │  ○    │ P3         │
│  □ Lighthouse 90+ on all pages    │  ○    │ P3         │
│  □ WCAG 2.1 AA compliance        │  ○    │ P3         │
│  □ Performance budget (<250K JS)  │  ○    │ P3         │
└────────────────────────────────────┴───────┴────────────┘
```

---

## 10. Investor Demo Framing Tips

### Narrative Arc for Demos

```
1. THE PROBLEM (30 seconds)
   "In sub-Saharan Africa, 80% of antibiotics are sold without
    prescription. We have NO visibility into which drugs are failing
    and where resistance is emerging."

2. THE SOLUTION (60 seconds)
   [Show Dashboard → Resistance Map]
   "UDARA turns every community health worker into a resistance
    surveillance node. Cases flow in via Telegram and WhatsApp in
    real-time. Our AI extracts drug names, organisms, and symptoms
    from free text and photos."

3. THE DATA (90 seconds)
   [Show Trend Chart → Click district on map → Show popup]
   "This choropleth shows resistance by district — updated hourly.
    You can see Ciprofloxacin resistance jumping from 35% to 68%
    in Kano State over 8 weeks. Our predictive model flagged this
    3 weeks before WHO's GLASS data confirmed it."

4. THE IMPACT (60 seconds)
   [Show Stats Cards → CHW Leaderboard]
   "In our pilot across 12 sites, 450 CHWs reported 14,000 cases
    in 16 weeks. We detected 23 resistance spikes before they became
    outbreaks. Average reporting time dropped from 14 days to 4 hours."

5. THE ASK (30 seconds)
   "We're raising to expand to 3 countries and 50 sites. Each new
    site costs $12K to deploy and generates surveillance data worth
    $50K+ per year to national health systems."
```

### Demo Data Preparation

| Scenario | Preparation | Visual Impact |
|----------|------------|---------------|
| Resistance spike detection | Pre-load 8-week trend showing clear uptick | Trend chart with markLines |
| Real-time case flow | Run 3 Telegram bots reporting simultaneously | Cases table auto-refreshing |
| District drill-down | GeoJSON with varied resistance values | Map zoom + popup with data |
| CHW engagement | Show leaderboard with gamification scores | Badge + progress animations |
| Offline resilience | Disconnect network → submit case → reconnect | Offline indicator + sync toast |

### Performance Targets for Demo

| Metric | Target | Measurement |
|--------|--------|-------------|
| First Contentful Paint | < 1.2s | Lighthouse |
| Time to Interactive | < 2.5s | Lighthouse |
| Map render time | < 800ms | Performance API |
| Chart render time | < 300ms | ECharts timing |
| Table 10K rows | < 500ms first paint | TanStack virtualization |
| Lighthouse Performance | > 90 | Lighthouse CI |

---

> **Next:** See [16-member-04-integration-bot-engineer.md](./16-member-04-integration-bot-engineer.md) for the bot engineering counterpart.
