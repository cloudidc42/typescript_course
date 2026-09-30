# ตอนที่ 29: React Query (TanStack Query) กับ TypeScript

## บทนำ

TanStack Query (เดิมชื่อ React Query) เป็น library สำหรับจัดการ server state ใน React applications โดยช่วยจัดการ data fetching, caching, synchronization และ background updates ได้อย่างมีประสิทธิภาพ เมื่อใช้ร่วมกับ TypeScript เราจะได้ type safety ครบถ้วนตั้งแต่ query จนถึง UI

---

## 1. การติดตั้งและตั้งค่า

### 1.1 การติดตั้ง

```bash
npm install @tanstack/react-query
npm install -D @tanstack/react-query-devtools
# สำหรับ TypeScript
npm install -D @types/react
```

### 1.2 Setup QueryClient

```typescript
// main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import App from './App';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 นาที
      gcTime: 10 * 60 * 1000,         // 10 นาที (garbage collection)
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  </React.StrictMode>
);
```

---

## 2. useQuery กับ TypeScript Generics

### 2.1 useQuery พื้นฐาน

```typescript
import { useQuery } from '@tanstack/react-query';

interface User {
  id: number;
  name: string;
  email: string;
  phone: string;
  address: {
    street: string;
    city: string;
    zipcode: string;
  };
}

// Function สำหรับ fetch data
const fetchUser = async (userId: number): Promise<User> => {
  const response = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`);
  if (!response.ok) {
    throw new Error(`HTTP Error: ${response.status}`);
  }
  return response.json();
};

// Component ที่ใช้ useQuery
const UserProfile: React.FC<{ userId: number }> = ({ userId }) => {
  const {
    data: user,           // User | undefined
    isLoading,            // boolean
    isError,              // boolean
    error,                // Error | null
    isSuccess,            // boolean
    isFetching,           // boolean
    refetch,              // function
    status,               // 'pending' | 'error' | 'success'
  } = useQuery<User, Error>({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: userId > 0,  // query จะทำงานเฉพาะเมื่อ userId > 0
    staleTime: 2 * 60 * 1000, // 2 นาที
  });

  if (isLoading) return <div>กำลังโหลดข้อมูล...</div>;
  if (isError) return <div>เกิดข้อผิดพลาด: {error.message}</div>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>อีเมล: {user.email}</p>
      <p>โทร: {user.phone}</p>
      <p>ที่อยู่: {user.address.street}, {user.address.city}</p>
      {isFetching && <span>กำลังอัปเดต...</span>}
      <button onClick={() => refetch()}>รีเฟรช</button>
    </div>
  );
};
```

### 2.2 Query Keys ที่ดี

```typescript
// Query Keys ควรเป็น unique และ hierarchical
const queryKeys = {
  users: {
    all: () => ['users'] as const,
    lists: () => [...queryKeys.users.all(), 'list'] as const,
    list: (filters: UserFilters) => [...queryKeys.users.lists(), filters] as const,
    details: () => [...queryKeys.users.all(), 'detail'] as const,
    detail: (id: number) => [...queryKeys.users.details(), id] as const,
  },
  posts: {
    all: () => ['posts'] as const,
    byUser: (userId: number) => [...queryKeys.posts.all(), 'byUser', userId] as const,
    detail: (id: number) => [...queryKeys.posts.all(), 'detail', id] as const,
  },
} as const;

// การใช้งาน
const { data: users } = useQuery({
  queryKey: queryKeys.users.list({ role: 'admin' }),
  queryFn: () => fetchUsers({ role: 'admin' }),
});

const { data: user } = useQuery({
  queryKey: queryKeys.users.detail(userId),
  queryFn: () => fetchUser(userId),
});
```

### 2.3 useQuery กับ Parameters

```typescript
interface PostFilters {
  userId?: number;
  category?: string;
  search?: string;
  page?: number;
  limit?: number;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  totalPages: number;
  hasMore: boolean;
}

interface Post {
  id: number;
  userId: number;
  title: string;
  body: string;
  category: string;
  tags: string[];
  createdAt: string;
}

