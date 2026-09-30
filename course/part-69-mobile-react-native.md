# ตอนที่ 69: React Native กับ TypeScript

## บทนำ

React Native ช่วยให้เราพัฒนาแอปมือถือสำหรับทั้ง iOS และ Android โดยใช้ JavaScript/TypeScript การใช้ TypeScript กับ React Native ช่วยให้โค้ดมีความปลอดภัยและบำรุงรักษาง่ายขึ้นมาก

## 69.1 React Native TypeScript Setup

### การสร้างโปรเจกต์

```bash
# สร้างโปรเจกต์ใหม่
npx react-native@latest init MyApp --template react-native-template-typescript

# หรือใช้ Expo
npx create-expo-app MyApp --template

cd MyApp
npm install
```

### tsconfig.json

```json
{
  "extends": "@react-native/typescript-config/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"],
      "@screens/*": ["src/screens/*"],
      "@hooks/*": ["src/hooks/*"],
      "@services/*": ["src/services/*"],
      "@types/*": ["src/types/*"],
      "@utils/*": ["src/utils/*"],
      "@store/*": ["src/store/*"]
    }
  }
}
```

### โครงสร้างโปรเจกต์

```
src/
  components/     # Reusable components
  screens/        # Screen components
  navigation/     # Navigation config
  hooks/          # Custom hooks
  services/       # API services
  store/          # State management
  types/          # TypeScript types
  utils/          # Utility functions
  constants/      # Constants
```

## 69.2 Navigation พร้อม Types (React Navigation)

```bash
npm install @react-navigation/native @react-navigation/stack @react-navigation/bottom-tabs
npm install react-native-screens react-native-safe-area-context
npm install react-native-gesture-handler
```

### Route Params Typing

```typescript
// navigation/types.ts
import { NativeStackScreenProps } from '@react-navigation/native-stack'
import { BottomTabScreenProps } from '@react-navigation/bottom-tabs'
import { CompositeScreenProps } from '@react-navigation/native'

// Root Stack Params
export type RootStackParamList = {
  Auth: undefined
  Main: undefined
  ProductDetail: { productId: number; fromCart?: boolean }
  ImageViewer: { images: string[]; initialIndex?: number }
  Modal: { title: string; message: string }
}

// Auth Stack Params
export type AuthStackParamList = {
  Login: { email?: string } | undefined
  Register: undefined
  ForgotPassword: { email?: string } | undefined
  OTPVerification: { phone: string; mode: 'login' | 'register' }
}

// Main Tab Params
export type MainTabParamList = {
  Home: undefined
  Products: { category?: string }
  Cart: undefined
  Profile: undefined
}

// Type helpers for screens
export type RootStackScreenProps<T extends keyof RootStackParamList> =
  NativeStackScreenProps<RootStackParamList, T>

export type AuthStackScreenProps<T extends keyof AuthStackParamList> =
  NativeStackScreenProps<AuthStackParamList, T>

export type MainTabScreenProps<T extends keyof MainTabParamList> =
  CompositeScreenProps<
    BottomTabScreenProps<MainTabParamList, T>,
    NativeStackScreenProps<RootStackParamList>
  >

// Augment useNavigation hook for better typing
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

### Navigation Setup

```typescript
// navigation/RootNavigator.tsx
import React from 'react'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs'
import { NavigationContainer } from '@react-navigation/native'
import { RootStackParamList, MainTabParamList } from './types'

const RootStack = createNativeStackNavigator<RootStackParamList>()
const MainTab = createBottomTabNavigator<MainTabParamList>()

function MainTabNavigator(): React.JSX.Element {
  return (
    <MainTab.Navigator>
      <MainTab.Screen name="Home" component={HomeScreen} />
      <MainTab.Screen name="Products" component={ProductsScreen} />
      <MainTab.Screen name="Cart" component={CartScreen} />
      <MainTab.Screen name="Profile" component={ProfileScreen} />
    </MainTab.Navigator>
  )
}

