# ตอนที่ 46: Next.js กับ TypeScript

## บทนำ

Next.js เป็น React framework ที่ทรงพลัง มี built-in support สำหรับ TypeScript ตั้งแต่เวอร์ชัน 9 ในบทนี้เราจะเรียนรู้การใช้ Next.js 14 กับ TypeScript อย่างครบถ้วน ตั้งแต่การตั้งค่าจนถึงการสร้าง full-stack application

---

## 46.1 Next.js 14 Setup กับ TypeScript

### การสร้าง Project ใหม่

```bash
# สร้าง Next.js project ด้วย TypeScript
npx create-next-app@latest my-nextjs-app --typescript

# หรือกับ options ต่างๆ
npx create-next-app@latest my-app \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"
```

### โครงสร้าง Project

```
my-nextjs-app/
├── src/
│   ├── app/                    # App Router
│   │   ├── (auth)/             # Route Group
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── register/
│   │   │       └── page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   └── dashboard/
│   │   │       └── page.tsx
│   │   ├── api/               # API Routes
│   │   │   └── users/
│   │   │       └── route.ts
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/
│   │   └── features/
│   ├── lib/
│   │   ├── db.ts
│   │   └── utils.ts
│   └── types/
│       └── index.ts
├── public/
├── next.config.ts
├── tsconfig.json
└── package.json
```

### package.json

```json
{
  "name": "my-nextjs-app",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "typescript": "^5.3.0",
    "eslint": "^8.0.0",
    "eslint-config-next": "14.1.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### next.config.ts

```typescript
// next.config.ts
import type { NextConfig } from "next";

const config: NextConfig = {
  experimental: {
    typedRoutes: true,
  },
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "images.unsplash.com",
      },
      {
        protocol: "https",
        hostname: "avatars.githubusercontent.com",
      }
    ]
  },
  logging: {
    fetches: {
      fullUrl: true
    }
  }
};

export default config;
```

---

## 46.2 App Router vs Pages Router

### App Router (แนะนำสำหรับ Next.js 13+)

```
app/
├── layout.tsx      # Root layout
├── page.tsx        # Home page (/)
├── loading.tsx     # Loading UI
├── error.tsx       # Error UI
├── not-found.tsx   # 404 page
├── blog/
│   ├── page.tsx    # /blog
│   └── [slug]/
│       └── page.tsx # /blog/[slug]
└── api/
    └── users/
        └── route.ts # /api/users
```

### Pages Router (Legacy)

```
pages/
├── index.tsx       # /
├── about.tsx       # /about
├── blog/
│   ├── index.tsx   # /blog
│   └── [slug].tsx  # /blog/[slug]
└── api/
    └── users.ts    # /api/users
```

---

## 46.3 Server Components Typing

### Root Layout

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: {
    default: "My App",
    template: "%s | My App"
  },
  description: "แอปพลิเคชัน TypeScript กับ Next.js",
  keywords: ["next.js", "typescript", "react"],
  authors: [{ name: "สมชาย" }],
  openGraph: {
    type: "website",
    locale: "th_TH",
    url: "https://myapp.com",
    siteName: "My App",
  },
  robots: {
    index: true,
    follow: true,
  }
};

interface RootLayoutProps {
  children: React.ReactNode;
}

export default function RootLayout({ children }: RootLayoutProps) {
  return (
    <html lang="th">
      <body className={inter.className}>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

### Server Component

```tsx
// src/app/users/page.tsx
// Server Component (default ใน App Router)

interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

async function getUsers(): Promise<User[]> {
  // Fetch ที่นี่ทำงานบน server
  const response = await fetch("https://api.example.com/users", {
    next: {
      revalidate: 3600, // revalidate ทุก 1 ชั่วโมง
      tags: ["users"]  // cache tag สำหรับ revalidation
    }
  });
  
  if (!response.ok) {
    throw new Error(`Failed to fetch users: ${response.statusText}`);
  }
  
  return response.json();
}

interface UsersPageProps {
  searchParams: {
    page?: string;
    search?: string;
  };
}

export default async function UsersPage({ searchParams }: UsersPageProps) {
  const users = await getUsers();
  const page = Number(searchParams.page ?? 1);
  const search = searchParams.search ?? "";
  
  const filteredUsers = users.filter(u => 
    u.name.toLowerCase().includes(search.toLowerCase())
  );
  
  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-2xl font-bold mb-4">รายชื่อผู้ใช้</h1>
      <p className="text-gray-500 mb-6">หน้า {page}</p>
      
      <div className="grid gap-4">
        {filteredUsers.map(user => (
          <div key={user.id} className="p-4 border rounded-lg">
            <h2 className="font-semibold">{user.name}</h2>
            <p className="text-gray-600">{user.email}</p>
            <span className="text-sm text-blue-600">{user.role}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Async Server Component with Error Handling

```tsx
// src/app/products/[id]/page.tsx
import { notFound } from "next/navigation";
import type { Metadata, ResolvingMetadata } from "next";

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  images: string[];
  category: string;
}

async function getProduct(id: string): Promise<Product | null> {
  try {
    const res = await fetch(`https://api.example.com/products/${id}`, {
      next: { revalidate: 60 }
    });
    
    if (res.status === 404) return null;
    if (!res.ok) throw new Error("Failed to fetch product");
    
    return res.json();
  } catch {
    return null;
  }
}