const fetchPosts = async (filters: PostFilters): Promise<PaginatedResponse<Post>> => {
  const params = new URLSearchParams();
  if (filters.userId) params.set('userId', filters.userId.toString());
  if (filters.category) params.set('category', filters.category);
  if (filters.search) params.set('search', filters.search);
  if (filters.page) params.set('page', filters.page.toString());
  if (filters.limit) params.set('limit', filters.limit.toString());

  const response = await fetch(`/api/posts?${params}`);
  if (!response.ok) throw new Error('ไม่สามารถโหลดโพสต์ได้');
  return response.json();
};

const PostList: React.FC = () => {
  const [filters, setFilters] = React.useState<PostFilters>({
    page: 1,
    limit: 10,
  });

  const { data, isLoading, isPlaceholderData } = useQuery({
    queryKey: ['posts', filters],
    queryFn: () => fetchPosts(filters),
    placeholderData: (previousData) => previousData, // เก็บ data เดิมไว้ขณะโหลด
  });

  return (
    <div>
      <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
        {isLoading ? (
          <div>กำลังโหลด...</div>
        ) : (
          data?.data.map(post => (
            <article key={post.id}>
              <h3>{post.title}</h3>
              <p>{post.body}</p>
              <div>{post.tags.map(tag => <span key={tag}>#{tag} </span>)}</div>
            </article>
          ))
        )}
      </div>

      <div className="pagination">
        <button
          onClick={() => setFilters(f => ({ ...f, page: (f.page ?? 1) - 1 }))}
          disabled={!filters.page || filters.page <= 1}
        >
          ก่อนหน้า
        </button>
        <span>หน้า {filters.page} จาก {data?.totalPages}</span>
        <button
          onClick={() => setFilters(f => ({ ...f, page: (f.page ?? 1) + 1 }))}
          disabled={!data?.hasMore}
        >
          ถัดไป
        </button>
      </div>
    </div>
  );
};
```

---

## 3. useMutation กับ TypeScript

### 3.1 useMutation พื้นฐาน

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface CreateUserData {
  name: string;
  email: string;
  role: 'admin' | 'user';
}

interface CreateUserResponse {
  id: number;
  name: string;
  email: string;
  role: string;
  createdAt: string;
}

const createUser = async (data: CreateUserData): Promise<CreateUserResponse> => {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message ?? 'ไม่สามารถสร้างผู้ใช้ได้');
  }
  return response.json();
};

const CreateUserForm: React.FC = () => {
  const queryClient = useQueryClient();
  const [formData, setFormData] = React.useState<CreateUserData>({
    name: '',
    email: '',
    role: 'user',
  });

  const mutation = useMutation<CreateUserResponse, Error, CreateUserData>({
    mutationFn: createUser,
    onSuccess: (data) => {
      // Invalidate และ refetch users list
      queryClient.invalidateQueries({ queryKey: ['users'] });
      // หรือ update cache โดยตรง
      queryClient.setQueryData<CreateUserResponse[]>(
        ['users'],
        (old) => old ? [...old, data] : [data]
      );
      alert(`สร้างผู้ใช้ ${data.name} สำเร็จ!`);
      setFormData({ name: '', email: '', role: 'user' });
    },
    onError: (error) => {
      alert(`เกิดข้อผิดพลาด: ${error.message}`);
    },
    onSettled: () => {
      // เรียกทุกครั้งไม่ว่า success หรือ error
      console.log('mutation สิ้นสุดแล้ว');
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate(formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="ชื่อ"
        value={formData.name}
        onChange={(e) => setFormData(f => ({ ...f, name: e.target.value }))}
        required
      />
      <input
        type="email"
        placeholder="อีเมล"
        value={formData.email}
        onChange={(e) => setFormData(f => ({ ...f, email: e.target.value }))}
        required
      />
      <select
        value={formData.role}
        onChange={(e) => setFormData(f => ({ ...f, role: e.target.value as any }))}
      >
        <option value="user">User</option>
        <option value="admin">Admin</option>
      </select>
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'กำลังสร้าง...' : 'สร้างผู้ใช้'}
      </button>
      {mutation.isError && <p>ข้อผิดพลาด: {mutation.error.message}</p>}
      {mutation.isSuccess && <p>สร้างสำเร็จ!</p>}
    </form>
  );
};
```

### 3.2 Delete Mutation

```typescript
interface DeleteUserVariables {
  userId: number;
  userName: string; // สำหรับแสดงใน UI
}

const deleteUser = async (userId: number): Promise<void> => {
  const response = await fetch(`/api/users/${userId}`, { method: 'DELETE' });
  if (!response.ok) throw new Error('ไม่สามารถลบผู้ใช้ได้');
};

const UserCard: React.FC<{ user: User }> = ({ user }) => {
  const queryClient = useQueryClient();

  const deleteMutation = useMutation<void, Error, DeleteUserVariables>({
    mutationFn: ({ userId }) => deleteUser(userId),
    onMutate: async ({ userId }) => {
      // ยกเลิก queries ที่กำลังทำงาน
      await queryClient.cancelQueries({ queryKey: ['users'] });
      
      // บันทึก state ก่อนหน้า
      const previousUsers = queryClient.getQueryData<User[]>(['users']);
      
      // Optimistic update
      queryClient.setQueryData<User[]>(['users'], (old) =>
        old ? old.filter(u => u.id !== userId) : []
      );
      
      return { previousUsers };
    },
    onError: (err, variables, context) => {
      // Rollback เมื่อเกิด error
      if (context?.previousUsers) {
        queryClient.setQueryData(['users'], context.previousUsers);
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  const handleDelete = () => {
    if (confirm(`ต้องการลบผู้ใช้ ${user.name} ใช่หรือไม่?`)) {
      deleteMutation.mutate({ userId: user.id, userName: user.name });
    }
  };

  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <button
        onClick={handleDelete}
        disabled={deleteMutation.isPending}
      >
        {deleteMutation.isPending ? 'กำลังลบ...' : 'ลบ'}
      </button>
    </div>
  );
};
```

---

## 4. Optimistic Updates

### 4.1 Optimistic Update กับ Todo

```typescript
interface Todo {
  id: number;
  title: string;
  completed: boolean;
  userId: number;
}

interface UpdateTodoVariables {
  id: number;
  completed: boolean;
}

const updateTodo = async ({ id, completed }: UpdateTodoVariables): Promise<Todo> => {
  const response = await fetch(`/api/todos/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ completed }),
  });
  if (!response.ok) throw new Error('ไม่สามารถอัปเดต todo ได้');
  return response.json();
};

