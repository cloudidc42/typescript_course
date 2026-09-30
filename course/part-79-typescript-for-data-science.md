# ตอนที่ 79: TypeScript for Data Science & Machine Learning

## บทนำ

TypeScript กำลังได้รับความนิยมในวงการ Data Science และ Machine Learning มากขึ้น โดยเฉพาะในฝั่ง Web และ Node.js ด้วยความช่วยเหลือของ libraries เช่น TensorFlow.js, Danfo.js และอื่นๆ

---

## 1. TensorFlow.js กับ TypeScript

### ตัวอย่างที่ 1: พื้นฐาน TensorFlow.js

```typescript
import * as tf from "@tensorflow/tfjs-node";

// สร้าง tensor พื้นฐาน
const scalar = tf.scalar(3.14);
console.log("Scalar:", scalar.arraySync());

const vector = tf.tensor1d([1, 2, 3, 4, 5]);
console.log("Vector:", vector.arraySync());

const matrix = tf.tensor2d([
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9],
]);
console.log("Matrix shape:", matrix.shape);

// Operations
const a = tf.tensor2d([[1, 2], [3, 4]]);
const b = tf.tensor2d([[5, 6], [7, 8]]);

// Element-wise operations
const sum = tf.add(a, b);
const product = tf.mul(a, b);
const diff = tf.sub(a, b);

console.log("Sum:", sum.arraySync());
console.log("Product:", product.arraySync());

// Matrix multiplication
const matmul = tf.matMul(a, b);
console.log("Matrix multiply:", matmul.arraySync());

// Reduction operations
const numbers = tf.tensor1d([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);
console.log("Mean:", numbers.mean().arraySync());
console.log("Sum:", numbers.sum().arraySync());
console.log("Max:", numbers.max().arraySync());
console.log("Min:", numbers.min().arraySync());
console.log("Std:", numbers.std().arraySync());
```

### ตัวอย่างที่ 2: Neural Network ง่ายๆ

```typescript
import * as tf from "@tensorflow/tfjs-node";

// ข้อมูล XOR problem
const xorInputs = tf.tensor2d([
  [0, 0],
  [0, 1],
  [1, 0],
  [1, 1],
]);

const xorOutputs = tf.tensor2d([[0], [1], [1], [0]]);

// สร้าง model
function createXORModel(): tf.Sequential {
  const model = tf.sequential();

  model.add(
    tf.layers.dense({
      inputShape: [2],
      units: 8,
      activation: "relu",
      kernelInitializer: "glorotUniform",
    })
  );

  model.add(
    tf.layers.dense({
      units: 4,
      activation: "relu",
    })
  );

  model.add(
    tf.layers.dense({
      units: 1,
      activation: "sigmoid",
    })
  );

  return model;
}

async function trainXOR(): Promise<void> {
  const model = createXORModel();

  model.compile({
    optimizer: tf.train.adam(0.01),
    loss: "binaryCrossentropy",
    metrics: ["accuracy"],
  });

  console.log("Model Summary:");
  model.summary();

  // Train
  const history = await model.fit(xorInputs, xorOutputs, {
    epochs: 1000,
    verbose: 0,
    callbacks: {
      onEpochEnd: (epoch, logs) => {
        if (epoch % 100 === 0) {
          console.log(
            `Epoch ${epoch}: loss = ${logs?.loss?.toFixed(4)}, accuracy = ${logs?.acc?.toFixed(4)}`
          );
        }
      },
    },
  });

  // Predict
  console.log("\nการทำนาย XOR:");
  const predictions = model.predict(xorInputs) as tf.Tensor;
  const results = predictions.arraySync() as number[][];

  [[0, 0], [0, 1], [1, 0], [1, 1]].forEach((input, i) => {
    const predicted = results[i][0] > 0.5 ? 1 : 0;
    console.log(
      `XOR(${input[0]}, ${input[1]}) = ${predicted} (confidence: ${(results[i][0] * 100).toFixed(1)}%)`
    );
  });
}

trainXOR().catch(console.error);
```

### ตัวอย่างที่ 3: Linear Regression

