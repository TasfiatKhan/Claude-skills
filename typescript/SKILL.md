---
name: typescript
description: Type API responses, props, navigation params, hooks, and context in TypeScript. Use when the user says "add types", "fix a TypeScript error", "type this response", or "what type should this be".
---

You are writing TypeScript types for a React Native + backend project.

## Typing API responses

Define an interface that exactly matches what the backend returns:

```typescript
// src/types/api.ts
export type ResponseOptionType = 'safe' | 'playful' | 'bold'

export interface ResponseOption {
  type: ResponseOptionType
  text: string
  note: string
}

export interface AIResponse {
  options: ResponseOption[]
  delivery: string
  record_id: number
}

// Usage in hook
const [response, setResponse] = useState<AIResponse | null>(null)

// After axios call
const data = res.data as AIResponse
```

## Typing component props

```typescript
interface ChipProps {
  label: string
  selected: boolean
  onPress: () => void
  color?: string           // optional — ? means undefined is fine
}

export default function Chip({ label, selected, onPress, color }: ChipProps) {
  // ...
}
```

## Typing navigation params

```typescript
// src/navigation/types.ts
export type MainStackParamList = {
  Home: undefined                            // no params
  Dashboard: undefined
  DetailView: { itemId: number | null }      // optional ID
  Settings: undefined
}

// In a screen component
import { NativeStackScreenProps } from '@react-navigation/native-stack'

type Props = NativeStackScreenProps<MainStackParamList, 'DetailView'>

export default function DetailScreen({ navigation, route }: Props) {
  const { itemId } = route.params   // typed — itemId is number | null
  navigation.navigate('Home')       // typed — only valid screen names accepted
}
```

## Typing context

```typescript
interface AuthContextValue {
  isAuthenticated: boolean
  isLoading: boolean
  signIn: (email: string, password: string) => Promise<void>
  signOut: () => Promise<void>
}

const AuthContext = createContext<AuthContextValue>({
  isAuthenticated: false,
  isLoading: true,
  signIn: async () => {},
  signOut: async () => {},
})
```

## Typing hooks

```typescript
// Return type annotation keeps the hook's interface clear
export function useProfile(): {
  profile: Profile | null
  isLoading: boolean
  update: (data: ProfileUpdate) => Promise<void>
} {
  const [profile, setProfile] = useState<Profile | null>(null)
  // ...
  return { profile, isLoading, update }
}
```

## Typing service functions

```typescript
// src/services/itemService.ts
export async function listItems(filter?: string): Promise<Item[]> {
  const res = await api.get<Item[]>('/api/items/', {
    params: { filter },
  })
  return res.data
}

export async function submitForm(
  id: number,
  payload: FormData | { text: string; context?: string }
): Promise<SubmitResponse> {
  const isFormData = payload instanceof FormData
  const res = await api.post<SubmitResponse>(
    `/api/items/${id}/submit/`,
    payload,
    isFormData ? { headers: { 'Content-Type': 'multipart/form-data' } } : undefined
  )
  return res.data
}
```

## Common patterns

```typescript
// Record — keyed object, useful for per-item state
const [itemFeedback, setItemFeedback] = useState<Record<number, string | null>>({})
const [expanded, setExpanded] = useState<Record<number, boolean>>({})

// Union types for state machines
type FormState = 'idle' | 'loading' | 'success' | 'error'
const [state, setState] = useState<FormState>('idle')

// Optional chaining + nullish coalescing
const name = user?.profile?.username ?? user?.email ?? 'Unknown'

// Type assertion (use sparingly — only when you know better than TS)
const data = res.data as AIResponse

// Discriminated union
type Option =
  | { type: 'safe'; text: string }
  | { type: 'playful'; text: string; tone: string }
```

## Type file organisation

```
src/types/
  api.ts        ← API response shapes (match backend exactly)
  models.ts     ← domain models (Profile, Item, etc.)
  navigation.ts ← stack param lists
```

One file per concern. Shared types imported across hooks, services, and screens.

## Common mistakes

- `any` everywhere — defeats the purpose. Use `unknown` if you genuinely don't know, then narrow.
- Not typing `useState` initial value — `useState(null)` gives `null` type, not `Profile | null`
- Forgetting `undefined` in navigation params — `DetailView: { itemId: number }` breaks when you navigate without params
- `as any` on FormData — sometimes needed for React Native's audio file object but document why
- Defining types inline in components — put shared types in `src/types/` and import them