export default function RootNavigator(): React.JSX.Element {
  const { isAuthenticated } = useAuth()
  
  return (
    <NavigationContainer>
      <RootStack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <>
            <RootStack.Screen name="Main" component={MainTabNavigator} />
            <RootStack.Screen 
              name="ProductDetail" 
              component={ProductDetailScreen}
              options={{ headerShown: true, title: 'รายละเอียดสินค้า' }}
            />
          </>
        ) : (
          <RootStack.Screen name="Auth" component={AuthNavigator} />
        )}
      </RootStack.Navigator>
    </NavigationContainer>
  )
}
```

### Screen Components

```typescript
// screens/ProductDetailScreen.tsx
import React, { useEffect, useState } from 'react'
import { View, Text, Image, ScrollView, TouchableOpacity, StyleSheet } from 'react-native'
import { RootStackScreenProps } from '@/navigation/types'

interface Product {
  id: number
  name: string
  description: string
  price: number
  images: string[]
  rating: number
  reviewCount: number
  inStock: boolean
}

type Props = RootStackScreenProps<'ProductDetail'>

export default function ProductDetailScreen({ route, navigation }: Props): React.JSX.Element {
  const { productId, fromCart } = route.params
  
  const [product, setProduct] = useState<Product | null>(null)
  const [loading, setLoading] = useState<boolean>(true)
  const [selectedImage, setSelectedImage] = useState<number>(0)
  const [quantity, setQuantity] = useState<number>(1)
  
  useEffect(() => {
    loadProduct()
  }, [productId])
  
  const loadProduct = async (): Promise<void> => {
    setLoading(true)
    try {
      const data = await productService.getById(productId)
      setProduct(data)
      navigation.setOptions({ title: data.name })
    } catch (error) {
      console.error('ไม่สามารถโหลดสินค้าได้:', error)
    } finally {
      setLoading(false)
    }
  }
  
  const handleAddToCart = (): void => {
    if (!product) return
    cartService.addItem({ productId: product.id, quantity })
    navigation.goBack()
  }
  
  const handleViewImages = (): void => {
    if (!product) return
    navigation.navigate('ImageViewer', {
      images: product.images,
      initialIndex: selectedImage
    })
  }
  
  if (loading) return <LoadingScreen />
  if (!product) return <ErrorScreen message="ไม่พบสินค้า" />
  
  return (
    <ScrollView style={styles.container}>
      <TouchableOpacity onPress={handleViewImages}>
        <Image 
          source={{ uri: product.images[selectedImage] }}
          style={styles.mainImage}
          resizeMode="cover"
        />
      </TouchableOpacity>
      
      <View style={styles.content}>
        <Text style={styles.name}>{product.name}</Text>
        <Text style={styles.price}>฿{product.price.toLocaleString()}</Text>
        
        <View style={styles.ratingRow}>
          <StarRating rating={product.rating} />
          <Text style={styles.reviewCount}>({product.reviewCount} รีวิว)</Text>
        </View>
        
        <Text style={styles.description}>{product.description}</Text>
        
        <QuantitySelector 
          value={quantity}
          min={1}
          max={10}
          onChange={setQuantity}
        />
        
        <TouchableOpacity
          style={[styles.addButton, !product.inStock && styles.disabledButton]}
          onPress={handleAddToCart}
          disabled={!product.inStock}
        >
          <Text style={styles.addButtonText}>
            {product.inStock ? 'เพิ่มในตะกร้า' : 'สินค้าหมด'}
          </Text>
        </TouchableOpacity>
      </View>
    </ScrollView>
  )
}
```

## 69.3 Component Props Typing

```typescript
// components/Button.tsx
import React from 'react'
import { 
  TouchableOpacity, Text, ActivityIndicator, 
  StyleSheet, ViewStyle, TextStyle
} from 'react-native'

type ButtonVariant = 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger'
type ButtonSize = 'sm' | 'md' | 'lg'

interface ButtonProps {
  label: string
  onPress: () => void
  variant?: ButtonVariant
  size?: ButtonSize
  disabled?: boolean
  loading?: boolean
  fullWidth?: boolean
  icon?: React.ReactNode
  style?: ViewStyle
  textStyle?: TextStyle
  testID?: string
}

