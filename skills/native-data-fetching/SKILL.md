---
name: native-data-fetching
description: Use when implementing network requests, API calls, data caching, or offline support in React Native or Expo apps, including authentication token management and request lifecycle handling
---

# Data Fetching in React Native

## Overview

React Native uses the standard Fetch API but requires additional patterns for production apps: caching, mutation handling, authentication, offline support, and request cancellation. TanStack Query (React Query) is the standard solution for managing server state.

**Core principle:** Separate server state (fetched data) from client state (UI state). Use React Query for server state and React state/context for client state.

## When to Use

- Making HTTP requests to REST or GraphQL APIs
- Caching API responses to avoid redundant network calls
- Handling optimistic updates and mutations
- Managing authentication tokens securely
- Supporting offline-first or intermittent connectivity
- Cancelling requests on component unmount or navigation

## Fetch API in React Native

### Basic Request

```tsx
async function fetchUser(id: string) {
  const response = await fetch(`https://api.example.com/users/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  return response.json();
}
```

### POST with JSON Body

```tsx
async function createPost(title: string, body: string, token: string) {
  const response = await fetch('https://api.example.com/posts', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ title, body }),
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message ?? 'Failed to create post');
  }

  return response.json();
}
```

### File Upload

```tsx
async function uploadImage(uri: string, token: string) {
  const formData = new FormData();
  formData.append('image', {
    uri,
    type: 'image/jpeg',
    name: 'photo.jpg',
  } as any);

  const response = await fetch('https://api.example.com/upload', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      // Do NOT set Content-Type; fetch sets it with boundary for FormData
    },
    body: formData,
  });

  return response.json();
}
```

## React Query (TanStack Query)

### Setup

```bash
npx expo install @tanstack/react-query
```

```tsx
// app/_layout.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // 5 minutes
      gcTime: 10 * 60 * 1000,       // 10 minutes (formerly cacheTime)
      retry: 2,
      refetchOnWindowFocus: false,   // Not relevant on mobile
    },
  },
});

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <Stack />
    </QueryClientProvider>
  );
}
```

### Queries

```tsx
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <ActivityIndicator />;
  if (error) return <Text>Error: {error.message}</Text>;

  return (
    <View>
      <Text>{data.name}</Text>
      <Text>{data.email}</Text>
    </View>
  );
}
```

### Dependent Queries

```tsx
function UserPosts({ userId }: { userId: string }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  const postsQuery = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchPostsByUser(userId),
    enabled: !!userQuery.data,  // Only fetch when user is loaded
  });

  // ...
}
```

### Infinite Queries (Pagination)

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';
import { FlatList } from 'react-native';

function InfiniteFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) => fetchFeed(pageParam),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined,
  });

  const items = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <FlatList
      data={items}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <FeedItem item={item} />}
      onEndReached={() => {
        if (hasNextPage) fetchNextPage();
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  );
}
```

### Mutations

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePostForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newPost: { title: string; body: string }) =>
      createPost(newPost.title, newPost.body),

    onSuccess: () => {
      // Invalidate and refetch posts list
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  const handleSubmit = () => {
    mutation.mutate({ title, body });
  };

  return (
    <View>
      {/* Form fields... */}
      <Pressable
        onPress={handleSubmit}
        disabled={mutation.isPending}
      >
        <Text>{mutation.isPending ? 'Posting...' : 'Submit'}</Text>
      </Pressable>
      {mutation.isError && (
        <Text style={{ color: 'red' }}>{mutation.error.message}</Text>
      )}
    </View>
  );
}
```

### Optimistic Updates

```tsx
const likeMutation = useMutation({
  mutationFn: (postId: string) => likePost(postId),

  onMutate: async (postId) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['posts'] });

    // Snapshot previous value
    const previousPosts = queryClient.getQueryData(['posts']);

    // Optimistically update
    queryClient.setQueryData(['posts'], (old: Post[]) =>
      old.map((post) =>
        post.id === postId
          ? { ...post, likes: post.likes + 1, liked: true }
          : post
      )
    );

    return { previousPosts };
  },

  onError: (_err, _postId, context) => {
    // Rollback on error
    queryClient.setQueryData(['posts'], context?.previousPosts);
  },

  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['posts'] });
  },
});
```

## Authentication with expo-secure-store

Store tokens securely using the device keychain (iOS) or EncryptedSharedPreferences (Android):

```bash
npx expo install expo-secure-store
```

```tsx
// lib/auth.ts
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