```typescript
import * as tf from "@tensorflow/tfjs-node";

// Type definitions
interface DataPoint {
  x: number;
  y: number;
}

interface TrainingConfig {
  learningRate: number;
  epochs: number;
  batchSize?: number;
}

interface TrainingResult {
  slope: number;
  intercept: number;
  finalLoss: number;
  history: number[];
}

class LinearRegressionModel {
  private model: tf.Sequential;

  constructor() {
    this.model = this.createModel();
  }

  private createModel(): tf.Sequential {
    const model = tf.sequential();
    model.add(tf.layers.dense({ inputShape: [1], units: 1 }));
    return model;
  }

  async train(
    data: DataPoint[],
    config: TrainingConfig
  ): Promise<TrainingResult> {
    const xs = tf.tensor1d(data.map((d) => d.x));
    const ys = tf.tensor1d(data.map((d) => d.y));

    this.model.compile({
      optimizer: tf.train.sgd(config.learningRate),
      loss: "meanSquaredError",
    });

    const lossHistory: number[] = [];

    await this.model.fit(xs, ys, {
      epochs: config.epochs,
      batchSize: config.batchSize ?? 32,
      verbose: 0,
      callbacks: {
        onEpochEnd: (_, logs) => {
          lossHistory.push(logs?.loss ?? 0);
        },
      },
    });

    // Extract weights
    const layer = this.model.layers[0];
    const weights = layer.getWeights();
    const slope = (weights[0].arraySync() as number[][])[0][0];
    const intercept = (weights[1].arraySync() as number[])[0];

    xs.dispose();
    ys.dispose();

    return {
      slope,
      intercept,
      finalLoss: lossHistory[lossHistory.length - 1],
      history: lossHistory,
    };
  }

  predict(x: number): number {
    const input = tf.tensor2d([[x]]);
    const output = this.model.predict(input) as tf.Tensor;
    const result = (output.arraySync() as number[][])[0][0];

    input.dispose();
    output.dispose();

    return result;
  }

  predictBatch(xs: number[]): number[] {
    const input = tf.tensor2d(xs.map((x) => [x]));
    const output = this.model.predict(input) as tf.Tensor;
    const results = (output.arraySync() as number[][]).map((r) => r[0]);

    input.dispose();
    output.dispose();

    return results;
  }
}

// ตัวอย่างการใช้งาน
async function demonstrateLinearRegression(): Promise<void> {
  // สร้างข้อมูล y = 2x + 5 + noise
  const generateData = (n: number): DataPoint[] => {
    const data: DataPoint[] = [];
    for (let i = 0; i < n; i++) {
      const x = Math.random() * 20 - 10;
      const noise = (Math.random() - 0.5) * 2;
      const y = 2 * x + 5 + noise;
      data.push({ x, y });
    }
    return data;
  };

  const trainingData = generateData(200);
  const testData = generateData(50);

  const model = new LinearRegressionModel();

  console.log("เริ่มฝึก Linear Regression...");
  const result = await model.train(trainingData, {
    learningRate: 0.01,
    epochs: 100,
  });

  console.log(`\nผลการฝึก:`);
  console.log(`  y = ${result.slope.toFixed(4)}x + ${result.intercept.toFixed(4)}`);
  console.log(`  (ค่าจริง: y = 2x + 5)`);
  console.log(`  Final Loss: ${result.finalLoss.toFixed(6)}`);

  // ทดสอบ
  console.log("\nการทำนาย:");
  [0, 5, 10, -5].forEach((x) => {
    const predicted = model.predict(x);
    const actual = 2 * x + 5;
    console.log(
      `  x=${x}: ทำนาย=${predicted.toFixed(2)}, จริง=${actual.toFixed(2)}`
    );
  });
}

demonstrateLinearRegression().catch(console.error);
```

---

## 2. Tensor Types

### ตัวอย่างที่ 4: Type-Safe Tensor Operations

```typescript
import * as tf from "@tensorflow/tfjs-node";

// Type-safe wrapper สำหรับ tensors
type TensorRank = 0 | 1 | 2 | 3 | 4;
type DataType = "float32" | "int32" | "bool" | "string";

interface TypedTensor<R extends TensorRank, D extends DataType = "float32"> {
  readonly rank: R;
  readonly dtype: D;
  readonly shape: number[];
  readonly tensor: tf.Tensor;
}

function createScalar(value: number): TypedTensor<0> {
  return {
    rank: 0,
    dtype: "float32",
    shape: [],
    tensor: tf.scalar(value),
  };
}

function createVector(values: number[]): TypedTensor<1> {
  return {
    rank: 1,
    dtype: "float32",
    shape: [values.length],
    tensor: tf.tensor1d(values),
  };
}

function createMatrix(values: number[][]): TypedTensor<2> {
  return {
    rank: 2,
    dtype: "float32",
    shape: [values.length, values[0].length],
    tensor: tf.tensor2d(values),
  };
}

// Type-safe matrix multiplication
function matmul(
  a: TypedTensor<2>,
  b: TypedTensor<2>
): TypedTensor<2> {
  if (a.shape[1] !== b.shape[0]) {
    throw new Error(
      `Matrix dimensions ไม่ตรง: [${a.shape}] × [${b.shape}]`
    );
  }

  return {
    rank: 2,
    dtype: "float32",
    shape: [a.shape[0], b.shape[1]],
    tensor: tf.matMul(a.tensor, b.tensor),
  };
}

// Type-safe reshape
function reshape<R extends TensorRank>(
  tensor: TypedTensor<TensorRank>,
  newShape: number[]
): TypedTensor<R> {
  return {
    rank: newShape.length as R,
    dtype: tensor.dtype,
    shape: newShape,
    tensor: tensor.tensor.reshape(newShape),
  };
}

// Batch operations
interface BatchResult<T> {
  results: T[];
  processingTime: number;
}

async function processBatch<T, R>(
  items: T[],
  processor: (item: T) => Promise<R>,
  batchSize: number = 32
): Promise<BatchResult<R>> {
  const startTime = Date.now();
  const results: R[] = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(processor));
    results.push(...batchResults);
  }

  return {
    results,
    processingTime: Date.now() - startTime,
  };
}

// ตัวอย่างการใช้งาน
const matA = createMatrix([[1, 2, 3], [4, 5, 6]]);
const matB = createMatrix([[7, 8], [9, 10], [11, 12]]);

const result = matmul(matA, matB);
console.log("Matrix multiply result:");
console.log(result.tensor.arraySync());
console.log("Shape:", result.shape);
```

---

## 3. Data Pipelines กับ TypeScript

### ตัวอย่างที่ 5: Functional Data Pipeline