const TodoItem: React.FC<{ todo: Todo }> = ({ todo }) => {
  const queryClient = useQueryClient();

  const toggleMutation = useMutation<Todo, Error, UpdateTodoVariables, { previousTodos: Todo[] | undefined }>({
    mutationFn: updateTodo,
    
    onMutate: async (variables) => {
      // ยกเลิก refetch ที่กำลังรอ
      await queryClient.cancelQueries({ queryKey: ['todos', todo.userId] });
      
      // Snapshot state ปัจจุบัน
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos', todo.userId]);
      
      // Optimistic update
      queryClient.setQueryData<Todo[]>(['todos', todo.userId], (old) =>
        old?.map(t =>
          t.id === variables.id
            ? { ...t, completed: variables.completed }
            : t
        )
      );
      
      // Return context สำหรับ rollback
      return { previousTodos };
    },
    
    onError: (_err, _variables, context) => {
      // Rollback
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos', todo.userId], context.previousTodos);
      }
    },
    
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos', todo.userId] });
    },
  });

  return (
    <li>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={(e) =>
          toggleMutation.mutate({ id: todo.id, completed: e.target.checked })
        }
      />
      <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
        {todo.title}
      </span>
    </li>
  );
};
```

---

## 5. Infinite Queries

### 5.1 useInfiniteQuery

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

interface InfinitePostsResponse {
  posts: Post[];
  nextCursor: string | null;
  prevCursor: string | null;
  hasMore: boolean;
}

const fetchInfinitePosts = async ({
  pageParam = null,
  queryKey,
}: {
  pageParam: string | null;
  queryKey: readonly unknown[];
}): Promise<InfinitePostsResponse> => {
  const [, category] = queryKey as [string, string?];
  const params = new URLSearchParams({ limit: '10' });
  if (pageParam) params.set('cursor', pageParam);
  if (category) params.set('category', category);

  const response = await fetch(`/api/posts?${params}`);
  if (!response.ok) throw new Error('ไม่สามารถโหลดโพสต์ได้');
  return response.json();
};

const InfinitePostFeed: React.FC<{ category?: string }> = ({ category }) => {
  const {
    data,
    fetchNextPage,
    fetchPreviousPage,
    hasNextPage,
    hasPreviousPage,
    isFetchingNextPage,
    isFetchingPreviousPage,
    isLoading,
    isError,
    error,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite', category],
    queryFn: fetchInfinitePosts,
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    getPreviousPageParam: (firstPage) => firstPage.prevCursor,
  });

  // Intersection Observer สำหรับ infinite scroll
  const loadMoreRef = React.useRef<HTMLDivElement>(null);

  React.useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.5 }
    );

    if (loadMoreRef.current) observer.observe(loadMoreRef.current);
    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <div>กำลังโหลด...</div>;
  if (isError) return <div>เกิดข้อผิดพลาด: {(error as Error).message}</div>;

  // รวม pages ทั้งหมดเข้าด้วยกัน
  const allPosts = data.pages.flatMap(page => page.posts);

  return (
    <div>
      {allPosts.map(post => (
        <article key={post.id}>
          <h3>{post.title}</h3>
          <p>{post.body}</p>
        </article>
      ))}

      {/* Trigger สำหรับ infinite scroll */}
      <div ref={loadMoreRef} style={{ height: 20 }}>
        {isFetchingNextPage && <div>กำลังโหลดเพิ่มเติม...</div>}
        {!hasNextPage && <div>แสดงครบทั้งหมดแล้ว</div>}
      </div>
    </div>
  );
};
```

