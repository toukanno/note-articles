# TypeScriptのデザインパターン

## はじめに

デザインパターンは、ソフトウェア設計における共通の課題に対する再利用可能な解決策です。TypeScriptの型システムを活用することで、これらのパターンをより安全かつ表現力豊かに実装できます。

## Builder パターン

複雑なオブジェクトを段階的に構築するパターンです。

```typescript
interface QueryConfig {
  table: string;
  select: string[];
  where: Record<string, unknown>;
  orderBy: { column: string; direction: "ASC" | "DESC" };
  limit: number;
  offset: number;
}

class QueryBuilder {
  private config: Partial<QueryConfig> = {};

  from(table: string): this {
    this.config.table = table;
    return this;
  }

  select(...columns: string[]): this {
    this.config.select = columns;
    return this;
  }

  where(conditions: Record<string, unknown>): this {
    this.config.where = { ...this.config.where, ...conditions };
    return this;
  }

  orderBy(column: string, direction: "ASC" | "DESC" = "ASC"): this {
    this.config.orderBy = { column, direction };
    return this;
  }

  limit(count: number): this {
    this.config.limit = count;
    return this;
  }

  offset(count: number): this {
    this.config.offset = count;
    return this;
  }

  build(): QueryConfig {
    if (!this.config.table) {
      throw new Error("Table is required");
    }
    return {
      table: this.config.table,
      select: this.config.select ?? ["*"],
      where: this.config.where ?? {},
      orderBy: this.config.orderBy ?? { column: "id", direction: "ASC" },
      limit: this.config.limit ?? 100,
      offset: this.config.offset ?? 0,
    };
  }
}

// 使用例
const query = new QueryBuilder()
  .from("users")
  .select("id", "name", "email")
  .where({ active: true })
  .orderBy("name", "ASC")
  .limit(20)
  .build();
```

### 型安全なBuilder（必須フィールドの保証）

```typescript
class TypedQueryBuilder<HasTable extends boolean = false> {
  private table?: string;
  private columns: string[] = ["*"];

  from(table: string): TypedQueryBuilder<true> {
    this.table = table;
    return this as unknown as TypedQueryBuilder<true>;
  }

  select(...columns: string[]): this {
    this.columns = columns;
    return this;
  }

  build(this: TypedQueryBuilder<true>): { table: string; columns: string[] } {
    return { table: this.table!, columns: this.columns };
  }
}

const builder = new TypedQueryBuilder();
// builder.build(); // エラー！from() が呼ばれていない
const query = builder.from("users").select("id", "name").build(); // OK
```

## Strategy パターン

アルゴリズムをカプセル化して交換可能にするパターンです。

```typescript
interface SortStrategy<T> {
  sort(items: T[]): T[];
}

class BubbleSort<T> implements SortStrategy<T> {
  constructor(private compare: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    const arr = [...items];
    for (let i = 0; i < arr.length; i++) {
      for (let j = 0; j < arr.length - i - 1; j++) {
        if (this.compare(arr[j], arr[j + 1]) > 0) {
          [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        }
      }
    }
    return arr;
  }
}

class QuickSort<T> implements SortStrategy<T> {
  constructor(private compare: (a: T, b: T) => number) {}

  sort(items: T[]): T[] {
    return [...items].sort(this.compare);
  }
}

class Sorter<T> {
  constructor(private strategy: SortStrategy<T>) {}

  setStrategy(strategy: SortStrategy<T>): void {
    this.strategy = strategy;
  }

  sort(items: T[]): T[] {
    return this.strategy.sort(items);
  }
}

// 使用例
const compare = (a: number, b: number) => a - b;
const sorter = new Sorter(new QuickSort(compare));
const sorted = sorter.sort([3, 1, 4, 1, 5, 9]);
```

### 関数ベースのStrategy

TypeScriptではクラスを使わず、関数で簡潔に表現することもできます。

```typescript
type PricingStrategy = (basePrice: number, quantity: number) => number;

const regularPricing: PricingStrategy = (price, qty) => price * qty;

const bulkPricing: PricingStrategy = (price, qty) => {
  const discount = qty >= 10 ? 0.1 : qty >= 5 ? 0.05 : 0;
  return price * qty * (1 - discount);
};

const premiumPricing: PricingStrategy = (price, qty) => price * qty * 0.8;

function calculateTotal(
  items: { price: number; quantity: number }[],
  strategy: PricingStrategy
): number {
  return items.reduce(
    (total, item) => total + strategy(item.price, item.quantity),
    0
  );
}
```

## Observer パターン

状態の変化を購読者に通知するパターンです。