```typescript
// Type-safe data pipeline
type Transform<A, B> = (input: A) => B;
type AsyncTransform<A, B> = (input: A) => Promise<B>;

class Pipeline<T> {
  private constructor(private readonly data: T) {}

  static of<T>(value: T): Pipeline<T> {
    return new Pipeline(value);
  }

  map<U>(transform: Transform<T, U>): Pipeline<U> {
    return new Pipeline(transform(this.data));
  }

  flatMap<U>(transform: Transform<T, Pipeline<U>>): Pipeline<U> {
    return transform(this.data);
  }

  filter(predicate: (value: T) => boolean): Pipeline<T | null> {
    return new Pipeline(predicate(this.data) ? this.data : null);
  }

  async mapAsync<U>(transform: AsyncTransform<T, U>): Promise<Pipeline<U>> {
    const result = await transform(this.data);
    return new Pipeline(result);
  }

  tap(fn: (value: T) => void): Pipeline<T> {
    fn(this.data);
    return this;
  }

  value(): T {
    return this.data;
  }
}

// Array Pipeline
class ArrayPipeline<T> {
  private constructor(private readonly items: T[]) {}

  static from<T>(items: T[]): ArrayPipeline<T> {
    return new ArrayPipeline(items);
  }

  map<U>(fn: (item: T, index: number) => U): ArrayPipeline<U> {
    return new ArrayPipeline(this.items.map(fn));
  }

  filter(fn: (item: T, index: number) => boolean): ArrayPipeline<T> {
    return new ArrayPipeline(this.items.filter(fn));
  }

  reduce<U>(fn: (acc: U, item: T, index: number) => U, initial: U): U {
    return this.items.reduce(fn, initial);
  }

  flatMap<U>(fn: (item: T) => U[]): ArrayPipeline<U> {
    return new ArrayPipeline(this.items.flatMap(fn));
  }

  groupBy<K extends string | number>(
    keyFn: (item: T) => K
  ): Map<K, T[]> {
    return this.items.reduce((groups, item) => {
      const key = keyFn(item);
      const group = groups.get(key) ?? [];
      group.push(item);
      groups.set(key, group);
      return groups;
    }, new Map<K, T[]>());
  }

  sort(compareFn?: (a: T, b: T) => number): ArrayPipeline<T> {
    return new ArrayPipeline([...this.items].sort(compareFn));
  }

  take(n: number): ArrayPipeline<T> {
    return new ArrayPipeline(this.items.slice(0, n));
  }

  skip(n: number): ArrayPipeline<T> {
    return new ArrayPipeline(this.items.slice(n));
  }

  unique(): ArrayPipeline<T> {
    return new ArrayPipeline([...new Set(this.items)]);
  }

  uniqueBy<K>(keyFn: (item: T) => K): ArrayPipeline<T> {
    const seen = new Set<K>();
    return this.filter((item) => {
      const key = keyFn(item);
      if (seen.has(key)) return false;
      seen.add(key);
      return true;
    });
  }

  chunk(size: number): ArrayPipeline<T[]> {
    const chunks: T[][] = [];
    for (let i = 0; i < this.items.length; i += size) {
      chunks.push(this.items.slice(i, i + size));
    }
    return new ArrayPipeline(chunks);
  }

  toArray(): T[] {
    return [...this.items];
  }

  count(): number {
    return this.items.length;
  }

  first(): T | undefined {
    return this.items[0];
  }

  last(): T | undefined {
    return this.items[this.items.length - 1];
  }
}

// ตัวอย่างการใช้งาน
interface SalesRecord {
  id: string;
  product: string;
  category: string;
  amount: number;
  date: Date;
  region: string;
  salesPerson: string;
}

const salesData: SalesRecord[] = [
  { id: "1", product: "โน้ตบุ๊ค", category: "Electronics", amount: 45000, date: new Date("2024-01-15"), region: "Bangkok", salesPerson: "สมชาย" },
  { id: "2", product: "สมาร์ทโฟน", category: "Electronics", amount: 25000, date: new Date("2024-01-20"), region: "Chiang Mai", salesPerson: "สมหญิง" },
  { id: "3", product: "โต๊ะทำงาน", category: "Furniture", amount: 12000, date: new Date("2024-02-05"), region: "Bangkok", salesPerson: "สมชาย" },
  { id: "4", product: "เก้าอี้", category: "Furniture", amount: 8000, date: new Date("2024-02-10"), region: "Phuket", salesPerson: "มานี" },
  { id: "5", product: "แท็บเล็ต", category: "Electronics", amount: 15000, date: new Date("2024-02-15"), region: "Chiang Mai", salesPerson: "สมหญิง" },
];

// Analysis pipeline
const analysis = ArrayPipeline.from(salesData)
  .filter((s) => s.amount > 10000)
  .map((s) => ({ ...s, monthYear: `${s.date.getFullYear()}-${String(s.date.getMonth() + 1).padStart(2, "0")}` }))
  .sort((a, b) => b.amount - a.amount);

console.log("ยอดขายสูงกว่า 10,000 เรียงตามมูลค่า:");
analysis.toArray().forEach((s) => {
  console.log(`  ${s.product}: ${s.amount.toLocaleString()} บาท (${s.region})`);
});

// Group by category
const byCategory = ArrayPipeline.from(salesData).groupBy((s) => s.category);
byCategory.forEach((records, category) => {
  const total = records.reduce((sum, r) => sum + r.amount, 0);
  console.log(`${category}: ${total.toLocaleString()} บาท (${records.length} รายการ)`);
});
```

---

## 4. CSV/JSON Processing

### ตัวอย่างที่ 6: CSV Parser ที่ปลอดภัย

