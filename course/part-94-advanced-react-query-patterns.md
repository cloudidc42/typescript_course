# Part 94: Advanced React Query Patterns

## บทนำ

React Query (TanStack Query) เป็น library ที่ทรงพลังสำหรับจัดการ server state ใน React applications บทนี้จะครอบคลุม patterns ขั้นสูงที่ใช้ในโปรเจกต์จริง

## การตั้งค่าพื้นฐาน

```typescript
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

function makeQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 นาที
        gcTime: 10 * 60 * 1000, // 10 นาที (เดิมคือ cacheTime)
        retry: (failureCount, error) => {
          // ไม่ retry สำหรับ 4xx errors
          if (error instanceof Error && 'status' in error) {
            const status = (error as any).status;
            if (status >= 400 && status < 500) return false;
          }
          return failureCount < 3;
        },
        refetchOnWindowFocus: true,
        refetchOnReconnect: true,
      },
      mutations: {
        retry: 1,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined = undefined;

function getQueryClient(): QueryClient {
  if (typeof window === 'undefined') {
    return makeQueryClient();
  } else {
    if (!browserQueryClient) browserQueryClient = makeQueryClient();
    return browserQueryClient;
  }
}

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => getQueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

## ส่วนที่ 1: Query Key Factory Pattern

### 1.1 ปัญหาของการจัดการ Query Keys

```typescript
// ❌ วิธีที่ไม่ดี - กระจัดกระจาย
useQuery({ queryKey: ['posts'] });
useQuery({ queryKey: ['posts', 1] });
useQuery({ queryKey: ['posts', 'category', 'typescript'] });
useMutation({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  }
});
```

### 1.2 Query Key Factory

```typescript
// lib/queryKeys.ts

// Factory pattern สำหรับ query keys
export const queryKeys = {
  // Posts
  posts: {
    all: () => ['posts'] as const,
    lists: () => [...queryKeys.posts.all(), 'list'] as const,
    list: (params: PostQueryParams) => [...queryKeys.posts.lists(), params] as const,
    details: () => [...queryKeys.posts.all(), 'detail'] as const,
    detail: (slug: string) => [...queryKeys.posts.details(), slug] as const,
    byCategory: (categorySlug: string) =>
      [...queryKeys.posts.lists(), 'category', categorySlug] as const,
    byTag: (tagSlug: string) =>
      [...queryKeys.posts.lists(), 'tag', tagSlug] as const,
    byAuthor: (authorId: string) =>
      [...queryKeys.posts.lists(), 'author', authorId] as const,
  },

  // Users
  users: {
    all: () => ['users'] as const,
    current: () => [...queryKeys.users.all(), 'current'] as const,
    detail: (id: string) => [...queryKeys.users.all(), id] as const,
    profile: (username: string) =>
      [...queryKeys.users.all(), 'profile', username] as const,
  },

  // Comments
  comments: {
    all: () => ['comments'] as const,
    byPost: (postId: string) =>
      [...queryKeys.comments.all(), 'post', postId] as const,
  },

  // Search
  search: {
    all: () => ['search'] as const,
    results: (query: string) =>
      [...queryKeys.search.all(), 'results', query] as const,
    suggestions: (query: string) =>
      [...queryKeys.search.all(), 'suggestions', query] as const,
  },
} as const;

interface PostQueryParams {
  page?: number;
  limit?: number;
  categorySlug?: string;
  tagSlug?: string;
  authorId?: string;
  search?: string;
}

// ใช้งาน
function usePosts(params: PostQueryParams = {}) {
  return useQuery({
    queryKey: queryKeys.posts.list(params),
    queryFn: () => fetchPosts(params),
  });
}

function usePost(slug: string) {
  return useQuery({
    queryKey: queryKeys.posts.detail(slug),
    queryFn: () => fetchPost(slug),
  });
}