---

## 6. Query Invalidation

### 6.1 Invalidation Patterns

```typescript
import { useQueryClient } from '@tanstack/react-query';

const AdminPanel: React.FC = () => {
  const queryClient = useQueryClient();

  // Invalidate ทุก queries ที่เกี่ยวกับ users
  const invalidateAllUsers = () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  };

  // Invalidate เฉพาะ user detail ของ user id 1
  const invalidateUser = (userId: number) => {
    queryClient.invalidateQueries({
      queryKey: ['users', 'detail', userId],
      exact: true,
    });
  };

  // Invalidate หลาย queries พร้อมกัน
  const handleUserUpdate = async (userId: number) => {
    await Promise.all([
      queryClient.invalidateQueries({ queryKey: ['users'] }),
      queryClient.invalidateQueries({ queryKey: ['posts', 'byUser', userId] }),
      queryClient.invalidateQueries({ queryKey: ['comments', 'byUser', userId] }),
    ]);
  };

  // Refetch immediately (ไม่รอ stale)
  const forceRefetch = () => {
    queryClient.refetchQueries({
      queryKey: ['users'],
      type: 'active',
    });
  };

  // Remove queries จาก cache
  const removeFromCache = (userId: number) => {
    queryClient.removeQueries({ queryKey: ['users', 'detail', userId] });
  };

  // Set data โดยตรง (ไม่ fetch)
  const prefillUserData = (user: User) => {
    queryClient.setQueryData(['users', 'detail', user.id], user);
  };

  return (
    <div>
      <button onClick={invalidateAllUsers}>รีเฟรชข้อมูลผู้ใช้ทั้งหมด</button>
      <button onClick={forceRefetch}>บังคับ Refetch</button>
    </div>
  );
};
```

---

## 7. Prefetching

### 7.1 Prefetch on Hover

