# Data Validation Library - API Reference

This document provides a complete API reference for the data validation library. For tutorials and usage patterns, see:
- [Getting Started Guide](./validation-library-getting-started.md) - Core concepts and basic usage
- [Intermediate Guide](./validation-library-intermediate.md) - Advanced patterns and scaling

## Table of Contents

- [ValidationFactory](#validationfactory)
- [Coercion Decorators](#coercion-decorators)
- [Validation Decorators](#validation-decorators)
- [AI Decorators](#ai-decorators)
- [Data Source Decorators](#data-source-decorators)
- [Context Decorators](#context-decorators)
- [Class-Level Decorators](#class-level-decorators)
- [Text Normalization](#text-normalization)
- [Matching Strategies](#matching-strategies)
- [Type Definitions](#type-definitions)
- [Error Types](#error-types)

---

## ValidationFactory

The central class for creating validated instances from raw data.

### Constructor

```typescript
new ValidationFactory(config?: ValidationConfig)
```

**ValidationConfig**

```typescript
interface ValidationConfig {
  aiHandler?: AIHandler;                    // Global AI transformation handler
  aiValidationHandler?: AIValidationHandler; // Global AI validation handler
  styles?: { [name: string]: ValidationStyle }; // Named validation styles
  globals?: GlobalValidationRule[];         // Global validation rules
  decoratorDefaults?: DecoratorDefaults;    // Default options for decorators
  defaultTransforms?: TypeDefaultTransforms; // CSS-like type defaults
}
```

### Methods

#### `create<T, Context>(targetClass, data, options?)`

Creates a validated instance from raw data.

```typescript
async create<T extends object, Context = any>(
  targetClass: new () => T,
  data: any,
  options?: ValidationOptions<Context>
): Promise<T>
```

**Parameters:**
- `targetClass` - The class to instantiate
- `data` - Raw input data (any shape)
- `options` - Validation options including context

**Example:**
```typescript
const factory = new ValidationFactory();
const user = await factory.create(UserRegistration, rawInput, {
  context: { allowedDomains: ['example.com'] }
});
```

#### `validate<T, Context>(instance, options?)`

Re-validates an existing instance.

```typescript
async validate<T extends object, Context = any>(
  instance: T,
  options?: ValidationOptions<Context>
): Promise<T>
```

#### `getMergedDecoratorOptions<T>(decoratorType, decoratorOptions?)`

Merges decorator options with factory defaults.

```typescript
getMergedDecoratorOptions<T>(
  decoratorType: string,
  decoratorOptions?: Partial<T>
): T
```

#### `getAIHandler(options)`

Gets the effective AI handler (per-request overrides global).

```typescript
getAIHandler(options: ValidationOptions): AIHandler | undefined
```

#### `getAIValidationHandler(options)`

Gets the effective AI validation handler.

```typescript
getAIValidationHandler(options: ValidationOptions): AIValidationHandler | undefined
```

### ValidationOptions

```typescript
interface ValidationOptions<Context = any> {
  context?: Context;                          // Runtime context data
  engine?: 'single-pass' | 'convergent';      // Validation engine (default: 'convergent')
  maxIterations?: number;                     // Max iterations for convergent engine
  aiHandler?: AIHandler;                      // Per-request AI handler
  aiValidationHandler?: AIValidationHandler;  // Per-request AI validation handler
}
```

---

## Coercion Decorators

Decorators that transform values. Execute top-to-bottom on each property.

### @Coerce(coercionFn, description?)

Generic coercion decorator. Multiple `@Coerce` decorators execute in order.

```typescript
@Coerce(coercionFn: (value: any) => T, description?: string)
```

**Example:**
```typescript
class Product {
  @Coerce((v) => v.toUpperCase(), 'Convert to uppercase')
  @Coerce((v) => v.trim(), 'Trim whitespace')
  sku: string;
}
```

### @CoerceType(targetType, options?)

Converts values to the specified type.

```typescript
@CoerceType(
  targetType: 'string' | 'number' | 'boolean' | 'date',
  options?: CoerceTypeOptions
)
```

**Boolean Options:**
```typescript
interface BooleanCoercionOptions {
  strict?: boolean;                    // Shorthand for strictness: 'strict'
  strictness?: 'strict' | 'standard'; // Default: 'standard'
  customMap?: (value: any) => boolean | undefined;
}
```

- **Standard mode** (default): Accepts true/false, 1/0, "true"/"false", "1"/"0", yes/no, y/n, on/off, t/f
- **Strict mode**: Accepts only true/false, 1/0, "true"/"false", "1"/"0"

**Date Options:**
```typescript
interface DateCoercionOptions {
  format?: 'loose' | 'iso' | 'iso-datetime' | 'iso-date' | 'timestamp' | RegExp;
  timezone?: 'utc' | 'local';    // For date-only strings
  parser?: (value: unknown) => Date;
  allowTimestamps?: boolean;     // Default: true for 'loose'
}
```

**Examples:**
```typescript
class Order {
  @CoerceType('number')
  quantity: number; // "42" -> 42

  @CoerceType('boolean', { strictness: 'strict' })
  confirmed: boolean; // "true" -> true, "yes" -> throws

  @CoerceType('date', { format: 'iso-date', timezone: 'utc' })
  orderDate: Date;
}
```

### @CoerceTrim()

Removes leading and trailing whitespace from strings.

```typescript
@CoerceTrim()
```

**Example:**
```typescript
class User {
  @CoerceTrim()
  name: string; // "  John Doe  " -> "John Doe"
}
```

### @CoerceCase(style)

Transforms string casing.

```typescript
@CoerceCase(style: CaseStyle)

type CaseStyle = 'lower' | 'upper' | 'title' | 'camel' | 'pascal' | 'snake' | 'kebab' | 'constant'
```

**Examples:**
```typescript
class Document {
  @CoerceCase('lower')
  email: string; // "JOHN@EXAMPLE.COM" -> "john@example.com"

  @CoerceCase('title')
  title: string; // "hello world" -> "Hello World"

  @CoerceCase('snake')
  fieldName: string; // "myFieldName" -> "my_field_name"

  @CoerceCase('constant')
  constant: string; // "my constant" -> "MY_CONSTANT"
}
```

### @CoerceRound(options?)

Rounds numbers to specified precision or nearest multiple.

```typescript
@CoerceRound(options?: RoundOptions)

interface RoundOptions {
  precision?: number;              // Decimal places (default: 0)
  mode?: 'round' | 'floor' | 'ceil'; // Rounding mode (default: 'round')
  toNearest?: number;              // Round to nearest multiple
}
```

**Examples:**
```typescript
class Product {
  @CoerceRound({ precision: 2 })
  price: number; // 19.999 -> 20.00

  @CoerceRound({ toNearest: 5, mode: 'ceil' })
  shippingCost: number; // 12 -> 15
}
```

### @CoerceFromSet(contextExtractor, options?)

Matches value to nearest item from a context-provided set.

```typescript
@CoerceFromSet<Context = any>(
  contextExtractor: (context: Context) => any[],
  options?: CoercionFromSetOptions
)

interface CoercionFromSetOptions {
  strategy?: MatchingStrategy;       // Default: 'fuzzy'
  caseSensitive?: boolean;           // Default: false
  fuzzyThreshold?: number;           // 0-1, default: 0.8
  customMatcher?: (value: string, candidate: string) => number;
}
```

**Example:**
```typescript
interface OrderContext {
  products: string[];
}

class Order {
  @CoerceFromSet<OrderContext>(
    (ctx) => ctx.products,
    { strategy: 'fuzzy', fuzzyThreshold: 0.7 }
  )
  product: string; // "Widgit" -> "Widget"
}

const order = await factory.create(Order, data, {
  context: { products: ['Widget', 'Gadget'] }
});
```

### @CoerceArrayElements(elementCoercion)

Applies a coercion function to each element in an array.

```typescript
@CoerceArrayElements(elementCoercion: (value: any) => any)
```

**Example:**
```typescript
class TagList {
  @CoerceArrayElements((tag) => tag.toLowerCase().trim())
  tags: string[];
}
```

---

## Validation Decorators

Decorators that validate values without transforming them (unless specified).

### @Validate(validationFn, description?, options?)

Generic validation decorator.

```typescript
@Validate<T = any>(
  validationFn: (value: T, obj: any) => boolean | string | Error,
  description?: string,
  options?: { canTransform?: boolean }
)
```

**Return Values:**
- `true` - Validation passed
- `false` - Validation failed (generic error)
- `string` - Validation failed with custom message
- `Error` - Validation failed with error object

**Example:**
```typescript
class User {
  @Validate(
    (age) => age >= 18 || 'Must be 18 or older',
    'Age validation'
  )
  age: number;
}
```

### @ValidateRequired(options?)

Ensures value is not null, undefined, or empty string. Runs **before** any coercions.

```typescript
@ValidateRequired(options?: CommonDecoratorOptions)

interface CommonDecoratorOptions {
  order?: number;        // Explicit execution order
  description?: string;  // Debugging description
}
```

**Example:**
```typescript
class Order {
  @ValidateRequired()
  orderId: string;
}
```

### @ValidateRange(min?, max?)

Validates that a number falls within a range (inclusive).

```typescript
@ValidateRange(min?: number, max?: number)
```

**Example:**
```typescript
class User {
  @ValidateRange(18, 120)
  age: number;

  @ValidateRange(0) // No upper limit
  balance: number;
}
```

### @ValidateLength(min?, max?)

Validates string or array length.

```typescript
@ValidateLength(min?: number, max?: number)
```

**Example:**
```typescript
class Post {
  @ValidateLength(1, 280)
  content: string;

  @ValidateLength(1, 10)
  tags: string[];
}
```

### @ValidatePattern(pattern, message?)

Validates string against a regular expression.

```typescript
@ValidatePattern(pattern: RegExp, message?: string)
```

**Example:**
```typescript
class User {
  @ValidatePattern(
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    'Invalid email format'
  )
  email: string;

  @ValidatePattern(/^\d{3}-\d{3}-\d{4}$/)
  phone: string;
}
```

### @ValidateAsync(validationFn, description?, options?)

Asynchronous validation for I/O operations (database lookups, API calls).

```typescript
@ValidateAsync<T = any>(
  validationFn: (value: T, obj: any) => Promise<boolean | string | Error>,
  description?: string,
  options?: { canTransform?: boolean }
)
```

**Example:**
```typescript
class User {
  @ValidateAsync(async (email, obj) => {
    const exists = await checkEmailInDatabase(email);
    return !exists || 'Email already registered';
  })
  email: string;
}
```

---

## AI Decorators

Decorators that leverage AI/LLM capabilities for transformation and validation.

### @AITransform(prompt, options?)

Transforms property value using AI. The AI output is then re-coerced through all subsequent coercion decorators.

```typescript
@AITransform<TMetadata = any>(
  prompt: PromptDefinition,
  options?: AITransformOptions<TMetadata>
)

type PromptDefinition =
  | string
  | ((params: AIHandlerParams) => string)
  | object;

interface AITransformOptions<TMetadata = any> {
  maxRetries?: number;        // Default: 2
  order?: number;
  metadata?: TMetadata;
  description?: string;
  dependsOn?: string[];       // Property dependencies
}
```

**Example:**
```typescript
const factory = new ValidationFactory({
  aiHandler: async (params, prompt) => {
    const response = await llm.complete(prompt);
    return response;
  }
});

class Order {
  @AITransform((params) =>
    `Extract the quantity as a number from: "${params.value}". Return only the number.`
  )
  @CoerceType('number')
  @ValidateRange(1, 1000)
  quantity: number;
}
```

**Automatic Retry:** If the AI output fails subsequent validation, the library automatically retries with the validation error as context.

### @AIValidate(prompt, options?)

Validates property value using AI without modifying it.

```typescript
@AIValidate<TMetadata = any>(
  prompt: PromptDefinition,
  options?: AIValidateOptions<TMetadata>
)

interface AIValidateOptions<TMetadata = any> {
  maxRetries?: number;        // Default: 1
  order?: number;
  metadata?: TMetadata;
  description?: string;
  dependsOn?: string[];
}
```

**Example:**
```typescript
class Review {
  @AIValidate((params) =>
    `Does this review contain inappropriate content? "${params.value}". Answer "valid" or explain the issue.`
  )
  content: string;
}
```

### AIHandlerParams

Parameters passed to AI handler functions.

```typescript
interface AIHandlerParams<TValue, TContext, TMetadata, TInstance> {
  value: TValue;                    // Current value
  instance: TInstance;              // Partial instance being built
  context: TContext;                // Runtime context
  propertyKey: string;              // Property name
  className: string;                // Class name
  previousError?: ValidationError; // Error from previous attempt
  attemptNumber: number;            // Current retry attempt (1-based)
  maxRetries: number;               // Maximum retries configured
  metadata: TMetadata;              // Custom metadata
  schema?: any;                     // Zod schema if available
}
```

---

## Data Source Decorators

Decorators that control where property values come from.

### @Copy(options?)

Copies property value directly from raw input data.

```typescript
@Copy(options?: CommonDecoratorOptions)
```

**Example:**
```typescript
class User {
  @Copy()
  id: string;
}
```

### @DerivedFrom(source, deriveFn?, options?)

Derives property value from one or more source properties. Supports JSONPath expressions.

```typescript
@DerivedFrom(
  source: string | string[],
  deriveFn?: (source: any, ctx: { raw: any; instance: any }) => any,
  options?: CommonDecoratorOptions
)
```

**Simple JSONPath:**
```typescript
class Order {
  @DerivedFrom('$.order_info.details.id')
  orderId: string;

  @DerivedFrom('$.customer.contact.email')
  customerEmail: string;

  @DerivedFrom('$.items[0].sku')
  firstSku: string;
}
```

**Fallback Paths:**
```typescript
class Order {
  @DerivedFrom(['$.order_id', '$.orderId', '$.id'])
  orderId: string; // First non-undefined value wins
}
```

**Custom Derivation Function:**
```typescript
class Order {
  @DerivedFrom('sku', (sku, ctx) => {
    return ctx.context.productPrices[sku] || 0;
  })
  basePrice: number;
}
```

### @RenameFrom(sourceKey, options?)

Simple property rename. Alias for `@DerivedFrom`.

```typescript
@RenameFrom(sourceKey: string, options?: { order?: number })
```

**Example:**
```typescript
class User {
  @RenameFrom('user_name')
  username: string;
}
```

### @JSONPath(path, options?)

Extracts value from raw input using JSONPath expression.

```typescript
@JSONPath(path: string, options?: JSONPathOptions)

interface JSONPathOptions {
  resultType?: 'value' | 'array'; // Default: 'value'
}
```

**Example:**
```typescript
class Order {
  @JSONPath('$.items[*].quantity', { resultType: 'array' })
  allQuantities: number[];
}
```

### @CollectProperties(options)

Collects properties from multiple JSONPath sources. Automatically excludes properties that have decorators on the current class.

```typescript
@CollectProperties(options: CollectPropertiesOptions)

interface CollectPropertiesOptions {
  sources: Array<{
    path: string;           // JSONPath expression
    exclude?: string[];     // Properties to exclude
  }>;
  includeDefinedProperties?: boolean; // Default: false
  transformFn?: (collected: any) => any;
  description?: string;
  order?: number;
}
```

**Example:**
```typescript
class Order {
  @DerivedFrom('$.order_info.details.id')
  orderId: string;

  @CollectProperties({ sources: [{ path: '$' }] })
  metadata: Record<string, any>; // Everything except orderId
}
```

### @Merge(options)

Merges values from other properties into this one. Runs after other value-populating decorators.

```typescript
@Merge(options: MergeOptions)

interface MergeOptions {
  sources: string[];                      // Property keys to merge from
  mergeFunction?: (values: any[]) => any; // Custom merge logic
}
```

**Default Merge Strategies:**
- Objects: `Object.assign()`
- Arrays: `concat()`
- Strings: `join(' ')`
- Primitives: Last non-undefined value

**Example:**
```typescript
class Profile {
  @Copy()
  firstName: string;

  @Copy()
  lastName: string;

  @Merge({
    sources: ['firstName', 'lastName'],
    mergeFunction: ([first, last]) => `${first} ${last}`
  })
  fullName: string;
}
```

---

## Context Decorators

Decorators that change the context in which subsequent decorators operate.

### @Keys()

Applies subsequent decorators to object keys.

```typescript
@Keys()
```

**Example:**
```typescript
class HttpRequest {
  @Keys()
  @CoerceTrim()
  @CoerceCase('lower')
  headers: Record<string, string>;
  // { "  Content-Type  ": "..." } -> { "content-type": "..." }
}
```

### @Values()

Applies subsequent decorators to object values or array elements.

```typescript
@Values()
```

**Example:**
```typescript
class TagContainer {
  @Values()
  @CoerceTrim()
  @CoerceCase('lower')
  tags: string[];
  // ["  TAG-ONE  ", "TAG-TWO"] -> ["tag-one", "tag-two"]
}
```

### @Split(separator, options?)

Splits string by delimiter, applies subsequent decorators to segments, returns array.

```typescript
@Split(separator: string, options?: SplitOptions)

interface SplitOptions {
  emptyStringBehavior?: 'empty-array' | 'single-empty-element';
  quotes?: string | string[];           // Quote chars to protect
  brackets?: BracketPair | BracketPair[]; // Bracket pairs to protect
  escapeChar?: string;                  // Default: '\\'
  stripQuotes?: boolean;                // Default: true
  stripBrackets?: boolean;              // Default: false
  description?: string;
}

type BracketPair = '()' | '[]' | '{}' | '<>' | string;
```

**Example:**
```typescript
class StyleRule {
  @Split(';', {
    quotes: ['"', "'"],
    brackets: ['()', '[]'],
    stripQuotes: true
  })
  @CoerceTrim()
  rules: string[];
  // 'color: rgb(255, 0, 0); font: "Times New Roman"'
  // -> ['color: rgb(255, 0, 0)', 'font: Times New Roman']
}
```

### @Delimited(separator, options?)

Splits string by delimiter, applies decorators to segments, rejoins with same delimiter.

```typescript
@Delimited(separator: string, options?: DelimitedOptions)

interface DelimitedOptions {
  quotes?: string | string[];
  brackets?: BracketPair | BracketPair[];
  escapeChar?: string;        // Default: '\\'
  stripQuotes?: boolean;      // Default: true
  stripBrackets?: boolean;    // Default: false
  description?: string;
}
```

**Example:**
```typescript
class CsvRow {
  @Delimited(',', { quotes: ['"'] })
  @CoerceTrim()
  row: string;
  // '"  value1  ", value2,  value3  ' -> 'value1,value2,value3'
}
```

---

## Class-Level Decorators

Decorators applied to classes or used for nested validation.

### @ValidatedClass(dataClass)

Explicitly marks property as containing a nested validated class. Usually auto-detected.

```typescript
@ValidatedClass(dataClass: Function)
```

**Example:**
```typescript
class Address {
  @ValidateRequired() @CoerceTrim() street: string;
  @ValidateRequired() @CoerceTrim() city: string;
}

class Customer {
  @ValidatedClass(Address)
  address: Address;
}
```

### @ValidatedClassArray(elementClass)

Marks property as array of validated class instances.

```typescript
@ValidatedClassArray(elementClass: Function)
```

**Example:**
```typescript
class OrderItem {
  @ValidateRequired() productId: string;
  @CoerceType('number') quantity: number;
}

class Order {
  @ValidatedClassArray(OrderItem)
  items: OrderItem[];
}
```

### @UseStyle(style)

Applies a reusable validation style to a property.

```typescript
@UseStyle(style: new () => any)
```

**Example:**
```typescript
// Define reusable style
class EmailStyle {
  @CoerceTrim()
  @CoerceCase('lower')
  @ValidatePattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
  value: string;
}

// Apply to multiple properties
class User {
  @UseStyle(EmailStyle)
  email: string;

  @UseStyle(EmailStyle)
  alternateEmail: string;
}
```

### @DefaultTransforms(transforms)

Sets type-based default transforms for a class. Creates a CSS-like cascade: Factory < Class < Property decorators.

```typescript
@DefaultTransforms(transforms: TypeDefaultTransforms)

interface TypeDefaultTransforms {
  string?: Function;
  number?: Function;
  boolean?: Function;
  [key: string]: Function | undefined;
}
```

**Example:**
```typescript
class TrimStyle { @CoerceTrim() value: string; }

@DefaultTransforms({
  string: TrimStyle
})
class User {
  @Copy() name: string;    // Auto-trimmed
  @Copy() email: string;   // Auto-trimmed
}
```

### @ObjectRule(validationFn, description?)

Adds object-level validation that runs after all property validations.

```typescript
@ObjectRule(
  validationFn: (obj: any) => boolean | string | Error | void,
  description?: string
)
```

**Example:**
```typescript
@ObjectRule(function(this: Order) {
  if (this.endDate < this.startDate) {
    return 'End date must be after start date';
  }
  return true;
}, 'Date range validation')
class Order {
  @CoerceType('date') startDate: Date;
  @CoerceType('date') endDate: Date;
}
```

### @CrossValidate(dependencies, validationFn, description?)

Property-level validation that depends on multiple properties.

```typescript
@CrossValidate(
  dependencies: string[],
  validationFn: (obj: any) => boolean | string | Error | void,
  description?: string
)
```

**Example:**
```typescript
class Subscription {
  @Copy() plan: string;

  @CrossValidate(['plan'], (obj) => {
    if (obj.plan === 'premium' && obj.seats < 5) {
      return 'Premium plan requires at least 5 seats';
    }
    return true;
  })
  @CoerceType('number')
  seats: number;
}
```

### @DependsOn(propertyKeys, options?)

Declares property dependencies for dynamic prompts and execution ordering.

```typescript
@DependsOn(propertyKeys: string[], options?: CommonDecoratorOptions)
```

**Example:**
```typescript
class Order {
  @Copy() sku: string;

  @DependsOn(['sku'])
  @DerivedFrom('sku', (sku, ctx) => ctx.context.prices[sku])
  price: number;
}
```

### @UseSinglePassValidation()

Uses single-pass validation engine (vs. default convergent).

```typescript
@UseSinglePassValidation()
```

### @UseConvergentValidation()

Uses convergent validation engine (default).

```typescript
@UseConvergentValidation()
```

### @ValidateClass(rules)

Applies validation styles to map keys/values.

```typescript
@ValidateClass(rules: Array<{
  on: 'key' | 'value';
  ofType: any;
  style: new () => any;
}>)
```

### @Examples(examples, description?)

Provides example values for error messages and AI context.

```typescript
@Examples(examples: any[], description?: string)
```

**Example:**
```typescript
class Order {
  @Examples(['ORD-001', 'ORD-002'], 'Order ID format')
  @ValidatePattern(/^ORD-\d{3}$/)
  orderId: string;
}
```

---

## Text Normalization

Built-in text normalizers for common formats.

### @NormalizeText(normalizerName)

Applies a registered text normalizer.

```typescript
@NormalizeText(normalizerName: string)
```

### @NormalizeTextChain(normalizerNames)

Applies multiple text normalizers in sequence.

```typescript
@NormalizeTextChain(normalizerNames: string[])
```

### Built-in Normalizers

| Name | Description | Example |
|------|-------------|---------|
| `email` | Lowercase and trim | `"  JOHN@EXAMPLE.COM  "` -> `"john@example.com"` |
| `phone` | Remove non-digit characters | `"(555) 123-4567"` -> `"5551234567"` |
| `phone-formatted` | Format to standard pattern | `"5551234567"` -> `"(555) 123-4567"` |
| `credit-card` | Remove spaces/dashes, mask | `"4111-1111-1111-1111"` -> `"************1111"` |
| `currency` | Normalize currency values | `"$1,234.56"` -> `"1234.56"` |
| `ssn` | Format SSN | `"123456789"` -> `"123-45-6789"` |
| `url` | Normalize URL | Adds protocol, normalizes case |
| `unicode-nfc` | Unicode NFC normalization | Composed form |
| `unicode-nfd` | Unicode NFD normalization | Decomposed form |
| `whitespace` | Normalize whitespace | Multiple spaces -> single space |
| `zip-code` | Format ZIP code | `"123456789"` -> `"12345-6789"` |

### Custom Normalizers

```typescript
interface TextNormalizer {
  name: string;
  description: string;
  normalize(input: string): string;
}

class TextNormalizerRegistry {
  static register(normalizer: TextNormalizer): void
  static get(name: string): TextNormalizer | undefined
  static list(): string[]
}
```

**Example:**
```typescript
TextNormalizerRegistry.register({
  name: 'slug',
  description: 'Convert to URL slug',
  normalize: (input) => input.toLowerCase().replace(/\s+/g, '-')
});

class Article {
  @NormalizeText('slug')
  urlSlug: string;
}
```

---

## Matching Strategies

Strategies for matching values in `@CoerceFromSet`.

### Available Strategies

| Strategy | Description |
|----------|-------------|
| `'exact'` | Exact match (case-sensitive/insensitive) |
| `'fuzzy'` | Levenshtein distance-based fuzzy matching |
| `'contains'` | Candidate contains input |
| `'beginsWith'` | Candidate starts with input |
| `'endsWith'` | Candidate ends with input |
| `'regex'` | Input treated as regex pattern |
| `'custom'` | Custom matcher function |

### Matching Functions

```typescript
function executeMatchingStrategy(
  input: string,
  candidates: string[],
  strategy: MatchingStrategy,
  caseSensitive: boolean,
  customMatcher?: (value: string, candidate: string) => number
): MatchResult[]

interface MatchResult {
  value: string;
  score: number;  // 0-1, higher is better
}

function findBestMatches(
  results: MatchResult[],
  threshold: number,
  ambiguityTolerance?: number  // Default: 0.1
): MatchResult[]
```

**Ambiguity Handling:** If multiple candidates score within `ambiguityTolerance` of each other, a `CoercionAmbiguityError` is thrown.

---

## Type Definitions

### AIHandler

```typescript
type AIHandler<TValue = any, TContext = any, TMetadata = any, TInstance = any, TPrompt = any> =
  (
    params: AIHandlerParams<TValue, TContext, TMetadata, TInstance>,
    prompt: TPrompt
  ) => Promise<TValue>;
```

### AIValidationHandler

```typescript
type AIValidationHandler<TValue = any, TContext = any, TMetadata = any, TInstance = any, TPrompt = any> =
  (
    params: AIHandlerParams<TValue, TContext, TMetadata, TInstance>,
    prompt: TPrompt
  ) => Promise<boolean | string | Error>;
```

### CommonDecoratorOptions

```typescript
interface CommonDecoratorOptions {
  order?: number;        // Explicit execution order (lower = earlier)
  description?: string;  // Description for debugging
}
```

### CoercionFunction

```typescript
type CoercionFunction<T = any> = (value: any) => T;
```

### ValidationFunction

```typescript
type ValidationFunction<T = any> = (value: T, obj: any) => boolean | string | Error;
```

### AsyncValidationFunction

```typescript
type AsyncValidationFunction<T = any> = (value: T, obj: any) => Promise<boolean | string | Error>;
```

### ObjectValidationFunction

```typescript
type ObjectValidationFunction = (obj: any) => boolean | string | Error | void;
```

---

## Error Types

### ValidationError

Thrown when validation fails. Extends `FFLLMFixableError`.

```typescript
class ValidationError extends FFLLMFixableError {
  message: string;
  propertyPath: string;
  rule: string;
  actualValue: any;
  examples?: any[];
  examplesDescription?: string;
}
```

**Example:**
```typescript
try {
  await factory.create(User, { age: 15 });
} catch (error) {
  if (error instanceof ValidationError) {
    console.log(error.message);      // "Age must be between 18 and 120"
    console.log(error.propertyPath); // "age"
    console.log(error.actualValue);  // 15
    console.log(error.rule);         // "ValidateRange"
  }
}
```

### FFError

Base class for all FireFoundry errors.

### FFLLMFixableError

Error that can potentially be fixed by LLM retry.

```typescript
class FFLLMFixableError extends Error {
  message: string;
  suggestedPromptAddition?: string;
}
```

### FFLLMNonFixableError

Error that cannot be fixed by LLM.

### ConvergenceTimeoutError

Thrown when convergent engine doesn't stabilize within `maxIterations`.

### OscillationError

Thrown when convergent engine detects oscillating values.

### CoercionAmbiguityError

Thrown when multiple equally valid coercion matches are found.

```typescript
class CoercionAmbiguityError extends FFLLMFixableError {
  candidates: MatchResult[];
}
```

---

## Complete Example

```typescript
import {
  ValidationFactory,
  ValidateRequired,
  CoerceTrim,
  CoerceType,
  CoerceCase,
  CoerceFromSet,
  ValidatePattern,
  ValidateRange,
  AITransform,
  DerivedFrom,
  UseStyle,
  DefaultTransforms,
  ObjectRule,
  ValidationError
} from '@firebrandanalytics/shared-utils/validation';

// Define reusable styles
class EmailStyle {
  @CoerceTrim()
  @CoerceCase('lower')
  @ValidatePattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
  value: string;
}

class SkuStyle {
  @CoerceTrim()
  @CoerceCase('upper')
  @ValidatePattern(/^[A-Z]{3}-\d{3}$/)
  value: string;
}

// Define context type
interface OrderContext {
  validSkus: string[];
  productPrices: Record<string, number>;
}

// Main class with validation
@DefaultTransforms({ string: class { @CoerceTrim() value: string; } })
@ObjectRule(function(this: Order) {
  if (this.quantity > 50 && !this.bulkApproved) {
    return 'Bulk orders over 50 require approval';
  }
  return true;
})
class Order {
  @ValidateRequired()
  orderId: string;

  @UseStyle(EmailStyle)
  customerEmail: string;

  @UseStyle(SkuStyle)
  @CoerceFromSet<OrderContext>(
    (ctx) => ctx.validSkus,
    { strategy: 'fuzzy', fuzzyThreshold: 0.7 }
  )
  sku: string;

  @AITransform((params) =>
    `Extract the quantity as a number from: "${params.value}". Return only the number.`
  )
  @CoerceType('number')
  @ValidateRange(1, 100)
  quantity: number;

  @DerivedFrom<Order, OrderContext>('sku', (sku, ctx) => {
    return ctx.context.productPrices[sku] || 0;
  })
  basePrice: number;

  @CoerceType('boolean')
  bulkApproved: boolean;

  // Business logic
  calculateTotal(): number {
    const discount = this.quantity > 10 ? 0.1 : 0;
    return this.basePrice * this.quantity * (1 - discount);
  }
}

// Usage
const factory = new ValidationFactory({
  aiHandler: async (params, prompt) => {
    // Your LLM integration
    return await llm.complete(prompt);
  }
});

const context: OrderContext = {
  validSkus: ['WID-001', 'GAD-002'],
  productPrices: { 'WID-001': 19.99, 'GAD-002': 29.99 }
};

try {
  const order = await factory.create(Order, {
    orderId: 'ORD-456',
    customerEmail: '  CUSTOMER@EXAMPLE.COM  ',
    sku: 'wid-001',
    quantity: 'about eight',
    bulkApproved: 'no'
  }, { context });

  console.log(order.calculateTotal());
} catch (error) {
  if (error instanceof ValidationError) {
    console.error(`Validation failed for ${error.propertyPath}: ${error.message}`);
  }
}
```

---

## Import Reference

```typescript
// Core
import { ValidationFactory } from '@firebrandanalytics/shared-utils/validation';

// Coercion decorators
import {
  Coerce,
  CoerceType,
  CoerceTrim,
  CoerceCase,
  CoerceRound,
  CoerceFromSet,
  CoerceArrayElements
} from '@firebrandanalytics/shared-utils/validation';

// Validation decorators
import {
  Validate,
  ValidateRequired,
  ValidateRange,
  ValidateLength,
  ValidatePattern,
  ValidateAsync,
  CrossValidate
} from '@firebrandanalytics/shared-utils/validation';

// AI decorators
import {
  AITransform,
  AIValidate
} from '@firebrandanalytics/shared-utils/validation';

// Data source decorators
import {
  Copy,
  DerivedFrom,
  RenameFrom,
  JSONPath,
  CollectProperties,
  Merge,
  DependsOn
} from '@firebrandanalytics/shared-utils/validation';

// Context decorators
import {
  Keys,
  Values,
  Split,
  Delimited
} from '@firebrandanalytics/shared-utils/validation';

// Class decorators
import {
  ValidatedClass,
  ValidatedClassArray,
  UseStyle,
  DefaultTransforms,
  ObjectRule,
  Examples,
  UseSinglePassValidation,
  UseConvergentValidation,
  ValidateClass
} from '@firebrandanalytics/shared-utils/validation';

// Text normalization
import {
  NormalizeText,
  NormalizeTextChain,
  TextNormalizerRegistry
} from '@firebrandanalytics/shared-utils/validation';

// Errors
import {
  ValidationError,
  FFError,
  FFLLMFixableError,
  FFLLMNonFixableError,
  ConvergenceTimeoutError,
  OscillationError,
  CoercionAmbiguityError
} from '@firebrandanalytics/shared-utils/validation';
```