```typescript
interface CSVParserOptions {
  delimiter?: string;
  hasHeader?: boolean;
  skipEmptyLines?: boolean;
  trimValues?: boolean;
  encoding?: string;
}

interface CSVRow {
  [key: string]: string;
}

class CSVParser {
  private options: Required<CSVParserOptions>;

  constructor(options: CSVParserOptions = {}) {
    this.options = {
      delimiter: options.delimiter ?? ",",
      hasHeader: options.hasHeader ?? true,
      skipEmptyLines: options.skipEmptyLines ?? true,
      trimValues: options.trimValues ?? true,
      encoding: options.encoding ?? "utf-8",
    };
  }

  parse(csvContent: string): CSVRow[] {
    const lines = csvContent.split("\n");
    const filteredLines = this.options.skipEmptyLines
      ? lines.filter((l) => l.trim().length > 0)
      : lines;

    if (filteredLines.length === 0) return [];

    let headers: string[] = [];
    let dataLines = filteredLines;

    if (this.options.hasHeader) {
      headers = this.parseLine(filteredLines[0]);
      dataLines = filteredLines.slice(1);
    }

    return dataLines.map((line) => {
      const values = this.parseLine(line);

      if (this.options.hasHeader) {
        const row: CSVRow = {};
        headers.forEach((header, i) => {
          row[header] = values[i] ?? "";
        });
        return row;
      }

      const row: CSVRow = {};
      values.forEach((value, i) => {
        row[String(i)] = value;
      });
      return row;
    });
  }

  private parseLine(line: string): string[] {
    const delimiter = this.options.delimiter;
    const values: string[] = [];
    let currentValue = "";
    let inQuotes = false;
    let i = 0;

    while (i < line.length) {
      const char = line[i];

      if (char === '"') {
        if (inQuotes && line[i + 1] === '"') {
          currentValue += '"';
          i += 2;
          continue;
        }
        inQuotes = !inQuotes;
      } else if (char === delimiter && !inQuotes) {
        values.push(
          this.options.trimValues ? currentValue.trim() : currentValue
        );
        currentValue = "";
      } else {
        currentValue += char;
      }

      i++;
    }

    values.push(
      this.options.trimValues ? currentValue.trim() : currentValue
    );

    return values;
  }

  parseToTyped<T>(
    csvContent: string,
    schema: { [K in keyof T]: (value: string) => T[K] }
  ): T[] {
    const rows = this.parse(csvContent);
    return rows.map((row) => {
      const result = {} as T;
      (Object.entries(schema) as [keyof T, (v: string) => T[keyof T]][]).forEach(
        ([key, converter]) => {
          result[key] = converter(row[key as string] ?? "");
        }
      );
      return result;
    });
  }

  serialize(data: CSVRow[], headers?: string[]): string {
    const keys = headers ?? (data.length > 0 ? Object.keys(data[0]) : []);

    const headerRow = keys.map((k) => this.escapeValue(k)).join(this.options.delimiter);
    const dataRows = data.map((row) =>
      keys.map((k) => this.escapeValue(row[k] ?? "")).join(this.options.delimiter)
    );

    return [headerRow, ...dataRows].join("\n");
  }

  private escapeValue(value: string): string {
    if (
      value.includes(this.options.delimiter) ||
      value.includes('"') ||
      value.includes("\n")
    ) {
      return `"${value.replace(/"/g, '""')}"`;
    }
    return value;
  }
}

// ตัวอย่างการใช้งาน
const csvData = `
id,name,age,email,salary
1,สมชาย,30,somchai@example.com,50000
2,สมหญิง,25,"สมหญิง มีนา",45000
3,มานี,35,manee@example.com,60000
4,วิชาย,28,vichai@example.com,55000
`.trim();

const parser = new CSVParser({ hasHeader: true, trimValues: true });

interface Employee {
  id: number;
  name: string;
  age: number;
  email: string;
  salary: number;
}

const employees = parser.parseToTyped<Employee>(csvData, {
  id: Number,
  name: String,
  age: Number,
  email: String,
  salary: Number,
});

console.log("Employees:", employees);

// Statistics
const avgSalary = employees.reduce((sum, e) => sum + e.salary, 0) / employees.length;
const maxSalary = Math.max(...employees.map((e) => e.salary));
const minSalary = Math.min(...employees.map((e) => e.salary));

console.log(`เงินเดือนเฉลี่ย: ${avgSalary.toLocaleString()}`);
console.log(`เงินเดือนสูงสุด: ${maxSalary.toLocaleString()}`);
console.log(`เงินเดือนต่ำสุด: ${minSalary.toLocaleString()}`);
```

---

## 5. Statistical Operations

### ตัวอย่างที่ 7: Statistics Library

```typescript
// Type-safe statistics library
class Statistics {
  // Measures of Central Tendency
  static mean(data: number[]): number {
    if (data.length === 0) return NaN;
    return data.reduce((sum, x) => sum + x, 0) / data.length;
  }

  static median(data: number[]): number {
    if (data.length === 0) return NaN;
    const sorted = [...data].sort((a, b) => a - b);
    const mid = Math.floor(sorted.length / 2);

    return sorted.length % 2 === 0
      ? (sorted[mid - 1] + sorted[mid]) / 2
      : sorted[mid];
  }

  static mode(data: number[]): number[] {
    if (data.length === 0) return [];

    const frequency = new Map<number, number>();
    data.forEach((x) => frequency.set(x, (frequency.get(x) ?? 0) + 1));

    const maxFreq = Math.max(...frequency.values());
    return Array.from(frequency.entries())
      .filter(([, freq]) => freq === maxFreq)
      .map(([value]) => value);
  }