// Invalidation ง่ายขึ้นมาก
function useUpdatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updatePost,
    onSuccess: (data, variables) => {
      // Invalidate list
      queryClient.invalidateQueries({
        queryKey: queryKeys.posts.lists(),
      });
      // Update specific post
      queryClient.setQueryData(
        queryKeys.posts.detail(variables.slug),
        data
      );
    },
  });
}
```

## ส่วนที่ 2: Dependent Queries

### 2.1 Sequential Queries

```typescript
// hooks/useUserWithPosts.ts

import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '../lib/queryKeys';

interface User {
  id: string;
  username: string;
  email: string;
}

interface Post {
  id: string;
  title: string;
  authorId: string;
}

function useUserWithPosts(username: string) {
  // Query 1: ดึงข้อมูลผู้ใช้ก่อน
  const userQuery = useQuery({
    queryKey: queryKeys.users.profile(username),
    queryFn: () => fetchUserByUsername(username),
    enabled: !!username,
  });

  // Query 2: ดึง posts เมื่อมี userId แล้ว
  const postsQuery = useQuery({
    queryKey: queryKeys.posts.byAuthor(userQuery.data?.id || ''),
    queryFn: () => fetchPostsByAuthor(userQuery.data!.id),
    enabled: !!userQuery.data?.id, // รอจนกว่าจะมี user data
  });

  return {
    user: userQuery.data,
    posts: postsQuery.data,
    isLoading: userQuery.isLoading || postsQuery.isLoading,
    isError: userQuery.isError || postsQuery.isError,
    error: userQuery.error || postsQuery.error,
  };
}

// ตัวอย่างการใช้
function UserProfile({ username }: { username: string }) {
  const { user, posts, isLoading, isError } = useUserWithPosts(username);

  if (isLoading) return <div>กำลังโหลด...</div>;
  if (isError) return <div>เกิดข้อผิดพลาด</div>;

  return (
    <div>
      <h1>{user?.username}</h1>
      <p>จำนวนบทความ: {posts?.length}</p>
    </div>
  );
}
```

### 2.2 Multi-level Dependencies

```typescript
// hooks/useNestedData.ts

function usePostWithAuthorAndComments(slug: string) {
  const postQuery = useQuery({
    queryKey: queryKeys.posts.detail(slug),
    queryFn: () => fetchPost(slug),
  });

  const authorQuery = useQuery({
    queryKey: queryKeys.users.detail(postQuery.data?.authorId || ''),
    queryFn: () => fetchUser(postQuery.data!.authorId),
    enabled: !!postQuery.data?.authorId,
  });

  const commentsQuery = useQuery({
    queryKey: queryKeys.comments.byPost(postQuery.data?.id || ''),
    queryFn: () => fetchCommentsByPost(postQuery.data!.id),
    enabled: !!postQuery.data?.id,
  });

  return {
    post: postQuery.data,
    author: authorQuery.data,
    comments: commentsQuery.data ?? [],
    isLoading:
      postQuery.isLoading ||
      (!!postQuery.data && authorQuery.isLoading) ||
      (!!postQuery.data && commentsQuery.isLoading),
    states: {
      post: postQuery.status,
      author: authorQuery.status,
      comments: commentsQuery.status,
    },
  };
}
```

## ส่วนที่ 3: Parallel Queries

### 3.1 useQueries

```typescript
// hooks/useMultipleUsers.ts

import { useQueries } from '@tanstack/react-query';

function useMultipleUsers(userIds: string[]) {
  const userQueries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: queryKeys.users.detail(id),
      queryFn: () => fetchUser(id),
      staleTime: 5 * 60 * 1000,
    })),
  });

  const isLoading = userQueries.some((q) => q.isLoading);
  const isError = userQueries.some((q) => q.isError);
  const users = userQueries
    .map((q) => q.data)
    .filter((u): u is User => u !== undefined);

  return { users, isLoading, isError, queries: userQueries };
}

