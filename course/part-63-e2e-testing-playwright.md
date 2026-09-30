# ส่วนที่ 63: E2E Testing with Playwright และ TypeScript

## บทนำ

Playwright เป็น framework สำหรับการทำ End-to-End testing ที่รองรับ browsers หลัก (Chromium, Firefox, WebKit) พร้อม TypeScript support แบบ first-class ในบทนี้เราจะเรียนรู้การใช้ Playwright อย่างครบถ้วน

---

## 1. การติดตั้งและตั้งค่า Playwright

### 1.1 ติดตั้ง Playwright

```bash
# ติดตั้ง Playwright
npm init playwright@latest

# หรือเพิ่มใน project ที่มีอยู่แล้ว
npm install --save-dev @playwright/test
npx playwright install
```

### 1.2 playwright.config.ts

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // ไดเรกทอรีที่เก็บ test files
  testDir: './e2e',
  
  // ใช้ parallel execution
  fullyParallel: true,
  
  // fail build เมื่อเหลือ test.only ใน CI
  forbidOnly: !!process.env.CI,
  
  // retry จำนวนครั้งใน CI
  retries: process.env.CI ? 2 : 0,
  
  // workers
  workers: process.env.CI ? 1 : undefined,
  
  // Reporter
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['list']
  ],
  
  use: {
    // Base URL
    baseURL: 'http://localhost:3000',
    
    // เก็บ screenshot เมื่อ fail
    screenshot: 'only-on-failure',
    
    // เก็บ video เมื่อ fail
    video: 'retain-on-failure',
    
    // เก็บ trace เมื่อ retry ครั้งแรก
    trace: 'on-first-retry',
    
    // viewport
    viewport: { width: 1280, height: 720 },
    
    // locale
    locale: 'th-TH',
    
    // timezone
    timezoneId: 'Asia/Bangkok'
  },

  // กำหนด browsers
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 13'] }
    }
  ],

  // เริ่ม dev server ก่อน test
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
});
```

---

## 2. พื้นฐาน Browser Automation

### 2.1 การนำทางและ Actions พื้นฐาน

```typescript
// e2e/basic-navigation.spec.ts
import { test, expect } from '@playwright/test';

test.describe('การนำทางพื้นฐาน', () => {
  test('ไปยัง URL และตรวจสอบ title', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveTitle(/หน้าแรก/);
  });

  test('คลิก link และนำทาง', async ({ page }) => {
    await page.goto('/');
    
    // คลิก link
    await page.click('text=เกี่ยวกับเรา');
    
    // ตรวจสอบ URL
    await expect(page).toHaveURL('/about');
  });

  test('กรอกข้อมูลใน form', async ({ page }) => {
    await page.goto('/contact');
    
    // กรอกข้อมูล
    await page.fill('#name', 'สมชาย ใจดี');
    await page.fill('#email', 'somchai@example.com');
    await page.fill('#message', 'ข้อความทดสอบจาก Playwright');
    
    // Submit
    await page.click('button[type="submit"]');
    
    // ตรวจสอบ success message
    await expect(page.locator('.success-message')).toBeVisible();
  });

  test('การใช้ keyboard', async ({ page }) => {
    await page.goto('/search');
    
    // กด keyboard shortcuts
    await page.keyboard.press('Control+a');
    await page.keyboard.type('TypeScript tutorial');
    await page.keyboard.press('Enter');
    
    // ตรวจสอบผลลัพธ์การค้นหา
    await expect(page.locator('.search-results')).toBeVisible();
  });

  test('Scroll และ viewport', async ({ page }) => {
    await page.goto('/long-page');
    
    // Scroll to bottom
    await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
    
    // ตรวจสอบ element ที่ lazy-load
    await expect(page.locator('#footer')).toBeVisible();
  });
});
```

### 2.2 การทำงานกับ Elements

```typescript
// e2e/element-interactions.spec.ts
import { test, expect } from '@playwright/test';