  // Measures of Dispersion
  static variance(data: number[], population = false): number {
    if (data.length < 2) return NaN;
    const mean = Statistics.mean(data);
    const squaredDiffs = data.map((x) => (x - mean) ** 2);
    const divisor = population ? data.length : data.length - 1;
    return squaredDiffs.reduce((sum, x) => sum + x, 0) / divisor;
  }

  static std(data: number[], population = false): number {
    return Math.sqrt(Statistics.variance(data, population));
  }

  static range(data: number[]): number {
    return Math.max(...data) - Math.min(...data);
  }

  static iqr(data: number[]): number {
    const sorted = [...data].sort((a, b) => a - b);
    const q1 = Statistics.quantile(sorted, 0.25);
    const q3 = Statistics.quantile(sorted, 0.75);
    return q3 - q1;
  }

  // Quantiles and Percentiles
  static quantile(sortedData: number[], p: number): number {
    if (p < 0 || p > 1) throw new Error("p ต้องอยู่ระหว่าง 0 ถึง 1");

    const n = sortedData.length;
    const index = p * (n - 1);
    const lower = Math.floor(index);
    const upper = Math.ceil(index);

    if (lower === upper) return sortedData[lower];
    return (
      sortedData[lower] * (upper - index) +
      sortedData[upper] * (index - lower)
    );
  }

  static percentile(data: number[], p: number): number {
    const sorted = [...data].sort((a, b) => a - b);
    return Statistics.quantile(sorted, p / 100);
  }

  // Distribution Analysis
  static zScore(value: number, mean: number, std: number): number {
    return (value - mean) / std;
  }

  static normalize(
    data: number[]
  ): { normalized: number[]; mean: number; std: number } {
    const mean = Statistics.mean(data);
    const std = Statistics.std(data);

    return {
      normalized: data.map((x) => (x - mean) / std),
      mean,
      std,
    };
  }

  static minMaxScale(
    data: number[],
    min = 0,
    max = 1
  ): { scaled: number[]; dataMin: number; dataMax: number } {
    const dataMin = Math.min(...data);
    const dataMax = Math.max(...data);
    const range = dataMax - dataMin;

    return {
      scaled: data.map((x) =>
        range === 0 ? min : min + ((x - dataMin) / range) * (max - min)
      ),
      dataMin,
      dataMax,
    };
  }

  // Correlation
  static correlation(x: number[], y: number[]): number {
    if (x.length !== y.length) throw new Error("ขนาด array ต้องเท่ากัน");

    const n = x.length;
    const meanX = Statistics.mean(x);
    const meanY = Statistics.mean(y);

    const numerator = x.reduce(
      (sum, xi, i) => sum + (xi - meanX) * (y[i] - meanY),
      0
    );

    const denomX = Math.sqrt(
      x.reduce((sum, xi) => sum + (xi - meanX) ** 2, 0)
    );
    const denomY = Math.sqrt(
      y.reduce((sum, yi) => sum + (yi - meanY) ** 2, 0)
    );

    return denomX * denomY === 0 ? 0 : numerator / (denomX * denomY);
  }

  // Summary statistics
  static describe(
    data: number[]
  ): {
    count: number;
    mean: number;
    std: number;
    min: number;
    q1: number;
    median: number;
    q3: number;
    max: number;
  } {
    const sorted = [...data].sort((a, b) => a - b);

    return {
      count: data.length,
      mean: Statistics.mean(data),
      std: Statistics.std(data),
      min: sorted[0],
      q1: Statistics.quantile(sorted, 0.25),
      median: Statistics.median(data),
      q3: Statistics.quantile(sorted, 0.75),
      max: sorted[sorted.length - 1],
    };
  }

  // Outlier Detection
  static detectOutliers(
    data: number[],
    method: "iqr" | "zscore" = "iqr",
    threshold = 1.5
  ): { outliers: number[]; clean: number[] } {
    if (method === "iqr") {
      const sorted = [...data].sort((a, b) => a - b);
      const q1 = Statistics.quantile(sorted, 0.25);
      const q3 = Statistics.quantile(sorted, 0.75);
      const iqr = q3 - q1;
      const lower = q1 - threshold * iqr;
      const upper = q3 + threshold * iqr;

      return {
        outliers: data.filter((x) => x < lower || x > upper),
        clean: data.filter((x) => x >= lower && x <= upper),
      };
    } else {
      // Z-score method
      const mean = Statistics.mean(data);
      const std = Statistics.std(data);

      return {
        outliers: data.filter((x) => Math.abs(Statistics.zScore(x, mean, std)) > threshold),
        clean: data.filter((x) => Math.abs(Statistics.zScore(x, mean, std)) <= threshold),
      };
    }
  }
}

// ตัวอย่างการใช้งาน
const salaries = [45000, 50000, 55000, 60000, 65000, 70000, 45000, 52000, 250000];

console.log("สถิติเงินเดือน:");
const stats = Statistics.describe(salaries);
Object.entries(stats).forEach(([key, value]) => {
  console.log(`  ${key}: ${typeof value === "number" ? value.toLocaleString() : value}`);
});

const outlierResult = Statistics.detectOutliers(salaries, "iqr", 1.5);
console.log("\nค่าผิดปกติ:", outlierResult.outliers);
console.log("ข้อมูลปกติ:", outlierResult.clean);