// Dashboard ที่โหลดหลาย section พร้อมกัน
function useDashboardData() {
  const queries = useQueries({
    queries: [
      {
        queryKey: ['dashboard', 'stats'],
        queryFn: fetchDashboardStats,
        staleTime: 5 * 60 * 1000,
      },
      {
        queryKey: ['dashboard', 'recentPosts'],
        queryFn: () => fetchPosts({ limit: 5, status: 'published' }),
        staleTime: 2 * 60 * 1000,
      },
      {
        queryKey: ['dashboard', 'topTags'],
        queryFn: fetchTopTags,
        staleTime: 10 * 60 * 1000,
      },
    ],
    combine: (results) => ({
      stats: results[0].data,
      recentPosts: results[1].data,
      topTags: results[2].data,
      isLoading: results.some((r) => r.isLoading),
      isError: results.some((r) => r.isError),
    }),
  });

  return queries;
}
```

## ส่วนที่ 4: Infinite Scroll

### 4.1 useInfiniteQuery

```typescript
// hooks/useInfinitePosts.ts

import { useInfiniteQuery } from '@tanstack/react-query';
import { useIntersectionObserver } from './useIntersectionObserver';
import { useRef, useEffect } from 'react';

interface PostsPage {
  data: Post[];
  meta: {
    page: number;
    hasNextPage: boolean;
    nextCursor?: string;
  };
}

// Cursor-based pagination
function useInfinitePostsByCursor(params?: { categorySlug?: string }) {
  return useInfiniteQuery({
    queryKey: [...queryKeys.posts.lists(), 'infinite', 'cursor', params],
    queryFn: async ({ pageParam }) => {
      const response = await fetchPostsCursor({
        cursor: pageParam as string | undefined,
        limit: 10,
        ...params,
      });
      return response as PostsPage;
    },
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.meta.nextCursor,
    getPreviousPageParam: undefined,
  });
}

// Page-based pagination
function useInfinitePostsByPage(params?: { categorySlug?: string }) {
  return useInfiniteQuery({
    queryKey: [...queryKeys.posts.lists(), 'infinite', 'page', params],
    queryFn: async ({ pageParam }) => {
      const response = await fetchPosts({
        page: pageParam as number,
        limit: 10,
        ...params,
      });
      return response;
    },
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.meta.hasNextPage ? lastPage.meta.page + 1 : undefined,
  });
}

// Component ที่ใช้ Infinite Scroll
function InfinitePostsList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
  } = useInfinitePostsByPage();

  const loadMoreRef = useRef<HTMLDivElement>(null);

  // Intersection Observer สำหรับ auto-load
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );

    if (loadMoreRef.current) {
      observer.observe(loadMoreRef.current);
    }

    return () => observer.disconnect();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <PostsSkeleton />;
  if (isError) return <ErrorMessage />;

  const allPosts = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div className="space-y-6">
      {allPosts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}

      {/* Sentinel element */}
      <div ref={loadMoreRef} className="h-4">
        {isFetchingNextPage && (
          <div className="flex justify-center py-4">
            <Spinner />
          </div>
        )}
        {!hasNextPage && allPosts.length > 0 && (
          <p className="text-center text-gray-500">โหลดครบแล้ว</p>
        )}
      </div>
    </div>
  );
}
```

## ส่วนที่ 5: Optimistic Updates

### 5.1 Optimistic Like

```typescript
// hooks/useToggleLike.ts

import { useMutation, useQueryClient } from '@tanstack/react-query';

interface LikeResult {
  liked: boolean;
  likesCount: number;
}