test.describe('การทำงานกับ Elements', () => {
  test('Select dropdown', async ({ page }) => {
    await page.goto('/form');
    
    // เลือกค่าใน select
    await page.selectOption('#country', 'TH');
    
    // ตรวจสอบค่าที่เลือก
    const value = await page.inputValue('#country');
    expect(value).toBe('TH');
  });

  test('Checkbox', async ({ page }) => {
    await page.goto('/form');
    
    // Check checkbox
    await page.check('#accept-terms');
    await expect(page.locator('#accept-terms')).toBeChecked();
    
    // Uncheck
    await page.uncheck('#accept-terms');
    await expect(page.locator('#accept-terms')).not.toBeChecked();
  });

  test('Radio buttons', async ({ page }) => {
    await page.goto('/form');
    
    await page.check('input[name="gender"][value="male"]');
    await expect(page.locator('input[name="gender"][value="male"]')).toBeChecked();
  });

  test('File upload', async ({ page }) => {
    await page.goto('/upload');
    
    // Upload file
    await page.setInputFiles('#file-input', './test-files/sample.pdf');
    
    // ตรวจสอบ filename
    await expect(page.locator('.filename')).toHaveText('sample.pdf');
  });

  test('Hover เพื่อแสดง tooltip', async ({ page }) => {
    await page.goto('/');
    
    await page.hover('.info-icon');
    await expect(page.locator('.tooltip')).toBeVisible();
  });

  test('Drag and Drop', async ({ page }) => {
    await page.goto('/kanban');
    
    const source = page.locator('.card:first-child');
    const target = page.locator('.column:nth-child(2)');
    
    await source.dragTo(target);
    
    // ตรวจสอบว่า card ย้ายไปแล้ว
    await expect(target.locator('.card')).toBeVisible();
  });
});
```

---

## 3. Page Object Model (POM) กับ TypeScript

### 3.1 สร้าง Base Page

```typescript
// e2e/pages/base.page.ts
import { Page, Locator, expect } from '@playwright/test';

export abstract class BasePage {
  protected readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async navigate(path: string = '/'): Promise<void> {
    await this.page.goto(path);
  }

  async waitForLoadState(): Promise<void> {
    await this.page.waitForLoadState('networkidle');
  }

  async getTitle(): Promise<string> {
    return this.page.title();
  }

  async takeScreenshot(name: string): Promise<void> {
    await this.page.screenshot({ path: `screenshots/${name}.png` });
  }

  protected locator(selector: string): Locator {
    return this.page.locator(selector);
  }

  async waitForSelector(selector: string, timeout: number = 5000): Promise<void> {
    await this.page.waitForSelector(selector, { timeout });
  }
}
```

### 3.2 Login Page Object

```typescript
// e2e/pages/login.page.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './base.page';

export class LoginPage extends BasePage {
  // Locators
  private readonly emailInput = this.locator('#email');
  private readonly passwordInput = this.locator('#password');
  private readonly submitButton = this.locator('button[type="submit"]');
  private readonly errorMessage = this.locator('.error-message');
  private readonly rememberMeCheckbox = this.locator('#remember-me');
  private readonly forgotPasswordLink = this.locator('a.forgot-password');

  constructor(page: Page) {
    super(page);
  }

  async goto(): Promise<void> {
    await this.navigate('/login');
    await this.waitForSelector('#email');
  }

  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async loginWithRememberMe(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.rememberMeCheckbox.check();
    await this.submitButton.click();
  }

  async getErrorMessage(): Promise<string> {
    await this.errorMessage.waitFor({ state: 'visible' });
    return this.errorMessage.textContent() ?? '';
  }

  async clickForgotPassword(): Promise<void> {
    await this.forgotPasswordLink.click();
  }

  async isLoginButtonDisabled(): Promise<boolean> {
    return this.submitButton.isDisabled();
  }

  async expectErrorVisible(message: string): Promise<void> {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toHaveText(message);
  }

  async expectRedirectedToDashboard(): Promise<void> {
    await expect(this.page).toHaveURL('/dashboard');
  }
}
```

### 3.3 Dashboard Page Object

```typescript
// e2e/pages/dashboard.page.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './base.page';

