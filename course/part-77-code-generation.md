# ตอนที่ 77: Code Generation (การสร้างโค้ดอัตโนมัติ)

## บทนำ

Code Generation คือกระบวนการสร้าง source code โดยอัตโนมัติจาก schemas, templates, หรือ specifications ต่างๆ ใน TypeScript ecosystem มีเครื่องมือหลายอย่างที่ช่วยให้เราทำ code generation ได้อย่างมีประสิทธิภาพ

---

## 1. Code Generation Overview

### ประโยชน์ของ Code Generation

1. **ลดโค้ดซ้ำ (DRY)** - สร้าง boilerplate อัตโนมัติ
2. **Type Safety** - สร้าง type-safe clients จาก API specs
3. **Consistency** - โค้ดที่สร้างอัตโนมัติมีรูปแบบสม่ำเสมอ
4. **Productivity** - ประหยัดเวลาในการเขียนโค้ดซ้ำๆ
5. **Schema-First Development** - เริ่มจาก schema แล้วสร้าง code

### เครื่องมือหลักๆ

```typescript
// เครื่องมือ Code Generation สำหรับ TypeScript
const tools = {
  // AST manipulation
  "typescript compiler API": "TypeScript compiler API สำหรับ parse/transform AST",
  "ts-morph": "High-level wrapper สำหรับ TypeScript Compiler API",

  // Schema to code
  "json-schema-to-typescript": "แปลง JSON Schema เป็น TypeScript types",
  "@openapitools/openapi-generator-cli": "แปลง OpenAPI spec เป็น client SDK",
  "graphql-code-generator": "แปลง GraphQL schema เป็น TypeScript types",

  // Template-based
  handlebars: "Template engine สำหรับสร้าง code",
  ejs: "Embedded JavaScript templates",
  mustache: "Logic-less templates",

  // ORM/Database
  prisma: "Type-safe database ORM พร้อม code generation",
  "typeorm-model-generator": "แปลง database schema เป็น TypeScript entities",
};
```

---

## 2. TypeScript AST (Abstract Syntax Tree)

### ตัวอย่างที่ 1: อ่าน TypeScript AST

```typescript
import * as ts from "typescript";
import * as fs from "fs";
import * as path from "path";

// ฟังก์ชัน helper สำหรับแสดง AST node
function printAST(node: ts.Node, indent: number = 0): void {
  const prefix = " ".repeat(indent * 2);
  const kind = ts.SyntaxKind[node.kind];
  console.log(`${prefix}${kind}`);
  node.forEachChild((child) => printAST(child, indent + 1));
}

// สร้าง TypeScript program จากโค้ด
function analyzeCode(code: string): void {
  const sourceFile = ts.createSourceFile(
    "temp.ts",
    code,
    ts.ScriptTarget.Latest,
    true
  );

  console.log("=== AST Structure ===");
  printAST(sourceFile);
}

// ตัวอย่างการใช้งาน
const sampleCode = `
interface User {
  id: number;
  name: string;
  email: string;
}

function getUser(id: number): User {
  return { id, name: "สมชาย", email: "somchai@example.com" };
}
`;

analyzeCode(sampleCode);
```

### ตัวอย่างที่ 2: Extract Interface Information จาก AST

```typescript
import * as ts from "typescript";

interface InterfaceInfo {
  name: string;
  properties: Array<{
    name: string;
    type: string;
    optional: boolean;
  }>;
}

function extractInterfaces(sourceCode: string): InterfaceInfo[] {
  const sourceFile = ts.createSourceFile(
    "temp.ts",
    sourceCode,
    ts.ScriptTarget.Latest,
    true
  );

  const interfaces: InterfaceInfo[] = [];

  function visit(node: ts.Node): void {
    if (ts.isInterfaceDeclaration(node)) {
      const interfaceInfo: InterfaceInfo = {
        name: node.name.text,
        properties: [],
      };

      node.members.forEach((member) => {
        if (ts.isPropertySignature(member) && member.name) {
          const name = member.name.getText(sourceFile);
          const typeNode = member.type;
          const type = typeNode
            ? typeNode.getText(sourceFile)
            : "unknown";
          const optional = !!member.questionToken;

          interfaceInfo.properties.push({ name, type, optional });
        }
      });

      interfaces.push(interfaceInfo);
    }

    ts.forEachChild(node, visit);
  }

  visit(sourceFile);
  return interfaces;
}

// ทดสอบ
const code = `
interface Product {
  id: number;
  name: string;
  price: number;
  description?: string;
  category: string;
  inStock: boolean;
}

interface Order {
  id: string;
  customerId: number;
  items: Array<{ productId: number; quantity: number; }>;
  totalAmount: number;
  status: "pending" | "confirmed" | "delivered";
  createdAt: Date;
}
`;

const extracted = extractInterfaces(code);
console.log(JSON.stringify(extracted, null, 2));
```

### ตัวอย่างที่ 3: Transform AST - เพิ่ม Getter Methods

```typescript
import * as ts from "typescript";

// Transform: เพิ่ม getter methods สำหรับ private properties
function addGetters(sourceCode: string): string {
  const sourceFile = ts.createSourceFile(
    "temp.ts",
    sourceCode,
    ts.ScriptTarget.Latest,
    true
  );

  const transformer: ts.TransformerFactory<ts.SourceFile> = (context) => {
    return (node) => {
      function visit(node: ts.Node): ts.Node {
        if (ts.isClassDeclaration(node)) {
          const newMembers: ts.ClassElement[] = [...node.members];

          // หา private properties แล้วสร้าง getter
          node.members.forEach((member) => {
            if (
              ts.isPropertyDeclaration(member) &&
              member.modifiers?.some(
                (m) => m.kind === ts.SyntaxKind.PrivateKeyword
              )
            ) {
              const name = (member.name as ts.Identifier).text;
              const getterName = name.startsWith("_")
                ? name.slice(1)
                : `get${name.charAt(0).toUpperCase()}${name.slice(1)}`;

              // สร้าง getter method
              const getter = ts.factory.createGetAccessorDeclaration(
                [ts.factory.createModifier(ts.SyntaxKind.PublicKeyword)],
                getterName,
                [],
                member.type,
                ts.factory.createBlock([
                  ts.factory.createReturnStatement(
                    ts.factory.createPropertyAccessExpression(
                      ts.factory.createThis(),
                      name
                    )
                  ),
                ])
              );

              newMembers.push(getter);
            }
          });

          return ts.factory.updateClassDeclaration(
            node,
            node.modifiers,
            node.name,
            node.typeParameters,
            node.heritageClauses,
            newMembers
          );
        }

        return ts.visitEachChild(node, visit, context);
      }

      return ts.visitNode(node, visit) as ts.SourceFile;
    };
  };

  const result = ts.transform(sourceFile, [transformer]);
  const printer = ts.createPrinter();
  return printer.printFile(result.transformed[0]);
}

const inputClass = `
class Person {
  private _name: string;
  private _age: number;
  