export async function saveTokens(accessToken: string, refreshToken: string) {
  await SecureStore.setItemAsync(TOKEN_KEY, accessToken);
  await SecureStore.setItemAsync(REFRESH_TOKEN_KEY, refreshToken);
}

export async function getAccessToken(): Promise<string | null> {
  return SecureStore.getItemAsync(TOKEN_KEY);
}

export async function clearTokens() {
  await SecureStore.deleteItemAsync(TOKEN_KEY);
  await SecureStore.deleteItemAsync(REFRESH_TOKEN_KEY);
}
```

### Authenticated Fetch Wrapper

```tsx
// lib/api.ts
import { getAccessToken, saveTokens, clearTokens } from './auth';

const BASE_URL = process.env.EXPO_PUBLIC_API_URL;

export async function apiFetch(path: string, options: RequestInit = {}) {
  const token = await getAccessToken();

  const response = await fetch(`${BASE_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  // Handle token expiry
  if (response.status === 401) {
    const refreshed = await refreshAccessToken();
    if (refreshed) {
      // Retry with new token
      return apiFetch(path, options);
    }
    // Refresh failed, clear tokens and redirect to login
    await clearTokens();
    throw new AuthError('Session expired');
  }

  if (!response.ok) {
    throw new ApiError(response.status, await response.text());
  }

  return response.json();
}
```

## Offline Support with NetInfo

```bash
npx expo install @react-native-community/netinfo
```

### Online Status Hook

```tsx
import NetInfo from '@react-native-community/netinfo';
import { onlineManager } from '@tanstack/react-query';

// Tell React Query about network status
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});
```

### Offline-Aware Component

```tsx
import { useNetInfo } from '@react-native-community/netinfo';

function DataScreen() {
  const netInfo = useNetInfo();
  const { data, isLoading } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems,
  });

  if (!netInfo.isConnected && !data) {
    return (
      <View style={styles.offline}>
        <Text>No internet connection</Text>
        <Text>Previously cached data unavailable</Text>
      </View>
    );
  }

  // ...
}
```

## Environment Configuration

```json
// app.json
{
  "expo": {
    "extra": {
      "eas": {
        "projectId": "your-project-id"
      }
    }
  }
}
```

Use `EXPO_PUBLIC_` prefix for client-visible environment variables:

```bash
# .env
EXPO_PUBLIC_API_URL=https://api.example.com
EXPO_PUBLIC_SENTRY_DSN=https://sentry.io/...
```

```tsx
// Access in code
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
```

For per-environment config with EAS:

```bash
eas env:create --name EXPO_PUBLIC_API_URL --value https://api.staging.example.com --environment preview
eas env:create --name EXPO_PUBLIC_API_URL --value https://api.example.com --environment production
```

## Request Cancellation

### With AbortController

```tsx
function SearchResults({ query }: { query: string }) {
  const { data } = useQuery({
    queryKey: ['search', query],
    queryFn: ({ signal }) => {
      // signal is automatically provided by React Query
      return fetch(`/api/search?q=${query}`, { signal }).then((r) => r.json());
    },
    enabled: query.length > 2,
  });

  // Query is automatically cancelled when:
  // - Component unmounts
  // - Query key changes (new search)
  // - Query is manually cancelled
}
```

### Manual Cancellation

```tsx
const queryClient = useQueryClient();

// Cancel all queries matching a key
queryClient.cancelQueries({ queryKey: ['search'] });

// Cancel specific query
queryClient.cancelQueries({ queryKey: ['search', 'react native'] });
```

## On-Device Storage

When the project needs key-value storage, ask the user which option fits their setup:

| Option | Pros | Cons | Requires Prebuild? |
|--------|------|------|--------------------|
| **react-native-mmkv** (recommended) | Synchronous API, ~30x faster than AsyncStorage, encryption support, Zustand integration | Requires native modules — needs `npx expo prebuild` or a dev client | **Yes** |
| **AsyncStorage** | Works in Expo Go out of the box, no prebuild needed, simple async API | Asynchronous only, slower, no encryption | **No** |

> **Which should I use?** If you're already using a dev client or have run `npx expo prebuild`, use MMKV. If you need to stay in Expo Go without prebuilding, use AsyncStorage.

### react-native-mmkv

Synchronous, ~30x faster than AsyncStorage, built on WeChat's battle-tested C++ library. **Requires a prebuild** (native module via Nitro Modules).

```bash
npx expo install react-native-mmkv react-native-nitro-modules
npx expo prebuild
```

### Basic Usage

```tsx
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// Write (synchronous)
storage.set('user.name', 'Jane');
storage.set('user.age', 28);
storage.set('onboarded', true);

// Read (synchronous)
const name = storage.getString('user.name');    // 'Jane'
const age = storage.getNumber('user.age');       // 28
const onboarded = storage.getBoolean('onboarded'); // true

// Delete
storage.delete('user.name');

// Check existence
if (storage.contains('user.age')) { /* ... */ }

// Clear all
storage.clearAll();
```

### Storing Objects

```tsx
// MMKV stores primitives — serialize objects as JSON
function setObject<T>(key: string, value: T) {
  storage.set(key, JSON.stringify(value));
}

function getObject<T>(key: string): T | undefined {
  const value = storage.getString(key);
  return value ? JSON.parse(value) : undefined;
}

// Usage
setObject('user.preferences', { theme: 'dark', language: 'en' });
const prefs = getObject<{ theme: string; language: string }>('user.preferences');
```

### With Zustand (Persisted State)

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

const mmkvStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'light' as 'light' | 'dark',
      setTheme: (theme: 'light' | 'dark') => set({ theme }),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => mmkvStorage),
    },
  ),
);
```

### Encrypted Storage

```tsx
const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'your-encryption-key',
});
```

> **Note:** For authentication tokens and secrets, prefer `expo-secure-store` (uses iOS Keychain / Android EncryptedSharedPreferences). Use MMKV for app data, preferences, caches, and non-sensitive state.

### AsyncStorage

Works in Expo Go without prebuild. Simple async key-value storage.

```bash
npx expo install @react-native-async-storage/async-storage
```

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// Write (asynchronous)
await AsyncStorage.setItem('user.name', 'Jane');

// Read (asynchronous)
const name = await AsyncStorage.getItem('user.name'); // 'Jane' | null

// Store objects (must serialize manually)
await AsyncStorage.setItem('prefs', JSON.stringify({ theme: 'dark' }));
const prefs = JSON.parse((await AsyncStorage.getItem('prefs')) ?? '{}');

// Delete
await AsyncStorage.removeItem('user.name');

// Clear all
await AsyncStorage.clear();
```