function useToggleLike(postId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: string) =>
      apiClient.post<LikeResult>(`/likes/posts/${postId}`).then(r => r.data),

    // Optimistic update
    onMutate: async (postId) => {
      // ยกเลิก queries ที่กำลังดึงข้อมูลอยู่
      await queryClient.cancelQueries({
        queryKey: queryKeys.posts.detail(postId),
      });

      // ดึง snapshot ก่อน update
      const previousPost = queryClient.getQueryData<Post>(
        queryKeys.posts.detail(postId)
      );

      // อัพเดทแบบ optimistic
      if (previousPost) {
        const isLiked = previousPost.isLiked;
        queryClient.setQueryData<Post>(
          queryKeys.posts.detail(postId),
          (old) =>
            old
              ? {
                  ...old,
                  isLiked: !isLiked,
                  likesCount: isLiked ? old.likesCount - 1 : old.likesCount + 1,
                }
              : old
        );
      }

      // อัพเดทใน list ด้วย
      queryClient.setQueriesData<{ data: Post[] }>(
        { queryKey: queryKeys.posts.lists() },
        (old) => {
          if (!old) return old;
          return {
            ...old,
            data: old.data.map((post) =>
              post.id === postId
                ? {
                    ...post,
                    isLiked: !post.isLiked,
                    likesCount: post.isLiked
                      ? post.likesCount - 1
                      : post.likesCount + 1,
                  }
                : post
            ),
          };
        }
      );

      return { previousPost };
    },

    // Rollback ถ้า error
    onError: (err, postId, context) => {
      if (context?.previousPost) {
        queryClient.setQueryData(
          queryKeys.posts.detail(postId),
          context.previousPost
        );
      }
      // แสดง toast error
      console.error('เกิดข้อผิดพลาดในการกดถูกใจ:', err);
    },

    // Refetch เพื่อให้แน่ใจว่าข้อมูลตรงกัน
    onSettled: (data, error, postId) => {
      queryClient.invalidateQueries({
        queryKey: queryKeys.posts.detail(postId),
      });
    },
  });
}
```

### 5.2 Optimistic Comment

```typescript
// hooks/useAddComment.ts

function useAddComment(postId: string) {
  const queryClient = useQueryClient();
  const currentUser = useCurrentUser();

  return useMutation({
    mutationFn: (content: string) =>
      apiClient
        .post<Comment>(`/posts/${postId}/comments`, { content })
        .then((r) => r.data),

    onMutate: async (content: string) => {
      await queryClient.cancelQueries({
        queryKey: queryKeys.comments.byPost(postId),
      });

      const previousComments = queryClient.getQueryData<Comment[]>(
        queryKeys.comments.byPost(postId)
      );

      // Optimistic comment (ยังไม่มี ID จริง)
      const optimisticComment: Comment = {
        id: `temp-${Date.now()}`,
        content,
        author: currentUser!,
        postId,
        createdAt: new Date(),
        updatedAt: new Date(),
        likesCount: 0,
        isOptimistic: true,
      };

      queryClient.setQueryData<Comment[]>(
        queryKeys.comments.byPost(postId),
        (old) => [optimisticComment, ...(old || [])]
      );

      // อัพเดท commentsCount ใน post
      queryClient.setQueryData<Post>(
        queryKeys.posts.detail(postId),
        (old) =>
          old ? { ...old, commentsCount: old.commentsCount + 1 } : old
      );

      return { previousComments, optimisticComment };
    },

    onError: (err, content, context) => {
      // Rollback
      if (context?.previousComments !== undefined) {
        queryClient.setQueryData(
          queryKeys.comments.byPost(postId),
          context.previousComments
        );
      }
      // Rollback post comment count
      queryClient.setQueryData<Post>(
        queryKeys.posts.detail(postId),
        (old) =>
          old ? { ...old, commentsCount: old.commentsCount - 1 } : old
      );
    },

    onSuccess: (newComment, content, context) => {
      // แทนที่ optimistic comment ด้วย real comment
      queryClient.setQueryData<Comment[]>(
        queryKeys.comments.byPost(postId),
        (old) =>
          old?.map((comment) =>
            comment.id === context?.optimisticComment.id
              ? newComment
              : comment
          ) || []
      );
    },
  });
}
```

## ส่วนที่ 6: Background Refetching Strategies

### 6.1 Polling

```typescript
// hooks/useRealtimeStats.ts

function useRealtimeStats(interval = 5000) {
  return useQuery({
    queryKey: ['dashboard', 'realtime-stats'],
    queryFn: fetchRealtimeStats,
    refetchInterval: interval,
    refetchIntervalInBackground: true, // Refetch แม้ tab ไม่ active
    staleTime: 0, // ถือว่า stale ทันที
  });
}