```typescript
const UserList: React.FC = () => {
  const queryClient = useQueryClient();
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  const prefetchUser = async (userId: number) => {
    await queryClient.prefetchQuery({
      queryKey: ['users', 'detail', userId],
      queryFn: () => fetchUser(userId),
      staleTime: 5 * 60 * 1000,
    });
  };

  return (
    <ul>
      {users?.map(user => (
        <li
          key={user.id}
          onMouseEnter={() => prefetchUser(user.id)}
        >
          <Link to={`/users/${user.id}`}>{user.name}</Link>
        </li>
      ))}
    </ul>
  );
};
```

### 7.2 Server-side Prefetching (SSR)

```typescript
// สำหรับ Next.js
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';

// Server Component
export default async function UsersPage() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ['users'],
    queryFn: fetchUsersOnServer,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <UserList />
    </HydrationBoundary>
  );
}
```

---

## 8. Error Handling

### 8.1 Error Boundaries กับ React Query

```typescript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';

interface ErrorFallbackProps {
  error: Error;
  resetErrorBoundary: () => void;
}

const ErrorFallback: React.FC<ErrorFallbackProps> = ({ error, resetErrorBoundary }) => (
  <div role="alert">
    <h2>เกิดข้อผิดพลาด</h2>
    <p>{error.message}</p>
    <button onClick={resetErrorBoundary}>ลองใหม่</button>
  </div>
);

const SafeUserData: React.FC = () => (
  <QueryErrorResetBoundary>
    {({ reset }) => (
      <ErrorBoundary onReset={reset} FallbackComponent={ErrorFallback}>
        <UserData />
      </ErrorBoundary>
    )}
  </QueryErrorResetBoundary>
);

// Component ที่ throw error เมื่อ query ล้มเหลว
const UserData: React.FC = () => {
  const { data } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    throwOnError: true, // โยน error ไปที่ ErrorBoundary
  });

  return <div>{data?.map(u => <div key={u.id}>{u.name}</div>)}</div>;
};
```

### 8.2 Typed Error Handling

```typescript
// Custom error class
class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code: string,
    public details?: Record<string, string[]>
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// Fetch ที่ throw ApiError
const fetchWithError = async <T>(url: string): Promise<T> => {
  const response = await fetch(url);
  
  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}));
    throw new ApiError(
      errorData.message ?? `HTTP Error ${response.status}`,
      response.status,
      errorData.code ?? 'UNKNOWN_ERROR',
      errorData.details
    );
  }
  
  return response.json();
};

// Component ที่ handle typed errors
const UserDetail: React.FC<{ userId: number }> = ({ userId }) => {
  const { data, error, isError } = useQuery<User, ApiError>({
    queryKey: ['user', userId],
    queryFn: () => fetchWithError<User>(`/api/users/${userId}`),
    retry: (failureCount, error) => {
      // ไม่ retry ถ้า 404 หรือ 403
      if (error.status === 404 || error.status === 403) return false;
      return failureCount < 3;
    },
  });

  if (isError) {
    if (error.status === 404) {
      return <div>ไม่พบผู้ใช้ที่ต้องการ</div>;
    }
    if (error.status === 403) {
      return <div>ไม่มีสิทธิ์เข้าถึงข้อมูลนี้</div>;
    }
    return <div>เกิดข้อผิดพลาด: {error.message}</div>;
  }

  return data ? <div>{data.name}</div> : null;
};
```

---

## 9. Custom Hooks

### 9.1 useUsers Hook