export class DashboardPage extends BasePage {
  private readonly welcomeMessage = this.locator('.welcome-message');
  private readonly navigationMenu = this.locator('nav.main-menu');
  private readonly userAvatar = this.locator('.user-avatar');
  private readonly logoutButton = this.locator('button.logout');
  private readonly statsCards = this.locator('.stats-card');
  private readonly searchInput = this.locator('input[placeholder="ค้นหา..."]');

  constructor(page: Page) {
    super(page);
  }

  async goto(): Promise<void> {
    await this.navigate('/dashboard');
  }

  async getWelcomeMessage(): Promise<string> {
    return await this.welcomeMessage.textContent() ?? '';
  }

  async logout(): Promise<void> {
    await this.userAvatar.click();
    await this.logoutButton.click();
    await expect(this.page).toHaveURL('/login');
  }

  async search(query: string): Promise<void> {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
  }

  async getStatsCount(): Promise<number> {
    return await this.statsCards.count();
  }

  async getStatValue(index: number): Promise<string> {
    const card = this.statsCards.nth(index);
    return await card.locator('.stat-value').textContent() ?? '';
  }

  async navigateTo(section: string): Promise<void> {
    await this.navigationMenu.locator(`a[href="/${section}"]`).click();
    await expect(this.page).toHaveURL(`/${section}`);
  }

  async expectToBeOnDashboard(): Promise<void> {
    await expect(this.page).toHaveURL('/dashboard');
    await expect(this.welcomeMessage).toBeVisible();
  }
}
```

### 3.4 Products Page Object

```typescript
// e2e/pages/products.page.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './base.page';

interface ProductInfo {
  name: string;
  price: string;
  category: string;
}

export class ProductsPage extends BasePage {
  private readonly productList = this.locator('.product-list');
  private readonly productCards = this.locator('.product-card');
  private readonly filterCategory = this.locator('#filter-category');
  private readonly sortSelect = this.locator('#sort-by');
  private readonly addToCartButtons = this.locator('button.add-to-cart');
  private readonly cartCount = this.locator('.cart-count');
  private readonly loadingSpinner = this.locator('.loading-spinner');

  constructor(page: Page) {
    super(page);
  }

  async goto(): Promise<void> {
    await this.navigate('/products');
    await this.waitForLoadState();
  }

  async getProductCount(): Promise<number> {
    await this.productList.waitFor({ state: 'visible' });
    return await this.productCards.count();
  }

  async filterByCategory(category: string): Promise<void> {
    await this.filterCategory.selectOption(category);
    await this.loadingSpinner.waitFor({ state: 'detached' });
  }

  async sortBy(option: string): Promise<void> {
    await this.sortSelect.selectOption(option);
    await this.loadingSpinner.waitFor({ state: 'detached' });
  }

  async getProductInfo(index: number): Promise<ProductInfo> {
    const card = this.productCards.nth(index);
    
    return {
      name: await card.locator('.product-name').textContent() ?? '',
      price: await card.locator('.product-price').textContent() ?? '',
      category: await card.locator('.product-category').textContent() ?? ''
    };
  }

  async addToCart(productIndex: number): Promise<void> {
    const currentCount = await this.getCartCount();
    await this.addToCartButtons.nth(productIndex).click();
    await expect(this.cartCount).toHaveText(String(currentCount + 1));
  }

  async getCartCount(): Promise<number> {
    const text = await this.cartCount.textContent() ?? '0';
    return parseInt(text);
  }

  async searchProduct(query: string): Promise<void> {
    await this.page.fill('input[type="search"]', query);
    await this.page.press('input[type="search"]', 'Enter');
    await this.loadingSpinner.waitFor({ state: 'detached' });
  }
}
```

### 3.5 การใช้ Page Objects ใน Tests

```typescript
// e2e/tests/authentication.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';

