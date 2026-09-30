# ตอนที่ 26: React Components และ Props กับ TypeScript

## บทนำ

การเขียน React Components ด้วย TypeScript ช่วยให้เราสามารถกำหนดชนิดของ props ได้อย่างชัดเจน ลดข้อผิดพลาด และทำให้โค้ดอ่านเข้าใจง่ายขึ้น ในบทนี้เราจะเรียนรู้เทคนิคต่างๆ ในการ type components และ props อย่างละเอียด

---

## 26.1 Functional Component Typing

### วิธีการ type Functional Components

```typescript
// src/components/examples/FunctionalComponents.tsx
import React from 'react';

// ===== วิธีที่ 1: React.FC<Props> =====
interface GreetingProps {
  name: string;
  greeting?: string;
}

const Greeting: React.FC<GreetingProps> = ({ name, greeting = 'สวัสดี' }) => {
  return <h1>{greeting}, {name}!</h1>;
};

// ===== วิธีที่ 2: Function declaration พร้อม return type =====
interface TitleProps {
  text: string;
  level?: 1 | 2 | 3 | 4 | 5 | 6;
}

function Title({ text, level = 1 }: TitleProps): JSX.Element {
  const Tag = `h${level}` as keyof JSX.IntrinsicElements;
  return <Tag>{text}</Tag>;
}

// ===== วิธีที่ 3: Arrow function โดยตรง =====
interface BadgeProps {
  label: string;
  color: 'red' | 'green' | 'blue' | 'yellow';
}

const Badge = ({ label, color }: BadgeProps) => (
  <span className={`badge badge-${color}`}>{label}</span>
);

// ===== วิธีที่ 4: component ที่ return null บางกรณี =====
interface ConditionalProps {
  show: boolean;
  message: string;
}

const ConditionalComponent: React.FC<ConditionalProps> = ({ show, message }) => {
  if (!show) return null;
  return <div className="message">{message}</div>;
};

// ===== วิธีที่ 5: React.VFC (Void Function Component) - deprecated ใน React 18 =====
// ใน React 18+ ใช้ React.FC แทน เพราะ children ถูกลบออกจาก implicit props

export { Greeting, Title, Badge, ConditionalComponent };
```

### Component ที่มี children

```typescript
// src/components/examples/WithChildren.tsx
import React from 'react';

// วิธีที่ 1: explicit children prop
interface ContainerProps {
  children: React.ReactNode;
  className?: string;
}

const Container: React.FC<ContainerProps> = ({ children, className }) => (
  <div className={`container ${className || ''}`}>{children}</div>
);

// วิธีที่ 2: React.PropsWithChildren
interface CardProps {
  title: string;
  footer?: React.ReactNode;
}

const Card: React.FC<React.PropsWithChildren<CardProps>> = ({
  title,
  footer,
  children
}) => (
  <div className="card">
    <div className="card-header"><h3>{title}</h3></div>
    <div className="card-body">{children}</div>
    {footer && <div className="card-footer">{footer}</div>}
  </div>
);

// วิธีที่ 3: children เป็น render function
interface DataProviderProps<T> {
  data: T;
  children: (data: T) => React.ReactNode;
}

function DataProvider<T>({ data, children }: DataProviderProps<T>) {
  return <>{children(data)}</>;
}

// การใช้งาน
const Usage: React.FC = () => (
  <>
    <Container className="main">
      <p>เนื้อหา</p>
    </Container>

    <Card title="หัวข้อ" footer={<button>ปุ่ม</button>}>
      <p>เนื้อหาในการ์ด</p>
    </Card>

    <DataProvider data={{ name: 'สมชาย', age: 25 }}>
      {(user) => <div>{user.name} - {user.age} ปี</div>}
    </DataProvider>
  </>
);

export { Container, Card, DataProvider };
```

---

## 26.2 Props Interfaces

### การออกแบบ Props Interfaces

```typescript
// src/types/component.types.ts
import React from 'react';

// Base props ที่หลาย components ใช้ร่วมกัน
export interface BaseProps {
  id?: string;
  className?: string;
  style?: React.CSSProperties;
  'data-testid'?: string;
}

// Polymorphic component type
export type PolymorphicProps<C extends React.ElementType, P = {}> = P &
  Omit<React.ComponentPropsWithoutRef<C>, keyof P> & {
    as?: C;
  };

// Size variant
export type Size = 'xs' | 'sm' | 'md' | 'lg' | 'xl';

// Color variant
export type ColorVariant =
  | 'primary'
  | 'secondary'
  | 'success'
  | 'warning'
  | 'danger'
  | 'info';
```

```typescript
// src/components/examples/PropsInterfaces.tsx
import React from 'react';
import { BaseProps, Size, ColorVariant } from '../../types/component.types';

// ===== Input Component =====
interface InputProps extends BaseProps {
  // Required props
  value: string;
  onChange: (value: string) => void;
  
  // Optional props
  type?: 'text' | 'email' | 'password' | 'number' | 'tel' | 'url';
  placeholder?: string;
  label?: string;
  error?: string;
  hint?: string;
  disabled?: boolean;
  readOnly?: boolean;
  size?: Size;
  
  // Advanced
  leftAddon?: React.ReactNode;
  rightAddon?: React.ReactNode;
  onBlur?: React.FocusEventHandler<HTMLInputElement>;
  onFocus?: React.FocusEventHandler<HTMLInputElement>;
  onKeyDown?: React.KeyboardEventHandler<HTMLInputElement>;
}

const Input: React.FC<InputProps> = ({
  id,
  className,
  value,
  onChange,
  type = 'text',
  placeholder,
  label,
  error,
  hint,
  disabled = false,
  readOnly = false,
  size = 'md',
  leftAddon,
  rightAddon,
  onBlur,
  onFocus,
  onKeyDown,
  'data-testid': testId,
  ...rest
}) => {
  const inputId = id || `input-${Math.random().toString(36).substr(2, 9)}`;
  
  return (
    <div className={`input-wrapper ${className || ''}`}>
      {label && (
        <label htmlFor={inputId} className="input-label">
          {label}
        </label>
      )}
      <div className={`input-container input-${size} ${error ? 'input-error' : ''}`}>
        {leftAddon && <span className="input-left-addon">{leftAddon}</span>}
        <input
          id={inputId}
          type={type}
          value={value}
          placeholder={placeholder}
          disabled={disabled}
          readOnly={readOnly}
          onChange={(e) => onChange(e.target.value)}
          onBlur={onBlur}
          onFocus={onFocus}
          onKeyDown={onKeyDown}
          data-testid={testId}
          className="input-element"
          {...rest}
        />
        {rightAddon && <span className="input-right-addon">{rightAddon}</span>}
      </div>
      {error && <p className="input-error-message">{error}</p>}
      {hint && !error && <p className="input-hint">{hint}</p>}
    </div>
  );
};

// ===== Alert Component =====
interface AlertProps extends BaseProps {
  variant: ColorVariant;
  title?: string;
  icon?: React.ReactNode;
  dismissible?: boolean;
  onDismiss?: () => void;
  children: React.ReactNode;
}

const Alert: React.FC<AlertProps> = ({
  variant,
  title,
  icon,
  dismissible = false,
  onDismiss,
  children,
  className
}) => (
  <div className={`alert alert-${variant} ${className || ''}`} role="alert">
    {icon && <span className="alert-icon">{icon}</span>}
    <div className="alert-content">
      {title && <strong className="alert-title">{title}</strong>}
      <div className="alert-body">{children}</div>
    </div>
    {dismissible && (
      <button
        className="alert-dismiss"
        onClick={onDismiss}
        aria-label="ปิด"
      >
        ×
      </button>
    )}
  </div>
);

export { Input, Alert };
```