export default function Button({
  label,
  onPress,
  variant = 'primary',
  size = 'md',
  disabled = false,
  loading = false,
  fullWidth = false,
  icon,
  style,
  textStyle,
  testID
}: ButtonProps): React.JSX.Element {
  const isDisabled = disabled || loading
  
  return (
    <TouchableOpacity
      testID={testID}
      style={[
        styles.base,
        styles[variant],
        styles[`size_${size}`],
        fullWidth && styles.fullWidth,
        isDisabled && styles.disabled,
        style
      ]}
      onPress={onPress}
      disabled={isDisabled}
      activeOpacity={0.8}
    >
      {loading ? (
        <ActivityIndicator color={variant === 'primary' ? '#fff' : '#007AFF'} />
      ) : (
        <>
          {icon && <View style={styles.iconWrapper}>{icon}</View>}
          <Text style={[styles.text, styles[`text_${variant}`], styles[`textSize_${size}`], textStyle]}>
            {label}
          </Text>
        </>
      )}
    </TouchableOpacity>
  )
}
```

## 69.4 StyleSheet Types

```typescript
import { StyleSheet, Platform, Dimensions, ViewStyle, TextStyle, ImageStyle } from 'react-native'

// Dimension helpers
const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window')

// Color palette
const Colors = {
  primary: '#007AFF',
  secondary: '#5856D6',
  success: '#34C759',
  warning: '#FF9500',
  danger: '#FF3B30',
  text: {
    primary: '#1C1C1E',
    secondary: '#6C6C70',
    disabled: '#AEAEB2',
    inverse: '#FFFFFF'
  },
  background: {
    primary: '#F2F2F7',
    secondary: '#FFFFFF',
    card: '#FFFFFF'
  },
  border: '#C6C6C8'
} as const

// Typography
const Typography = {
  h1: {
    fontSize: 34,
    fontWeight: '700' as const,
    lineHeight: 41,
    color: Colors.text.primary
  },
  h2: {
    fontSize: 28,
    fontWeight: '600' as const,
    lineHeight: 34,
    color: Colors.text.primary
  },
  body: {
    fontSize: 17,
    fontWeight: '400' as const,
    lineHeight: 22,
    color: Colors.text.primary
  },
  caption: {
    fontSize: 12,
    fontWeight: '400' as const,
    lineHeight: 16,
    color: Colors.text.secondary
  }
} as const

// Spacing
const Spacing = {
  xs: 4,
  sm: 8,
  md: 16,
  lg: 24,
  xl: 32,
  xxl: 48
} as const

// Shadows
const Shadows = {
  sm: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.1,
      shadowRadius: 2
    },
    android: { elevation: 2 }
  }),
  md: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.15,
      shadowRadius: 8
    },
    android: { elevation: 4 }
  })
} as const

// Type-safe stylesheet
type NamedStyles<T> = { [P in keyof T]: ViewStyle | TextStyle | ImageStyle }

function createStyles<T extends NamedStyles<T>>(styles: T): T {
  return StyleSheet.create(styles)
}

const styles = createStyles({
  container: {
    flex: 1,
    backgroundColor: Colors.background.primary
  },
  card: {
    backgroundColor: Colors.background.card,
    borderRadius: 12,
    padding: Spacing.md,
    marginHorizontal: Spacing.md,
    marginVertical: Spacing.sm,
    ...Shadows.md
  },
  title: {
    ...Typography.h2,
    marginBottom: Spacing.sm
  }
})
```

## 69.5 Platform-Specific Code

```typescript
import { Platform, PlatformIOSStatic, PlatformAndroidStatic } from 'react-native'

// Platform constants
const isIOS: boolean = Platform.OS === 'ios'
const isAndroid: boolean = Platform.OS === 'android'
const iosVersion = isIOS ? (Platform as PlatformIOSStatic).Version : null
const androidVersion = isAndroid ? (Platform as PlatformAndroidStatic).Version : null

