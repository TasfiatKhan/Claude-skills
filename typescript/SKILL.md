---
name: typescript
description: Type API responses, props, navigation params, hooks, and context in TypeScript. Use when the user says "add types", "fix a TypeScript error", "type this response", or "what type should this be".
---

You are writing TypeScript types for a React Native + Django project.

## Typing API responses

Define an interface that exactly matches what the backend returns:

```typescript
// src/types/humor.ts
export type AIOptionType = 'safe' | 'playful' | 'bold'

export interface AIOption {
  type: AIOptionType
  text: string
  note: string
}

export interface AIResponse {
  options: AIOption[]
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
  Home: undefined                              // no params
  TextingMode: undefined
  MomentDetail: { momentId: number | null }   // optional ID
  SavedResponses: undefined
}

// In a screen component
import { NativeStackScreenProps } from '@react-navigation/native-stack'

type Props = NativeStackScreenProps<MainStackParamList, 'MomentDetail'>

export default function MomentDetailScreen({ navigation, route }: Props) {
  const { momentId } = route.params   // typed — momentId is number | null
  navigation.navigate('Home')          // typed — only valid screen names accepted
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
// src/services/momentsService.ts
export async function listMoments(archived: boolean): Promise<Moment[]> {
  const res = await api.get<Moment[]>('/api/moments/', {
    params: { archived },
  })
  return res.data
}

export async function continueMoment(
  momentId: number,
  payload: FormData | { new_input: string; environment?: string }
): Promise<MomentContinueResponse> {
  const isFormData = payload instanceof FormData
  const res = await api.post<MomentContinueResponse>(
    `/api/moments/${momentId}/continue/`,
    payload,
    isFormData ? { headers: { 'Content-Type': 'multipart/form-data' } } : undefined
  )
  return res.data
}
```

## Common patterns

```typescript
// Record — keyed object, useful for per-item state
const [msgFeedback, setMsgFeedback] = useState<Record<number, string | null>>({})
const [deliveryExpanded, setDeliveryExpanded] = useState<Record<number, boolean>>({})

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

## Witly type files

```
src/types/
  humor.ts      ← AIOption, AIOptionType, AIResponse
  profile.ts    ← Profile, ProfileUpdate, HUMOR_STYLES
  moments.ts    ← Moment, MomentMessage, MomentContinueResponse
```

## Common mistakes

- `any` everywhere — defeats the purpose. Use `unknown` if you genuinely don't know, then narrow.
- Not typing `useState` initial value — `useState(null)` gives `null` type, not `Profile | null`
- Forgetting `undefined` in navigation params — `MomentDetail: { momentId: number }` breaks when you navigate without params
- `as any` on FormData — sometimes needed for React Native's audio file object but document why
- Defining types inline in components — put shared types in `src/types/` and import them