// Conditional polling
function useOnlineStatus() {
  const [isPolling, setIsPolling] = useState(true);

  const query = useQuery({
    queryKey: ['online-status'],
    queryFn: checkOnlineStatus,
    refetchInterval: isPolling ? 10000 : false,
    refetchIntervalInBackground: false,
  });

  return {
    ...query,
    startPolling: () => setIsPolling(true),
    stopPolling: () => setIsPolling(false),
  };
}
```

### 6.2 Stale-while-revalidate

```typescript
// hooks/usePosts.ts

function usePostsWithSWR(params: PostQueryParams = {}) {
  return useQuery({
    queryKey: queryKeys.posts.list(params),
    queryFn: () => fetchPosts(params),
    staleTime: 30 * 1000,       // ข้อมูลสดอยู่ 30 วินาที
    gcTime: 5 * 60 * 1000,      // เก็บ cache 5 นาที
    refetchOnWindowFocus: true,  // Refetch เมื่อกลับมา focus
    placeholderData: (prev) => prev, // แสดงข้อมูลเก่าระหว่างโหลด
  });
}

// ใช้ keepPreviousData pattern (pagination)
function usePaginatedPosts(page: number) {
  return useQuery({
    queryKey: queryKeys.posts.list({ page }),
    queryFn: () => fetchPosts({ page }),
    placeholderData: keepPreviousData,
    staleTime: 60 * 1000,
  });
}
```

## ส่วนที่ 7: React Query กับ Suspense

### 7.1 Suspense Mode

```typescript
// hooks/usePostSuspense.ts

import { useSuspenseQuery, useSuspenseInfiniteQuery } from '@tanstack/react-query';

// ใช้ useSuspenseQuery แทน useQuery + Suspense
function usePostSuspense(slug: string) {
  return useSuspenseQuery({
    queryKey: queryKeys.posts.detail(slug),
    queryFn: () => fetchPost(slug),
  });
}