---

## 26.3 Children Prop Types

### ReactNode, ReactElement, และ ReactChild

```typescript
// src/components/examples/ChildrenPropTypes.tsx
import React from 'react';

// ===== ReactNode =====
// ครอบคลุม: elements, strings, numbers, booleans, null, undefined, arrays, fragments
interface LayoutProps {
  header: React.ReactNode;
  sidebar: React.ReactNode;
  footer: React.ReactNode;
  children: React.ReactNode;  // ใช้บ่อยที่สุด
}

const Layout: React.FC<LayoutProps> = ({ header, sidebar, footer, children }) => (
  <div className="layout">
    <header className="layout-header">{header}</header>
    <div className="layout-body">
      <aside className="layout-sidebar">{sidebar}</aside>
      <main className="layout-main">{children}</main>
    </div>
    <footer className="layout-footer">{footer}</footer>
  </div>
);

// ===== ReactElement =====
// เฉพาะ JSX elements เท่านั้น (ไม่รับ string, number, null)
interface TabsProps {
  children: React.ReactElement | React.ReactElement[];
  activeTab?: string;
  onTabChange?: (tabId: string) => void;
}

const Tabs: React.FC<TabsProps> = ({ children, activeTab }) => {
  const tabs = React.Children.toArray(children) as React.ReactElement[];
  
  return (
    <div className="tabs">
      <div className="tab-list">
        {tabs.map((tab) => (
          <button
            key={tab.props.id}
            className={`tab-button ${activeTab === tab.props.id ? 'active' : ''}`}
          >
            {tab.props.label}
          </button>
        ))}
      </div>
      <div className="tab-panels">
        {tabs.find(tab => tab.props.id === activeTab)}
      </div>
    </div>
  );
};

// ===== Children ที่เป็น function =====
interface WithLoadingProps {
  loading: boolean;
  error?: Error;
  children: () => React.ReactNode;  // render function
}

const WithLoading: React.FC<WithLoadingProps> = ({ loading, error, children }) => {
  if (loading) return <div className="spinner">กำลังโหลด...</div>;
  if (error) return <div className="error">{error.message}</div>;
  return <>{children()}</>;
};

// ===== Children ที่เป็น array =====
interface GridProps {
  columns?: number;
  children: React.ReactNode[];  // ต้องการ array
  gap?: number;
}

const Grid: React.FC<GridProps> = ({ columns = 3, children, gap = 16 }) => (
  <div
    className="grid"
    style={{
      display: 'grid',
      gridTemplateColumns: `repeat(${columns}, 1fr)`,
      gap: `${gap}px`
    }}
  >
    {children}
  </div>
);

// ===== cloneElement กับ TypeScript =====
interface EnhancedChildrenProps {
  children: React.ReactElement;
  extraProp: string;
}

const EnhancedChildren: React.FC<EnhancedChildrenProps> = ({ children, extraProp }) => {
  return React.cloneElement(children, { 'data-enhanced': extraProp });
};

export { Layout, Tabs, WithLoading, Grid, EnhancedChildren };
```

---

## 26.4 Optional และ Required Props

```typescript
// src/components/examples/OptionalRequired.tsx
import React from 'react';

// ===== Required Props =====
interface StrictProps {
  // ทั้งหมดเป็น required
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}

// ===== Optional Props =====
interface FlexibleProps {
  // บาง props เป็น optional
  title: string;       // required
  subtitle?: string;   // optional
  description?: string;
  imageUrl?: string;
  tags?: string[];
  maxLength?: number;
  onClick?: () => void;
}

// ===== การใช้ Intersection Types =====
interface RequiredFields {
  id: string;
  name: string;
}

interface OptionalFields {
  avatar?: string;
  bio?: string;
  website?: string;
  location?: string;
}

type UserProfileProps = RequiredFields & OptionalFields;

const UserProfile: React.FC<UserProfileProps> = ({
  id,
  name,
  avatar,
  bio,
  website,
  location
}) => (
  <div className="user-profile">
    {avatar && <img src={avatar} alt={name} className="avatar" />}
    <h2>{name}</h2>
    {bio && <p className="bio">{bio}</p>}
    {website && <a href={website}>{website}</a>}
    {location && <span className="location">📍 {location}</span>}
  </div>
);

// ===== Partial และ Required utility types =====
interface FullConfig {
  apiUrl: string;
  timeout: number;
  retries: number;
  headers: Record<string, string>;
  debug: boolean;
}

// ทุก fields เป็น optional
type PartialConfig = Partial<FullConfig>;

// ทุก fields เป็น required
type RequiredConfig = Required<PartialConfig>;

// แค่บาง fields
type MinimalConfig = Pick<FullConfig, 'apiUrl'> & Partial<Omit<FullConfig, 'apiUrl'>>;

// Component ที่รับ partial config
interface ConfigProviderProps {
  config: MinimalConfig;
  children: React.ReactNode;
}

const ConfigProvider: React.FC<ConfigProviderProps> = ({ config, children }) => {
  const mergedConfig: FullConfig = {
    apiUrl: config.apiUrl,
    timeout: config.timeout ?? 5000,
    retries: config.retries ?? 3,
    headers: config.headers ?? {},
    debug: config.debug ?? false
  };

  return (
    <div data-api-url={mergedConfig.apiUrl}>
      {children}
    </div>
  );
};

export { UserProfile, ConfigProvider };
export type { UserProfileProps, MinimalConfig };
```