test.describe('Authentication Flow', () => {
  let loginPage: LoginPage;
  let dashboardPage: DashboardPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    dashboardPage = new DashboardPage(page);
    await loginPage.goto();
  });

  test('login สำเร็จด้วย credentials ที่ถูกต้อง', async () => {
    await loginPage.login('user@example.com', 'Password1!');
    await loginPage.expectRedirectedToDashboard();
    
    const welcomeMsg = await dashboardPage.getWelcomeMessage();
    expect(welcomeMsg).toContain('ยินดีต้อนรับ');
  });

  test('login ไม่สำเร็จด้วย credentials ที่ผิด', async () => {
    await loginPage.login('user@example.com', 'wrongpassword');
    await loginPage.expectErrorVisible('อีเมลหรือรหัสผ่านไม่ถูกต้อง');
  });

  test('logout สำเร็จ', async () => {
    await loginPage.login('user@example.com', 'Password1!');
    await dashboardPage.expectToBeOnDashboard();
    await dashboardPage.logout();
    
    await expect(loginPage.page).toHaveURL('/login');
  });
});
```

---

## 4. Selectors และ Locators

### 4.1 Locator Strategies

```typescript
// e2e/tests/locators.spec.ts
import { test, expect, Page } from '@playwright/test';

test.describe('Locator Strategies', () => {
  test('CSS Selectors', async ({ page }) => {
    await page.goto('/');
    
    // Class selector
    await expect(page.locator('.header')).toBeVisible();
    
    // ID selector
    await expect(page.locator('#main-nav')).toBeVisible();
    
    // Attribute selector
    await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
    
    // Combined selector
    await expect(page.locator('button.primary[type="submit"]')).toBeEnabled();
  });

  test('Text selectors', async ({ page }) => {
    await page.goto('/');
    
    // Exact text
    await expect(page.getByText('หน้าแรก', { exact: true })).toBeVisible();
    
    // Partial text
    await expect(page.getByText('ยินดีต้อนรับ')).toBeVisible();
    
    // Role with name
    await expect(page.getByRole('button', { name: 'เข้าสู่ระบบ' })).toBeVisible();
  });

  test('ARIA Locators', async ({ page }) => {
    await page.goto('/form');
    
    // By label
    await page.getByLabel('ชื่อผู้ใช้').fill('สมชาย');
    
    // By placeholder
    await page.getByPlaceholder('กรอกอีเมล').fill('test@example.com');
    
    // By role
    await page.getByRole('textbox', { name: 'รหัสผ่าน' }).fill('password123');
  });

  test('chaining locators', async ({ page }) => {
    await page.goto('/products');
    
    // หา element ใน element
    const productList = page.locator('.product-list');
    const firstProduct = productList.locator('.product-card').first();
    const productName = firstProduct.locator('.product-name');
    
    await expect(productName).toBeVisible();
  });

  test('data-testid selectors (recommended)', async ({ page }) => {
    await page.goto('/');
    
    // ใช้ data-testid เป็น convention
    const header = page.getByTestId('page-header');
    await expect(header).toBeVisible();
  });
});
```

---

## 5. Assertions ขั้นสูง

```typescript
// e2e/tests/assertions.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Playwright Assertions', () => {
  test('ตรวจสอบ visibility', async ({ page }) => {
    await page.goto('/');
    
    // toBeVisible / toBeHidden
    await expect(page.locator('.header')).toBeVisible();
    await expect(page.locator('.loading')).toBeHidden();
    
    // toBeAttached (in DOM แต่อาจ hidden)
    await expect(page.locator('[data-loaded]')).toBeAttached();
  });

  test('ตรวจสอบ text content', async ({ page }) => {
    await page.goto('/');
    
    // toHaveText - exact match
    await expect(page.locator('h1')).toHaveText('ยินดีต้อนรับ');
    
    // toContainText - partial match
    await expect(page.locator('.description')).toContainText('TypeScript');
    
    // toHaveValue - input value
    await page.fill('#search', 'test query');
    await expect(page.locator('#search')).toHaveValue('test query');
  });

  test('ตรวจสอบ attributes', async ({ page }) => {
    await page.goto('/');
    
    // toHaveAttribute
    await expect(page.locator('img.logo')).toHaveAttribute('alt', 'Logo');
    
    // toHaveClass
    await expect(page.locator('.active-menu-item')).toHaveClass(/active/);
    
    // toHaveCount
    await expect(page.locator('.menu-item')).toHaveCount(5);
  });

  test('ตรวจสอบ element state', async ({ page }) => {
    await page.goto('/form');
    
    // toBeEnabled / toBeDisabled
    await expect(page.locator('#submit-btn')).toBeEnabled();
    await expect(page.locator('#disabled-input')).toBeDisabled();
    
    // toBeChecked
    await page.check('#checkbox');
    await expect(page.locator('#checkbox')).toBeChecked();
    
    // toBeEditable
    await expect(page.locator('#text-input')).toBeEditable();
  });

  test('ตรวจสอบ URL และ title', async ({ page }) => {
    await page.goto('/');
    
    await expect(page).toHaveURL('http://localhost:3000/');
    await expect(page).toHaveTitle(/หน้าแรก/);
    
    // URL pattern
    await expect(page).toHaveURL(/localhost/);
  });

  test('Soft assertions', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Soft assertions ไม่หยุดการทดสอบเมื่อ fail
    await expect.soft(page.locator('.widget-1')).toBeVisible();
    await expect.soft(page.locator('.widget-2')).toBeVisible();
    await expect.soft(page.locator('.widget-3')).toBeVisible();
    
    // ตรวจสอบทุก soft assertion ณ สิ้นสุด test
    expect(test.info().errors).toHaveLength(0);
  });
});
```

---

## 6. Network Interception

### 6.1 Mock API Responses

```typescript
// e2e/tests/network.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Network Interception', () => {
  test('Mock API response', async ({ page }) => {
    // Intercept API call และส่ง mock response
    await page.route('/api/users', async route => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify([
          { id: '1', name: 'สมชาย', email: 'somchai@test.com' },
          { id: '2', name: 'สมหญิง', email: 'somying@test.com' }
        ])
      });
    });

    await page.goto('/users');
    
    // ตรวจสอบว่าแสดงข้อมูลจาก mock
    await expect(page.locator('.user-item')).toHaveCount(2);
    await expect(page.locator('.user-item').first()).toContainText('สมชาย');
  });

  test('Modify request', async ({ page }) => {
    // เพิ่ม header
    await page.route('/api/**', async route => {
      const headers = {
        ...route.request().headers(),
        'X-Test-Header': 'playwright-test'
      };
      await route.continue({ headers });
    });

    await page.goto('/');
  });

  test('Test error handling', async ({ page }) => {
    // Mock 500 error
    await page.route('/api/users', route => {
      route.fulfill({
        status: 500,
        body: JSON.stringify({ error: 'Internal Server Error' })
      });
    });

    await page.goto('/users');
    
    // ตรวจสอบว่าแสดง error state
    await expect(page.locator('.error-message')).toBeVisible();
    await expect(page.locator('.error-message')).toContainText('เกิดข้อผิดพลาด');
  });

  test('Intercept และ monitor requests', async ({ page }) => {
    const requests: string[] = [];
    
    // ดักจับ requests
    page.on('request', request => {
      if (request.url().includes('/api/')) {
        requests.push(request.url());
      }
    });

    await page.goto('/dashboard');
    await page.waitForLoadState('networkidle');
    
    // ตรวจสอบว่า requests ที่คาดหวังถูกเรียก
    expect(requests.some(url => url.includes('/api/users'))).toBe(true);
  });

  test('Wait for specific request', async ({ page }) => {
    // รอ API call ที่ specific
    const [request] = await Promise.all([
      page.waitForRequest('/api/products'),
      page.goto('/products')
    ]);

    expect(request.method()).toBe('GET');
  });

  test('Wait for response', async ({ page }) => {
    const [response] = await Promise.all([
      page.waitForResponse('/api/products'),
      page.goto('/products')
    ]);

    expect(response.status()).toBe(200);
    const data = await response.json();
    expect(Array.isArray(data)).toBe(true);
  });
});
```

---

## 7. Authentication Testing

### 7.1 Session Management

```typescript
// e2e/auth.setup.ts
import { test as setup } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '../.auth/user.json');