// Dynamic Metadata
interface ProductPageProps {
  params: { id: string };
}

export async function generateMetadata(
  { params }: ProductPageProps,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const product = await getProduct(params.id);
  
  if (!product) {
    return { title: "สินค้าไม่พบ" };
  }
  
  const previousImages = (await parent).openGraph?.images ?? [];
  
  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      description: product.description,
      images: [...product.images, ...previousImages]
    }
  };
}

// Static Generation
export async function generateStaticParams() {
  const res = await fetch("https://api.example.com/products");
  const products: Product[] = await res.json();
  
  return products.map(p => ({ id: p.id }));
}

export default async function ProductPage({ params }: ProductPageProps) {
  const product = await getProduct(params.id);
  
  if (!product) {
    notFound(); // แสดง 404 page
  }
  
  return (
    <div className="max-w-4xl mx-auto px-4 py-8">
      <div className="grid md:grid-cols-2 gap-8">
        <div>
          {product.images[0] && (
            <img
              src={product.images[0]}
              alt={product.name}
              className="w-full rounded-lg"
            />
          )}
        </div>
        <div>
          <h1 className="text-3xl font-bold">{product.name}</h1>
          <p className="text-gray-600 mt-2">{product.description}</p>
          <p className="text-2xl font-bold text-blue-600 mt-4">
            ฿{product.price.toLocaleString("th-TH")}
          </p>
          <span className="inline-block bg-gray-100 text-gray-700 px-3 py-1 rounded-full text-sm mt-2">
            {product.category}
          </span>
        </div>
      </div>
    </div>
  );
}
```

---

## 46.4 Client Components

```tsx
// src/components/features/SearchBar.tsx
"use client"; // ทำให้เป็น Client Component

import { useState, useCallback, useTransition } from "react";
import { useRouter, useSearchParams, usePathname } from "next/navigation";

interface SearchBarProps {
  placeholder?: string;
  debounceMs?: number;
}

export function SearchBar({ 
  placeholder = "ค้นหา...", 
  debounceMs = 300 
}: SearchBarProps) {
  const router = useRouter();
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const [isPending, startTransition] = useTransition();
  const [value, setValue] = useState(searchParams.get("search") ?? "");

  const handleSearch = useCallback((term: string) => {
    const params = new URLSearchParams(searchParams.toString());
    
    if (term) {
      params.set("search", term);
    } else {
      params.delete("search");
    }
    
    // Reset to page 1 when searching
    params.delete("page");
    
    startTransition(() => {
      router.push(`${pathname}?${params.toString()}`);
    });
  }, [searchParams, pathname, router]);

  return (
    <div className="relative">
      <input
        type="text"
        value={value}
        onChange={(e) => {
          setValue(e.target.value);
          handleSearch(e.target.value);
        }}
        placeholder={placeholder}
        className={[
          "w-full px-4 py-2 pl-10 border rounded-lg",
          "focus:outline-none focus:ring-2 focus:ring-blue-500",
          isPending ? "opacity-70" : ""
        ].join(" ")}
      />
      <svg
        className="absolute left-3 top-2.5 h-5 w-5 text-gray-400"
        fill="none"
        viewBox="0 0 24 24"
        stroke="currentColor"
      >
        <path
          strokeLinecap="round"
          strokeLinejoin="round"
          strokeWidth={2}
          d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
        />
      </svg>
      {isPending && (
        <div className="absolute right-3 top-2.5">
          <div className="animate-spin h-5 w-5 border-2 border-blue-500 rounded-full border-t-transparent" />
        </div>
      )}
    </div>
  );
}
```

### Counter Client Component

```tsx
// src/components/ui/Counter.tsx
"use client";

import { useState, useEffect } from "react";

interface CounterProps {
  initialCount?: number;
  step?: number;
  min?: number;
  max?: number;
  onChange?: (count: number) => void;
  label?: string;
}