  constructor(name: string, age: number) {
    this._name = name;
    this._age = age;
  }
}
`;

const transformed = addGetters(inputClass);
console.log("Transformed code:");
console.log(transformed);
```

---

## 3. ts-morph สำหรับ Code Manipulation

### ตัวอย่างที่ 4: ใช้ ts-morph สร้าง TypeScript Files

```typescript
import { Project, StructureKind, Scope } from "ts-morph";

// สร้าง project ใหม่
const project = new Project({
  tsConfigFilePath: "./tsconfig.json",
  skipAddingFilesFromTsConfig: true,
});

// สร้าง interface file
function generateInterfaceFile(
  outputPath: string,
  interfaceName: string,
  properties: Array<{ name: string; type: string; optional?: boolean }>
): void {
  const sourceFile = project.createSourceFile(outputPath, "", {
    overwrite: true,
  });

  // เพิ่ม interface
  sourceFile.addInterface({
    name: interfaceName,
    isExported: true,
    properties: properties.map((prop) => ({
      name: prop.name,
      type: prop.type,
      hasQuestionToken: prop.optional ?? false,
    })),
  });

  // บันทึกไฟล์
  sourceFile.saveSync();
  console.log(`สร้างไฟล์ ${outputPath} สำเร็จ`);
}

// สร้าง class file
function generateClassFile(
  outputPath: string,
  className: string,
  properties: Array<{ name: string; type: string; optional?: boolean }>
): void {
  const sourceFile = project.createSourceFile(outputPath, "", {
    overwrite: true,
  });

  // เพิ่ม class
  const classDecl = sourceFile.addClass({
    name: className,
    isExported: true,
  });

  // เพิ่ม constructor
  const constructorDecl = classDecl.addConstructor({
    parameters: properties.map((prop) => ({
      name: `_${prop.name}`,
      type: prop.type,
      scope: Scope.Private,
      isReadonly: true,
    })),
  });

  // เพิ่ม getter methods
  properties.forEach((prop) => {
    classDecl.addGetAccessor({
      name: prop.name,
      returnType: prop.type,
      scope: Scope.Public,
      statements: [`return this._${prop.name};`],
    });
  });

  // เพิ่ม toJSON method
  classDecl.addMethod({
    name: "toJSON",
    returnType: "Record<string, unknown>",
    statements: [
      `return { ${properties.map((p) => `${p.name}: this._${p.name}`).join(", ")} };`,
    ],
  });

  sourceFile.saveSync();
  console.log(`สร้าง class ${className} สำเร็จ`);
}

// ตัวอย่างการใช้งาน
generateInterfaceFile("./generated/IUser.ts", "IUser", [
  { name: "id", type: "string" },
  { name: "name", type: "string" },
  { name: "email", type: "string" },
  { name: "age", type: "number", optional: true },
  { name: "role", type: '"admin" | "user" | "moderator"' },
]);

generateClassFile("./generated/User.ts", "User", [
  { name: "id", type: "string" },
  { name: "name", type: "string" },
  { name: "email", type: "string" },
]);
```

### ตัวอย่างที่ 5: Refactoring Tool ด้วย ts-morph

```typescript
import { Project, SyntaxKind } from "ts-morph";

// Rename all occurrences of a symbol
function renameSymbol(
  projectPath: string,
  oldName: string,
  newName: string
): void {
  const project = new Project({
    tsConfigFilePath: `${projectPath}/tsconfig.json`,
  });

  const sourceFiles = project.getSourceFiles();
  let changesCount = 0;

  sourceFiles.forEach((sourceFile) => {
    // หา identifiers ทั้งหมดที่ตรงกับ oldName
    const identifiers = sourceFile
      .getDescendantsOfKind(SyntaxKind.Identifier)
      .filter((id) => id.getText() === oldName);

    if (identifiers.length > 0) {
      identifiers.forEach((id) => {
        id.rename(newName);
        changesCount++;
      });

      sourceFile.saveSync();
      console.log(
        `แก้ไข ${identifiers.length} จุด ใน ${sourceFile.getFilePath()}`
      );
    }
  });

  console.log(`รวมแก้ไข ${changesCount} จุดทั้งหมด`);
}

// Extract method refactoring
function extractRepeatedCode(sourceCode: string): string {
  const project = new Project();
  const sourceFile = project.createSourceFile("temp.ts", sourceCode);

  // หา string literals ที่ซ้ำกัน
  const stringLiterals = new Map<string, number>();

  sourceFile
    .getDescendantsOfKind(SyntaxKind.StringLiteral)
    .forEach((literal) => {
      const value = literal.getLiteralValue();
      stringLiterals.set(value, (stringLiterals.get(value) ?? 0) + 1);
    });

  // แสดงรายการ strings ที่ซ้ำ
  console.log("String literals ที่ซ้ำกัน:");
  stringLiterals.forEach((count, value) => {
    if (count > 1) {
      console.log(`  "${value}" พบ ${count} ครั้ง`);
    }
  });

  return sourceFile.getFullText();
}

// Import organizer
function organizeImports(sourceCode: string): string {
  const project = new Project();
  const sourceFile = project.createSourceFile("temp.ts", sourceCode);

  sourceFile.organizeImports();
  return sourceFile.getFullText();
}

// ตัวอย่างการใช้ organizeImports
const messyCode = `
import { z } from 'zod';
import express from 'express';
import { User } from './models/User';
import * as fs from 'fs';
import { readFile } from 'fs/promises';
import { Router } from 'express';
import { z as zod } from 'zod';