---

## 26.5 Default Props

```typescript
// src/components/examples/DefaultProps.tsx
import React from 'react';

// ===== วิธีที่ 1: Default parameter values (แนะนำ) =====
interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
  onClick?: () => void;
}

const Button: React.FC<ButtonProps> = ({
  children,
  variant = 'primary',  // default value
  size = 'md',          // default value
  disabled = false,     // default value
  loading = false,
  onClick
}) => (
  <button
    className={`btn btn-${variant} btn-${size}`}
    disabled={disabled || loading}
    onClick={onClick}
  >
    {loading ? 'กำลังโหลด...' : children}
  </button>
);

// ===== วิธีที่ 2: defaultProps (legacy - ไม่แนะนำใน modern React) =====
interface LegacyProps {
  message: string;
  type?: string;
  timeout?: number;
}

class LegacyToast extends React.Component<LegacyProps> {
  static defaultProps: Partial<LegacyProps> = {
    type: 'info',
    timeout: 3000
  };

  render() {
    const { message, type, timeout } = this.props;
    return (
      <div className={`toast toast-${type}`} data-timeout={timeout}>
        {message}
      </div>
    );
  }
}

// ===== วิธีที่ 3: defaultProps กับ Functional Component =====
// (ไม่แนะนำใน TypeScript เพราะมี issues กับ type inference)
const Notification: React.FC<LegacyProps> & {
  defaultProps: Partial<LegacyProps>;
} = ({ message, type, timeout }) => (
  <div className={`notification notification-${type}`}>
    {message} (หายใน {timeout}ms)
  </div>
);

Notification.defaultProps = {
  type: 'info',
  timeout: 5000
};

// ===== วิธีที่ 4: Object spread สำหรับ default config =====
interface ChartConfig {
  width?: number;
  height?: number;
  margins?: {
    top: number;
    right: number;
    bottom: number;
    left: number;
  };
  colors?: string[];
  showGrid?: boolean;
  showLegend?: boolean;
}

interface ChartProps {
  data: Array<{ label: string; value: number }>;
  config?: ChartConfig;
}

const DEFAULT_CONFIG: Required<ChartConfig> = {
  width: 600,
  height: 400,
  margins: { top: 20, right: 20, bottom: 40, left: 40 },
  colors: ['#3b82f6', '#10b981', '#f59e0b', '#ef4444'],
  showGrid: true,
  showLegend: true
};

const Chart: React.FC<ChartProps> = ({ data, config = {} }) => {
  const finalConfig = { ...DEFAULT_CONFIG, ...config };
  
  return (
    <svg
      width={finalConfig.width}
      height={finalConfig.height}
      className="chart"
    >
      {/* Chart rendering */}
      {data.map((item, index) => (
        <rect
          key={item.label}
          fill={finalConfig.colors[index % finalConfig.colors.length]}
          x={index * 50}
          y={0}
          width={40}
          height={item.value}
        />
      ))}
    </svg>
  );
};

export { Button, LegacyToast, Notification, Chart };
```

---

## 26.6 Prop Drilling กับ Types

```typescript
// src/components/examples/PropDrilling.tsx
import React from 'react';

// ===== ปัญหา Prop Drilling =====
interface UserData {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  preferences: {
    theme: 'light' | 'dark';
    language: string;
    notifications: boolean;
  };
}

// Level 1
interface AppProps {
  user: UserData;
  onLogout: () => void;
  onUpdatePreferences: (prefs: UserData['preferences']) => void;
}

const App: React.FC<AppProps> = ({ user, onLogout, onUpdatePreferences }) => (
  <div className="app">
    <Header user={user} onLogout={onLogout} />
    <Main user={user} onUpdatePreferences={onUpdatePreferences} />
  </div>
);

// Level 2
interface HeaderProps {
  user: UserData;
  onLogout: () => void;
}

const Header: React.FC<HeaderProps> = ({ user, onLogout }) => (
  <header>
    <UserMenu user={user} onLogout={onLogout} />
  </header>
);

// Level 3
interface UserMenuProps {
  user: UserData;
  onLogout: () => void;
}

const UserMenu: React.FC<UserMenuProps> = ({ user, onLogout }) => (
  <div className="user-menu">
    <UserAvatar user={user} />
    <button onClick={onLogout}>ออกจากระบบ</button>
  </div>
);

// Level 4
interface UserAvatarProps {
  user: Pick<UserData, 'name' | 'avatar'>;
}

const UserAvatar: React.FC<UserAvatarProps> = ({ user }) => (
  <div className="avatar">
    {user.avatar
      ? <img src={user.avatar} alt={user.name} />
      : <span>{user.name[0].toUpperCase()}</span>
    }
  </div>
);

// Level 2
interface MainProps {
  user: UserData;
  onUpdatePreferences: (prefs: UserData['preferences']) => void;
}

const Main: React.FC<MainProps> = ({ user, onUpdatePreferences }) => (
  <main>
    <SettingsPanel
      preferences={user.preferences}
      onUpdate={onUpdatePreferences}
    />
  </main>
);

// Level 3
interface SettingsPanelProps {
  preferences: UserData['preferences'];
  onUpdate: (prefs: UserData['preferences']) => void;
}

const SettingsPanel: React.FC<SettingsPanelProps> = ({ preferences, onUpdate }) => (
  <div className="settings">
    <h2>การตั้งค่า</h2>
    <label>
      <input
        type="checkbox"
        checked={preferences.notifications}
        onChange={(e) => onUpdate({ ...preferences, notifications: e.target.checked })}
      />
      รับการแจ้งเตือน
    </label>
  </div>
);

export { App };
```

---

## 26.7 forwardRef Typing