setup('authenticate', async ({ page }) => {
  // ทำการ login
  await page.goto('/login');
  await page.fill('#email', process.env.TEST_USER_EMAIL ?? 'test@example.com');
  await page.fill('#password', process.env.TEST_USER_PASSWORD ?? 'password123');
  await page.click('button[type="submit"]');
  
  // รอให้ redirect ไป dashboard
  await page.waitForURL('/dashboard');
  
  // บันทึก authentication state
  await page.context().storageState({ path: authFile });
});
```

```typescript
// playwright.config.ts - ใช้ auth state
import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/
    },
    {
      name: 'authenticated',
      testMatch: /.*\.spec\.ts/,
      dependencies: ['setup'],
      use: {
        storageState: '.auth/user.json'
      }
    }
  ]
});
```

```typescript
// e2e/tests/authenticated.spec.ts
import { test, expect } from '@playwright/test';

// test นี้จะใช้ session ที่ login ไว้แล้ว
test.describe('Authenticated User Tests', () => {
  test('เข้าถึง dashboard ได้โดยไม่ต้อง login', async ({ page }) => {
    await page.goto('/dashboard');
    
    // ไม่ควร redirect ไป login
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('.user-info')).toBeVisible();
  });

  test('แก้ไขโปรไฟล์', async ({ page }) => {
    await page.goto('/profile');
    
    await page.fill('#display-name', 'ชื่อใหม่ สกุลใหม่');
    await page.click('button[type="submit"]');
    
    await expect(page.locator('.success-toast')).toBeVisible();
  });
});
```

### 7.2 Testing Admin vs User Roles

```typescript
// e2e/auth/admin.setup.ts
import { test as setup } from '@playwright/test';