// Platform-specific values
function platformValue<T>(options: { ios: T; android: T; default?: T }): T {
  if (isIOS) return options.ios
  if (isAndroid) return options.android
  return options.default ?? options.ios
}

const hitSlop = platformValue({
  ios: { top: 8, bottom: 8, left: 8, right: 8 },
  android: { top: 10, bottom: 10, left: 10, right: 10 }
})

// Platform-specific components
type PlatformSpecificProps = {
  ios: React.ComponentType<any>
  android: React.ComponentType<any>
}

function PlatformSpecific({ ios: IOSComponent, android: AndroidComponent }: PlatformSpecificProps) {
  if (isIOS) return <IOSComponent />
  return <AndroidComponent />
}

// Status bar handling
import { StatusBar, StatusBarStyle } from 'react-native'

function useStatusBar(style: StatusBarStyle = 'dark-content'): void {
  useEffect(() => {
    StatusBar.setBarStyle(style, true)
    
    if (isAndroid) {
      StatusBar.setBackgroundColor('transparent', true)
      StatusBar.setTranslucent(true)
    }
    
    return () => {
      StatusBar.setBarStyle('default', true)
    }
  }, [style])
}
```

## 69.6 Native Modules พร้อม TypeScript

```typescript
// NativeModules typing
import { NativeModules, NativeEventEmitter } from 'react-native'

// ประกาศ interface ของ native module
interface BiometricAuthModule {
  authenticate(reason: string): Promise<{ success: boolean; error?: string }>
  isAvailable(): Promise<boolean>
  getBiometryType(): Promise<'FaceID' | 'TouchID' | 'Fingerprint' | 'None'>
}

interface CameraModule {
  takePicture(options: CameraOptions): Promise<CameraResult>
  startRecording(options: VideoOptions): void
  stopRecording(): Promise<VideoResult>
}

interface CameraOptions {
  quality: 'low' | 'medium' | 'high'
  flashMode: 'off' | 'on' | 'auto'
  facing: 'front' | 'back'
}

interface CameraResult {
  uri: string
  width: number
  height: number
  fileSize: number
  type: 'image/jpeg' | 'image/png'
}

interface VideoOptions {
  quality: 'low' | 'medium' | 'high'
  maxDuration?: number
  maxFileSize?: number
}

interface VideoResult {
  uri: string
  duration: number
  fileSize: number
}

// สร้าง typed wrappers
const BiometricAuth: BiometricAuthModule = NativeModules.BiometricAuth

const useBiometricAuth = () => {
  const [isAvailable, setIsAvailable] = useState<boolean>(false)
  const [biometryType, setBiometryType] = useState<string>('None')
  
  useEffect(() => {
    const checkAvailability = async () => {
      const available = await BiometricAuth.isAvailable()
      setIsAvailable(available)
      
      if (available) {
        const type = await BiometricAuth.getBiometryType()
        setBiometryType(type)
      }
    }
    
    checkAvailability()
  }, [])
  
  const authenticate = async (reason: string): Promise<boolean> => {
    if (!isAvailable) return false
    
    try {
      const result = await BiometricAuth.authenticate(reason)
      return result.success
    } catch (error) {
      console.error('Biometric auth failed:', error)
      return false
    }
  }
  
  return { isAvailable, biometryType, authenticate }
}
```

## 69.7 State Management กับ Redux Toolkit

```typescript
// store/slices/productsSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit'

interface Product {
  id: number
  name: string
  price: number
  category: string
  inStock: boolean
}

interface ProductsState {
  items: Product[]
  selectedProduct: Product | null
  loading: boolean
  error: string | null
  searchQuery: string
  category: string
}

const initialState: ProductsState = {
  items: [],
  selectedProduct: null,
  loading: false,
  error: null,
  searchQuery: '',
  category: ''
}