const app = express();
const router = Router();
`;

const organized = organizeImports(messyCode);
console.log("Organized imports:");
console.log(organized);
```

---

## 4. Generating TypeScript จาก JSON Schema

### ตัวอย่างที่ 6: JSON Schema Parser และ Type Generator

```typescript
interface JSONSchema {
  type?: string | string[];
  properties?: Record<string, JSONSchema>;
  required?: string[];
  items?: JSONSchema;
  enum?: unknown[];
  $ref?: string;
  allOf?: JSONSchema[];
  anyOf?: JSONSchema[];
  oneOf?: JSONSchema[];
  description?: string;
  title?: string;
  format?: string;
  minimum?: number;
  maximum?: number;
  minLength?: number;
  maxLength?: number;
  pattern?: string;
  additionalProperties?: boolean | JSONSchema;
  definitions?: Record<string, JSONSchema>;
  $defs?: Record<string, JSONSchema>;
}

class TypeScriptGenerator {
  private generatedTypes = new Map<string, string>();
  private schema: JSONSchema;

  constructor(schema: JSONSchema) {
    this.schema = schema;
  }

  generate(typeName: string = "Root"): string {
    const typeStr = this.generateType(this.schema, typeName);
    const definitions = this.generateDefinitions();

    return [definitions, typeStr].filter(Boolean).join("\n\n");
  }

  private generateDefinitions(): string {
    const defs = this.schema.$defs ?? this.schema.definitions ?? {};
    return Object.entries(defs)
      .map(([name, schema]) => {
        const typeName = this.toPascalCase(name);
        return this.generateType(schema, typeName);
      })
      .join("\n\n");
  }

  private generateType(schema: JSONSchema, name: string): string {
    if (schema.$ref) {
      const refName = this.resolveRef(schema.$ref);
      return `type ${name} = ${refName};`;
    }

    if (schema.enum) {
      const values = schema.enum
        .map((v) => (typeof v === "string" ? `"${v}"` : String(v)))
        .join(" | ");
      return `type ${name} = ${values};`;
    }

    if (schema.oneOf || schema.anyOf) {
      const variants = (schema.oneOf ?? schema.anyOf ?? [])
        .map((s, i) => this.generateInlineType(s))
        .join(" | ");
      return `type ${name} = ${variants};`;
    }

    if (schema.allOf) {
      const parts = schema.allOf.map((s) => this.generateInlineType(s)).join(" & ");
      return `type ${name} = ${parts};`;
    }

    if (schema.type === "object" || schema.properties) {
      return this.generateInterface(schema, name);
    }

    const inlineType = this.generateInlineType(schema);
    return `type ${name} = ${inlineType};`;
  }

  private generateInterface(schema: JSONSchema, name: string): string {
    const properties = schema.properties ?? {};
    const required = new Set(schema.required ?? []);

    const lines: string[] = [];

    if (schema.description) {
      lines.push(`/** ${schema.description} */`);
    }

    lines.push(`export interface ${name} {`);

    Object.entries(properties).forEach(([propName, propSchema]) => {
      const isRequired = required.has(propName);
      const optional = isRequired ? "" : "?";
      const propType = this.generateInlineType(propSchema);

      if (propSchema.description) {
        lines.push(`  /** ${propSchema.description} */`);
      }

      lines.push(`  ${propName}${optional}: ${propType};`);
    });

    if (schema.additionalProperties === true) {
      lines.push(`  [key: string]: unknown;`);
    } else if (
      schema.additionalProperties &&
      typeof schema.additionalProperties === "object"
    ) {
      const valueType = this.generateInlineType(schema.additionalProperties);
      lines.push(`  [key: string]: ${valueType};`);
    }

    lines.push("}");
    return lines.join("\n");
  }

  private generateInlineType(schema: JSONSchema): string {
    if (schema.$ref) {
      return this.resolveRef(schema.$ref);
    }

    if (schema.enum) {
      return schema.enum
        .map((v) => (typeof v === "string" ? `"${v}"` : String(v)))
        .join(" | ");
    }

    if (schema.oneOf || schema.anyOf) {
      return (schema.oneOf ?? schema.anyOf ?? [])
        .map((s) => this.generateInlineType(s))
        .join(" | ");
    }

    if (schema.allOf) {
      return schema.allOf.map((s) => this.generateInlineType(s)).join(" & ");
    }

    const types = Array.isArray(schema.type) ? schema.type : [schema.type];

    const typeStrings = types
      .filter(Boolean)
      .map((t) => {
        switch (t) {
          case "string":
            if (schema.format === "date-time") return "Date";
            if (schema.format === "date") return "string"; // ISO date string
            return "string";
          case "number":
          case "integer":
            return "number";
          case "boolean":
            return "boolean";
          case "null":
            return "null";
          case "array":
            if (schema.items) {
              return `Array<${this.generateInlineType(schema.items)}>`;
            }
            return "unknown[]";
          case "object":
            if (schema.properties) {
              const props = Object.entries(schema.properties)
                .map(([k, v]) => `${k}: ${this.generateInlineType(v)}`)
                .join("; ");
              return `{ ${props} }`;
            }
            return "Record<string, unknown>";
          default:
            return "unknown";
        }
      });

    return typeStrings.length === 0
      ? "unknown"
      : typeStrings.join(" | ");
  }

  private resolveRef(ref: string): string {
    const name = ref.split("/").pop() ?? "";
    return this.toPascalCase(name);
  }

  private toPascalCase(str: string): string {
    return str
      .split(/[-_\s]/)
      .map((part) => part.charAt(0).toUpperCase() + part.slice(1))
      .join("");
  }
}

// ตัวอย่างการใช้งาน
const userSchema: JSONSchema = {
  title: "User",
  description: "ข้อมูลผู้ใช้งาน",
  type: "object",
  required: ["id", "name", "email"],
  properties: {
    id: {
      type: "integer",
      description: "รหัสผู้ใช้",
    },
    name: {
      type: "string",
      description: "ชื่อผู้ใช้",
      minLength: 1,
      maxLength: 100,
    },
    email: {
      type: "string",
      format: "email",
      description: "อีเมล",
    },
    age: {
      type: ["integer", "null"],
      description: "อายุ",
      minimum: 0,
      maximum: 150,
    },
    role: {
      type: "string",
      enum: ["admin", "user", "moderator"],
      description: "บทบาท",
    },
    createdAt: {
      type: "string",
      format: "date-time",
      description: "วันที่สร้าง",
    },
    address: {
      type: "object",
      properties: {
        street: { type: "string" },
        city: { type: "string" },
        country: { type: "string" },
        zipCode: { type: "string" },
      },
    },
    tags: {
      type: "array",
      items: { type: "string" },
    },
  },
};

const generator = new TypeScriptGenerator(userSchema);
const generated = generator.generate("User");
console.log("Generated TypeScript:");
console.log(generated);
```

---

## 5. Generating จาก OpenAPI Spec

### ตัวอย่างที่ 7: OpenAPI to TypeScript Generator

```typescript
interface OpenAPISpec {
  openapi: string;
  info: {
    title: string;
    version: string;
    description?: string;
  };
  paths: Record<string, PathItem>;
  components?: {
    schemas?: Record<string, JSONSchema>;
    parameters?: Record<string, unknown>;
    responses?: Record<string, unknown>;
  };
}

interface PathItem {
  get?: Operation;
  post?: Operation;
  put?: Operation;
  delete?: Operation;
  patch?: Operation;
}

interface Operation {
  operationId?: string;
  summary?: string;
  description?: string;
  tags?: string[];
  parameters?: Parameter[];
  requestBody?: RequestBody;
  responses: Record<string, Response>;
}

interface Parameter {
  name: string;
  in: "query" | "path" | "header" | "cookie";
  required?: boolean;
  schema?: JSONSchema;
  description?: string;
}

interface RequestBody {
  required?: boolean;
  content: Record<string, { schema?: JSONSchema }>;
}

interface Response {
  description: string;
  content?: Record<string, { schema?: JSONSchema }>;
}

class OpenAPITypeScriptGenerator {
  private spec: OpenAPISpec;
  private generatedSchemas = new Map<string, string>();

  constructor(spec: OpenAPISpec) {
    this.spec = spec;
  }

  generateAll(): string {
    const parts: string[] = [];

    // Header comment
    parts.push(`// Generated from ${this.spec.info.title} v${this.spec.info.version}`);
    parts.push(`// DO NOT EDIT - This file is auto-generated`);
    parts.push("");

    // Generate schema types
    if (this.spec.components?.schemas) {
      parts.push("// =============");
      parts.push("// Schema Types");
      parts.push("// =============");
      parts.push("");

      Object.entries(this.spec.components.schemas).forEach(([name, schema]) => {
        const generated = this.generateSchemaType(name, schema);
        parts.push(generated);
        parts.push("");
      });
    }

    // Generate API client
    parts.push("// ==========");
    parts.push("// API Client");
    parts.push("// ==========");
    parts.push("");
    parts.push(this.generateAPIClient());

    return parts.join("\n");
  }

  private generateSchemaType(name: string, schema: JSONSchema): string {
    const gen = new TypeScriptGenerator(schema);
    return gen.generate(name);
  }

  private generateAPIClient(): string {
    const lines: string[] = [];

    lines.push(`export class APIClient {`);
    lines.push(`  private baseUrl: string;`);
    lines.push(`  private headers: Record<string, string>;`);
    lines.push(``);
    lines.push(`  constructor(baseUrl: string, headers: Record<string, string> = {}) {`);
    lines.push(`    this.baseUrl = baseUrl;`);
    lines.push(`    this.headers = headers;`);
    lines.push(`  }`);
    lines.push(``);

    // Generate methods for each path
    Object.entries(this.spec.paths).forEach(([path, pathItem]) => {
      const methods = ["get", "post", "put", "delete", "patch"] as const;

      methods.forEach((method) => {
        const operation = pathItem[method];
        if (!operation) return;

        const methodCode = this.generateMethod(path, method, operation);
        lines.push(methodCode);
        lines.push("");
      });
    });

    // Add fetch helper
    lines.push(`  private async request<T>(method: string, path: string, options: {`);
    lines.push(`    params?: Record<string, string>;`);
    lines.push(`    body?: unknown;`);
    lines.push(`  } = {}): Promise<T> {`);
    lines.push(`    const url = new URL(path, this.baseUrl);`);
    lines.push(`    if (options.params) {`);
    lines.push(`      Object.entries(options.params).forEach(([k, v]) => {`);
    lines.push(`        url.searchParams.set(k, v);`);
    lines.push(`      });`);
    lines.push(`    }`);
    lines.push(``);
    lines.push(`    const response = await fetch(url.toString(), {`);
    lines.push(`      method,`);
    lines.push(`      headers: {`);
    lines.push(`        "Content-Type": "application/json",`);
    lines.push(`        ...this.headers,`);
    lines.push(`      },`);
    lines.push(`      body: options.body ? JSON.stringify(options.body) : undefined,`);
    lines.push(`    });`);
    lines.push(``);
    lines.push(`    if (!response.ok) {`);
    lines.push(`      throw new Error(\`HTTP \${response.status}: \${response.statusText}\`);`);
    lines.push(`    }`);
    lines.push(``);
    lines.push(`    return response.json();`);
    lines.push(`  }`);
    lines.push(`}`);

    return lines.join("\n");
  }

  private generateMethod(
    path: string,
    method: string,
    operation: Operation
  ): string {
    const operationId =
      operation.operationId ?? this.generateOperationId(method, path);
    const methodName = this.toCamelCase(operationId);

    const pathParams = (operation.parameters ?? []).filter(
      (p) => p.in === "path"
    );
    const queryParams = (operation.parameters ?? []).filter(
      (p) => p.in === "query"
    );

    const lines: string[] = [];

    if (operation.summary) {
      lines.push(`  /** ${operation.summary} */`);
    }

    // Build parameters
    const params: string[] = [];

    pathParams.forEach((p) => {
      const type = this.getParamType(p);
      const optional = p.required ? "" : "?";
      params.push(`${p.name}${optional}: ${type}`);
    });

    if (queryParams.length > 0) {
      const queryType = queryParams
        .map((p) => {
          const type = this.getParamType(p);
          const optional = p.required ? "" : "?";
          return `${p.name}${optional}: ${type}`;
        })
        .join("; ");
      params.push(`query?: { ${queryType} }`);
    }

    if (operation.requestBody) {
      const bodySchema =
        operation.requestBody.content["application/json"]?.schema;
      const bodyType = bodySchema ? this.schemaToType(bodySchema) : "unknown";
      params.push(`body?: ${bodyType}`);
    }

    // Return type
    const successResponse = operation.responses["200"] ?? operation.responses["201"];
    const returnType = successResponse?.content?.["application/json"]?.schema
      ? this.schemaToType(successResponse.content["application/json"].schema)
      : "void";

    // Build path with params
    const interpolatedPath = path.replace(/\{(\w+)\}/g, "${$1}");

    lines.push(`  async ${methodName}(${params.join(", ")}): Promise<${returnType}> {`);
    lines.push(
      `    return this.request<${returnType}>("${method.toUpperCase()}", \`${interpolatedPath}\`, {`
    );

    if (queryParams.length > 0) {
      lines.push(`      params: query as Record<string, string>,`);
    }

    if (operation.requestBody) {
      lines.push(`      body,`);
    }

    lines.push(`    });`);
    lines.push(`  }`);

    return lines.join("\n");
  }

  private getParamType(param: Parameter): string {
    if (!param.schema) return "string";
    return this.schemaToType(param.schema);
  }

  private schemaToType(schema: JSONSchema): string {
    if (schema.$ref) {
      return schema.$ref.split("/").pop() ?? "unknown";
    }

    switch (schema.type) {
      case "string": return "string";
      case "number":
      case "integer": return "number";
      case "boolean": return "boolean";
      case "array":
        return schema.items
          ? `${this.schemaToType(schema.items)}[]`
          : "unknown[]";
      case "object": return "Record<string, unknown>";
      default: return "unknown";
    }
  }

  private generateOperationId(method: string, path: string): string {
    const parts = path
      .split("/")
      .filter(Boolean)
      .map((p) => (p.startsWith("{") ? "by" + p.slice(1, -1) : p));

    return method + parts.map((p) => p.charAt(0).toUpperCase() + p.slice(1)).join("");
  }

  private toCamelCase(str: string): string {
    return str.charAt(0).toLowerCase() + str.slice(1);
  }
}