// Correlation
const x = [1, 2, 3, 4, 5];
const y = [2, 4, 5, 4, 5];
console.log(`\nCorrelation: ${Statistics.correlation(x, y).toFixed(4)}`);
```

---

## 6. Observable Data Streams

### ตัวอย่างที่ 8: RxJS-like Observable

```typescript
// Simple Observable implementation
type Observer<T> = {
  next: (value: T) => void;
  error?: (err: unknown) => void;
  complete?: () => void;
};

type Subscription = {
  unsubscribe: () => void;
};

class Observable<T> {
  constructor(
    private _subscribe: (observer: Observer<T>) => () => void
  ) {}

  subscribe(observer: Observer<T>): Subscription {
    const unsubscribe = this._subscribe(observer);
    return { unsubscribe };
  }

  // Operators
  map<U>(transform: (value: T) => U): Observable<U> {
    return new Observable<U>((observer) => {
      return this._subscribe({
        next: (value) => observer.next(transform(value)),
        error: observer.error,
        complete: observer.complete,
      });
    });
  }

  filter(predicate: (value: T) => boolean): Observable<T> {
    return new Observable<T>((observer) => {
      return this._subscribe({
        next: (value) => {
          if (predicate(value)) observer.next(value);
        },
        error: observer.error,
        complete: observer.complete,
      });
    });
  }

  take(n: number): Observable<T> {
    return new Observable<T>((observer) => {
      let count = 0;
      const sub = this._subscribe({
        next: (value) => {
          if (count < n) {
            observer.next(value);
            count++;
            if (count === n) {
              observer.complete?.();
            }
          }
        },
        error: observer.error,
        complete: observer.complete,
      });
      return sub;
    });
  }

  scan<R>(
    accumulator: (acc: R, value: T, index: number) => R,
    seed: R
  ): Observable<R> {
    return new Observable<R>((observer) => {
      let acc = seed;
      let index = 0;

      return this._subscribe({
        next: (value) => {
          acc = accumulator(acc, value, index++);
          observer.next(acc);
        },
        error: observer.error,
        complete: observer.complete,
      });
    });
  }

  debounceTime(ms: number): Observable<T> {
    return new Observable<T>((observer) => {
      let timer: ReturnType<typeof setTimeout> | null = null;

      return this._subscribe({
        next: (value) => {
          if (timer) clearTimeout(timer);
          timer = setTimeout(() => {
            observer.next(value);
            timer = null;
          }, ms);
        },
        error: observer.error,
        complete: observer.complete,
      });
    });
  }

  // Static methods
  static interval(period: number): Observable<number> {
    return new Observable<number>((observer) => {
      let count = 0;
      const id = setInterval(() => {
        observer.next(count++);
      }, period);

      return () => clearInterval(id);
    });
  }

  static from<T>(values: T[]): Observable<T> {
    return new Observable<T>((observer) => {
      values.forEach((v) => observer.next(v));
      observer.complete?.();
      return () => {};
    });
  }

  static of<T>(...values: T[]): Observable<T> {
    return Observable.from(values);
  }
}

// Data stream processing
interface StockPrice {
  symbol: string;
  price: number;
  volume: number;
  timestamp: Date;
}

// จำลอง stock price stream
function createStockStream(symbol: string): Observable<StockPrice> {
  let price = 100 + Math.random() * 900;

  return Observable.interval(100).map((tick) => {
    // Random walk
    price += (Math.random() - 0.5) * 5;
    price = Math.max(1, price);

    return {
      symbol,
      price: Math.round(price * 100) / 100,
      volume: Math.floor(Math.random() * 1000),
      timestamp: new Date(),
    };
  });
}

// Moving Average indicator
function movingAverage(
  stream: Observable<StockPrice>,
  period: number
): Observable<{ symbol: string; ma: number; price: number }> {
  return stream
    .scan<StockPrice[]>((history, current) => {
      const newHistory = [...history, current];
      return newHistory.slice(-period);
    }, [])
    .filter((history) => history.length === period)
    .map((history) => {
      const avg = history.reduce((sum, h) => sum + h.price, 0) / history.length;
      return {
        symbol: history[history.length - 1].symbol,
        ma: Math.round(avg * 100) / 100,
        price: history[history.length - 1].price,
      };
    });
}

// ตัวอย่างการใช้งาน
const ptttStream = createStockStream("PTT");
const maStream = movingAverage(ptttStream, 5);

const sub = maStream.take(10).subscribe({
  next: (data) => {
    console.log(
      `${data.symbol}: ราคา=${data.price}, MA5=${data.ma}`
    );
  },
  complete: () => console.log("Stream complete"),
});

// Cleanup หลัง 2 วินาที
setTimeout(() => sub.unsubscribe(), 2000);
```

---

## 7. Type-Safe Data Processing

### ตัวอย่างที่ 9: Data Transformation Pipeline

```typescript
// Data transformation types
type Transformer<Input, Output> = (input: Input) => Output;
type AsyncTransformerFn<Input, Output> = (input: Input) => Promise<Output>;
type Validator<T> = (value: T) => string | null;

// Schema-based transformer
interface Schema<T> {
  fields: {
    [K in keyof T]: {
      type: "string" | "number" | "boolean" | "date" | "array" | "object";
      required?: boolean;
      transform?: (value: unknown) => T[K];
      validate?: Validator<T[K]>;
    };
  };
}

class DataTransformer<T> {
  constructor(private schema: Schema<T>) {}