#### With Zustand (Persisted State)

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'light' as 'light' | 'dark',
      setTheme: (theme: 'light' | 'dark') => set({ theme }),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => AsyncStorage),
    },
  ),
);
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Storing tokens in AsyncStorage | Use `expo-secure-store` for sensitive credentials |
| Using AsyncStorage when prebuild is available | Use `react-native-mmkv` — synchronous, ~30x faster |
| Setting `refetchOnWindowFocus: true` on mobile | Mobile apps don't have window focus events like web; set to `false` |
| Not handling 401 token expiry | Implement token refresh in your fetch wrapper |
| Fetching in `useEffect` without cleanup | Use React Query or AbortController to cancel on unmount |
| Hardcoding API URLs | Use `EXPO_PUBLIC_*` env vars with EAS environments |
| Not setting `staleTime` | Default `0` means every mount refetches; set appropriate stale time |
| Missing error boundaries | Wrap screens with error boundaries for network failures |
| Using `cacheTime` (renamed) | Use `gcTime` (garbage collection time) in v5 |

## Quick Reference

| Task | Pattern |
|------|---------|
| Basic fetch | `fetch(url).then(r => r.json())` |
| Setup React Query | `<QueryClientProvider client={queryClient}>` |
| Query data | `useQuery({ queryKey, queryFn })` |
| Mutate data | `useMutation({ mutationFn, onSuccess })` |
| Paginated list | `useInfiniteQuery` with `FlatList.onEndReached` |
| Store token | `SecureStore.setItemAsync(key, value)` |
| Network status | `NetInfo.addEventListener` + `onlineManager` |
| Cancel request | Pass `signal` from `queryFn` to `fetch` |
| Invalidate cache | `queryClient.invalidateQueries({ queryKey })` |
| Local storage | `MMKV` from `react-native-mmkv` (not AsyncStorage) |
| Env variable | `process.env.EXPO_PUBLIC_API_URL` |