export function Counter({
  initialCount = 0,
  step = 1,
  min = -Infinity,
  max = Infinity,
  onChange,
  label = "จำนวน"
}: CounterProps) {
  const [count, setCount] = useState(initialCount);

  useEffect(() => {
    onChange?.(count);
  }, [count, onChange]);

  const increment = () => setCount(c => Math.min(c + step, max));
  const decrement = () => setCount(c => Math.max(c - step, min));
  const reset = () => setCount(initialCount);

  return (
    <div className="flex items-center gap-3">
      {label && (
        <span className="text-sm font-medium text-gray-700">{label}:</span>
      )}
      <div className="flex items-center border rounded-lg overflow-hidden">
        <button
          onClick={decrement}
          disabled={count <= min}
          className={[
            "px-3 py-2 text-gray-600",
            "hover:bg-gray-100 transition-colors",
            "disabled:opacity-50 disabled:cursor-not-allowed"
          ].join(" ")}
          aria-label="ลดลง"
        >
          −
        </button>
        <span
          className="px-4 py-2 min-w-[3rem] text-center font-semibold"
          aria-live="polite"
          aria-label={`${label}: ${count}`}
        >
          {count}
        </span>
        <button
          onClick={increment}
          disabled={count >= max}
          className={[
            "px-3 py-2 text-gray-600",
            "hover:bg-gray-100 transition-colors",
            "disabled:opacity-50 disabled:cursor-not-allowed"
          ].join(" ")}
          aria-label="เพิ่มขึ้น"
        >
          +
        </button>
      </div>
      {count !== initialCount && (
        <button
          onClick={reset}
          className="text-sm text-blue-600 hover:underline"
        >
          รีเซ็ต
        </button>
      )}
    </div>
  );
}
```

---

## 46.5 Layout Components

### Dashboard Layout

```tsx
// src/app/(dashboard)/layout.tsx
import { Suspense } from "react";
import { redirect } from "next/navigation";
import { getSession } from "@/lib/auth";
import { Sidebar } from "@/components/features/Sidebar";
import { Header } from "@/components/features/Header";
import { LoadingSpinner } from "@/components/ui/LoadingSpinner";

interface DashboardLayoutProps {
  children: React.ReactNode;
}

export default async function DashboardLayout({ children }: DashboardLayoutProps) {
  const session = await getSession();
  
  if (!session) {
    redirect("/login");
  }
  
  return (
    <div className="flex h-screen bg-gray-50">
      <Sidebar user={session.user} />
      <div className="flex-1 flex flex-col overflow-hidden">
        <Header user={session.user} />
        <main className="flex-1 overflow-y-auto p-6">
          <Suspense fallback={<LoadingSpinner />}>
            {children}
          </Suspense>
        </main>
      </div>
    </div>
  );
}
```

### Nested Layout

```tsx
// src/app/(dashboard)/settings/layout.tsx
interface SettingsLayoutProps {
  children: React.ReactNode;
}

const settingsNavItems = [
  { href: "/settings/profile", label: "โปรไฟล์" },
  { href: "/settings/account", label: "บัญชี" },
  { href: "/settings/notifications", label: "การแจ้งเตือน" },
  { href: "/settings/security", label: "ความปลอดภัย" },
];

export default function SettingsLayout({ children }: SettingsLayoutProps) {
  return (
    <div className="max-w-4xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">การตั้งค่า</h1>
      <div className="flex gap-6">
        <nav className="w-48 shrink-0">
          <ul className="space-y-1">
            {settingsNavItems.map(item => (
              <li key={item.href}>
                <a
                  href={item.href}
                  className="block px-3 py-2 rounded-md text-sm hover:bg-gray-100"
                >
                  {item.label}
                </a>
              </li>
            ))}
          </ul>
        </nav>
        <div className="flex-1">{children}</div>
      </div>
    </div>
  );
}
```

---

## 46.6 Route Handlers (API Routes)

### GET Request Handler

```typescript
// src/app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

const querySchema = z.object({
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(10),
  search: z.string().optional(),
  role: z.enum(["admin", "user", "moderator"]).optional(),
  sortBy: z.enum(["name", "email", "createdAt"]).default("createdAt"),
  sortOrder: z.enum(["asc", "desc"]).default("desc")
});

export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    
    // Parse และ validate query parameters
    const query = querySchema.safeParse(Object.fromEntries(searchParams));
    
    if (!query.success) {
      return NextResponse.json(
        { error: "Invalid query parameters", details: query.error.flatten() },
        { status: 400 }
      );
    }
    
    const { page, pageSize, search, role, sortBy, sortOrder } = query.data;
    
    // TODO: Fetch from database
    const users = await fetchUsers({ page, pageSize, search, role, sortBy, sortOrder });
    
    return NextResponse.json({
      data: users.data,
      pagination: {
        page,
        pageSize,
        total: users.total,
        totalPages: Math.ceil(users.total / pageSize)
      }
    });
  } catch (error) {
    console.error("Error fetching users:", error);
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    );
  }
}

// POST Request Handler
const createUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  password: z.string().min(8),
  role: z.enum(["admin", "user", "moderator"]).default("user")
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    
    const validation = createUserSchema.safeParse(body);
    
    if (!validation.success) {
      return NextResponse.json(
        { error: "Validation failed", details: validation.error.flatten() },
        { status: 422 }
      );
    }
    
    // TODO: Create user in database
    const user = await createUser(validation.data);
    
    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    if (error instanceof Error && error.message.includes("duplicate")) {
      return NextResponse.json(
        { error: "Email already exists" },
        { status: 409 }
      );
    }
    
    return NextResponse.json(
      { error: "Internal Server Error" },
      { status: 500 }
    );
  }
}

// Helper functions (ตัวอย่าง)
async function fetchUsers(params: any) {
  return { data: [], total: 0 };
}

async function createUser(data: any) {
  return { id: "1", ...data };
}
```

### Dynamic Route Handler

```typescript
// src/app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";