// ตัวอย่าง OpenAPI spec
const petStoreSpec: OpenAPISpec = {
  openapi: "3.0.0",
  info: {
    title: "Pet Store API",
    version: "1.0.0",
    description: "API สำหรับจัดการสัตว์เลี้ยง",
  },
  paths: {
    "/pets": {
      get: {
        operationId: "listPets",
        summary: "ดูรายการสัตว์เลี้ยงทั้งหมด",
        parameters: [
          {
            name: "limit",
            in: "query",
            schema: { type: "integer" },
            description: "จำนวนที่ต้องการ",
          },
          {
            name: "species",
            in: "query",
            schema: { type: "string" },
          },
        ],
        responses: {
          "200": {
            description: "รายการสัตว์เลี้ยง",
            content: {
              "application/json": {
                schema: { type: "array", items: { $ref: "#/components/schemas/Pet" } },
              },
            },
          },
        },
      },
      post: {
        operationId: "createPet",
        summary: "เพิ่มสัตว์เลี้ยงใหม่",
        requestBody: {
          required: true,
          content: {
            "application/json": {
              schema: { $ref: "#/components/schemas/NewPet" },
            },
          },
        },
        responses: {
          "201": {
            description: "สัตว์เลี้ยงที่สร้างแล้ว",
            content: {
              "application/json": {
                schema: { $ref: "#/components/schemas/Pet" },
              },
            },
          },
        },
      },
    },
    "/pets/{petId}": {
      get: {
        operationId: "getPet",
        summary: "ดูข้อมูลสัตว์เลี้ยง",
        parameters: [
          {
            name: "petId",
            in: "path",
            required: true,
            schema: { type: "integer" },
          },
        ],
        responses: {
          "200": {
            description: "ข้อมูลสัตว์เลี้ยง",
            content: {
              "application/json": {
                schema: { $ref: "#/components/schemas/Pet" },
              },
            },
          },
        },
      },
    },
  },
  components: {
    schemas: {
      Pet: {
        type: "object",
        required: ["id", "name", "species"],
        properties: {
          id: { type: "integer", description: "รหัสสัตว์เลี้ยง" },
          name: { type: "string", description: "ชื่อสัตว์เลี้ยง" },
          species: { type: "string", description: "ประเภทสัตว์" },
          age: { type: "integer", description: "อายุ" },
          ownerId: { type: "integer", description: "รหัสเจ้าของ" },
        },
      },
      NewPet: {
        type: "object",
        required: ["name", "species"],
        properties: {
          name: { type: "string" },
          species: { type: "string" },
          age: { type: "integer" },
          ownerId: { type: "integer" },
        },
      },
    },
  },
};