```typescript
interface UseUsersOptions {
  filters?: UserFilters;
  enabled?: boolean;
}

interface UseUsersReturn {
  users: User[];
  isLoading: boolean;
  isError: boolean;
  error: Error | null;
  total: number;
  refetch: () => void;
}

export function useUsers({ filters, enabled = true }: UseUsersOptions = {}): UseUsersReturn {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['users', 'list', filters],
    queryFn: () => fetchUsers(filters),
    enabled,
    select: (data) => ({
      users: data.data,
      total: data.total,
    }),
  });

  return {
    users: data?.users ?? [],
    total: data?.total ?? 0,
    isLoading,
    isError,
    error: error as Error | null,
    refetch,
  };
}

// useUser hook สำหรับ single user
export function useUser(userId: number) {
  return useQuery({
    queryKey: ['users', 'detail', userId],
    queryFn: () => fetchUser(userId),
    enabled: userId > 0,
    staleTime: 2 * 60 * 1000,
  });
}

// useCreateUser hook
export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createUser,
    onSuccess: (newUser) => {
      queryClient.invalidateQueries({ queryKey: ['users', 'list'] });
      queryClient.setQueryData(['users', 'detail', newUser.id], newUser);
    },
  });
}

// useUpdateUser hook
export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<User> }) =>
      updateUser(id, data),
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: ['users', 'detail', id] });
      const previous = queryClient.getQueryData<User>(['users', 'detail', id]);
      queryClient.setQueryData<User>(['users', 'detail', id], (old) =>
        old ? { ...old, ...data } : undefined
      );
      return { previous };
    },
    onError: (_err, { id }, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['users', 'detail', id], context.previous);
      }
    },
    onSettled: (_data, _err, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['users', 'detail', id] });
    },
  });
}

// useDeleteUser hook
export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (userId: number) => deleteUser(userId),
    onSuccess: (_data, userId) => {
      queryClient.removeQueries({ queryKey: ['users', 'detail', userId] });
      queryClient.invalidateQueries({ queryKey: ['users', 'list'] });
    },
  });
}
```

---

## 10. ตัวอย่าง CRUD สมบูรณ์

### 10.1 Product CRUD