```typescript
// src/components/common/Input/ForwardRefInput.tsx
import React, { forwardRef, useImperativeHandle } from 'react';

// ===== พื้นฐาน forwardRef =====
interface BasicInputProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
  className?: string;
}

// forwardRef<Ref type, Props type>
const BasicInput = forwardRef<HTMLInputElement, BasicInputProps>(
  ({ value, onChange, placeholder, disabled, className }, ref) => (
    <input
      ref={ref}
      value={value}
      onChange={(e) => onChange(e.target.value)}
      placeholder={placeholder}
      disabled={disabled}
      className={`input ${className || ''}`}
    />
  )
);

BasicInput.displayName = 'BasicInput';

// ===== forwardRef กับ useImperativeHandle =====
interface TextEditorHandle {
  focus: () => void;
  blur: () => void;
  clear: () => void;
  getValue: () => string;
  setValue: (value: string) => void;
  select: () => void;
}

interface TextEditorProps {
  defaultValue?: string;
  onChange?: (value: string) => void;
  rows?: number;
  readOnly?: boolean;
}

const TextEditor = forwardRef<TextEditorHandle, TextEditorProps>(
  ({ defaultValue = '', onChange, rows = 5, readOnly = false }, ref) => {
    const textareaRef = React.useRef<HTMLTextAreaElement>(null);
    const [value, setValue] = React.useState(defaultValue);

    useImperativeHandle(ref, () => ({
      focus: () => textareaRef.current?.focus(),
      blur: () => textareaRef.current?.blur(),
      clear: () => {
        setValue('');
        onChange?.('');
      },
      getValue: () => value,
      setValue: (newValue: string) => {
        setValue(newValue);
        onChange?.(newValue);
      },
      select: () => textareaRef.current?.select()
    }));

    const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
      setValue(e.target.value);
      onChange?.(e.target.value);
    };

    return (
      <textarea
        ref={textareaRef}
        value={value}
        onChange={handleChange}
        rows={rows}
        readOnly={readOnly}
        className="text-editor"
      />
    );
  }
);

TextEditor.displayName = 'TextEditor';

// ===== การใช้งาน forwardRef =====
const ForwardRefUsage: React.FC = () => {
  const inputRef = React.useRef<HTMLInputElement>(null);
  const editorRef = React.useRef<TextEditorHandle>(null);

  const handleFocusInput = () => {
    inputRef.current?.focus();
  };

  const handleClearEditor = () => {
    editorRef.current?.clear();
  };

  const handleGetEditorValue = () => {
    const value = editorRef.current?.getValue();
    console.log('Editor value:', value);
  };

  return (
    <div>
      <BasicInput
        ref={inputRef}
        value=""
        onChange={(v) => console.log(v)}
        placeholder="กรอกข้อมูล"
      />
      <button onClick={handleFocusInput}>Focus Input</button>

      <TextEditor
        ref={editorRef}
        defaultValue="เนื้อหาเริ่มต้น"
        onChange={(v) => console.log('Changed:', v)}
      />
      <button onClick={handleClearEditor}>ล้าง</button>
      <button onClick={handleGetEditorValue}>ดูค่า</button>
    </div>
  );
};

export { BasicInput, TextEditor };
export type { TextEditorHandle, BasicInputProps, TextEditorProps };
```

---

## 26.8 HOC (Higher Order Component) Typing

```typescript
// src/hoc/withAuth.tsx
import React, { ComponentType, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

// Interface สำหรับ user authentication
interface AuthUser {
  id: string;
  name: string;
  email: string;
  roles: string[];
}

// Props ที่ HOC จะ inject
interface WithAuthProps {
  user: AuthUser;
  isAuthenticated: boolean;
}

// HOC สำหรับ authentication
function withAuth<P extends WithAuthProps>(
  WrappedComponent: ComponentType<P>
): ComponentType<Omit<P, keyof WithAuthProps>> {
  const WithAuthComponent = (props: Omit<P, keyof WithAuthProps>) => {
    // Mock auth hook (ใช้ context จริงในการใช้งานจริง)
    const user: AuthUser | null = null; // จาก context
    const isAuthenticated = user !== null;
    const navigate = useNavigate();

    useEffect(() => {
      if (!isAuthenticated) {
        navigate('/login');
      }
    }, [isAuthenticated, navigate]);

    if (!isAuthenticated || !user) {
      return <div>กำลังตรวจสอบสิทธิ์...</div>;
    }

    return (
      <WrappedComponent
        {...(props as P)}
        user={user}
        isAuthenticated={isAuthenticated}
      />
    );
  };

  WithAuthComponent.displayName = `WithAuth(${
    WrappedComponent.displayName || WrappedComponent.name
  })`;

  return WithAuthComponent;
}

// HOC สำหรับ loading state
interface WithLoadingProps {
  isLoading: boolean;
  loadingFallback?: React.ReactNode;
}

function withLoading<P extends object>(
  WrappedComponent: ComponentType<P>,
  defaultFallback?: React.ReactNode
) {
  const WithLoadingComponent: React.FC<P & WithLoadingProps> = ({
    isLoading,
    loadingFallback,
    ...props
  }) => {
    if (isLoading) {
      return (
        <div className="loading-wrapper">
          {loadingFallback || defaultFallback || <span>กำลังโหลด...</span>}
        </div>
      );
    }

    return <WrappedComponent {...(props as P)} />;
  };

  WithLoadingComponent.displayName = `WithLoading(${
    WrappedComponent.displayName || WrappedComponent.name
  })`;

  return WithLoadingComponent;
}

// HOC สำหรับ error boundary
function withErrorBoundary<P extends object>(
  WrappedComponent: ComponentType<P>,
  FallbackComponent?: ComponentType<{ error: Error; reset: () => void }>
) {
  class WithErrorBoundary extends React.Component<
    P,
    { hasError: boolean; error: Error | null }
  > {
    static displayName = `WithErrorBoundary(${
      WrappedComponent.displayName || WrappedComponent.name
    })`;

    state = { hasError: false, error: null as Error | null };

    static getDerivedStateFromError(error: Error) {
      return { hasError: true, error };
    }

    reset = () => {
      this.setState({ hasError: false, error: null });
    };

    render() {
      if (this.state.hasError && this.state.error) {
        if (FallbackComponent) {
          return (
            <FallbackComponent error={this.state.error} reset={this.reset} />
          );
        }
        return (
          <div>
            <p>เกิดข้อผิดพลาด: {this.state.error.message}</p>
            <button onClick={this.reset}>ลองใหม่</button>
          </div>
        );
      }
      return <WrappedComponent {...this.props} />;
    }
  }

  return WithErrorBoundary;
}

// ===== การใช้งาน HOCs =====
interface DashboardProps extends WithAuthProps {
  title: string;
}

const Dashboard: React.FC<DashboardProps> = ({ title, user }) => (
  <div>
    <h1>{title}</h1>
    <p>ยินดีต้อนรับ, {user.name}</p>
  </div>
);

const AuthDashboard = withAuth(Dashboard);

interface ProductListProps {
  products: Array<{ id: string; name: string }>;
}

const ProductList: React.FC<ProductListProps> = ({ products }) => (
  <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
);

const LoadingProductList = withLoading(ProductList);
const SafeProductList = withErrorBoundary(LoadingProductList);

export { withAuth, withLoading, withErrorBoundary, AuthDashboard, SafeProductList };
```