interface Params {
  id: string;
}

export async function GET(
  request: NextRequest,
  { params }: { params: Params }
) {
  const { id } = params;
  
  if (!id) {
    return NextResponse.json(
      { error: "User ID is required" },
      { status: 400 }
    );
  }
  
  // TODO: Fetch user from database
  const user = await getUserById(id);
  
  if (!user) {
    return NextResponse.json(
      { error: "User not found" },
      { status: 404 }
    );
  }
  
  return NextResponse.json(user);
}

const updateUserSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  email: z.string().email().optional(),
  role: z.enum(["admin", "user", "moderator"]).optional()
}).refine(data => Object.keys(data).length > 0, {
  message: "At least one field must be provided"
});

export async function PATCH(
  request: NextRequest,
  { params }: { params: Params }
) {
  const { id } = params;
  
  const body = await request.json();
  const validation = updateUserSchema.safeParse(body);
  
  if (!validation.success) {
    return NextResponse.json(
      { error: "Validation failed", details: validation.error.flatten() },
      { status: 422 }
    );
  }
  
  const user = await updateUser(id, validation.data);
  
  if (!user) {
    return NextResponse.json(
      { error: "User not found" },
      { status: 404 }
    );
  }
  
  return NextResponse.json(user);
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: Params }
) {
  const { id } = params;
  
  const deleted = await deleteUser(id);
  
  if (!deleted) {
    return NextResponse.json(
      { error: "User not found" },
      { status: 404 }
    );
  }
  
  return new NextResponse(null, { status: 204 });
}

// Placeholder functions
async function getUserById(id: string) { return null; }
async function updateUser(id: string, data: any) { return null; }
async function deleteUser(id: string) { return false; }
```

---

## 46.7 Server Actions

```typescript
// src/app/actions/user.ts
"use server";

import { revalidatePath, revalidateTag } from "next/cache";
import { redirect } from "next/navigation";
import { z } from "zod";

type ActionState = {
  success: boolean;
  message?: string;
  errors?: Record<string, string[]>;
  data?: unknown;
};

// Schema validation
const profileSchema = z.object({
  name: z.string().min(2, "ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"),
  bio: z.string().max(500, "ประวัติต้องไม่เกิน 500 ตัวอักษร").optional(),
  website: z.string().url("URL ไม่ถูกต้อง").optional().or(z.literal(""))
});

export async function updateProfileAction(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  try {
    // Parse form data
    const rawData = {
      name: formData.get("name") as string,
      bio: formData.get("bio") as string | undefined,
      website: formData.get("website") as string | undefined
    };
    
    // Validate
    const validation = profileSchema.safeParse(rawData);
    
    if (!validation.success) {
      return {
        success: false,
        errors: validation.error.flatten().fieldErrors
      };
    }
    
    // Get current user (ต้องมี auth)
    const session = await getSession();
    if (!session) {
      return { success: false, message: "ไม่ได้เข้าสู่ระบบ" };
    }
    
    // Update in database
    await updateUserProfile(session.user.id, validation.data);
    
    // Revalidate cache
    revalidatePath(`/profile/${session.user.id}`);
    revalidateTag("user-profile");
    
    return {
      success: true,
      message: "อัปเดตโปรไฟล์สำเร็จ"
    };
  } catch (error) {
    return {
      success: false,
      message: "เกิดข้อผิดพลาด กรุณาลองใหม่"
    };
  }
}

export async function deleteAccountAction(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  const confirmation = formData.get("confirmation") as string;
  
  if (confirmation !== "DELETE") {
    return {
      success: false,
      message: "กรุณาพิมพ์ DELETE เพื่อยืนยันการลบบัญชี"
    };
  }
  
  const session = await getSession();
  if (!session) {
    return { success: false, message: "ไม่ได้เข้าสู่ระบบ" };
  }
  
  await deleteUserAccount(session.user.id);
  
  // Redirect after successful deletion
  redirect("/");
}

// Placeholder functions
async function getSession() { return null as any; }
async function updateUserProfile(id: string, data: any) {}
async function deleteUserAccount(id: string) {}
```

### การใช้ Server Actions กับ Form

```tsx
// src/app/(dashboard)/settings/profile/page.tsx
"use client";

import { useFormState, useFormStatus } from "react-dom";
import { updateProfileAction } from "@/app/actions/user";

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button
      type="submit"
      disabled={pending}
      className={[
        "px-4 py-2 bg-blue-600 text-white rounded-md",
        "hover:bg-blue-700 transition-colors",
        "disabled:opacity-50 disabled:cursor-not-allowed",
        "flex items-center gap-2"
      ].join(" ")}
    >
      {pending && (
        <div className="animate-spin h-4 w-4 border-2 border-white rounded-full border-t-transparent" />
      )}
      {pending ? "กำลังบันทึก..." : "บันทึกการเปลี่ยนแปลง"}
    </button>
  );
}