setup('authenticate as admin', async ({ page }) => {
  await page.goto('/login');
  await page.fill('#email', 'admin@example.com');
  await page.fill('#password', 'AdminPass1!');
  await page.click('[type="submit"]');
  await page.waitForURL('/admin');
  await page.context().storageState({ path: '.auth/admin.json' });
});
```

```typescript
// e2e/tests/admin.spec.ts
import { test, expect } from '@playwright/test';

test.use({ storageState: '.auth/admin.json' });

test.describe('Admin Panel', () => {
  test('admin เห็นเมนูจัดการ', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page.locator('nav a[href="/admin"]')).toBeVisible();
  });

  test('admin ลบผู้ใช้ได้', async ({ page }) => {
    await page.goto('/admin/users');
    
    const userCount = await page.locator('.user-row').count();
    
    await page.locator('.user-row').first().locator('button.delete').click();
    await page.locator('.confirm-dialog button.confirm').click();
    
    await expect(page.locator('.user-row')).toHaveCount(userCount - 1);
  });
});
```

---

## 8. Mobile Testing

```typescript
// e2e/tests/mobile.spec.ts
import { test, expect, devices } from '@playwright/test';

// กำหนด device emulation
test.use({ ...devices['iPhone 14 Pro'] });

test.describe('Mobile Tests', () => {
  test('แสดง mobile menu', async ({ page }) => {
    await page.goto('/');
    
    // ตรวจสอบว่า hamburger menu แสดง
    await expect(page.locator('.hamburger-menu')).toBeVisible();
    
    // ตรวจสอบว่า desktop menu ซ่อนอยู่
    await expect(page.locator('.desktop-nav')).toBeHidden();
  });

  test('เปิด mobile menu', async ({ page }) => {
    await page.goto('/');
    
    await page.tap('.hamburger-menu');
    
    await expect(page.locator('.mobile-menu')).toBeVisible();
    await expect(page.locator('.mobile-menu a')).toHaveCount(5);
  });

  test('Swipe gestures', async ({ page }) => {
    await page.goto('/gallery');
    
    const gallery = page.locator('.gallery-slider');
    
    // Swipe left
    await gallery.dispatchEvent('touchstart', {
      touches: [{ clientX: 300, clientY: 200 }]
    });
    await gallery.dispatchEvent('touchend', {
      changedTouches: [{ clientX: 100, clientY: 200 }]
    });
    
    // ตรวจสอบว่า slide เปลี่ยน
    await expect(page.locator('.slide.active')).toHaveAttribute('data-index', '1');
  });

  test('Touch interactions', async ({ page }) => {
    await page.goto('/map');
    
    // Pinch to zoom
    await page.locator('.map').dispatchEvent('wheel', {
      deltaY: -100,
      ctrlKey: true
    });
  });
});
```

---

## 9. Screenshots และ Videos

### 9.1 Manual Screenshots

```typescript
// e2e/tests/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Testing', () => {
  test('Screenshot comparison', async ({ page }) => {
    await page.goto('/');
    
    // Full page screenshot
    await expect(page).toHaveScreenshot('homepage-full.png', {
      fullPage: true,
      maxDiffPixelRatio: 0.01
    });
  });

  test('Element screenshot', async ({ page }) => {
    await page.goto('/');
    
    const header = page.locator('header');
    
    // Screenshot ของ specific element
    await expect(header).toHaveScreenshot('header.png', {
      maxDiffPixels: 50
    });
  });

  test('Screenshot กับ mask', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Mask dynamic content (เวลา, ข้อมูลที่เปลี่ยนแปลง)
    await expect(page).toHaveScreenshot('dashboard.png', {
      mask: [
        page.locator('.current-time'),
        page.locator('.last-updated'),
        page.locator('.user-avatar')
      ]
    });
  });

  test('บันทึก screenshot ใน test', async ({ page }) => {
    await page.goto('/');
    
    // บันทึก screenshot เป็น buffer
    const screenshot = await page.screenshot();
    
    // หรือบันทึกเป็นไฟล์
    await page.screenshot({ path: 'test-results/homepage.png' });
    
    expect(screenshot).toBeDefined();
  });
});
```

### 9.2 Video Recording

```typescript
// playwright.config.ts - เปิด video recording
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    video: {
      mode: 'retain-on-failure', // บันทึกเฉพาะตอน fail
      size: { width: 1280, height: 720 }
    }
  }
});
```

---

## 10. CI/CD Integration

### 10.1 GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 60

    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps

      - name: Build application
        run: npm run build

      - name: Start application
        run: npm run preview &
        env:
          PORT: 3000

      - name: Wait for app to be ready
        run: npx wait-on http://localhost:3000

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload traces on failure
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-traces
          path: test-results/
```