  transform(raw: Record<string, unknown>): { data: T | null; errors: string[] } {
    const errors: string[] = [];
    const result: Partial<T> = {};

    for (const [key, fieldSchema] of Object.entries(this.schema.fields) as [
      keyof T,
      Schema<T>["fields"][keyof T]
    ][]) {
      const rawValue = raw[key as string];

      if (rawValue === undefined || rawValue === null) {
        if (fieldSchema.required) {
          errors.push(`Field "${String(key)}" จำเป็นต้องมีค่า`);
        }
        continue;
      }

      try {
        let value: T[keyof T];

        if (fieldSchema.transform) {
          value = fieldSchema.transform(rawValue);
        } else {
          value = this.convertType(rawValue, fieldSchema.type) as T[keyof T];
        }

        if (fieldSchema.validate) {
          const error = fieldSchema.validate(value);
          if (error) {
            errors.push(`Field "${String(key)}": ${error}`);
            continue;
          }
        }

        result[key] = value;
      } catch (e) {
        errors.push(`Field "${String(key)}": ไม่สามารถแปลงค่าได้ - ${(e as Error).message}`);
      }
    }

    return {
      data: errors.length === 0 ? (result as T) : null,
      errors,
    };
  }

  transformMany(
    rows: Record<string, unknown>[]
  ): { valid: T[]; invalid: Array<{ row: Record<string, unknown>; errors: string[] }> } {
    const valid: T[] = [];
    const invalid: Array<{ row: Record<string, unknown>; errors: string[] }> = [];

    rows.forEach((row) => {
      const result = this.transform(row);
      if (result.data !== null) {
        valid.push(result.data);
      } else {
        invalid.push({ row, errors: result.errors });
      }
    });

    return { valid, invalid };
  }

  private convertType(
    value: unknown,
    type: string
  ): unknown {
    switch (type) {
      case "string":
        return String(value);
      case "number":
        const num = Number(value);
        if (isNaN(num)) throw new Error(`"${value}" ไม่ใช่ตัวเลข`);
        return num;
      case "boolean":
        if (typeof value === "boolean") return value;
        if (value === "true") return true;
        if (value === "false") return false;
        throw new Error(`"${value}" ไม่ใช่ boolean`);
      case "date":
        const date = new Date(value as string);
        if (isNaN(date.getTime())) throw new Error(`"${value}" ไม่ใช่วันที่ที่ถูกต้อง`);
        return date;
      default:
        return value;
    }
  }
}

// ตัวอย่างการใช้งาน
interface CleanedProduct {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
  createdAt: Date;
}

const productTransformer = new DataTransformer<CleanedProduct>({
  fields: {
    id: {
      type: "number",
      required: true,
      validate: (v) => (v > 0 ? null : "ID ต้องมากกว่า 0"),
    },
    name: {
      type: "string",
      required: true,
      validate: (v) => (v.length > 0 ? null : "ชื่อต้องไม่ว่าง"),
    },
    price: {
      type: "number",
      required: true,
      validate: (v) => (v >= 0 ? null : "ราคาต้องไม่ติดลบ"),
    },
    inStock: {
      type: "boolean",
      required: false,
      transform: (v) => v === true || v === "true" || v === 1,
    },
    createdAt: {
      type: "date",
      required: false,
      transform: (v) => (v instanceof Date ? v : new Date(v as string)),
    },
  },
});

const rawProducts = [
  { id: "1", name: "โน้ตบุ๊ค", price: "45000", inStock: "true", createdAt: "2024-01-15" },
  { id: "2", name: "สมาร์ทโฟน", price: "25000", inStock: true, createdAt: "2024-01-20" },
  { id: "invalid", name: "", price: "-1000", inStock: "yes" }, // ข้อมูลผิด
];

const { valid, invalid } = productTransformer.transformMany(rawProducts);
console.log("ข้อมูลที่ถูกต้อง:", valid);
console.log("ข้อมูลที่ผิด:", invalid);
```

---

## 8. Chart.js / D3 กับ TypeScript

### ตัวอย่างที่ 10: Type-Safe Chart Configuration

```typescript
// Type-safe Chart.js configuration
interface ChartDataset {
  label: string;
  data: number[];
  backgroundColor?: string | string[];
  borderColor?: string | string[];
  borderWidth?: number;
  fill?: boolean;
  tension?: number;
}

interface ChartConfig {
  type: "line" | "bar" | "pie" | "doughnut" | "radar" | "scatter";
  data: {
    labels: string[];
    datasets: ChartDataset[];
  };
  options?: {
    responsive?: boolean;
    scales?: {
      y?: {
        beginAtZero?: boolean;
        title?: { display: boolean; text: string };
      };
      x?: {
        title?: { display: boolean; text: string };
      };
    };
    plugins?: {
      legend?: { position?: "top" | "bottom" | "left" | "right" };
      title?: { display: boolean; text: string };
    };
  };
}

// Chart builder
class ChartBuilder {
  private config: Partial<ChartConfig> = {};
  private datasets: ChartDataset[] = [];
  private labels: string[] = [];

  type(chartType: ChartConfig["type"]): this {
    this.config.type = chartType;
    return this;
  }

  withLabels(labels: string[]): this {
    this.labels = labels;
    return this;
  }

  addDataset(dataset: ChartDataset): this {
    this.datasets.push(dataset);
    return this;
  }

  withTitle(title: string): this {
    this.config.options = {
      ...this.config.options,
      plugins: {
        ...this.config.options?.plugins,
        title: { display: true, text: title },
      },
    };
    return this;
  }

  withYAxisTitle(title: string): this {
    this.config.options = {
      ...this.config.options,
      scales: {
        ...this.config.options?.scales,
        y: {
          ...this.config.options?.scales?.y,
          title: { display: true, text: title },
        },
      },
    };
    return this;
  }