```typescript
type Listener<T> = (data: T) => void;
type Unsubscribe = () => void;

class EventEmitter<EventMap extends Record<string, unknown>> {
  private listeners = new Map<keyof EventMap, Set<Listener<any>>>();

  on<K extends keyof EventMap>(event: K, listener: Listener<EventMap[K]>): Unsubscribe {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener);

    return () => {
      this.listeners.get(event)?.delete(listener);
    };
  }

  emit<K extends keyof EventMap>(event: K, data: EventMap[K]): void {
    this.listeners.get(event)?.forEach((listener) => listener(data));
  }
}

// 使用例
interface StoreEvents {
  userLoggedIn: { userId: string; timestamp: number };
  userLoggedOut: { userId: string };
  cartUpdated: { items: string[]; total: number };
}

const store = new EventEmitter<StoreEvents>();

const unsubscribe = store.on("userLoggedIn", (data) => {
  console.log(`User ${data.userId} logged in at ${data.timestamp}`);
});

store.emit("userLoggedIn", { userId: "123", timestamp: Date.now() });
unsubscribe(); // 購読解除
```

## Repository パターン

データアクセスのロジックを抽象化するパターンです。

```typescript
interface Entity {
  id: string;
}

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, "id">): Promise<T>;
  update(id: string, data: Partial<Omit<T, "id">>): Promise<T>;
  delete(id: string): Promise<void>;
}

interface User extends Entity {
  name: string;
  email: string;
  role: "admin" | "user";
}

// インメモリ実装
class InMemoryUserRepository implements Repository<User> {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findAll(filter?: Partial<User>): Promise<User[]> {
    let results = Array.from(this.users.values());
    if (filter) {
      results = results.filter((user) =>
        Object.entries(filter).every(
          ([key, value]) => user[key as keyof User] === value
        )
      );
    }
    return results;
  }

  async create(data: Omit<User, "id">): Promise<User> {
    const user: User = { id: crypto.randomUUID(), ...data };
    this.users.set(user.id, user);
    return user;
  }

  async update(id: string, data: Partial<Omit<User, "id">>): Promise<User> {
    const user = this.users.get(id);
    if (!user) throw new Error(`User ${id} not found`);
    const updated = { ...user, ...data };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<void> {
    this.users.delete(id);
  }
}
```

## State パターン

オブジェクトの内部状態によって振る舞いを変えるパターンです。

```typescript
interface OrderState {
  readonly name: string;
  next(order: Order): void;
  cancel(order: Order): void;
}

class PendingState implements OrderState {
  readonly name = "pending";

  next(order: Order): void {
    order.setState(new ProcessingState());
  }

  cancel(order: Order): void {
    order.setState(new CancelledState());
  }
}

class ProcessingState implements OrderState {
  readonly name = "processing";

  next(order: Order): void {
    order.setState(new ShippedState());
  }

  cancel(order: Order): void {
    order.setState(new CancelledState());
  }
}

class ShippedState implements OrderState {
  readonly name = "shipped";

  next(order: Order): void {
    order.setState(new DeliveredState());
  }

  cancel(_order: Order): void {
    throw new Error("配送中の注文はキャンセルできません");
  }
}

class DeliveredState implements OrderState {
  readonly name = "delivered";

  next(_order: Order): void {
    throw new Error("配送済みの注文です");
  }

  cancel(_order: Order): void {
    throw new Error("配送済みの注文はキャンセルできません");
  }
}

class CancelledState implements OrderState {
  readonly name = "cancelled";

  next(_order: Order): void {
    throw new Error("キャンセル済みの注文です");
  }

  cancel(_order: Order): void {
    throw new Error("既にキャンセル済みです");
  }
}

class Order {
  private state: OrderState = new PendingState();

  setState(state: OrderState): void {
    console.log(`${this.state.name} → ${state.name}`);
    this.state = state;
  }

  next(): void {
    this.state.next(this);
  }

  cancel(): void {
    this.state.cancel(this);
  }

  get currentState(): string {
    return this.state.name;
  }
}

// 使用例
const order = new Order();
order.next();   // pending → processing
order.next();   // processing → shipped
order.next();   // shipped → delivered
```

## Dependency Injection（DI）パターン

依存関係を外部から注入するパターンです。

```typescript
// インターフェースで依存を定義
interface Logger {
  info(message: string): void;
  error(message: string): void;
}

interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

interface EmailService {
  send(to: string, subject: string, body: string): Promise<void>;
}

// 実装は外部から注入
class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService,
    private readonly logger: Logger
  ) {}

  async registerUser(name: string, email: string): Promise<User> {
    this.logger.info(`Registering user: ${email}`);

    const user: User = { id: crypto.randomUUID(), name, email, role: "user" };
    await this.userRepo.save(user);
    await this.emailService.send(
      email,
      "ようこそ",
      `${name}さん、登録ありがとうございます。`
    );

    return user;
  }
}

// テスト時にモックを注入できる
const mockLogger: Logger = {
  info: () => {},
  error: () => {},
};
```

## まとめ

- Builder パターン: 型パラメータで必須フィールドの保証が可能
- Strategy パターン: 関数型で簡潔に表現できる
- Observer パターン: ジェネリクスで型安全なイベントシステム
- Repository パターン: ジェネリックインターフェースでデータアクセスを抽象化
- State パターン: インターフェースで状態ごとの振る舞いを定義
- DI パターン: インターフェースでテスタブルなコードを実現

次回は最終回「TypeScript 5.x の最新機能」について詳しく解説します。