// Component
function PostDetail({ slug }: { slug: string }) {
  const { data: post } = usePostSuspense(slug);
  // ไม่ต้อง check loading/error เพราะ Suspense handle ให้

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

// Parent component ใช้ Suspense + ErrorBoundary
function PostPage({ slug }: { slug: string }) {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<PostSkeleton />}>
        <PostDetail slug={slug} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### 7.2 Multiple Suspense Boundaries

```typescript
// Parallel Suspense
function Dashboard() {
  return (
    <div className="grid grid-cols-2 gap-4">
      {/* แต่ละ section โหลดแยกกัน */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsPanel />
      </Suspense>

      <Suspense fallback={<PostsSkeleton />}>
        <RecentPostsPanel />
      </Suspense>

      <Suspense fallback={<TagsSkeleton />}>
        <TopTagsPanel />
      </Suspense>
    </div>
  );
}

function StatsPanel() {
  const { data: stats } = useSuspenseQuery({
    queryKey: ['dashboard', 'stats'],
    queryFn: fetchStats,
  });

  return (
    <div>
      <h2>สถิติ</h2>
      <p>โพสต์ทั้งหมด: {stats.totalPosts}</p>
      <p>ผู้อ่านทั้งหมด: {stats.totalReaders}</p>
    </div>
  );
}
```

## ส่วนที่ 8: Testing React Query

### 8.1 Setup Testing Environment

```typescript
// test/setup.ts

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render, RenderOptions } from '@testing-library/react';
import { ReactNode } from 'react';

function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,        // ไม่ retry ใน tests
        gcTime: Infinity,    // ไม่ garbage collect
        staleTime: 0,        // ถือว่า stale ทันที
      },
      mutations: {
        retry: false,
      },
    },
  });
}

interface WrapperProps {
  children: ReactNode;
}

function createWrapper(queryClient?: QueryClient) {
  const client = queryClient || createTestQueryClient();

  function Wrapper({ children }: WrapperProps) {
    return (
      <QueryClientProvider client={client}>
        {children}
      </QueryClientProvider>
    );
  }

  return Wrapper;
}

export function renderWithQuery(
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'> & { queryClient?: QueryClient }
) {
  const { queryClient, ...renderOptions } = options || {};
  const client = queryClient || createTestQueryClient();

  return {
    ...render(ui, {
      wrapper: createWrapper(client),
      ...renderOptions,
    }),
    queryClient: client,
  };
}
```

### 8.2 Testing Hooks

```typescript
// hooks/usePosts.test.ts

import { renderHook, waitFor } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { usePosts } from './usePosts';
import { createWrapper } from '../test/setup';

// Mock server
const server = setupServer(
  http.get('/api/posts', ({ request }) => {
    const url = new URL(request.url);
    const page = parseInt(url.searchParams.get('page') || '1');

    return HttpResponse.json({
      data: [
        {
          id: '1',
          title: 'บทความทดสอบ',
          slug: 'test-post',
          excerpt: 'เนื้อหาย่อ',
        },
      ],
      meta: {
        page,
        limit: 10,
        total: 1,
        totalPages: 1,
        hasNextPage: false,
        hasPrevPage: false,
      },
    });
  }),

  http.post('/api/posts', async ({ request }) => {
    const body = await request.json() as Record<string, unknown>;
    return HttpResponse.json(
      { id: '2', ...body },
      { status: 201 }
    );
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('usePosts', () => {
  it('should fetch posts', async () => {
    const { result } = renderHook(() => usePosts(), {
      wrapper: createWrapper(),
    });

    // Initial loading state
    expect(result.current.isLoading).toBe(true);

    // Wait for data
    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data?.data).toHaveLength(1);
    expect(result.current.data?.data[0].title).toBe('บทความทดสอบ');
  });

  it('should handle error', async () => {
    server.use(
      http.get('/api/posts', () => {
        return HttpResponse.json(
          { error: 'Server Error' },
          { status: 500 }
        );
      })
    );

    const { result } = renderHook(() => usePosts(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => expect(result.current.isError).toBe(true));
  });
});
```

### 8.3 Testing Components

```typescript
// components/PostCard.test.tsx

import { screen, fireEvent, waitFor } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { PostCard } from './PostCard';
import { renderWithQuery } from '../test/setup';

const mockPost: Post = {
  id: '1',
  title: 'บทความทดสอบ TypeScript',
  slug: 'test-typescript-post',
  excerpt: 'นี่คือเนื้อหาย่อของบทความ',
  content: '<p>เนื้อหาเต็ม</p>',
  status: 'published',
  likesCount: 42,
  commentsCount: 8,
  viewsCount: 1000,
  readingTime: 5,
  isLiked: false,
  author: { id: '1', username: 'author', displayName: 'ผู้เขียน' },
  category: { id: '1', name: 'TypeScript', slug: 'typescript' },
  tags: [],
  publishedAt: new Date('2024-01-15'),
  createdAt: new Date('2024-01-15'),
  updatedAt: new Date('2024-01-15'),
};

const server = setupServer(
  http.post('/api/likes/posts/:postId', ({ params }) => {
    return HttpResponse.json({
      liked: true,
      likesCount: 43,
    });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('PostCard', () => {
  it('renders post information', () => {
    renderWithQuery(<PostCard post={mockPost} />);

    expect(screen.getByText('บทความทดสอบ TypeScript')).toBeInTheDocument();
    expect(screen.getByText('นี่คือเนื้อหาย่อของบทความ')).toBeInTheDocument();
    expect(screen.getByText('42')).toBeInTheDocument(); // likes count
  });

  it('handles like click with optimistic update', async () => {
    renderWithQuery(<PostCard post={mockPost} />);

    const likeButton = screen.getByRole('button', { name: /ถูกใจ/i });
    fireEvent.click(likeButton);

    // Optimistic update ทันที
    await waitFor(() => {
      expect(screen.getByText('43')).toBeInTheDocument();
    });
  });
});
```

## ส่วนที่ 9: Prefetching Strategies

### 9.1 Prefetch on Hover

```typescript
// components/PostLink.tsx

import Link from 'next/link';
import { useQueryClient } from '@tanstack/react-query';
import { queryKeys } from '../lib/queryKeys';

interface PostLinkProps {
  post: { slug: string; title: string };
  children: React.ReactNode;
}

function PostLink({ post, children }: PostLinkProps) {
  const queryClient = useQueryClient();

  const prefetchPost = () => {
    // Prefetch เมื่อ hover
    queryClient.prefetchQuery({
      queryKey: queryKeys.posts.detail(post.slug),
      queryFn: () => fetchPost(post.slug),
      staleTime: 5 * 60 * 1000, // 5 นาที
    });
  };

  return (
    <Link
      href={`/posts/${post.slug}`}
      onMouseEnter={prefetchPost}
      onFocus={prefetchPost}
    >
      {children}
    </Link>
  );
}

// Prefetch next page
function usePrefetchNextPage(
  currentPage: number,
  hasNextPage: boolean,
  params: PostQueryParams
) {
  const queryClient = useQueryClient();

  useEffect(() => {
    if (hasNextPage) {
      queryClient.prefetchQuery({
        queryKey: queryKeys.posts.list({ ...params, page: currentPage + 1 }),
        queryFn: () => fetchPosts({ ...params, page: currentPage + 1 }),
        staleTime: 30 * 1000,
      });
    }
  }, [currentPage, hasNextPage, queryClient, params]);
}
```

### 9.2 Server-side Prefetching (Next.js)

```typescript
// app/posts/page.tsx

import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';
import { queryKeys } from '@/lib/queryKeys';
import { PostsList } from '@/components/PostsList';

export default async function PostsPage() {
  const queryClient = new QueryClient();

  // Prefetch ข้อมูลบน server
  await queryClient.prefetchQuery({
    queryKey: queryKeys.posts.list({}),
    queryFn: () => fetchPosts({}),
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostsList />
    </HydrationBoundary>
  );
}

// app/posts/[slug]/page.tsx
export default async function PostPage({ params }: { params: { slug: string } }) {
  const queryClient = new QueryClient();

  await Promise.all([
    queryClient.prefetchQuery({
      queryKey: queryKeys.posts.detail(params.slug),
      queryFn: () => fetchPost(params.slug),
    }),
    queryClient.prefetchQuery({
      queryKey: queryKeys.comments.byPost(params.slug),
      queryFn: () => fetchComments(params.slug),
    }),
  ]);

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostDetail slug={params.slug} />
      <CommentsList postSlug={params.slug} />
    </HydrationBoundary>
  );
}
```

## ส่วนที่ 10: Custom Query Hooks Patterns

### 10.1 Factory Hook Pattern

```typescript
// hooks/createResourceHooks.ts

import {
  useQuery,
  useMutation,
  useQueryClient,
  UseQueryOptions,
} from '@tanstack/react-query';

interface ResourceConfig<T, CreateDto, UpdateDto> {
  resource: string;
  fetchAll: (params?: Record<string, unknown>) => Promise<T[]>;
  fetchById: (id: string) => Promise<T>;
  create: (data: CreateDto) => Promise<T>;
  update: (id: string, data: UpdateDto) => Promise<T>;
  remove: (id: string) => Promise<void>;
}

function createResourceHooks<T, CreateDto, UpdateDto>(
  config: ResourceConfig<T, CreateDto, UpdateDto>
) {
  const keys = {
    all: [config.resource] as const,
    lists: () => [...keys.all, 'list'] as const,
    list: (params?: Record<string, unknown>) =>
      [...keys.lists(), params] as const,
    detail: (id: string) => [...keys.all, 'detail', id] as const,
  };

  function useList(
    params?: Record<string, unknown>,
    options?: Omit<UseQueryOptions, 'queryKey' | 'queryFn'>
  ) {
    return useQuery({
      queryKey: keys.list(params),
      queryFn: () => config.fetchAll(params),
      ...options,
    });
  }

  function useDetail(
    id: string,
    options?: Omit<UseQueryOptions, 'queryKey' | 'queryFn'>
  ) {
    return useQuery({
      queryKey: keys.detail(id),
      queryFn: () => config.fetchById(id),
      enabled: !!id,
      ...options,
    });
  }

  function useCreate() {
    const queryClient = useQueryClient();

    return useMutation({
      mutationFn: config.create,
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: keys.lists() });
      },
    });
  }

  function useUpdate() {
    const queryClient = useQueryClient();

    return useMutation({
      mutationFn: ({ id, data }: { id: string; data: UpdateDto }) =>
        config.update(id, data),
      onSuccess: (data, { id }) => {
        queryClient.invalidateQueries({ queryKey: keys.lists() });
        queryClient.setQueryData(keys.detail(id), data);
      },
    });
  }

  function useRemove() {
    const queryClient = useQueryClient();

    return useMutation({
      mutationFn: config.remove,
      onSuccess: (_, id) => {
        queryClient.invalidateQueries({ queryKey: keys.lists() });
        queryClient.removeQueries({ queryKey: keys.detail(id) });
      },
    });
  }

  return { useList, useDetail, useCreate, useUpdate, useRemove, keys };
}

// สร้าง hooks สำหรับ resources
export const postHooks = createResourceHooks<Post, CreatePostDto, UpdatePostDto>({
  resource: 'posts',
  fetchAll: fetchPosts,
  fetchById: (id) => fetchPost(id),
  create: createPost,
  update: updatePost,
  remove: deletePost,
});

export const userHooks = createResourceHooks<User, CreateUserDto, UpdateUserDto>({
  resource: 'users',
  fetchAll: fetchUsers,
  fetchById: fetchUser,
  create: createUser,
  update: updateUser,
  remove: deleteUser,
});

// ใช้งาน
function PostManagement() {
  const { data: posts } = postHooks.useList();
  const createPost = postHooks.useCreate();
  const deletePost = postHooks.useRemove();

  return (
    <div>
      {posts?.map(post => (
        <div key={post.id}>
          <span>{post.title}</span>
          <button onClick={() => deletePost.mutate(post.id)}>
            ลบ
          </button>
        </div>
      ))}
    </div>
  );
}
```

## ส่วนที่ 11: Server State vs Client State

```typescript
// store/uiStore.ts - Client state ด้วย Zustand

import { create } from 'zustand';

interface UIState {
  // Client state - ไม่ต้องการ server
  sidebarOpen: boolean;
  selectedPostId: string | null;
  theme: 'light' | 'dark';
  searchQuery: string;

  // Actions
  toggleSidebar: () => void;
  selectPost: (id: string | null) => void;
  setTheme: (theme: 'light' | 'dark') => void;
  setSearchQuery: (query: string) => void;
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: false,
  selectedPostId: null,
  theme: 'light',
  searchQuery: '',

  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  selectPost: (id) => set({ selectedPostId: id }),
  setTheme: (theme) => set({ theme }),
  setSearchQuery: (query) => set({ searchQuery: query }),
}));

// การใช้งานร่วมกัน
function BlogApp() {
  // Server state - React Query
  const { data: posts } = usePosts();
  const { data: currentUser } = useCurrentUser();

  // Client state - Zustand
  const { sidebarOpen, selectedPostId, toggleSidebar } = useUIStore();

  return (
    <div className={sidebarOpen ? 'with-sidebar' : ''}>
      <button onClick={toggleSidebar}>เมนู</button>
      {posts?.data.map(post => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  );
}
```

## สรุป

React Query Patterns ที่สำคัญ:

1. **Query Key Factory** - จัดการ keys แบบ type-safe
2. **Dependent Queries** - ดึงข้อมูลแบบ sequential
3. **Parallel Queries** - ดึงข้อมูลพร้อมกัน
4. **Infinite Scroll** - ใช้ useInfiniteQuery
5. **Optimistic Updates** - อัพเดท UI ก่อน API response
6. **Prefetching** - โหลดข้อมูลล่วงหน้า
7. **Suspense Integration** - ใช้ useSuspenseQuery
8. **Testing** - ใช้ MSW + Testing Library
9. **Factory Pattern** - สร้าง reusable hooks
10. **Server vs Client State** - แยกประเภท state ให้ชัดเจน