  build(): ChartConfig {
    if (!this.config.type) {
      throw new Error("Chart type ต้องถูกระบุ");
    }

    return {
      type: this.config.type,
      data: {
        labels: this.labels,
        datasets: this.datasets,
      },
      options: this.config.options ?? {},
    };
  }
}

// ตัวอย่างการสร้าง chart configs
const monthlySalesChart = new ChartBuilder()
  .type("bar")
  .withLabels(["ม.ค.", "ก.พ.", "มี.ค.", "เม.ย.", "พ.ค.", "มิ.ย."])
  .addDataset({
    label: "ยอดขาย 2024",
    data: [65000, 72000, 68000, 85000, 79000, 92000],
    backgroundColor: "rgba(54, 162, 235, 0.5)",
    borderColor: "rgba(54, 162, 235, 1)",
    borderWidth: 1,
  })
  .addDataset({
    label: "ยอดขาย 2023",
    data: [58000, 65000, 61000, 75000, 70000, 82000],
    backgroundColor: "rgba(255, 99, 132, 0.5)",
    borderColor: "rgba(255, 99, 132, 1)",
    borderWidth: 1,
  })
  .withTitle("ยอดขายรายเดือน (บาท)")
  .withYAxisTitle("ยอดขาย (บาท)")
  .build();

console.log("Chart config:", JSON.stringify(monthlySalesChart, null, 2));

// D3 type definitions (simplified)
interface D3Selection {
  attr: (name: string, value: string | number | ((d: unknown) => string | number)) => D3Selection;
  style: (name: string, value: string | ((d: unknown) => string)) => D3Selection;
  text: (value: string | ((d: unknown) => string)) => D3Selection;
  append: (type: string) => D3Selection;
  data: (data: unknown[]) => D3Selection;
  enter: () => D3Selection;
  on: (event: string, handler: (event: Event, d: unknown) => void) => D3Selection;
}

interface D3Scale {
  domain: (values: [unknown, unknown]) => D3Scale;
  range: (values: [number, number]) => D3Scale;
  (value: unknown): number;
}

// D3 chart builder
function createBarChart(
  data: Array<{ label: string; value: number }>,
  options: {
    width: number;
    height: number;
    margin: { top: number; right: number; bottom: number; left: number };
  }
): string {
  const { width, height, margin } = options;
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;

  const maxValue = Math.max(...data.map((d) => d.value));
  const barWidth = innerWidth / data.length;

  // สร้าง SVG string
  const bars = data.map((d, i) => {
    const barHeight = (d.value / maxValue) * innerHeight;
    const x = i * barWidth;
    const y = innerHeight - barHeight;

    return `
      <g transform="translate(${x}, 0)">
        <rect
          x="${margin.left + 2}"
          y="${margin.top + y}"
          width="${barWidth - 4}"
          height="${barHeight}"
          fill="steelblue"
          opacity="0.8"
        />
        <text
          x="${margin.left + barWidth / 2}"
          y="${margin.top + y - 5}"
          text-anchor="middle"
          font-size="12"
        >${d.value.toLocaleString()}</text>
        <text
          x="${margin.left + barWidth / 2}"
          y="${margin.top + innerHeight + 20}"
          text-anchor="middle"
          font-size="11"
        >${d.label}</text>
      </g>
    `;
  }).join("");

  return `
    <svg width="${width}" height="${height}" xmlns="http://www.w3.org/2000/svg">
      ${bars}
      <line
        x1="${margin.left}"
        y1="${margin.top + innerHeight}"
        x2="${margin.left + innerWidth}"
        y2="${margin.top + innerHeight}"
        stroke="black"
        stroke-width="1"
      />
    </svg>
  `;
}

const salesData = [
  { label: "ม.ค.", value: 65000 },
  { label: "ก.พ.", value: 72000 },
  { label: "มี.ค.", value: 68000 },
  { label: "เม.ย.", value: 85000 },
  { label: "พ.ค.", value: 79000 },
  { label: "มิ.ย.", value: 92000 },
];

const svgChart = createBarChart(salesData, {
  width: 600,
  height: 400,
  margin: { top: 30, right: 30, bottom: 50, left: 50 },
});

console.log("SVG Chart สร้างแล้ว (ขนาด:", svgChart.length, "chars)");
```

---

## สรุป

TypeScript สำหรับ Data Science และ Machine Learning มีความสามารถหลายอย่าง:

1. **TensorFlow.js** - สร้างและฝึก neural networks ใน JavaScript/TypeScript
2. **Type-Safe Tensor Operations** - wrapper ที่ปลอดภัยสำหรับ tensor operations
3. **Data Pipelines** - functional pipeline สำหรับประมวลผลข้อมูล
4. **CSV/JSON Processing** - parse และ validate ข้อมูลจากหลายรูปแบบ
5. **Statistical Operations** - คำนวณสถิติต่างๆ
6. **Observable Streams** - ประมวลผลข้อมูล real-time
7. **Data Transformation** - แปลงและ validate ข้อมูล
8. **Visualization** - สร้าง charts ด้วย Chart.js และ D3

ข้อดีของ TypeScript ใน Data Science:
- Type safety ช่วยจับ bugs ที่เกี่ยวกับ data types
- Good IDE support
- สามารถ share code กับ frontend ได้
- Strong async support สำหรับ data streaming

---

*จบตอนที่ 79 - TypeScript for Data Science & Machine Learning*