const apiGenerator = new OpenAPITypeScriptGenerator(petStoreSpec);
console.log(apiGenerator.generateAll());
```

---

## 6. Handlebars Templates สำหรับ Code Generation

### ตัวอย่างที่ 8: Handlebars Code Generator

```typescript
// ติดตั้ง: npm install handlebars
import Handlebars from "handlebars";
import * as fs from "fs";
import * as path from "path";

// Register custom helpers
Handlebars.registerHelper("toCamelCase", (str: string) => {
  return str.charAt(0).toLowerCase() + str.slice(1).replace(/_(\w)/g, (_, c) => c.toUpperCase());
});

Handlebars.registerHelper("toPascalCase", (str: string) => {
  return str.replace(/(^|_)(\w)/g, (_, _sep, c) => c.toUpperCase());
});

Handlebars.registerHelper("toSnakeCase", (str: string) => {
  return str.replace(/([A-Z])/g, "_$1").toLowerCase().replace(/^_/, "");
});

Handlebars.registerHelper("pluralize", (str: string) => {
  if (str.endsWith("y")) return str.slice(0, -1) + "ies";
  if (str.endsWith("s")) return str + "es";
  return str + "s";
});

Handlebars.registerHelper("eq", (a: unknown, b: unknown) => a === b);
Handlebars.registerHelper("or", (a: unknown, b: unknown) => a || b);
Handlebars.registerHelper("and", (a: unknown, b: unknown) => a && b);
Handlebars.registerHelper("not", (a: unknown) => !a);