// Async thunk
export const fetchProducts = createAsyncThunk(
  'products/fetchAll',
  async (params: { category?: string; query?: string }, { rejectWithValue }) => {
    try {
      const response = await productApi.getProducts(params)
      return response.data
    } catch (error: any) {
      return rejectWithValue(error.message || 'ไม่สามารถโหลดสินค้าได้')
    }
  }
)

export const fetchProductById = createAsyncThunk(
  'products/fetchById',
  async (id: number, { rejectWithValue }) => {
    try {
      return await productApi.getById(id)
    } catch (error: any) {
      return rejectWithValue(error.message)
    }
  }
)

const productsSlice = createSlice({
  name: 'products',
  initialState,
  reducers: {
    setSearchQuery: (state, action: PayloadAction<string>) => {
      state.searchQuery = action.payload
    },
    setCategory: (state, action: PayloadAction<string>) => {
      state.category = action.payload
    },
    clearSelected: (state) => {
      state.selectedProduct = null
    },
    clearError: (state) => {
      state.error = null
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true
        state.error = null
      })
      .addCase(fetchProducts.fulfilled, (state, action: PayloadAction<Product[]>) => {
        state.loading = false
        state.items = action.payload
      })
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false
        state.error = action.payload as string
      })
      .addCase(fetchProductById.fulfilled, (state, action: PayloadAction<Product>) => {
        state.selectedProduct = action.payload
      })
  }
})

export const { setSearchQuery, setCategory, clearSelected, clearError } = productsSlice.actions
export default productsSlice.reducer

// store/index.ts
import { configureStore } from '@reduxjs/toolkit'
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux'
import productsReducer from './slices/productsSlice'
import cartReducer from './slices/cartSlice'
import authReducer from './slices/authSlice'

export const store = configureStore({
  reducer: {
    products: productsReducer,
    cart: cartReducer,
    auth: authReducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['auth/setUser'],
        ignoredPaths: ['auth.user.createdAt']
      }
    })
})

export type RootState = ReturnType<typeof store.getState>
export type AppDispatch = typeof store.dispatch

// Typed hooks
export const useAppDispatch = (): AppDispatch => useDispatch<AppDispatch>()
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector
```

## 69.8 API Calls พร้อม TypeScript

```typescript
// services/apiClient.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios'
import AsyncStorage from '@react-native-async-storage/async-storage'

interface ApiClientConfig {
  baseURL: string
  timeout?: number
}

class ApiClient {
  private instance: AxiosInstance
  
  constructor(config: ApiClientConfig) {
    this.instance = axios.create({
      baseURL: config.baseURL,
      timeout: config.timeout ?? 10000,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      }
    })
    
    this.setupInterceptors()
  }
  
  private setupInterceptors(): void {
    // Request interceptor
    this.instance.interceptors.request.use(async (config) => {
      const token = await AsyncStorage.getItem('@auth_token')
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
      }
      return config
    })
    
    // Response interceptor
    this.instance.interceptors.response.use(
      (response: AxiosResponse) => response,
      async (error) => {
        if (error.response?.status === 401) {
          await AsyncStorage.removeItem('@auth_token')
          // Navigate to login
        }
        return Promise.reject(error)
      }
    )
  }
  
  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.instance.get<T>(url, config)
    return response.data
  }
  
  async post<T, D = unknown>(url: string, data?: D, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.instance.post<T>(url, data, config)
    return response.data
  }
  
  async put<T, D = unknown>(url: string, data?: D): Promise<T> {
    const response = await this.instance.put<T>(url, data)
    return response.data
  }
  
  async delete<T>(url: string): Promise<T> {
    const response = await this.instance.delete<T>(url)
    return response.data
  }
}

export const apiClient = new ApiClient({
  baseURL: process.env.API_BASE_URL || 'https://api.example.com'
})

// services/productService.ts
interface ProductListParams {
  page?: number
  perPage?: number
  category?: string
  search?: string
  sortBy?: 'name' | 'price' | 'rating'
  sortOrder?: 'asc' | 'desc'
}

interface PaginatedResponse<T> {
  data: T[]
  page: number
  perPage: number
  total: number
  totalPages: number
}