export default function ProfileSettingsPage() {
  const initialState = { success: false };
  const [state, formAction] = useFormState(updateProfileAction, initialState);
  
  return (
    <div className="max-w-2xl">
      <h2 className="text-xl font-semibold mb-6">แก้ไขโปรไฟล์</h2>
      
      {state.success && (
        <div className="mb-4 p-3 bg-green-50 border border-green-200 rounded-md text-green-700">
          {state.message}
        </div>
      )}
      
      {!state.success && state.message && (
        <div className="mb-4 p-3 bg-red-50 border border-red-200 rounded-md text-red-700">
          {state.message}
        </div>
      )}
      
      <form action={formAction} className="space-y-4">
        <div>
          <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">
            ชื่อ-นามสกุล
          </label>
          <input
            id="name"
            name="name"
            type="text"
            required
            className={[
              "w-full px-3 py-2 border rounded-md",
              "focus:outline-none focus:ring-2 focus:ring-blue-500",
              state.errors?.name ? "border-red-300" : "border-gray-300"
            ].join(" ")}
          />
          {state.errors?.name && (
            <p className="mt-1 text-sm text-red-600">{state.errors.name[0]}</p>
          )}
        </div>
        
        <div>
          <label htmlFor="bio" className="block text-sm font-medium text-gray-700 mb-1">
            ประวัติย่อ
          </label>
          <textarea
            id="bio"
            name="bio"
            rows={4}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          {state.errors?.bio && (
            <p className="mt-1 text-sm text-red-600">{state.errors.bio[0]}</p>
          )}
        </div>
        
        <div>
          <label htmlFor="website" className="block text-sm font-medium text-gray-700 mb-1">
            เว็บไซต์
          </label>
          <input
            id="website"
            name="website"
            type="url"
            placeholder="https://example.com"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          {state.errors?.website && (
            <p className="mt-1 text-sm text-red-600">{state.errors.website[0]}</p>
          )}
        </div>
        
        <SubmitButton />
      </form>
    </div>
  );
}
```

---

## 46.8 Dynamic Routes Typing

### Catch-all Routes

```tsx
// src/app/docs/[...slug]/page.tsx
interface DocsPageProps {
  params: {
    slug: string[];
  };
}

export async function generateStaticParams() {
  const docPaths = [
    { slug: ["getting-started"] },
    { slug: ["api", "reference"] },
    { slug: ["guide", "typescript", "setup"] }
  ];
  
  return docPaths;
}

export default async function DocsPage({ params }: DocsPageProps) {
  const { slug } = params;
  const path = slug.join("/");
  
  // โหลด markdown content จาก slug path
  const content = await loadDocContent(path);
  
  return (
    <div className="prose max-w-none">
      <nav aria-label="breadcrumb">
        <ol className="flex gap-2 text-sm text-gray-500">
          <li><a href="/docs">Docs</a></li>
          {slug.map((segment, i) => (
            <li key={i} className="flex items-center gap-2">
              <span>/</span>
              <a href={`/docs/${slug.slice(0, i + 1).join("/")}`}>
                {segment}
              </a>
            </li>
          ))}
        </ol>
      </nav>
      <div dangerouslySetInnerHTML={{ __html: content }} />
    </div>
  );
}

async function loadDocContent(path: string): Promise<string> {
  return `<h1>${path}</h1>`;
}
```

### Optional Catch-all Routes

```tsx
// src/app/[[...path]]/page.tsx
// [[...path]] - optional catch-all

interface OptionalCatchAllProps {
  params: {
    path?: string[];
  };
}

export default function OptionalCatchAllPage({ params }: OptionalCatchAllProps) {
  const { path } = params;
  
  if (!path || path.length === 0) {
    return <div>หน้าแรก</div>;
  }
  
  return <div>เส้นทาง: {path.join("/")}</div>;
}
```

---

## 46.9 Middleware

```typescript
// src/middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

// Routes ที่ต้อง authentication
const protectedRoutes = ["/dashboard", "/profile", "/settings"];

// Routes สำหรับ non-authenticated users เท่านั้น
const authRoutes = ["/login", "/register"];

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  
  // ดึง token จาก cookie
  const token = request.cookies.get("auth-token")?.value;
  const isAuthenticated = !!token;
  
  // ตรวจสอบว่าเป็น protected route
  const isProtectedRoute = protectedRoutes.some(route => 
    pathname.startsWith(route)
  );
  
  // ตรวจสอบว่าเป็น auth route
  const isAuthRoute = authRoutes.some(route => 
    pathname.startsWith(route)
  );
  
  // Redirect to login ถ้าไม่ได้ authenticate แต่เข้า protected route
  if (isProtectedRoute && !isAuthenticated) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(loginUrl);
  }
  
  // Redirect to dashboard ถ้า authenticate แล้วและเข้า auth route
  if (isAuthRoute && isAuthenticated) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }
  
  // เพิ่ม security headers
  const response = NextResponse.next();
  
  response.headers.set("X-Frame-Options", "DENY");
  response.headers.set("X-Content-Type-Options", "nosniff");
  response.headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
  
  return response;
}