// Template สำหรับ Repository class
const repositoryTemplate = `
import { {{toPascalCase entityName}} } from '../entities/{{entityName}}';
import { Repository, EntityManager } from 'typeorm';

export class {{toPascalCase entityName}}Repository {
  constructor(private manager: EntityManager) {}

  async findById(id: {{idType}}): Promise<{{toPascalCase entityName}} | null> {
    return this.manager.findOne({{toPascalCase entityName}}, { where: { id } });
  }

  async findAll(options?: {
    limit?: number;
    offset?: number;
    {{#each filterFields}}
    {{name}}?: {{type}};
    {{/each}}
  }): Promise<{{toPascalCase entityName}}[]> {
    const qb = this.manager.createQueryBuilder({{toPascalCase entityName}}, '{{toCamelCase entityName}}');
    
    {{#each filterFields}}
    if (options?.{{name}} !== undefined) {
      qb.andWhere('{{../toCamelCase ../entityName}}.{{name}} = :{{name}}', { {{name}}: options.{{name}} });
    }
    {{/each}}
    
    if (options?.limit) qb.take(options.limit);
    if (options?.offset) qb.skip(options.offset);
    
    return qb.getMany();
  }

  async create(data: Omit<{{toPascalCase entityName}}, 'id' | 'createdAt' | 'updatedAt'>): Promise<{{toPascalCase entityName}}> {
    const entity = this.manager.create({{toPascalCase entityName}}, data);
    return this.manager.save(entity);
  }

  async update(id: {{idType}}, data: Partial<Omit<{{toPascalCase entityName}}, 'id'>>): Promise<{{toPascalCase entityName}} | null> {
    await this.manager.update({{toPascalCase entityName}}, id, data);
    return this.findById(id);
  }

  async delete(id: {{idType}}): Promise<boolean> {
    const result = await this.manager.delete({{toPascalCase entityName}}, id);
    return (result.affected ?? 0) > 0;
  }

  async count(filter?: Partial<{{toPascalCase entityName}}>): Promise<number> {
    return this.manager.count({{toPascalCase entityName}}, { where: filter });
  }
}
`.trim();

// Compile template
const repositoryTemplateCompiled = Handlebars.compile(repositoryTemplate);

// ข้อมูล entity
const productEntity = {
  entityName: "product",
  idType: "number",
  filterFields: [
    { name: "category", type: "string" },
    { name: "inStock", type: "boolean" },
    { name: "minPrice", type: "number" },
    { name: "maxPrice", type: "number" },
  ],
};

// สร้าง code
const generatedRepository = repositoryTemplateCompiled(productEntity);
console.log("Generated Repository:");
console.log(generatedRepository);
```

### ตัวอย่างที่ 9: CRUD API Generator

```typescript
const apiRouterTemplate = `
import express, { Router, Request, Response } from 'express';
import { {{toPascalCase entityName}}Service } from '../services/{{entityName}}Service';
import { validate{{toPascalCase entityName}} } from '../validators/{{entityName}}Validator';

const router = Router();
const service = new {{toPascalCase entityName}}Service();

// GET /{{pluralize entityName}}
router.get('/', async (req: Request, res: Response) => {
  try {
    const {
      limit = '10',
      offset = '0',
      {{#each queryParams}}
      {{name}},
      {{/each}}
    } = req.query as Record<string, string>;

    const items = await service.findAll({
      limit: parseInt(limit),
      offset: parseInt(offset),
      {{#each queryParams}}
      {{#if (eq type "number")}}
      {{name}}: {{name}} ? parseFloat({{name}}) : undefined,
      {{else}}
      {{name}},
      {{/if}}
      {{/each}}
    });

    res.json({ data: items, count: items.length });
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// GET /{{pluralize entityName}}/:id
router.get('/:id', async (req: Request, res: Response) => {
  try {
    const id = {{#if (eq idType "number")}}parseInt(req.params.id){{else}}req.params.id{{/if}};
    const item = await service.findById(id);
    
    if (!item) {
      return res.status(404).json({ error: '{{toPascalCase entityName}} ไม่พบ' });
    }
    
    res.json(item);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// POST /{{pluralize entityName}}
router.post('/', async (req: Request, res: Response) => {
  try {
    const validation = validate{{toPascalCase entityName}}(req.body);
    if (!validation.success) {
      return res.status(400).json({ errors: validation.errors });
    }
    
    const item = await service.create(req.body);
    res.status(201).json(item);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// PUT /{{pluralize entityName}}/:id
router.put('/:id', async (req: Request, res: Response) => {
  try {
    const id = {{#if (eq idType "number")}}parseInt(req.params.id){{else}}req.params.id{{/if}};
    const item = await service.update(id, req.body);
    
    if (!item) {
      return res.status(404).json({ error: '{{toPascalCase entityName}} ไม่พบ' });
    }
    
    res.json(item);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// DELETE /{{pluralize entityName}}/:id
router.delete('/:id', async (req: Request, res: Response) => {
  try {
    const id = {{#if (eq idType "number")}}parseInt(req.params.id){{else}}req.params.id{{/if}};
    const deleted = await service.delete(id);
    
    if (!deleted) {
      return res.status(404).json({ error: '{{toPascalCase entityName}} ไม่พบ' });
    }
    
    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
`.trim();

const routerTemplateCompiled = Handlebars.compile(apiRouterTemplate);

const userEntityConfig = {
  entityName: "user",
  idType: "number",
  queryParams: [
    { name: "role", type: "string" },
    { name: "isActive", type: "boolean" },
    { name: "search", type: "string" },
  ],
};

const generatedRouter = routerTemplateCompiled(userEntityConfig);
console.log("\nGenerated API Router:");
console.log(generatedRouter);
```

---

## 7. Prisma-like Schema Processors

### ตัวอย่างที่ 10: Schema DSL Parser

```typescript
// Custom schema DSL parser คล้าย Prisma
interface FieldDefinition {
  name: string;
  type: string;
  isOptional: boolean;
  isArray: boolean;
  isPrimaryKey: boolean;
  isUnique: boolean;
  hasDefault: boolean;
  defaultValue?: unknown;
  isRelation: boolean;
  relationTarget?: string;
  attributes: string[];
}