---

## 11. Complete E2E Test Suite

### 11.1 E-commerce Application Tests

```typescript
// e2e/tests/e-commerce.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { ProductsPage } from '../pages/products.page';
import { CartPage } from '../pages/cart.page';
import { CheckoutPage } from '../pages/checkout.page';

test.describe('E-commerce Purchase Flow', () => {
  test('Complete purchase journey', async ({ page }) => {
    const loginPage = new LoginPage(page);
    const productsPage = new ProductsPage(page);

    // Step 1: Login
    await loginPage.goto();
    await loginPage.login('buyer@example.com', 'BuyerPass1!');
    await loginPage.expectRedirectedToDashboard();

    // Step 2: Browse products
    await productsPage.goto();
    await expect(productsPage.page).toHaveURL('/products');

    // Step 3: Filter products
    await productsPage.filterByCategory('electronics');
    const productCount = await productsPage.getProductCount();
    expect(productCount).toBeGreaterThan(0);

    // Step 4: Add to cart
    await productsPage.addToCart(0);
    expect(await productsPage.getCartCount()).toBe(1);

    // Step 5: View cart
    await page.click('.cart-icon');
    await expect(page).toHaveURL('/cart');

    // Step 6: Proceed to checkout
    await page.click('button.checkout');
    await expect(page).toHaveURL('/checkout');

    // Step 7: Fill shipping info
    await page.fill('#address', '123 ถนนสุขุมวิท');
    await page.fill('#city', 'กรุงเทพมหานคร');
    await page.selectOption('#province', 'กรุงเทพ');
    await page.fill('#postal-code', '10110');

    // Step 8: Fill payment
    await page.fill('#card-number', '4111111111111111');
    await page.fill('#card-expiry', '12/25');
    await page.fill('#card-cvv', '123');

    // Step 9: Confirm order
    await page.click('button.place-order');
    await page.waitForURL('/order-confirmation');

    // Step 10: Verify order
    await expect(page.locator('.order-success')).toBeVisible();
    await expect(page.locator('.order-number')).toBeVisible();
  });

  test('Add multiple products and checkout', async ({ page }) => {
    // Login and navigate
    await page.goto('/products');
    
    // Add 3 products
    for (let i = 0; i < 3; i++) {
      await page.locator('.add-to-cart').nth(i).click();
      await page.waitForTimeout(500);
    }

    expect(
      parseInt(await page.locator('.cart-count').textContent() ?? '0')
    ).toBe(3);
  });
});
```