export const productService = {
  getProducts: (params?: ProductListParams): Promise<PaginatedResponse<Product>> => {
    return apiClient.get('/products', { params })
  },
  
  getById: (id: number): Promise<Product> => {
    return apiClient.get(`/products/${id}`)
  },
  
  create: (data: CreateProductDto): Promise<Product> => {
    return apiClient.post('/products', data)
  },
  
  update: (id: number, data: UpdateProductDto): Promise<Product> => {
    return apiClient.put(`/products/${id}`, data)
  },
  
  delete: (id: number): Promise<void> => {
    return apiClient.delete(`/products/${id}`)
  }
}
```

## 69.9 AsyncStorage พร้อม Types

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage'

// Type-safe AsyncStorage wrapper
class TypedAsyncStorage {
  private prefix: string
  
  constructor(prefix: string = '@app') {
    this.prefix = prefix
  }
  
  private getKey(key: string): string {
    return `${this.prefix}_${key}`
  }
  
  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await AsyncStorage.getItem(this.getKey(key))
      return value ? JSON.parse(value) as T : null
    } catch (error) {
      console.error(`Error getting ${key}:`, error)
      return null
    }
  }
  
  async set<T>(key: string, value: T): Promise<void> {
    try {
      await AsyncStorage.setItem(this.getKey(key), JSON.stringify(value))
    } catch (error) {
      console.error(`Error setting ${key}:`, error)
    }
  }
  
  async remove(key: string): Promise<void> {
    try {
      await AsyncStorage.removeItem(this.getKey(key))
    } catch (error) {
      console.error(`Error removing ${key}:`, error)
    }
  }
  
  async multiGet<T extends Record<string, any>>(keys: (keyof T)[]): Promise<Partial<T>> {
    const storageKeys = (keys as string[]).map(k => this.getKey(k))
    const pairs = await AsyncStorage.multiGet(storageKeys)
    
    const result: Partial<T> = {}
    pairs.forEach(([key, value]) => {
      const originalKey = key.replace(`${this.prefix}_`, '') as keyof T
      if (value) {
        result[originalKey] = JSON.parse(value)
      }
    })
    
    return result
  }
  
  async clear(): Promise<void> {
    const keys = await AsyncStorage.getAllKeys()
    const appKeys = keys.filter(k => k.startsWith(this.prefix))
    await AsyncStorage.multiRemove(appKeys)
  }
}

const storage = new TypedAsyncStorage()

// Custom hooks สำหรับ AsyncStorage
function useAsyncStorageState<T>(
  key: string,
  defaultValue: T
): [T, (value: T) => Promise<void>, boolean] {
  const [state, setState] = useState<T>(defaultValue)
  const [loading, setLoading] = useState<boolean>(true)
  
  useEffect(() => {
    const loadValue = async () => {
      const stored = await storage.get<T>(key)
      if (stored !== null) setState(stored)
      setLoading(false)
    }
    
    loadValue()
  }, [key])
  
  const setValue = useCallback(async (value: T): Promise<void> => {
    setState(value)
    await storage.set(key, value)
  }, [key])
  
  return [state, setValue, loading]
}

// การใช้งาน
const [userSettings, setUserSettings, loadingSettings] = useAsyncStorageState(
  'userSettings',
  { theme: 'light', language: 'th', notifications: true }
)
```

## 69.10 Push Notifications Types