interface ModelDefinition {
  name: string;
  fields: FieldDefinition[];
  tableName: string;
}

class SchemaParser {
  private static readonly PRIMITIVE_TYPES = new Set([
    "String", "Int", "Float", "Boolean", "DateTime", "Json", "Bytes",
  ]);

  parse(schemaText: string): ModelDefinition[] {
    const models: ModelDefinition[] = [];
    const modelRegex = /model\s+(\w+)\s*\{([^}]+)\}/g;
    let match;

    while ((match = modelRegex.exec(schemaText)) !== null) {
      const modelName = match[1];
      const fieldsText = match[2];
      const fields = this.parseFields(fieldsText);

      models.push({
        name: modelName,
        fields,
        tableName: this.toSnakeCase(modelName) + "s",
      });
    }

    return models;
  }

  private parseFields(fieldsText: string): FieldDefinition[] {
    const fields: FieldDefinition[] = [];
    const lines = fieldsText.split("\n").map((l) => l.trim()).filter(Boolean);

    lines.forEach((line) => {
      if (line.startsWith("//") || line.startsWith("@@")) return;

      const field = this.parseField(line);
      if (field) fields.push(field);
    });

    return fields;
  }

  private parseField(line: string): FieldDefinition | null {
    const fieldRegex = /^(\w+)\s+(\w+)(\[\])?(\?)?(.*)$/;
    const match = fieldRegex.exec(line);
    if (!match) return null;

    const [, name, type, isArrayStr, isOptionalStr, attributesStr] = match;
    const attributes = this.parseAttributes(attributesStr ?? "");

    return {
      name,
      type: type,
      isOptional: !!isOptionalStr,
      isArray: !!isArrayStr,
      isPrimaryKey: attributes.includes("@id"),
      isUnique: attributes.includes("@unique"),
      hasDefault: attributes.some((a) => a.startsWith("@default")),
      defaultValue: this.extractDefault(attributes),
      isRelation: !SchemaParser.PRIMITIVE_TYPES.has(type),
      relationTarget: !SchemaParser.PRIMITIVE_TYPES.has(type) ? type : undefined,
      attributes,
    };
  }

  private parseAttributes(str: string): string[] {
    const attrRegex = /@\w+(\([^)]*\))?/g;
    const attrs: string[] = [];
    let match;

    while ((match = attrRegex.exec(str)) !== null) {
      attrs.push(match[0]);
    }

    return attrs;
  }

  private extractDefault(attributes: string[]): unknown {
    const defaultAttr = attributes.find((a) => a.startsWith("@default"));
    if (!defaultAttr) return undefined;

    const match = /\((.+)\)/.exec(defaultAttr);
    if (!match) return undefined;

    const value = match[1];
    if (value === "true") return true;
    if (value === "false") return false;
    if (/^\d+$/.test(value)) return parseInt(value);
    if (/^\d+\.\d+$/.test(value)) return parseFloat(value);
    if (value.startsWith('"') && value.endsWith('"')) return value.slice(1, -1);
    return value;
  }

  private toSnakeCase(str: string): string {
    return str.replace(/([A-Z])/g, "_$1").toLowerCase().replace(/^_/, "");
  }
}

// Generator จาก schema
class TypeScriptFromSchema {
  generate(models: ModelDefinition[]): string {
    return models.map((model) => this.generateModel(model)).join("\n\n");
  }

  private generateModel(model: ModelDefinition): string {
    const lines: string[] = [];

    lines.push(`// ${model.name} - table: ${model.tableName}`);
    lines.push(`export interface ${model.name} {`);

    model.fields
      .filter((f) => !f.isRelation)
      .forEach((field) => {
        const optional = field.isOptional ? "?" : "";
        const type = this.mapToTSType(field);
        const arrayStr = field.isArray ? "[]" : "";
        lines.push(`  ${field.name}${optional}: ${type}${arrayStr};`);
      });

    lines.push("}");

    // สร้าง create input type
    lines.push("");
    lines.push(`export type Create${model.name}Input = Omit<${model.name}, 'id' | 'createdAt' | 'updatedAt'>;`);
    lines.push(`export type Update${model.name}Input = Partial<Create${model.name}Input>;`);

    return lines.join("\n");
  }

  private mapToTSType(field: FieldDefinition): string {
    switch (field.type) {
      case "String": return "string";
      case "Int":
      case "Float": return "number";
      case "Boolean": return "boolean";
      case "DateTime": return "Date";
      case "Json": return "Record<string, unknown>";
      case "Bytes": return "Buffer";
      default: return field.type;
    }
  }
}

// ตัวอย่าง schema
const schemaText = `
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  role      String   @default("user")
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
  profile   Profile?
}

model Post {
  id          Int      @id @default(autoincrement())
  title       String
  content     String?
  published   Boolean  @default(false)
  authorId    Int
  author      User
  createdAt   DateTime @default(now())
}

model Profile {
  id     Int    @id @default(autoincrement())
  bio    String?
  userId Int    @unique
  user   User
}
`;

const parser = new SchemaParser();
const models = parser.parse(schemaText);

console.log("Parsed models:", JSON.stringify(models, null, 2));

const tsGen = new TypeScriptFromSchema();
console.log("\nGenerated TypeScript:");
console.log(tsGen.generate(models));
```

---

## 8. Code Refactoring Tools

### ตัวอย่างที่ 11: Automatic Import Organizer

```typescript
interface ImportStatement {
  defaultImport?: string;
  namedImports: string[];
  namespaceImport?: string;
  source: string;
  isTypeOnly: boolean;
}