```typescript
// api/products.ts
interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
  category: string;
  stock: number;
  imageUrl: string;
  isActive: boolean;
}

type CreateProductData = Omit<Product, 'id' | 'isActive'>;
type UpdateProductData = Partial<Omit<Product, 'id'>>;

const API_URL = '/api/products';

const productsApi = {
  getAll: async (params?: { page?: number; category?: string }): Promise<PaginatedResponse<Product>> => {
    const url = new URL(API_URL, window.location.origin);
    if (params?.page) url.searchParams.set('page', params.page.toString());
    if (params?.category) url.searchParams.set('category', params.category);
    const res = await fetch(url.toString());
    if (!res.ok) throw new Error('ไม่สามารถโหลดสินค้าได้');
    return res.json();
  },

  getById: async (id: number): Promise<Product> => {
    const res = await fetch(`${API_URL}/${id}`);
    if (!res.ok) throw new Error(`ไม่พบสินค้า #${id}`);
    return res.json();
  },

  create: async (data: CreateProductData): Promise<Product> => {
    const res = await fetch(API_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error('ไม่สามารถสร้างสินค้าได้');
    return res.json();
  },

  update: async (id: number, data: UpdateProductData): Promise<Product> => {
    const res = await fetch(`${API_URL}/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) throw new Error('ไม่สามารถอัปเดตสินค้าได้');
    return res.json();
  },

  delete: async (id: number): Promise<void> => {
    const res = await fetch(`${API_URL}/${id}`, { method: 'DELETE' });
    if (!res.ok) throw new Error('ไม่สามารถลบสินค้าได้');
  },
};

// hooks/useProducts.ts
export function useProducts(params?: { page?: number; category?: string }) {
  return useQuery({
    queryKey: ['products', params],
    queryFn: () => productsApi.getAll(params),
  });
}

export function useProduct(id: number) {
  return useQuery({
    queryKey: ['products', id],
    queryFn: () => productsApi.getById(id),
    enabled: id > 0,
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: productsApi.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}

export function useUpdateProduct(id: number) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: UpdateProductData) => productsApi.update(id, data),
    onMutate: async (data) => {
      await queryClient.cancelQueries({ queryKey: ['products', id] });
      const previous = queryClient.getQueryData<Product>(['products', id]);
      queryClient.setQueryData<Product>(['products', id], old =>
        old ? { ...old, ...data } : undefined
      );
      return { previous };
    },
    onError: (_err, _data, context) => {
      if (context?.previous) {
        queryClient.setQueryData(['products', id], context.previous);
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['products', id] });
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}

export function useDeleteProduct() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: productsApi.delete,
    onSuccess: (_data, id) => {
      queryClient.removeQueries({ queryKey: ['products', id] });
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}

// ProductManager Component
const ProductManager: React.FC = () => {
  const [page, setPage] = React.useState(1);
  const [editingId, setEditingId] = React.useState<number | null>(null);

  const { data, isLoading } = useProducts({ page });
  const createMutation = useCreateProduct();
  const deleteMutation = useDeleteProduct();

  const handleCreate = async (data: CreateProductData) => {
    await createMutation.mutateAsync(data);
  };

  const handleDelete = async (id: number, name: string) => {
    if (confirm(`ลบสินค้า "${name}" ใช่หรือไม่?`)) {
      await deleteMutation.mutateAsync(id);
    }
  };

  if (isLoading) return <div>กำลังโหลด...</div>;

  return (
    <div>
      <h1>จัดการสินค้า</h1>
      
      <CreateProductForm onSubmit={handleCreate} isLoading={createMutation.isPending} />
      
      <table>
        <thead>
          <tr>
            <th>ชื่อสินค้า</th>
            <th>ราคา</th>
            <th>หมวดหมู่</th>
            <th>สต็อก</th>
            <th>การดำเนินการ</th>
          </tr>
        </thead>
        <tbody>
          {data?.data.map(product => (
            <tr key={product.id}>
              <td>{product.name}</td>
              <td>฿{product.price.toFixed(2)}</td>
              <td>{product.category}</td>
              <td>{product.stock}</td>
              <td>
                <button onClick={() => setEditingId(product.id)}>แก้ไข</button>
                <button onClick={() => handleDelete(product.id, product.name)}>
                  ลบ
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>

      {editingId && (
        <EditProductModal
          productId={editingId}
          onClose={() => setEditingId(null)}
        />
      )}
    </div>
  );
};
```

---

## 11. Advanced Features

### 11.1 Dependent Queries

```typescript
// Query ที่ขึ้นอยู่กับ query อื่น
const UserPostsAndComments: React.FC<{ userId: number }> = ({ userId }) => {
  // Query 1: Get user
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Query 2: Get posts (depends on user)
  const { data: posts } = useQuery({
    queryKey: ['posts', 'byUser', user?.id],
    queryFn: () => fetchPostsByUser(user!.id),
    enabled: !!user, // รอให้ user โหลดก่อน
  });

  // Query 3: Get first post's comments (depends on posts)
  const firstPostId = posts?.[0]?.id;
  const { data: comments } = useQuery({
    queryKey: ['comments', 'byPost', firstPostId],
    queryFn: () => fetchCommentsByPost(firstPostId!),
    enabled: !!firstPostId, // รอให้ posts โหลดก่อน
  });

  return (
    <div>
      <h2>{user?.name}</h2>
      <h3>โพสต์ ({posts?.length ?? 0})</h3>
      {posts?.map(post => <div key={post.id}>{post.title}</div>)}
      <h3>ความคิดเห็นล่าสุด</h3>
      {comments?.map(comment => <div key={comment.id}>{comment.body}</div>)}
    </div>
  );
};
```

### 11.2 Parallel Queries

```typescript
// ทำหลาย queries พร้อมกัน
import { useQueries } from '@tanstack/react-query';

const MultipleUsers: React.FC<{ userIds: number[] }> = ({ userIds }) => {
  const userQueries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
    combine: (results) => ({
      users: results.map(r => r.data).filter(Boolean) as User[],
      isLoading: results.some(r => r.isLoading),
      isError: results.some(r => r.isError),
    }),
  });

  if (userQueries.isLoading) return <div>กำลังโหลด...</div>;

  return (
    <div>
      {userQueries.users.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
};
```

### 11.3 Background Refetching

```typescript
const LiveDashboard: React.FC = () => {
  const { data: stats, dataUpdatedAt } = useQuery({
    queryKey: ['dashboard', 'stats'],
    queryFn: fetchDashboardStats,
    refetchInterval: 30 * 1000,  // refetch ทุก 30 วินาที
    refetchIntervalInBackground: true,  // refetch แม้ tab ไม่ active
  });

  return (
    <div>
      <h2>Dashboard แบบ Real-time</h2>
      <p>อัปเดตล่าสุด: {new Date(dataUpdatedAt).toLocaleTimeString('th-TH')}</p>
      {stats && (
        <div>
          <p>ผู้ใช้ออนไลน์: {stats.onlineUsers}</p>
          <p>คำสั่งซื้อวันนี้: {stats.todayOrders}</p>
          <p>รายได้วันนี้: ฿{stats.todayRevenue.toLocaleString()}</p>
        </div>
      )}
    </div>
  );
};
```

---

## 12. React Query DevTools

### 12.1 การตั้งค่า DevTools

```typescript
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router>
        <Routes />
      </Router>
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools
          initialIsOpen={false}
          buttonPosition="bottom-right"
          position="bottom"
        />
      )}
    </QueryClientProvider>
  );
}
```

---

## 13. TypeScript Tips สำหรับ React Query

### 13.1 Generic Query Function

```typescript
// Reusable generic query function
async function fetchResource<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`/api${endpoint}`, {
    headers: { 'Content-Type': 'application/json', ...options?.headers },
    ...options,
  });
  
  if (!response.ok) {
    const error = await response.json().catch(() => ({ message: 'Request failed' }));
    throw new Error(error.message);
  }
  
  return response.json();
}

// การใช้งาน
const { data: users } = useQuery<User[]>({
  queryKey: ['users'],
  queryFn: () => fetchResource<User[]>('/users'),
});

const { data: products } = useQuery<Product[]>({
  queryKey: ['products'],
  queryFn: () => fetchResource<Product[]>('/products'),
});
```

### 13.2 useQuery สำหรับ Form Default Values

```typescript
// Pattern สำหรับ edit form ที่โหลด data มาก่อน
interface EditUserFormProps {
  userId: number;
  onSuccess: () => void;
}

const EditUserForm: React.FC<EditUserFormProps> = ({ userId, onSuccess }) => {
  const { data: user, isLoading } = useUser(userId);
  const updateMutation = useUpdateUser();

  const [formData, setFormData] = React.useState<Partial<User>>({});

  // อัปเดต form เมื่อ user data โหลดเสร็จ
  React.useEffect(() => {
    if (user) {
      setFormData({
        name: user.name,
        email: user.email,
        role: user.role,
      });
    }
  }, [user]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await updateMutation.mutateAsync({ id: userId, data: formData });
    onSuccess();
  };

  if (isLoading) return <div>กำลังโหลด...</div>;

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name ?? ''}
        onChange={(e) => setFormData(f => ({ ...f, name: e.target.value }))}
        placeholder="ชื่อ"
      />
      <input
        value={formData.email ?? ''}
        onChange={(e) => setFormData(f => ({ ...f, email: e.target.value }))}
        placeholder="อีเมล"
      />
      <button type="submit" disabled={updateMutation.isPending}>
        {updateMutation.isPending ? 'กำลังบันทึก...' : 'บันทึก'}
      </button>
    </form>
  );
};
```

---

## 14. สรุป

TanStack Query กับ TypeScript เป็นคู่ที่ทำงานได้ดีมากสำหรับ server state management:

**ข้อดีหลัก:**
1. **Type Safety**: Generic types ทำให้ data, error, และ variables ถูก type อย่างถูกต้อง
2. **Caching อัตโนมัติ**: ลด API calls ที่ไม่จำเป็น
3. **Background Updates**: ข้อมูลสดเสมอโดยไม่ต้องใช้ manual polling
4. **Optimistic Updates**: UX ดีขึ้นโดย update UI ก่อน server response
5. **DevTools**: Debug ง่ายด้วย built-in devtools

**Best Practices:**
- ใช้ Query Key factories เพื่อจัดการ keys อย่างเป็นระบบ
- สร้าง custom hooks ครอบ useQuery/useMutation
- Handle errors ด้วย typed error classes
- ใช้ Optimistic Updates สำหรับ interactive UI
- แยก server state (React Query) และ client state (Zustand/Context)

---

*จบบทที่ 29 - React Query กับ TypeScript*