```typescript
import messaging, { FirebaseMessagingTypes } from '@react-native-firebase/messaging'
import PushNotification, { 
  PushNotification as PushNotificationT,
  ReceivedNotification
} from 'react-native-push-notification'

// Notification payload types
interface NotificationPayload {
  type: 'ORDER_UPDATE' | 'PROMOTION' | 'MESSAGE' | 'ALERT'
  title: string
  body: string
  data?: Record<string, string>
}

interface OrderUpdateNotification extends NotificationPayload {
  type: 'ORDER_UPDATE'
  data: {
    orderId: string
    status: 'confirmed' | 'shipped' | 'delivered' | 'cancelled'
  }
}

interface PromotionNotification extends NotificationPayload {
  type: 'PROMOTION'
  data: {
    promotionId: string
    discount: string
    validUntil: string
  }
}

type AppNotification = OrderUpdateNotification | PromotionNotification

// Notification service
class NotificationService {
  async requestPermission(): Promise<boolean> {
    const authStatus = await messaging().requestPermission()
    return (
      authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
      authStatus === messaging.AuthorizationStatus.PROVISIONAL
    )
  }
  
  async getFCMToken(): Promise<string | null> {
    try {
      return await messaging().getToken()
    } catch (error) {
      console.error('Cannot get FCM token:', error)
      return null
    }
  }
  
  setupForegroundHandler(): () => void {
    return messaging().onMessage(async (remoteMessage: FirebaseMessagingTypes.RemoteMessage) => {
      const notification = this.parseNotification(remoteMessage)
      if (notification) {
        this.showLocalNotification(notification)
      }
    })
  }
  
  private parseNotification(
    remoteMessage: FirebaseMessagingTypes.RemoteMessage
  ): AppNotification | null {
    if (!remoteMessage.notification || !remoteMessage.data) return null
    
    const type = remoteMessage.data.type as AppNotification['type']
    const payload: AppNotification = {
      type,
      title: remoteMessage.notification.title ?? '',
      body: remoteMessage.notification.body ?? '',
      data: remoteMessage.data as any
    }
    
    return payload
  }
  
  private showLocalNotification(notification: AppNotification): void {
    PushNotification.localNotification({
      channelId: 'default',
      title: notification.title,
      message: notification.body,
      userInfo: notification.data
    })
  }
  
  handleNotificationPress(notification: ReceivedNotification): void {
    const data = notification.data as AppNotification['data']
    if (!data) return
    
    // Navigate based on notification type
    switch ((notification.data as any).type) {
      case 'ORDER_UPDATE':
        navigationRef.navigate('OrderDetail', { 
          orderId: (data as OrderUpdateNotification['data']).orderId 
        })
        break
      case 'PROMOTION':
        navigationRef.navigate('Promotions')
        break
    }
  }
}
```

## 69.11 โครงสร้างแอปสมบูรณ์

```typescript
// App.tsx
import React, { useEffect } from 'react'
import { StatusBar } from 'react-native'
import { Provider } from 'react-redux'
import { PersistGate } from 'redux-persist/integration/react'
import { store, persistor } from '@/store'
import RootNavigator from '@/navigation/RootNavigator'
import { NotificationService } from '@/services/NotificationService'

const notificationService = new NotificationService()

export default function App(): React.JSX.Element {
  useEffect(() => {
    // ตั้งค่า notifications
    const setupNotifications = async () => {
      const permitted = await notificationService.requestPermission()
      if (permitted) {
        const token = await notificationService.getFCMToken()
        if (token) {
          await userService.updatePushToken(token)
        }
        
        // Setup handlers
        const unsubscribeForeground = notificationService.setupForegroundHandler()
        return () => unsubscribeForeground()
      }
    }
    
    setupNotifications()
  }, [])
  
  return (
    <Provider store={store}>
      <PersistGate loading={null} persistor={persistor}>
        <StatusBar barStyle="dark-content" />
        <RootNavigator />
      </PersistGate>
    </Provider>
  )
}
```

## สรุป

React Native กับ TypeScript ให้ประโยชน์มากมาย:

1. **Navigation typing** ด้วย React Navigation's typed params
2. **Component props** ที่ type-safe ป้องกัน runtime errors
3. **Redux Toolkit** กับ typed dispatch และ selectors
4. **Native modules** ที่มี typed interfaces
5. **AsyncStorage** ที่ type-safe
6. **Push notifications** ที่มี typed payloads
7. **Platform-specific code** ที่จัดการได้ถูกต้อง

TypeScript ช่วยให้การพัฒนา mobile app เป็นเรื่องที่น่าเพลิดเพลินและลด bugs ได้อย่างมีนัยสำคัญ