function parseImports(code: string): ImportStatement[] {
  const importRegex =
    /import\s+(?:(type)\s+)?(?:(\w+)\s*,\s*)?(?:\{([^}]*)\}|(\*\s+as\s+\w+))?\s+from\s+['"]([^'"]+)['"]/g;

  const imports: ImportStatement[] = [];
  let match;

  while ((match = importRegex.exec(code)) !== null) {
    const [, typeKeyword, defaultImport, namedStr, namespaceStr, source] =
      match;

    const namedImports = namedStr
      ? namedStr
          .split(",")
          .map((s) => s.trim())
          .filter(Boolean)
      : [];

    const namespaceImport = namespaceStr
      ? namespaceStr.replace("* as ", "").trim()
      : undefined;

    imports.push({
      defaultImport: defaultImport?.trim() || undefined,
      namedImports,
      namespaceImport,
      source,
      isTypeOnly: !!typeKeyword,
    });
  }

  return imports;
}

function organizeImports(imports: ImportStatement[]): ImportStatement[] {
  // แยก imports ตามประเภท
  const nodeBuiltins = imports.filter((i) =>
    ["fs", "path", "os", "crypto", "http", "https", "events", "stream", "util"].includes(
      i.source
    )
  );

  const externalPackages = imports.filter(
    (i) =>
      !i.source.startsWith(".") &&
      !i.source.startsWith("/") &&
      !nodeBuiltins.includes(i)
  );

  const localImports = imports.filter(
    (i) => i.source.startsWith(".") || i.source.startsWith("/")
  );

  // เรียงแต่ละกลุ่มตาม alphabet
  const sortFn = (a: ImportStatement, b: ImportStatement) =>
    a.source.localeCompare(b.source);

  return [
    ...nodeBuiltins.sort(sortFn),
    ...externalPackages.sort(sortFn),
    ...localImports.sort(sortFn),
  ];
}

function renderImports(imports: ImportStatement[]): string {
  return imports
    .map((imp) => {
      const typeStr = imp.isTypeOnly ? "type " : "";
      const parts: string[] = [];

      if (imp.defaultImport) parts.push(imp.defaultImport);
      if (imp.namespaceImport) parts.push(`* as ${imp.namespaceImport}`);
      if (imp.namedImports.length > 0) {
        if (imp.namedImports.length <= 3) {
          parts.push(`{ ${imp.namedImports.join(", ")} }`);
        } else {
          const named = imp.namedImports.map((n) => `  ${n}`).join(",\n");
          parts.push(`{\n${named},\n}`);
        }
      }

      return `import ${typeStr}${parts.join(", ")} from '${imp.source}';`;
    })
    .join("\n");
}

// ตัวอย่างการใช้งาน
const messyImports = `
import { join } from 'path';
import axios from 'axios';
import { useState, useEffect } from 'react';
import * as fs from 'fs';
import { z } from 'zod';
import { User } from './models/User';
import { createRouter } from './router';
import express from 'express';
import { readFile, writeFile } from 'fs/promises';
import { ProductService } from '../services/ProductService';
`;

const parsed = parseImports(messyImports);
const organized = organizeImports(parsed);
const rendered = renderImports(organized);

console.log("Organized imports:");
console.log(rendered);
```

---

## 9. Advanced Code Generation Patterns

### ตัวอย่างที่ 12: Type-Safe Mock Generator

```typescript
import * as ts from "typescript";

// Generate mock data จาก TypeScript interface
function generateMockFromInterface(interfaceCode: string): string {
  const sourceFile = ts.createSourceFile(
    "temp.ts",
    interfaceCode,
    ts.ScriptTarget.Latest,
    true
  );

  const mocks: string[] = [];

  function generateMockValue(typeNode: ts.TypeNode): string {
    if (ts.isStringKeyword(typeNode)) return `"mock-string"`;
    if (ts.isNumberKeyword(typeNode)) return `42`;
    if (ts.isBooleanKeyword(typeNode)) return `true`;
    if (ts.isArrayTypeNode(typeNode)) {
      const elementMock = generateMockValue(typeNode.elementType);
      return `[${elementMock}]`;
    }
    if (ts.isTypeReferenceNode(typeNode)) {
      const name = typeNode.typeName.getText(sourceFile);
      if (name === "Date") return `new Date()`;
      if (name === "string") return `"mock-string"`;
      if (name === "number") return `42`;
      return `{} as ${name}`;
    }
    if (ts.isLiteralTypeNode(typeNode)) {
      return typeNode.literal.getText(sourceFile);
    }
    if (ts.isUnionTypeNode(typeNode)) {
      return generateMockValue(typeNode.types[0]);
    }
    return "undefined";
  }

  function visit(node: ts.Node): void {
    if (ts.isInterfaceDeclaration(node)) {
      const name = node.name.text;
      const props: string[] = [];

      node.members.forEach((member) => {
        if (ts.isPropertySignature(member) && member.name && member.type) {
          const propName = member.name.getText(sourceFile);
          const mockValue = generateMockValue(member.type);
          props.push(`  ${propName}: ${mockValue}`);
        }
      });

      mocks.push(
        `export const mock${name}: ${name} = {\n${props.join(",\n")}\n};`
      );
    }

    ts.forEachChild(node, visit);
  }

  visit(sourceFile);
  return mocks.join("\n\n");
}

// ทดสอบ
const interfaceCode = `
interface User {
  id: number;
  name: string;
  email: string;
  isActive: boolean;
  createdAt: Date;
  tags: string[];
  role: "admin" | "user";
}
`;

const mockCode = generateMockFromInterface(interfaceCode);
console.log("Generated mocks:");
console.log(mockCode);
```

---

## สรุป

Code Generation ใน TypeScript เป็นเครื่องมือทรงพลังที่:

1. **AST Manipulation** - ใช้ TypeScript Compiler API และ ts-morph สำหรับ transform code
2. **Schema-to-Code** - แปลง JSON Schema, OpenAPI spec เป็น TypeScript types และ client code
3. **Template-Based Generation** - ใช้ Handlebars สำหรับ code templates
4. **Schema Processors** - สร้าง custom schema parsers คล้าย Prisma
5. **Refactoring Tools** - สร้าง tools สำหรับ organize imports และ refactor code

ประโยชน์สำคัญ:
- ลดโค้ด boilerplate ที่ต้องเขียนซ้ำ
- รับประกัน type safety ตั้งแต่ compile time
- Consistency ในโค้ดที่สร้างอัตโนมัติ
- เร่งความเร็วในการพัฒนา

---

*จบตอนที่ 77 - Code Generation*