// กำหนด paths ที่ middleware จะทำงาน
export const config = {
  matcher: [
    // รัน middleware บนทุก path ยกเว้น static files
    "/((?!_next/static|_next/image|favicon.ico|public/).*)"
  ]
};
```

### Middleware พร้อม Rate Limiting

```typescript
// src/middleware.ts (เพิ่มเติม)
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

// Simple in-memory rate limiter (ใน production ควรใช้ Redis)
const rateLimitMap = new Map<string, { count: number; timestamp: number }>();

function rateLimit(ip: string, limit: number, windowMs: number): boolean {
  const now = Date.now();
  const windowStart = now - windowMs;
  
  const current = rateLimitMap.get(ip);
  
  if (!current || current.timestamp < windowStart) {
    rateLimitMap.set(ip, { count: 1, timestamp: now });
    return true; // อนุญาต
  }
  
  if (current.count >= limit) {
    return false; // เกิน limit
  }
  
  current.count++;
  return true;
}

export function middleware(request: NextRequest) {
  // Rate limiting สำหรับ API routes
  if (request.nextUrl.pathname.startsWith("/api/")) {
    const ip = request.ip ?? request.headers.get("x-forwarded-for") ?? "unknown";
    
    // 100 requests ต่อ 60 วินาที
    const allowed = rateLimit(ip, 100, 60 * 1000);
    
    if (!allowed) {
      return NextResponse.json(
        { error: "Too many requests" },
        { 
          status: 429,
          headers: {
            "Retry-After": "60"
          }
        }
      );
    }
  }
  
  return NextResponse.next();
}
```

---

## 46.10 Data Fetching Patterns

### Parallel Data Fetching

```tsx
// src/app/dashboard/page.tsx

async function getDashboardData() {
  // Fetch ข้อมูลพร้อมกัน
  const [users, products, orders, stats] = await Promise.all([
    fetch("/api/users?limit=5").then(r => r.json()),
    fetch("/api/products?limit=5").then(r => r.json()),
    fetch("/api/orders?limit=5").then(r => r.json()),
    fetch("/api/stats/summary").then(r => r.json())
  ]);
  
  return { users, products, orders, stats };
}