---

## 26.9 Render Props Pattern

```typescript
// src/components/patterns/RenderProps.tsx
import React, { useState, useEffect } from 'react';

// ===== Basic Render Props =====
interface MouseTrackerRenderProps {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  render: (props: MouseTrackerRenderProps) => React.ReactNode;
  // หรือใช้ children as function
  children?: (props: MouseTrackerRenderProps) => React.ReactNode;
}

const MouseTracker: React.FC<MouseTrackerProps> = ({ render, children }) => {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  useEffect(() => {
    const handleMouseMove = (e: MouseEvent) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };

    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);

  const renderFn = render || children;
  return <>{renderFn?.(position)}</>;
};

// ===== Data Fetcher with Render Props =====
interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

interface DataFetcherProps<T> {
  url: string;
  children: (state: FetchState<T>) => React.ReactNode;
  initialData?: T;
}

function DataFetcher<T>({ url, children, initialData }: DataFetcherProps<T>) {
  const [state, setState] = useState<Omit<FetchState<T>, 'refetch'>>({
    data: initialData || null,
    loading: false,
    error: null
  });

  const fetchData = async () => {
    setState(prev => ({ ...prev, loading: true, error: null }));
    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP Error: ${response.status}`);
      const data = await response.json() as T;
      setState({ data, loading: false, error: null });
    } catch (error) {
      setState(prev => ({
        ...prev,
        loading: false,
        error: error as Error
      }));
    }
  };

  useEffect(() => {
    void fetchData();
  }, [url]);

  return <>{children({ ...state, refetch: fetchData })}</>;
}

// ===== Toggle with Render Props =====
interface ToggleRenderProps {
  isOn: boolean;
  toggle: () => void;
  setOn: () => void;
  setOff: () => void;
}

interface ToggleProps {
  initialOn?: boolean;
  onToggle?: (isOn: boolean) => void;
  children: (props: ToggleRenderProps) => React.ReactNode;
}

const Toggle: React.FC<ToggleProps> = ({ initialOn = false, onToggle, children }) => {
  const [isOn, setIsOn] = useState(initialOn);

  const toggle = () => {
    const newState = !isOn;
    setIsOn(newState);
    onToggle?.(newState);
  };

  const setOn = () => {
    setIsOn(true);
    onToggle?.(true);
  };

  const setOff = () => {
    setIsOn(false);
    onToggle?.(false);
  };

  return <>{children({ isOn, toggle, setOn, setOff })}</>;
};

// ===== การใช้งาน =====
const RenderPropsUsage: React.FC = () => (
  <div>
    <MouseTracker render={({ x, y }) => (
      <div>ตำแหน่ง Mouse: {x}, {y}</div>
    )} />

    <DataFetcher<{ id: number; name: string }[]> url="/api/users">
      {({ data, loading, error, refetch }) => {
        if (loading) return <div>กำลังโหลด...</div>;
        if (error) return <div>Error: {error.message}</div>;
        return (
          <div>
            <ul>{data?.map(u => <li key={u.id}>{u.name}</li>)}</ul>
            <button onClick={refetch}>รีโหลด</button>
          </div>
        );
      }}
    </DataFetcher>

    <Toggle initialOn={false} onToggle={(v) => console.log('Toggled:', v)}>
      {({ isOn, toggle }) => (
        <button onClick={toggle}>
          {isOn ? 'ปิด' : 'เปิด'}
        </button>
      )}
    </Toggle>
  </div>
);

export { MouseTracker, DataFetcher, Toggle };
```

---

## 26.10 Compound Components Pattern

```typescript
// src/components/patterns/CompoundComponents.tsx
import React, { createContext, useContext, useState } from 'react';

// ===== Accordion Compound Component =====

// Context
interface AccordionContextValue {
  expandedId: string | null;
  toggle: (id: string) => void;
}

const AccordionContext = createContext<AccordionContextValue | null>(null);

const useAccordion = () => {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('useAccordion must be used within Accordion');
  }
  return context;
};

// Root component
interface AccordionProps {
  children: React.ReactNode;
  defaultExpandedId?: string;
  onExpandChange?: (id: string | null) => void;
}