### 11.2 Form Validation Tests

```typescript
// e2e/tests/form-validation.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Form Validation', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/register');
  });

  test('แสดง validation errors เมื่อ submit ว่าง', async ({ page }) => {
    await page.click('button[type="submit"]');
    
    await expect(page.locator('#name-error')).toHaveText('กรุณากรอกชื่อ');
    await expect(page.locator('#email-error')).toHaveText('กรุณากรอกอีเมล');
    await expect(page.locator('#password-error')).toHaveText('กรุณากรอกรหัสผ่าน');
  });

  test('validate email format', async ({ page }) => {
    await page.fill('#email', 'not-valid');
    await page.click('button[type="submit"]');
    
    await expect(page.locator('#email-error')).toHaveText('รูปแบบอีเมลไม่ถูกต้อง');
  });

  test('password strength indicator', async ({ page }) => {
    await page.fill('#password', 'weak');
    await expect(page.locator('.password-strength')).toHaveClass(/weak/);
    
    await page.fill('#password', 'Strong1!Pass');
    await expect(page.locator('.password-strength')).toHaveClass(/strong/);
  });

  test('successful registration', async ({ page }) => {
    await page.fill('#name', 'สมชาย ใจดี');
    await page.fill('#email', `test${Date.now()}@example.com`);
    await page.fill('#password', 'SecurePass1!');
    await page.fill('#confirm-password', 'SecurePass1!');
    await page.check('#accept-terms');
    
    await page.click('button[type="submit"]');
    
    await expect(page).toHaveURL('/verify-email');
  });
});
```

---

## 12. Custom Fixtures

```typescript
// e2e/fixtures/test-fixtures.ts
import { test as base, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';
import { DashboardPage } from '../pages/dashboard.page';

interface TestFixtures {
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
  authenticatedPage: void;
}

export const test = base.extend<TestFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  dashboardPage: async ({ page }, use) => {
    const dashboardPage = new DashboardPage(page);
    await use(dashboardPage);
  },

  authenticatedPage: async ({ page }, use) => {
    // Auto-login fixture
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('user@example.com', 'Password1!');
    await page.waitForURL('/dashboard');
    await use();
  }
});

export { expect };
```

```typescript
// e2e/tests/with-fixtures.spec.ts
import { test, expect } from '../fixtures/test-fixtures';

test.describe('With Fixtures', () => {
  test('ใช้ loginPage fixture', async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login('user@example.com', 'Password1!');
    await loginPage.expectRedirectedToDashboard();
  });

  test('ใช้ authenticatedPage fixture', async ({ authenticatedPage, page }) => {
    // authenticatedPage จัดการ login ให้แล้ว
    await expect(page).toHaveURL('/dashboard');
  });
});
```

---

## สรุปบทที่ 63

ในบทนี้เราได้เรียนรู้:

1. **Playwright Setup** - การติดตั้งและตั้งค่า
2. **Browser Automation** - navigation, clicks, form filling
3. **Page Object Model** - การจัดการโค้ดด้วย POM
4. **Selectors** - CSS, text, ARIA, data-testid
5. **Assertions** - การตรวจสอบ state ต่างๆ
6. **Network Interception** - mock APIs และ monitoring
7. **Authentication Testing** - session management
8. **Mobile Testing** - device emulation
9. **Screenshots/Videos** - visual testing
10. **CI/CD Integration** - GitHub Actions
11. **Custom Fixtures** - reusable test setup
12. **Complete E2E Suite** - e-commerce flow

---

## แบบฝึกหัด

1. สร้าง Page Object สำหรับ blog application
2. เขียน E2E tests สำหรับ authentication flow
3. ตั้งค่า visual regression testing
4. เขียน test สำหรับ mobile responsive design
5. สร้าง CI/CD pipeline ที่รัน E2E tests

---

*ต่อไป: ส่วนที่ 64 - AWS with TypeScript*