export default async function DashboardPage() {
  const { users, products, orders, stats } = await getDashboardData();
  
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold">แดชบอร์ด</h1>
      
      <div className="grid grid-cols-2 lg:grid-cols-4 gap-4">
        {Object.entries(stats).map(([key, value]) => (
          <div key={key} className="p-4 bg-white rounded-lg shadow-sm border">
            <p className="text-sm text-gray-500 capitalize">{key}</p>
            <p className="text-2xl font-bold">{String(value)}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Streaming กับ Suspense

```tsx
// src/app/dashboard/page.tsx (with Streaming)
import { Suspense } from "react";

async function RecentUsers() {
  // Simulate slow fetch
  const users = await fetch("/api/users?limit=5").then(r => r.json());
  
  return (
    <ul className="space-y-2">
      {users.map((user: any) => (
        <li key={user.id} className="flex items-center gap-3 p-3 bg-gray-50 rounded">
          <div className="w-8 h-8 bg-blue-500 rounded-full flex items-center justify-center text-white text-sm">
            {user.name.charAt(0)}
          </div>
          <div>
            <p className="font-medium">{user.name}</p>
            <p className="text-sm text-gray-500">{user.email}</p>
          </div>
        </li>
      ))}
    </ul>
  );
}

function UsersSkeleton() {
  return (
    <ul className="space-y-2">
      {Array.from({ length: 5 }).map((_, i) => (
        <li key={i} className="flex items-center gap-3 p-3 bg-gray-50 rounded animate-pulse">
          <div className="w-8 h-8 bg-gray-200 rounded-full" />
          <div className="flex-1">
            <div className="h-4 bg-gray-200 rounded w-1/3 mb-1" />
            <div className="h-3 bg-gray-200 rounded w-1/2" />
          </div>
        </li>
      ))}
    </ul>
  );
}

export default function DashboardPage() {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold">แดชบอร์ด</h1>
      
      <div className="grid grid-cols-2 gap-6">
        <div className="bg-white rounded-lg p-4 shadow-sm border">
          <h2 className="font-semibold mb-4">ผู้ใช้ล่าสุด</h2>
          <Suspense fallback={<UsersSkeleton />}>
            <RecentUsers />
          </Suspense>
        </div>
      </div>
    </div>
  );
}
```

---

## 46.11 Pages Router - getStaticProps/getServerSideProps Typing

```tsx
// pages/blog/[slug].tsx (Pages Router)
import { GetStaticProps, GetStaticPaths, NextPage } from "next";

interface Post {
  slug: string;
  title: string;
  content: string;
  publishedAt: string;
  author: string;
}

interface PostPageProps {
  post: Post;
  relatedPosts: Post[];
}

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetchAllPosts();
  
  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: "blocking" // หรือ false หรือ true
  };
};

export const getStaticProps: GetStaticProps<PostPageProps> = async ({ params }) => {
  const slug = params?.slug as string;
  
  const [post, relatedPosts] = await Promise.all([
    fetchPostBySlug(slug),
    fetchRelatedPosts(slug)
  ]);
  
  if (!post) {
    return { notFound: true };
  }
  
  return {
    props: {
      post,
      relatedPosts
    },
    revalidate: 3600 // ISR: revalidate ทุก 1 ชั่วโมง
  };
};

const PostPage: NextPage<PostPageProps> = ({ post, relatedPosts }) => {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      
      <section>
        <h2>บทความที่เกี่ยวข้อง</h2>
        {relatedPosts.map(related => (
          <a key={related.slug} href={`/blog/${related.slug}`}>
            {related.title}
          </a>
        ))}
      </section>
    </article>
  );
};

export default PostPage;

// Placeholder functions
async function fetchAllPosts(): Promise<Post[]> { return []; }
async function fetchPostBySlug(slug: string): Promise<Post | null> { return null; }
async function fetchRelatedPosts(slug: string): Promise<Post[]> { return []; }
```

### getServerSideProps Typing

```tsx
// pages/profile.tsx
import { GetServerSideProps, NextPage } from "next";
import { getSession } from "next-auth/react";

interface ProfileProps {
  user: {
    id: string;
    name: string;
    email: string;
    joinedAt: string;
  };
}

export const getServerSideProps: GetServerSideProps<ProfileProps> = async (ctx) => {
  const session = await getSession(ctx);
  
  if (!session) {
    return {
      redirect: {
        destination: "/login",
        permanent: false
      }
    };
  }
  
  const user = await fetchUserProfile(session.user.id);
  
  if (!user) {
    return { notFound: true };
  }
  
  return {
    props: {
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        joinedAt: user.createdAt.toISOString()
      }
    }
  };
};

const ProfilePage: NextPage<ProfileProps> = ({ user }) => {
  return (
    <div>
      <h1>โปรไฟล์: {user.name}</h1>
      <p>Email: {user.email}</p>
      <p>สมัครเมื่อ: {new Date(user.joinedAt).toLocaleDateString("th-TH")}</p>
    </div>
  );
};

export default ProfilePage;

async function fetchUserProfile(id: string): Promise<any> { return null; }
```

---

## 46.12 Metadata API

```typescript
// src/app/blog/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from "next";

interface Props {
  params: { slug: string };
  searchParams: { [key: string]: string | string[] | undefined };
}

// Static Metadata
export const metadata: Metadata = {
  title: "บล็อก",
  description: "บทความเกี่ยวกับ TypeScript และ Next.js"
};

// Dynamic Metadata
export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const slug = params.slug;
  const post = await fetchPost(slug);
  
  if (!post) {
    return {
      title: "ไม่พบบทความ"
    };
  }
  
  // เข้าถึง metadata ของ parent
  const previousImages = (await parent).openGraph?.images ?? [];
  
  return {
    title: post.title,
    description: post.excerpt,
    authors: [{ name: post.author }],
    publishedTime: post.publishedAt,
    
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: "article",
      publishedTime: post.publishedAt,
      authors: [post.author],
      images: [
        {
          url: post.coverImage,
          width: 1200,
          height: 630,
          alt: post.title
        },
        ...previousImages
      ]
    },
    
    twitter: {
      card: "summary_large_image",
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage]
    },
    
    alternates: {
      canonical: `/blog/${slug}`
    }
  };
}

async function fetchPost(slug: string): Promise<any> { return null; }
```

---

## 46.13 Image Optimization

```tsx
// src/components/ui/OptimizedImage.tsx
import Image from "next/image";
import { useState } from "react";

interface OptimizedImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  priority?: boolean;
  className?: string;
  fill?: boolean;
  sizes?: string;
  quality?: number;
  placeholder?: "blur" | "empty";
  blurDataURL?: string;
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false,
  className,
  fill = false,
  sizes,
  quality = 80,
  placeholder = "empty",
  blurDataURL
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);
  
  if (hasError) {
    return (
      <div
        className={`bg-gray-200 flex items-center justify-center ${className}`}
        style={!fill ? { width, height } : undefined}
      >
        <span className="text-gray-400 text-sm">ไม่สามารถโหลดรูปได้</span>
      </div>
    );
  }
  
  return (
    <div className={`relative overflow-hidden ${className}`}>
      <Image
        src={src}
        alt={alt}
        width={fill ? undefined : width}
        height={fill ? undefined : height}
        fill={fill}
        sizes={sizes ?? `(max-width: 768px) 100vw, ${width}px`}
        quality={quality}
        priority={priority}
        placeholder={placeholder}
        blurDataURL={blurDataURL}
        className={`transition-opacity duration-300 ${
          isLoading ? "opacity-0" : "opacity-100"
        }`}
        onLoad={() => setIsLoading(false)}
        onError={() => {
          setIsLoading(false);
          setHasError(true);
        }}
      />
      {isLoading && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse" />
      )}
    </div>
  );
}
```

---

## 46.14 Complete App Example

### E-Commerce Product Listing

```tsx
// src/app/shop/page.tsx
import { Suspense } from "react";
import { SearchBar } from "@/components/features/SearchBar";

interface SearchParams {
  category?: string;
  search?: string;
  sort?: string;
  page?: string;
}

interface Product {
  id: string;
  name: string;
  price: number;
  image: string;
  category: string;
  rating: number;
  reviewCount: number;
}

async function getProducts(params: SearchParams): Promise<{
  products: Product[];
  total: number;
}> {
  const query = new URLSearchParams();
  if (params.category) query.set("category", params.category);
  if (params.search) query.set("search", params.search);
  if (params.sort) query.set("sort", params.sort);
  query.set("page", params.page ?? "1");
  
  const res = await fetch(`/api/products?${query}`, {
    next: { revalidate: 60 }
  });
  
  if (!res.ok) throw new Error("Failed to fetch products");
  return res.json();
}

interface ShopPageProps {
  searchParams: SearchParams;
}

export default async function ShopPage({ searchParams }: ShopPageProps) {
  const { products, total } = await getProducts(searchParams);
  const currentPage = Number(searchParams.page ?? 1);
  const pageSize = 12;
  const totalPages = Math.ceil(total / pageSize);
  
  return (
    <div className="max-w-7xl mx-auto px-4 py-8">
      <div className="flex flex-col gap-6">
        {/* Header */}
        <div className="flex items-center justify-between">
          <h1 className="text-2xl font-bold">ร้านค้า</h1>
          <span className="text-gray-500 text-sm">
            แสดง {products.length} จาก {total} รายการ
          </span>
        </div>
        
        {/* Filters */}
        <div className="flex flex-col sm:flex-row gap-4">
          <Suspense>
            <SearchBar placeholder="ค้นหาสินค้า..." />
          </Suspense>
          
          <select
            className="px-3 py-2 border border-gray-300 rounded-md text-sm"
            defaultValue={searchParams.sort ?? "newest"}
          >
            <option value="newest">ใหม่ล่าสุด</option>
            <option value="price_asc">ราคา: ต่ำ - สูง</option>
            <option value="price_desc">ราคา: สูง - ต่ำ</option>
            <option value="rating">คะแนนสูงสุด</option>
          </select>
        </div>
        
        {/* Products Grid */}
        {products.length > 0 ? (
          <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-4">
            {products.map(product => (
              <a
                key={product.id}
                href={`/shop/${product.id}`}
                className="group"
              >
                <div className="bg-white rounded-lg overflow-hidden border border-gray-200 hover:shadow-md transition-shadow">
                  <div className="aspect-square bg-gray-100 relative overflow-hidden">
                    <img
                      src={product.image}
                      alt={product.name}
                      className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
                    />
                  </div>
                  <div className="p-3">
                    <h3 className="font-medium text-sm line-clamp-2">{product.name}</h3>
                    <p className="text-blue-600 font-bold mt-1">
                      ฿{product.price.toLocaleString("th-TH")}
                    </p>
                    <div className="flex items-center gap-1 mt-1 text-xs text-gray-500">
                      <span>⭐ {product.rating.toFixed(1)}</span>
                      <span>({product.reviewCount.toLocaleString()} รีวิว)</span>
                    </div>
                  </div>
                </div>
              </a>
            ))}
          </div>
        ) : (
          <div className="text-center py-16 text-gray-500">
            <p className="text-lg">ไม่พบสินค้า</p>
            <p className="text-sm mt-1">ลองค้นหาด้วยคำอื่น</p>
          </div>
        )}
        
        {/* Pagination */}
        {totalPages > 1 && (
          <nav className="flex justify-center gap-2" aria-label="pagination">
            {Array.from({ length: totalPages }, (_, i) => i + 1).map(page => (
              <a
                key={page}
                href={`?${new URLSearchParams({ ...searchParams, page: String(page) })}`}
                className={[
                  "px-3 py-2 rounded-md text-sm",
                  page === currentPage
                    ? "bg-blue-600 text-white"
                    : "bg-white border border-gray-300 hover:bg-gray-50"
                ].join(" ")}
                aria-current={page === currentPage ? "page" : undefined}
              >
                {page}
              </a>
            ))}
          </nav>
        )}
      </div>
    </div>
  );
}
```

---

## สรุปบทที่ 46

ในบทนี้เราได้เรียนรู้:

1. **Setup** - การสร้างและตั้งค่า Next.js 14 กับ TypeScript
2. **App Router** - โครงสร้างและการใช้งาน App Router
3. **Server Components** - การ fetch data บน server
4. **Client Components** - การใช้ state และ events
5. **Layouts** - การสร้าง shared layouts
6. **Route Handlers** - API routes ใหม่
7. **Server Actions** - การจัดการ form mutations
8. **Dynamic Routes** - การสร้าง dynamic pages
9. **Middleware** - การดัก requests
10. **Data Fetching** - patterns ต่างๆ
11. **Pages Router** - getStaticProps/getServerSideProps
12. **Metadata API** - SEO optimization
13. **Image Optimization** - การใช้ next/image
14. **Complete Example** - E-commerce app

Next.js 14 กับ TypeScript ทำให้เราสร้าง:
- Type-safe full-stack applications
- Optimized server-side rendering
- Progressive enhancement
- Excellent developer experience