const Accordion: React.FC<AccordionProps> & {
  Item: typeof AccordionItem;
  Header: typeof AccordionHeader;
  Panel: typeof AccordionPanel;
} = ({ children, defaultExpandedId = null, onExpandChange }) => {
  const [expandedId, setExpandedId] = useState<string | null>(defaultExpandedId);

  const toggle = (id: string) => {
    const newId = expandedId === id ? null : id;
    setExpandedId(newId);
    onExpandChange?.(newId);
  };

  return (
    <AccordionContext.Provider value={{ expandedId, toggle }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
};

// Item context
interface AccordionItemContextValue {
  id: string;
  isExpanded: boolean;
}

const AccordionItemContext = createContext<AccordionItemContextValue | null>(null);

// Item component
interface AccordionItemProps {
  id: string;
  children: React.ReactNode;
}

const AccordionItem: React.FC<AccordionItemProps> = ({ id, children }) => {
  const { expandedId } = useAccordion();
  const isExpanded = expandedId === id;

  return (
    <AccordionItemContext.Provider value={{ id, isExpanded }}>
      <div className={`accordion-item ${isExpanded ? 'expanded' : ''}`}>
        {children}
      </div>
    </AccordionItemContext.Provider>
  );
};

// Header component
interface AccordionHeaderProps {
  children: React.ReactNode;
  className?: string;
}

const AccordionHeader: React.FC<AccordionHeaderProps> = ({ children, className }) => {
  const accordion = useAccordion();
  const itemContext = useContext(AccordionItemContext);

  if (!itemContext) {
    throw new Error('AccordionHeader must be used within AccordionItem');
  }

  return (
    <button
      className={`accordion-header ${className || ''}`}
      onClick={() => accordion.toggle(itemContext.id)}
      aria-expanded={itemContext.isExpanded}
    >
      {children}
      <span className="accordion-icon">{itemContext.isExpanded ? '▲' : '▼'}</span>
    </button>
  );
};

// Panel component
interface AccordionPanelProps {
  children: React.ReactNode;
  className?: string;
}

const AccordionPanel: React.FC<AccordionPanelProps> = ({ children, className }) => {
  const itemContext = useContext(AccordionItemContext);

  if (!itemContext) {
    throw new Error('AccordionPanel must be used within AccordionItem');
  }

  if (!itemContext.isExpanded) return null;

  return (
    <div className={`accordion-panel ${className || ''}`}>
      {children}
    </div>
  );
};

// Assign sub-components
Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Panel = AccordionPanel;

// ===== การใช้งาน =====
const AccordionUsage: React.FC = () => (
  <Accordion defaultExpandedId="item-1">
    <Accordion.Item id="item-1">
      <Accordion.Header>ส่วนที่ 1</Accordion.Header>
      <Accordion.Panel>
        <p>เนื้อหาส่วนที่ 1</p>
      </Accordion.Panel>
    </Accordion.Item>

    <Accordion.Item id="item-2">
      <Accordion.Header>ส่วนที่ 2</Accordion.Header>
      <Accordion.Panel>
        <p>เนื้อหาส่วนที่ 2</p>
      </Accordion.Panel>
    </Accordion.Item>
  </Accordion>
);

export { Accordion, AccordionUsage };
```

---

## 26.11 Class Component Typing (Legacy)

```typescript
// src/components/legacy/ClassComponents.tsx
import React, { Component, PureComponent } from 'react';

// ===== Basic Class Component =====
interface CounterProps {
  initialCount?: number;
  step?: number;
  onCountChange?: (count: number) => void;
}

interface CounterState {
  count: number;
  history: number[];
}

class Counter extends Component<CounterProps, CounterState> {
  // static defaultProps
  static defaultProps: Partial<CounterProps> = {
    initialCount: 0,
    step: 1
  };

  constructor(props: CounterProps) {
    super(props);
    this.state = {
      count: props.initialCount || 0,
      history: [props.initialCount || 0]
    };
  }

  // Lifecycle methods
  componentDidMount(): void {
    document.title = `Count: ${this.state.count}`;
  }

  componentDidUpdate(prevProps: CounterProps, prevState: CounterState): void {
    if (prevState.count !== this.state.count) {
      document.title = `Count: ${this.state.count}`;
      this.props.onCountChange?.(this.state.count);
    }
  }

  componentWillUnmount(): void {
    document.title = 'App';
  }

  // Methods
  increment = (): void => {
    this.setState(prevState => ({
      count: prevState.count + (this.props.step || 1),
      history: [...prevState.history, prevState.count + (this.props.step || 1)]
    }));
  };

  decrement = (): void => {
    this.setState(prevState => ({
      count: prevState.count - (this.props.step || 1),
      history: [...prevState.history, prevState.count - (this.props.step || 1)]
    }));
  };

  reset = (): void => {
    const initial = this.props.initialCount || 0;
    this.setState({ count: initial, history: [initial] });
  };

  render(): React.ReactNode {
    const { count, history } = this.state;
    return (
      <div className="counter">
        <h2>Count: {count}</h2>
        <div className="controls">
          <button onClick={this.decrement}>-</button>
          <button onClick={this.reset}>Reset</button>
          <button onClick={this.increment}>+</button>
        </div>
        <div className="history">
          <h4>ประวัติ: {history.join(' → ')}</h4>
        </div>
      </div>
    );
  }
}

// ===== PureComponent =====
interface UserListItemProps {
  user: {
    id: string;
    name: string;
    email: string;
  };
  isSelected: boolean;
  onSelect: (id: string) => void;
}

class UserListItem extends PureComponent<UserListItemProps> {
  handleClick = (): void => {
    this.props.onSelect(this.props.user.id);
  };

  render(): React.ReactNode {
    const { user, isSelected } = this.props;
    return (
      <div
        className={`user-item ${isSelected ? 'selected' : ''}`}
        onClick={this.handleClick}
      >
        <strong>{user.name}</strong>
        <span>{user.email}</span>
      </div>
    );
  }
}

// ===== getDerivedStateFromProps =====
interface AnimatedValueProps {
  targetValue: number;
  duration?: number;
}

interface AnimatedValueState {
  currentValue: number;
  isAnimating: boolean;
}

class AnimatedValue extends Component<AnimatedValueProps, AnimatedValueState> {
  private animationFrame: number | null = null;

  state: AnimatedValueState = {
    currentValue: this.props.targetValue,
    isAnimating: false
  };

  static getDerivedStateFromProps(
    props: AnimatedValueProps,
    state: AnimatedValueState
  ): Partial<AnimatedValueState> | null {
    if (props.targetValue !== state.currentValue && !state.isAnimating) {
      return { isAnimating: true };
    }
    return null;
  }

  componentDidUpdate(prevProps: AnimatedValueProps): void {
    if (prevProps.targetValue !== this.props.targetValue) {
      this.animateTo(this.props.targetValue);
    }
  }

  private animateTo(target: number): void {
    const start = this.state.currentValue;
    const diff = target - start;
    const duration = this.props.duration || 300;
    const startTime = performance.now();

    const animate = (currentTime: number) => {
      const elapsed = currentTime - startTime;
      const progress = Math.min(elapsed / duration, 1);
      const current = start + diff * progress;

      this.setState({ currentValue: Math.round(current) });

      if (progress < 1) {
        this.animationFrame = requestAnimationFrame(animate);
      } else {
        this.setState({ isAnimating: false });
      }
    };

    this.animationFrame = requestAnimationFrame(animate);
  }

  componentWillUnmount(): void {
    if (this.animationFrame !== null) {
      cancelAnimationFrame(this.animationFrame);
    }
  }

  render(): React.ReactNode {
    return (
      <span className={`animated-value ${this.state.isAnimating ? 'animating' : ''}`}>
        {this.state.currentValue}
      </span>
    );
  }
}

export { Counter, UserListItem, AnimatedValue };
```

---

## 26.12 Event Handler Types

```typescript
// src/components/examples/EventHandlers.tsx
import React, { useState, useCallback } from 'react';

// ===== Mouse Events =====
const MouseEventExamples: React.FC = () => {
  const handleClick: React.MouseEventHandler<HTMLButtonElement> = (e) => {
    console.log('Clicked:', e.currentTarget.id);
    e.stopPropagation();
  };

  const handleContextMenu: React.MouseEventHandler<HTMLDivElement> = (e) => {
    e.preventDefault();
    console.log('Right clicked at:', e.clientX, e.clientY);
  };

  const handleMouseEnter: React.MouseEventHandler<HTMLDivElement> = (e) => {
    e.currentTarget.style.backgroundColor = '#f0f0f0';
  };

  const handleMouseLeave: React.MouseEventHandler<HTMLDivElement> = (e) => {
    e.currentTarget.style.backgroundColor = '';
  };

  return (
    <div
      onContextMenu={handleContextMenu}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
    >
      <button id="btn-1" onClick={handleClick}>คลิก</button>
    </div>
  );
};

// ===== Keyboard Events =====
const KeyboardEventExamples: React.FC = () => {
  const [value, setValue] = useState('');

  const handleKeyDown: React.KeyboardEventHandler<HTMLInputElement> = (e) => {
    if (e.key === 'Enter') {
      console.log('Submit:', e.currentTarget.value);
    }
    if (e.key === 'Escape') {
      setValue('');
    }
    if (e.ctrlKey && e.key === 'a') {
      e.preventDefault();
      e.currentTarget.select();
    }
  };

  const handleKeyPress: React.KeyboardEventHandler<HTMLInputElement> = (e) => {
    // Block special characters
    if (!/[a-zA-Z0-9 ]/.test(e.key)) {
      e.preventDefault();
    }
  };

  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
      onKeyDown={handleKeyDown}
      onKeyPress={handleKeyPress}
      placeholder="พิมพ์ข้อความ..."
    />
  );
};

// ===== Form Events =====
interface FormData {
  username: string;
  email: string;
  password: string;
  remember: boolean;
}

const FormEventExamples: React.FC = () => {
  const [form, setForm] = useState<FormData>({
    username: '',
    email: '',
    password: '',
    remember: false
  });

  const handleChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
    const { name, value, type, checked } = e.target;
    setForm(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value
    }));
  };

  const handleSelectChange: React.ChangeEventHandler<HTMLSelectElement> = (e) => {
    console.log('Selected:', e.target.value);
  };

  const handleTextAreaChange: React.ChangeEventHandler<HTMLTextAreaElement> = (e) => {
    console.log('Text:', e.target.value);
  };

  const handleSubmit: React.FormEventHandler<HTMLFormElement> = (e) => {
    e.preventDefault();
    console.log('Form data:', form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        name="username"
        value={form.username}
        onChange={handleChange}
        placeholder="ชื่อผู้ใช้"
      />
      <input
        type="email"
        name="email"
        value={form.email}
        onChange={handleChange}
        placeholder="อีเมล"
      />
      <input
        type="password"
        name="password"
        value={form.password}
        onChange={handleChange}
        placeholder="รหัสผ่าน"
      />
      <label>
        <input
          type="checkbox"
          name="remember"
          checked={form.remember}
          onChange={handleChange}
        />
        จดจำฉัน
      </label>
      <button type="submit">เข้าสู่ระบบ</button>
    </form>
  );
};

