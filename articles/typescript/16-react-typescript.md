# React × TypeScript 実践ガイド

## はじめに

ReactとTypeScriptの組み合わせは、現代のフロントエンド開発において事実上のスタンダードです。コンポーネントのpropsやstateに型をつけることで、バグの早期発見と優れた開発体験を実現できます。

## コンポーネントの型付け

### 関数コンポーネント

```typescript
// propsの型を定義
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: "primary" | "secondary" | "danger";
}

// 関数コンポーネント
function Button({ label, onClick, disabled = false, variant = "primary" }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {label}
    </button>
  );
}
```

### children を含むコンポーネント

```typescript
interface CardProps {
  title: string;
  children: React.ReactNode;
}

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

// 使用例
<Card title="ユーザー情報">
  <p>名前：太郎</p>
  <p>年齢：25</p>
</Card>
```

### よく使うReactの型

```typescript
// children の型
React.ReactNode      // 何でもレンダリング可能な型（最も一般的）
React.ReactElement   // JSX要素のみ
React.FC             // 関数コンポーネント型（非推奨気味）

// イベントの型
React.MouseEvent<HTMLButtonElement>
React.ChangeEvent<HTMLInputElement>
React.FormEvent<HTMLFormElement>
React.KeyboardEvent<HTMLInputElement>

// スタイルの型
React.CSSProperties

// Ref の型
React.RefObject<HTMLDivElement>
```

## イベントハンドラーの型付け

### よくあるイベント

```typescript
function Form() {
  // input の onChange
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  // form の onSubmit
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // フォーム送信処理
  };

  // button の onClick
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    console.log("clicked", e.clientX, e.clientY);
  };

  // select の onChange
  const handleSelect = (e: React.ChangeEvent<HTMLSelectElement>) => {
    console.log(e.target.value);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" onChange={handleChange} />
      <select onChange={handleSelect}>
        <option value="a">A</option>
        <option value="b">B</option>
      </select>
      <button onClick={handleClick}>送信</button>
    </form>
  );
}
```

## Hooks の型付け

### useState

```typescript
// 型推論に任せる（推奨）
const [count, setCount] = useState(0);           // number
const [name, setName] = useState("太郎");         // string

// 明示的に型を指定
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<string[]>([]);

// リテラルユニオン
const [status, setStatus] = useState<"idle" | "loading" | "error">("idle");
```

### useRef

```typescript
// DOM要素の参照
const inputRef = useRef<HTMLInputElement>(null);
const divRef = useRef<HTMLDivElement>(null);

function FocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleClick = () => {
    inputRef.current?.focus();
  };

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>フォーカス</button>
    </>
  );
}

// ミュータブルな値の保持
const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
```

### useReducer

```typescript
interface State {
  count: number;
  error: string | null;
}

type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset" }
  | { type: "setError"; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "increment":
      return { ...state, count: state.count + 1 };
    case "decrement":
      return { ...state, count: state.count - 1 };
    case "reset":
      return { count: 0, error: null };
    case "setError":
      return { ...state, error: action.payload };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, error: null });

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
    </div>
  );
}
```

### useContext

```typescript
interface ThemeContextType {
  theme: "light" | "dark";
  toggleTheme: () => void;
}

const ThemeContext = React.createContext<ThemeContextType | null>(null);

// カスタムフック
function useTheme(): ThemeContextType {
  const context = React.useContext(ThemeContext);
  if (!context) {
    throw new Error("useTheme must be used within ThemeProvider");
  }
  return context;
}

// Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");

  const toggleTheme = () => {
    setTheme((prev) => (prev === "light" ? "dark" : "light"));
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## ジェネリックコンポーネント

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// 使用例
interface User {
  id: number;
  name: string;
}

<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

## HTML属性の継承

コンポーネントが標準のHTML属性も受け入れるようにするパターンです。

```typescript
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
  isLoading?: boolean;
}

function Button({ variant = "primary", isLoading, children, ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      disabled={isLoading || rest.disabled}
      {...rest}
    >
      {isLoading ? "読み込み中..." : children}
    </button>
  );
}

// 標準のbutton属性がすべて使える
<Button variant="primary" type="submit" aria-label="送信">
  送信
</Button>
```

## forwardRef の型付け

```typescript
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...rest }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} {...rest} />
        {error && <span className="error">{error}</span>}
      </div>
    );
  }
);

Input.displayName = "Input";
```

## カスタムフックの型

```typescript
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await fetch(url);
      const json = await response.json();
      setData(json as T);
    } catch (e) {
      setError(e instanceof Error ? e : new Error(String(e)));
    } finally {
      setLoading(false);
    }
  }, [url]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}

// 使用例
function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error } = useFetch<User>(`/api/users/${userId}`);

  if (loading) return <p>読み込み中...</p>;
  if (error) return <p>エラー: {error.message}</p>;
  if (!user) return null;

  return <h1>{user.name}</h1>;
}
```

## まとめ

- コンポーネントのpropsはinterfaceで型定義する
- `React.ReactNode` はchildrenの型として最も一般的
- Hooksの型は多くの場合、型推論に任せられる
- `useContext` は null チェック付きのカスタムフックで包むのが安全
- ジェネリックコンポーネントで再利用性の高いUIが作れる
- `React.ButtonHTMLAttributes` 等でHTML属性を継承できる
- カスタムフックの戻り値には明示的に型をつけるとよい

次回は「TypeScriptのコンパイラオプション」について詳しく解説します。