// ===== Drag Events =====
interface DraggableItem {
  id: string;
  label: string;
}

const DragEventExamples: React.FC = () => {
  const [items, setItems] = useState<DraggableItem[]>([
    { id: '1', label: 'รายการ 1' },
    { id: '2', label: 'รายการ 2' },
    { id: '3', label: 'รายการ 3' }
  ]);
  const [draggedId, setDraggedId] = useState<string | null>(null);

  const handleDragStart: React.DragEventHandler<HTMLDivElement> = (e) => {
    setDraggedId(e.currentTarget.id);
    e.dataTransfer.effectAllowed = 'move';
  };

  const handleDragOver: React.DragEventHandler<HTMLDivElement> = (e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
  };

  const handleDrop: React.DragEventHandler<HTMLDivElement> = (e) => {
    e.preventDefault();
    console.log('Dropped on:', e.currentTarget.id);
    setDraggedId(null);
  };

  return (
    <div className="drag-container">
      {items.map(item => (
        <div
          key={item.id}
          id={item.id}
          draggable
          onDragStart={handleDragStart}
          onDragOver={handleDragOver}
          onDrop={handleDrop}
          className={`drag-item ${draggedId === item.id ? 'dragging' : ''}`}
        >
          {item.label}
        </div>
      ))}
    </div>
  );
};

// ===== Custom event handlers with generics =====
type InputChangeHandler = React.ChangeEventHandler<
  HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement
>;

const createChangeHandler = <T extends Record<string, string>>(
  setState: React.Dispatch<React.SetStateAction<T>>
): InputChangeHandler => {
  return (e) => {
    const { name, value } = e.target;
    setState(prev => ({ ...prev, [name]: value }));
  };
};

export {
  MouseEventExamples,
  KeyboardEventExamples,
  FormEventExamples,
  DragEventExamples,
  createChangeHandler
};
```

---

## 26.13 ตัวอย่าง Component Library

```typescript
// src/components/library/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
export { Table } from './Table';
export { Pagination } from './Pagination';

// src/components/library/Table.tsx
import React, { useState } from 'react';

// Generic Table Component
interface Column<T> {
  key: keyof T | string;
  header: string;
  width?: string;
  sortable?: boolean;
  render?: (value: T[keyof T], row: T, index: number) => React.ReactNode;
  align?: 'left' | 'center' | 'right';
}

interface TableProps<T extends { id: string | number }> {
  data: T[];
  columns: Column<T>[];
  loading?: boolean;
  emptyMessage?: string;
  rowKey?: keyof T;
  onRowClick?: (row: T) => void;
  selectedRows?: Array<T['id']>;
  onSelectRows?: (ids: Array<T['id']>) => void;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  onSort?: (key: string, order: 'asc' | 'desc') => void;
  pagination?: {
    currentPage: number;
    totalPages: number;
    onPageChange: (page: number) => void;
  };
}

function Table<T extends { id: string | number }>({
  data,
  columns,
  loading = false,
  emptyMessage = 'ไม่มีข้อมูล',
  rowKey = 'id',
  onRowClick,
  selectedRows = [],
  onSelectRows,
  sortBy,
  sortOrder,
  onSort,
  pagination
}: TableProps<T>) {
  const handleSort = (key: string) => {
    if (onSort) {
      const newOrder = sortBy === key && sortOrder === 'asc' ? 'desc' : 'asc';
      onSort(key, newOrder);
    }
  };

  const handleSelectAll = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (onSelectRows) {
      onSelectRows(e.target.checked ? data.map(row => row.id) : []);
    }
  };

  const handleSelectRow = (id: T['id']) => {
    if (onSelectRows) {
      const newSelected = selectedRows.includes(id)
        ? selectedRows.filter(s => s !== id)
        : [...selectedRows, id];
      onSelectRows(newSelected);
    }
  };

  if (loading) {
    return (
      <div className="table-loading">
        <div className="spinner" />
        <p>กำลังโหลดข้อมูล...</p>
      </div>
    );
  }

  return (
    <div className="table-container">
      <table className="table">
        <thead>
          <tr>
            {onSelectRows && (
              <th className="table-checkbox-col">
                <input
                  type="checkbox"
                  checked={selectedRows.length === data.length && data.length > 0}
                  onChange={handleSelectAll}
                />
              </th>
            )}
            {columns.map(col => (
              <th
                key={String(col.key)}
                style={{ width: col.width, textAlign: col.align || 'left' }}
                className={col.sortable ? 'sortable' : ''}
                onClick={() => col.sortable && handleSort(String(col.key))}
              >
                {col.header}
                {col.sortable && sortBy === String(col.key) && (
                  <span className={`sort-icon ${sortOrder}`}>
                    {sortOrder === 'asc' ? '↑' : '↓'}
                  </span>
                )}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.length === 0 ? (
            <tr>
              <td
                colSpan={columns.length + (onSelectRows ? 1 : 0)}
                className="table-empty"
              >
                {emptyMessage}
              </td>
            </tr>
          ) : (
            data.map((row, index) => (
              <tr
                key={String(row[rowKey])}
                className={[
                  onRowClick ? 'clickable' : '',
                  selectedRows.includes(row.id) ? 'selected' : ''
                ].join(' ')}
                onClick={() => onRowClick?.(row)}
              >
                {onSelectRows && (
                  <td className="table-checkbox-col" onClick={(e) => e.stopPropagation()}>
                    <input
                      type="checkbox"
                      checked={selectedRows.includes(row.id)}
                      onChange={() => handleSelectRow(row.id)}
                    />
                  </td>
                )}
                {columns.map(col => {
                  const value = row[col.key as keyof T];
                  return (
                    <td
                      key={String(col.key)}
                      style={{ textAlign: col.align || 'left' }}
                    >
                      {col.render
                        ? col.render(value, row, index)
                        : String(value ?? '')
                      }
                    </td>
                  );
                })}
              </tr>
            ))
          )}
        </tbody>
      </table>
      
      {pagination && (
        <div className="table-pagination">
          <button
            disabled={pagination.currentPage <= 1}
            onClick={() => pagination.onPageChange(pagination.currentPage - 1)}
          >
            ก่อนหน้า
          </button>
          <span>หน้า {pagination.currentPage} / {pagination.totalPages}</span>
          <button
            disabled={pagination.currentPage >= pagination.totalPages}
            onClick={() => pagination.onPageChange(pagination.currentPage + 1)}
          >
            ถัดไป
          </button>
        </div>
      )}
    </div>
  );
}

// ===== การใช้งาน Table =====
interface Employee {
  id: number;
  name: string;
  department: string;
  salary: number;
  active: boolean;
}

const EmployeeTable: React.FC = () => {
  const employees: Employee[] = [
    { id: 1, name: 'สมชาย ใจดี', department: 'IT', salary: 35000, active: true },
    { id: 2, name: 'สมหญิง รักดี', department: 'HR', salary: 28000, active: true },
    { id: 3, name: 'สมศักดิ์ มีใจ', department: 'Finance', salary: 42000, active: false }
  ];

  const columns: Column<Employee>[] = [
    { key: 'name', header: 'ชื่อ-สกุล', sortable: true },
    { key: 'department', header: 'แผนก', sortable: true },
    {
      key: 'salary',
      header: 'เงินเดือน',
      align: 'right',
      sortable: true,
      render: (value) => `฿${Number(value).toLocaleString()}`
    },
    {
      key: 'active',
      header: 'สถานะ',
      align: 'center',
      render: (value) => (
        <span className={`badge ${value ? 'badge-success' : 'badge-danger'}`}>
          {value ? 'ใช้งาน' : 'ไม่ใช้งาน'}
        </span>
      )
    }
  ];

  return (
    <Table
      data={employees}
      columns={columns}
      onRowClick={(emp) => console.log('Clicked:', emp.name)}
    />
  );
};

export { Table, EmployeeTable };
export type { Column, TableProps };
```

---

## สรุปบทที่ 26

ในบทนี้เราได้เรียนรู้:
- วิธีการ type Functional Components แบบต่างๆ
- การออกแบบ Props Interfaces ที่ดี
- ReactNode, ReactElement และ children prop types
- การใช้ optional และ required props
- Default props ด้วย default parameter values
- Prop drilling กับ TypeScript types
- forwardRef typing และ useImperativeHandle
- Higher Order Components (HOC) typing
- Render Props pattern กับ TypeScript
- Compound Components pattern
- Class component typing
- Event handler types ครบถ้วน
- ตัวอย่าง component library พร้อม Generic Table

**ขั้นตอนต่อไป**: ในบทที่ 27 เราจะเรียนรู้เกี่ยวกับ React Hooks กับ TypeScript อย่างละเอียด
