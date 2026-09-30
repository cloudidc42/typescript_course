# ส่วนที่ 50: TypeScript Compiler API

## บทนำ

TypeScript Compiler API เป็นชุดเครื่องมือที่ทรงพลังที่ช่วยให้นักพัฒนาสามารถอ่าน วิเคราะห์ และแปลงโค้ด TypeScript ได้อย่างเป็นโปรแกรม บทนี้จะสอนวิธีใช้ Compiler API เพื่อสร้างเครื่องมือต่างๆ เช่น linters, code generators, และ transformers

---

## 1. TypeScript Compiler API Overview (ภาพรวม)

### 1.1 การติดตั้งและ Setup

```bash
# ติดตั้ง TypeScript
npm install typescript
npm install --save-dev @types/node ts-node

# หรือถ้าใช้ ts-morph (wrapper ที่ง่ายกว่า)
npm install ts-morph
```

```typescript
// สิ่งที่สำคัญใน TypeScript Compiler API:
// 1. ts.Program - ตัวแทนของ TypeScript program ทั้งหมด
// 2. ts.SourceFile - ตัวแทนของไฟล์ TypeScript
// 3. ts.Node - node ใน Abstract Syntax Tree (AST)
// 4. ts.TypeChecker - ใช้สำหรับ type information
// 5. ts.Transformer - แปลง AST
// 6. ts.Printer - แปลง AST กลับเป็น source code

import ts from 'typescript';

// ตัวอย่างง่ายๆ - อ่าน TypeScript file
const sourceCode = `
  const greeting: string = "Hello, TypeScript!";
  function add(a: number, b: number): number {
    return a + b;
  }
`;

// สร้าง SourceFile
const sourceFile = ts.createSourceFile(
  'example.ts',
  sourceCode,
  ts.ScriptTarget.ES2020,
  true // setParentNodes = true
);

console.log('AST Kind:', ts.SyntaxKind[sourceFile.kind]);
console.log('Statements count:', sourceFile.statements.length);
```

---

## 2. Creating a TypeScript Program (การสร้าง TypeScript Program)

### 2.1 Basic Program Creation

```typescript
import ts from 'typescript';
import path from 'path';

// ✅ สร้าง TypeScript Program จากไฟล์จริง
function createProgram(filePaths: string[]): ts.Program {
  const compilerOptions: ts.CompilerOptions = {
    target: ts.ScriptTarget.ES2020,
    module: ts.ModuleKind.CommonJS,
    strict: true,
    noImplicitAny: true,
    strictNullChecks: true,
    lib: ['lib.es2020.d.ts'],
  };
  
  return ts.createProgram(filePaths, compilerOptions);
}

// ✅ สร้าง Program จาก tsconfig.json
function createProgramFromTsConfig(tsConfigPath: string): ts.Program {
  const configFile = ts.readConfigFile(tsConfigPath, ts.sys.readFile);
  
  if (configFile.error) {
    throw new Error(`Error reading tsconfig: ${configFile.error.messageText}`);
  }
  
  const parsedConfig = ts.parseJsonConfigFileContent(
    configFile.config,
    ts.sys,
    path.dirname(tsConfigPath)
  );
  
  if (parsedConfig.errors.length > 0) {
    throw new Error(`Error parsing tsconfig: ${parsedConfig.errors[0].messageText}`);
  }
  
  return ts.createProgram(
    parsedConfig.fileNames,
    parsedConfig.options
  );
}

// ✅ การใช้งาน Program
function analyzeProgram(program: ts.Program): void {
  const checker = program.getTypeChecker();
  const diagnostics = ts.getPreEmitDiagnostics(program);
  
  // รายงาน type errors
  if (diagnostics.length > 0) {
    console.log('\n=== Type Errors ===');
    diagnostics.forEach(diag => {
      const message = ts.flattenDiagnosticMessageText(diag.messageText, '\n');
      if (diag.file && diag.start !== undefined) {
        const { line, character } = diag.file.getLineAndCharacterOfPosition(diag.start);
        console.log(`${diag.file.fileName}:${line + 1}:${character + 1} - ${message}`);
      } else {
        console.log(message);
      }
    });
  }
  
  // วิเคราะห์แต่ละ source file
  program.getSourceFiles().forEach(sourceFile => {
    if (!sourceFile.isDeclarationFile) {
      console.log(`\nAnalyzing: ${sourceFile.fileName}`);
      
      ts.forEachChild(sourceFile, node => {
        if (ts.isFunctionDeclaration(node) && node.name) {
          const symbol = checker.getSymbolAtLocation(node.name);
          if (symbol) {
            const type = checker.getTypeOfSymbolAtLocation(symbol, node);
            console.log(`Function: ${symbol.name} -> ${checker.typeToString(type)}`);
          }
        }
      });
    }
  });
}
```

### 2.2 Custom Compiler Host

```typescript
import ts from 'typescript';

// ✅ Custom Compiler Host สำหรับ in-memory compilation
function createInMemoryCompilerHost(
  files: Map<string, string>
): ts.CompilerHost {
  const defaultHost = ts.createCompilerHost({});
  
  return {
    ...defaultHost,
    
    getSourceFile: (fileName, languageVersion) => {
      const content = files.get(fileName);
      if (content !== undefined) {
        return ts.createSourceFile(fileName, content, languageVersion, true);
      }
      return defaultHost.getSourceFile(fileName, languageVersion);
    },
    
    fileExists: (fileName) => {
      return files.has(fileName) || defaultHost.fileExists(fileName);
    },
    
    readFile: (fileName) => {
      return files.get(fileName) ?? defaultHost.readFile(fileName);
    },
    
    writeFile: (fileName, content) => {
      files.set(fileName, content);
    },
    
    getCurrentDirectory: () => process.cwd(),
    getDirectories: defaultHost.getDirectories,
    getDefaultLibFileName: defaultHost.getDefaultLibFileName,
    useCaseSensitiveFileNames: defaultHost.useCaseSensitiveFileNames,
    getCanonicalFileName: defaultHost.getCanonicalFileName,
    getNewLine: defaultHost.getNewLine,
  };
}

// ตัวอย่างการใช้งาน in-memory compilation
function compileInMemory(source: string): string {
  const files = new Map<string, string>();
  files.set('main.ts', source);
  
  const host = createInMemoryCompilerHost(files);
  const program = ts.createProgram(['main.ts'], {
    target: ts.ScriptTarget.ES2020,
    module: ts.ModuleKind.CommonJS,
  }, host);
  
  let output = '';
  program.emit(undefined, (fileName, content) => {
    if (fileName.endsWith('.js')) {
      output = content;
    }
  });
  
  return output;
}

// ทดสอบ
const tsCode = `
  interface User {
    name: string;
    age: number;
  }
  
  function greet(user: User): string {
    return \`Hello, \${user.name}! You are \${user.age} years old.\`;
  }
  
  console.log(greet({ name: 'Alice', age: 30 }));
`;

const jsOutput = compileInMemory(tsCode);
console.log('Compiled JavaScript:');
console.log(jsOutput);
```

---

## 3. AST Traversal (การท่อง AST)

### 3.1 การเดิน AST

```typescript
import ts from 'typescript';

// ✅ เดิน AST และเก็บข้อมูล
interface FunctionInfo {
  name: string;
  parameters: ParameterInfo[];
  returnType: string;
  isAsync: boolean;
  lineNumber: number;
}

interface ParameterInfo {
  name: string;
  type: string;
  isOptional: boolean;
  hasDefault: boolean;
}

class ASTAnalyzer {
  private functions: FunctionInfo[] = [];
  private classes: string[] = [];
  private imports: string[] = [];
  
  analyze(sourceFile: ts.SourceFile, checker: ts.TypeChecker): void {
    this.visit(sourceFile, sourceFile, checker);
  }

  private visit(
    node: ts.Node,
    sourceFile: ts.SourceFile,
    checker: ts.TypeChecker
  ): void {
    if (ts.isFunctionDeclaration(node) || ts.isMethodDeclaration(node)) {
      this.extractFunctionInfo(node, sourceFile, checker);
    } else if (ts.isClassDeclaration(node) && node.name) {
      this.classes.push(node.name.text);
    } else if (ts.isImportDeclaration(node)) {
      const moduleSpecifier = node.moduleSpecifier;
      if (ts.isStringLiteral(moduleSpecifier)) {
        this.imports.push(moduleSpecifier.text);
      }
    }
    
    ts.forEachChild(node, child => this.visit(child, sourceFile, checker));
  }

  private extractFunctionInfo(
    node: ts.FunctionDeclaration | ts.MethodDeclaration,
    sourceFile: ts.SourceFile,
    checker: ts.TypeChecker
  ): void {
    if (!node.name) return;
    
    const name = ts.isIdentifier(node.name) ? node.name.text : 'unknown';
    const position = sourceFile.getLineAndCharacterOfPosition(node.getStart());
    const isAsync = node.modifiers?.some(
      m => m.kind === ts.SyntaxKind.AsyncKeyword
    ) ?? false;
    
    const parameters: ParameterInfo[] = node.parameters.map(param => {
      const paramType = checker.getTypeAtLocation(param);
      return {
        name: ts.isIdentifier(param.name) ? param.name.text : 'unknown',
        type: checker.typeToString(paramType),
        isOptional: !!param.questionToken,
        hasDefault: !!param.initializer,
      };
    });
    
    const signature = checker.getSignatureFromDeclaration(node);
    const returnType = signature 
      ? checker.typeToString(checker.getReturnTypeOfSignature(signature))
      : 'unknown';
    
    this.functions.push({
      name,
      parameters,
      returnType,
      isAsync,
      lineNumber: position.line + 1,
    });
  }

  getReport(): {
    functions: FunctionInfo[];
    classes: string[];
    imports: string[];
  } {
    return {
      functions: this.functions,
      classes: this.classes,
      imports: this.imports,
    };
  }
}

// ✅ การใช้งาน
function analyzeFile(filePath: string): void {
  const program = ts.createProgram([filePath], {
    target: ts.ScriptTarget.ES2020,
  });
  
  const checker = program.getTypeChecker();
  const sourceFile = program.getSourceFile(filePath);
  
  if (!sourceFile) {
    console.error(`Cannot find file: ${filePath}`);
    return;
  }
  
  const analyzer = new ASTAnalyzer();
  analyzer.analyze(sourceFile, checker);
  
  const report = analyzer.getReport();
  
  console.log('\n=== Code Analysis Report ===');
  console.log(`\nFunctions (${report.functions.length}):`);
  report.functions.forEach(fn => {
    const asyncTag = fn.isAsync ? 'async ' : '';
    const params = fn.parameters.map(p => `${p.name}: ${p.type}`).join(', ');
    console.log(`  Line ${fn.lineNumber}: ${asyncTag}${fn.name}(${params}): ${fn.returnType}`);
  });
  
  console.log(`\nClasses (${report.classes.length}): ${report.classes.join(', ')}`);
  console.log(`\nImports (${report.imports.length}): ${report.imports.join(', ')}`);
}
```

### 3.2 Node Visitor Pattern

```typescript
import ts from 'typescript';

// ✅ Visitor pattern สำหรับ AST traversal
type NodeVisitor = {
  [K in ts.SyntaxKind]?: (node: ts.Node) => void;
};

class TypedASTVisitor {
  constructor(private visitors: NodeVisitor) {}

  visit(node: ts.Node): void {
    const visitor = this.visitors[node.kind];
    if (visitor) {
      visitor(node);
    }
    ts.forEachChild(node, child => this.visit(child));
  }
}

// ตัวอย่าง: นับจำนวน node แต่ละประเภท
function countNodeTypes(sourceFile: ts.SourceFile): Map<string, number> {
  const counts = new Map<string, number>();
  
  function visit(node: ts.Node): void {
    const kindName = ts.SyntaxKind[node.kind];
    counts.set(kindName, (counts.get(kindName) ?? 0) + 1);
    ts.forEachChild(node, visit);
  }
  
  visit(sourceFile);
  return counts;
}

// ✅ ค้นหา node ตาม predicate
function findNodes<T extends ts.Node>(
  sourceFile: ts.Node,
  predicate: (node: ts.Node) => node is T
): T[] {
  const results: T[] = [];
  
  function visit(node: ts.Node): void {
    if (predicate(node)) {
      results.push(node);
    }
    ts.forEachChild(node, visit);
  }
  
  visit(sourceFile);
  return results;
}

// ใช้งาน
const sourceFile = ts.createSourceFile(
  'test.ts',
  `
    const a = 1;
    let b = "hello";
    var c = true;
    function foo() {}
    class Bar {}
  `,
  ts.ScriptTarget.ES2020,
  true
);

// หา variable declarations ทั้งหมด
const variables = findNodes(
  sourceFile,
  (node): node is ts.VariableDeclaration => ts.isVariableDeclaration(node)
);

console.log('Variables found:', variables.length);
variables.forEach(v => {
  if (ts.isIdentifier(v.name)) {
    console.log(`  - ${v.name.text}`);
  }
});
```

---

## 4. Node Types (ประเภทของ Node)

### 4.1 การตรวจสอบ Node Types ที่สำคัญ

```typescript
import ts from 'typescript';

// ✅ Type Guards สำหรับ Node types ต่างๆ
function exploreNodeTypes(node: ts.Node, depth = 0): void {
  const indent = '  '.repeat(depth);
  const kindName = ts.SyntaxKind[node.kind];
  
  let info = '';
  
  if (ts.isIdentifier(node)) {
    info = ` "${node.text}"`;
  } else if (ts.isStringLiteral(node)) {
    info = ` "${node.text}"`;
  } else if (ts.isNumericLiteral(node)) {
    info = ` ${node.text}`;
  } else if (ts.isFunctionDeclaration(node) && node.name) {
    info = ` ${node.name.text}()`;
  } else if (ts.isClassDeclaration(node) && node.name) {
    info = ` class ${node.name.text}`;
  } else if (ts.isInterfaceDeclaration(node)) {
    info = ` interface ${node.name.text}`;
  } else if (ts.isTypeAliasDeclaration(node)) {
    info = ` type ${node.name.text}`;
  }
  
  console.log(`${indent}${kindName}${info}`);
  
  if (depth < 3) { // จำกัด depth เพื่อไม่ให้ output ยาวเกิน
    ts.forEachChild(node, child => exploreNodeTypes(child, depth + 1));
  }
}

// ✅ สร้าง Node ต่างๆ ด้วย Factory
function createNodeExamples(): void {
  const factory = ts.factory;
  
  // สร้าง identifier
  const id = factory.createIdentifier('myVariable');
  
  // สร้าง string literal
  const str = factory.createStringLiteral('Hello, World!');
  
  // สร้าง number literal
  const num = factory.createNumericLiteral('42');
  
  // สร้าง variable declaration
  const varDecl = factory.createVariableDeclaration(
    factory.createIdentifier('x'),
    undefined,
    factory.createKeywordTypeNode(ts.SyntaxKind.NumberKeyword),
    factory.createNumericLiteral('10')
  );
  
  // สร้าง function
  const func = factory.createFunctionDeclaration(
    undefined, // modifiers
    undefined, // asterisk token
    factory.createIdentifier('greet'),
    undefined, // type parameters
    [
      factory.createParameterDeclaration(
        undefined,
        undefined,
        factory.createIdentifier('name'),
        undefined,
        factory.createKeywordTypeNode(ts.SyntaxKind.StringKeyword)
      ),
    ],
    factory.createKeywordTypeNode(ts.SyntaxKind.StringKeyword), // return type
    factory.createBlock([
      factory.createReturnStatement(
        factory.createTemplateExpression(
          factory.createTemplateHead('Hello, '),
          [
            factory.createTemplateSpan(
              factory.createIdentifier('name'),
              factory.createTemplateTail('!')
            ),
          ]
        )
      ),
    ])
  );
  
  // Print ออกมาเป็น TypeScript code
  const printer = ts.createPrinter();
  const resultFile = ts.createSourceFile('output.ts', '', ts.ScriptTarget.Latest);
  
  const funcCode = printer.printNode(
    ts.EmitHint.Unspecified,
    func,
    resultFile
  );
  
  console.log('Generated function:');
  console.log(funcCode);
}

createNodeExamples();
```

---

## 5. Type Checker (การตรวจสอบ Type)

### 5.1 การใช้งาน TypeChecker

```typescript
import ts from 'typescript';
import fs from 'fs';

// ✅ ใช้ TypeChecker เพื่อ extract type information
class TypeExtractor {
  constructor(
    private program: ts.Program,
    private checker: ts.TypeChecker
  ) {}

  getTypeInfo(sourceFile: ts.SourceFile): TypeInfo[] {
    const infos: TypeInfo[] = [];
    
    const visit = (node: ts.Node): void => {
      if (ts.isVariableDeclaration(node) && ts.isIdentifier(node.name)) {
        const type = this.checker.getTypeAtLocation(node.name);
        const typeString = this.checker.typeToString(type);
        const position = sourceFile.getLineAndCharacterOfPosition(node.getStart());
        
        infos.push({
          name: node.name.text,
          type: typeString,
          kind: 'variable',
          line: position.line + 1,
          column: position.character + 1,
        });
      }
      
      if (ts.isFunctionDeclaration(node) && node.name) {
        const symbol = this.checker.getSymbolAtLocation(node.name);
        if (symbol) {
          const type = this.checker.getTypeOfSymbolAtLocation(
            symbol,
            node.name
          );
          const position = sourceFile.getLineAndCharacterOfPosition(node.getStart());
          
          infos.push({
            name: node.name.text,
            type: this.checker.typeToString(type),
            kind: 'function',
            line: position.line + 1,
            column: position.character + 1,
          });
        }
      }
      
      ts.forEachChild(node, visit);
    };
    
    visit(sourceFile);
    return infos;
  }

  getSymbolInfo(node: ts.Node): SymbolInfo | null {
    const symbol = this.checker.getSymbolAtLocation(node);
    if (!symbol) return null;
    
    return {
      name: symbol.getName(),
      flags: this.getSymbolFlagNames(symbol.flags),
      declarations: symbol.declarations?.map(d => ({
        kind: ts.SyntaxKind[d.kind],
        fileName: d.getSourceFile().fileName,
      })) ?? [],
    };
  }

  private getSymbolFlagNames(flags: ts.SymbolFlags): string[] {
    const names: string[] = [];
    const flagMap: [ts.SymbolFlags, string][] = [
      [ts.SymbolFlags.Function, 'Function'],
      [ts.SymbolFlags.Class, 'Class'],
      [ts.SymbolFlags.Interface, 'Interface'],
      [ts.SymbolFlags.TypeAlias, 'TypeAlias'],
      [ts.SymbolFlags.Variable, 'Variable'],
      [ts.SymbolFlags.Property, 'Property'],
      [ts.SymbolFlags.Method, 'Method'],
    ];
    
    for (const [flag, name] of flagMap) {
      if (flags & flag) names.push(name);
    }
    
    return names;
  }
}

interface TypeInfo {
  name: string;
  type: string;
  kind: 'variable' | 'function' | 'class' | 'interface';
  line: number;
  column: number;
}

interface SymbolInfo {
  name: string;
  flags: string[];
  declarations: Array<{ kind: string; fileName: string }>;
}
```

---

## 6. Symbol Resolution (การ Resolve Symbols)

### 6.1 การทำงานกับ Symbols

```typescript
import ts from 'typescript';

// ✅ Symbol resolution และ cross-reference analysis
class SymbolAnalyzer {
  private usageMap = new Map<string, UsageInfo[]>();
  
  constructor(
    private program: ts.Program,
    private checker: ts.TypeChecker
  ) {}

  buildUsageMap(sourceFiles: ts.SourceFile[]): void {
    for (const sourceFile of sourceFiles) {
      this.analyzeSourceFile(sourceFile);
    }
  }

  private analyzeSourceFile(sourceFile: ts.SourceFile): void {
    const visit = (node: ts.Node): void => {
      if (ts.isIdentifier(node)) {
        const symbol = this.checker.getSymbolAtLocation(node);
        
        if (symbol) {
          const declarations = symbol.getDeclarations();
          if (declarations && declarations.length > 0) {
            const declFile = declarations[0].getSourceFile().fileName;
            const position = sourceFile.getLineAndCharacterOfPosition(node.getStart());
            
            const key = `${declFile}:${symbol.getName()}`;
            const existing = this.usageMap.get(key) ?? [];
            
            existing.push({
              fileName: sourceFile.fileName,
              line: position.line + 1,
              column: position.character + 1,
              usage: node.parent ? ts.SyntaxKind[node.parent.kind] : 'unknown',
            });
            
            this.usageMap.set(key, existing);
          }
        }
      }
      
      ts.forEachChild(node, visit);
    };
    
    visit(sourceFile);
  }

  findUnusedExports(sourceFile: ts.SourceFile): string[] {
    const exportedSymbols: string[] = [];
    
    // หา exported symbols
    const moduleSymbol = this.checker.getSymbolAtLocation(sourceFile);
    if (moduleSymbol) {
      const exports = this.checker.getExportsOfModule(moduleSymbol);
      exports.forEach(sym => exportedSymbols.push(sym.getName()));
    }
    
    // ตรวจสอบว่า export แต่ละตัวถูกใช้ที่ไหนบ้าง
    const unusedExports: string[] = [];
    for (const exported of exportedSymbols) {
      const key = `${sourceFile.fileName}:${exported}`;
      const usages = this.usageMap.get(key) ?? [];
      
      // ถ้ามีแค่ 1 usage (ที่ declaration นั่นเอง) ถือว่าไม่ถูกใช้
      if (usages.length <= 1) {
        unusedExports.push(exported);
      }
    }
    
    return unusedExports;
  }

  getUsages(symbolName: string, fileName: string): UsageInfo[] {
    const key = `${fileName}:${symbolName}`;
    return this.usageMap.get(key) ?? [];
  }
}

interface UsageInfo {
  fileName: string;
  line: number;
  column: number;
  usage: string;
}
```

---

## 7. Building Custom Transformers (การสร้าง Custom Transformers)

### 7.1 Custom AST Transformer

```typescript
import ts from 'typescript';

// ✅ Transformer ที่เพิ่ม logging อัตโนมัติ
function createLoggingTransformer(
  context: ts.TransformationContext
): ts.Transformer<ts.SourceFile> {
  return (sourceFile: ts.SourceFile) => {
    function visit(node: ts.Node): ts.Node {
      if (ts.isFunctionDeclaration(node) && node.name && node.body) {
        const funcName = node.name.text;
        
        // สร้าง console.log statement
        const logStatement = ts.factory.createExpressionStatement(
          ts.factory.createCallExpression(
            ts.factory.createPropertyAccessExpression(
              ts.factory.createIdentifier('console'),
              ts.factory.createIdentifier('log')
            ),
            undefined,
            [ts.factory.createStringLiteral(`[LOG] Calling ${funcName}`)]
          )
        );
        
        // เพิ่ม log ที่ต้นของ function body
        const newBody = ts.factory.updateBlock(
          node.body,
          [logStatement, ...node.body.statements]
        );
        
        return ts.factory.updateFunctionDeclaration(
          node,
          node.modifiers,
          node.asteriskToken,
          node.name,
          node.typeParameters,
          node.parameters,
          node.type,
          newBody
        );
      }
      
      return ts.visitEachChild(node, visit, context);
    }
    
    return ts.visitNode(sourceFile, visit) as ts.SourceFile;
  };
}

// ✅ Transformer ที่ลบ console.log ใน production
function createRemoveConsoleTransformer(
  context: ts.TransformationContext
): ts.Transformer<ts.SourceFile> {
  return (sourceFile: ts.SourceFile) => {
    function visit(node: ts.Node): ts.Node | undefined {
      if (ts.isExpressionStatement(node)) {
        const expr = node.expression;
        
        if (
          ts.isCallExpression(expr) &&
          ts.isPropertyAccessExpression(expr.expression) &&
          ts.isIdentifier(expr.expression.expression) &&
          expr.expression.expression.text === 'console'
        ) {
          // ลบ console.* calls
          return undefined;
        }
      }
      
      return ts.visitEachChild(node, visit, context);
    }
    
    return ts.visitNode(sourceFile, visit) as ts.SourceFile;
  };
}

// ✅ รัน transformations
function applyTransformations(
  sourceCode: string,
  transformers: Array<(ctx: ts.TransformationContext) => ts.Transformer<ts.SourceFile>>
): string {
  const sourceFile = ts.createSourceFile(
    'temp.ts',
    sourceCode,
    ts.ScriptTarget.ES2020,
    true
  );
  
  const result = ts.transform(sourceFile, transformers);
  const transformedFile = result.transformed[0];
  
  const printer = ts.createPrinter();
  const output = printer.printFile(transformedFile);
  
  result.dispose();
  return output;
}

// ตัวอย่างการใช้งาน
const inputCode = `
function calculateTotal(items: number[]): number {
  console.log('Calculating total...');
  const total = items.reduce((sum, item) => sum + item, 0);
  console.log('Total:', total);
  return total;
}

function greet(name: string): string {
  return \`Hello, \${name}!\`;
}
`;

// ใช้ logging transformer
const withLogging = applyTransformations(inputCode, [createLoggingTransformer]);
console.log('With logging:');
console.log(withLogging);

// ลบ console.log
const withoutConsole = applyTransformations(inputCode, [createRemoveConsoleTransformer]);
console.log('\nWithout console.log:');
console.log(withoutConsole);
```

---

## 8. Code Generation (การสร้างโค้ด)

### 8.1 TypeScript Code Generator

```typescript
import ts from 'typescript';

// ✅ สร้าง TypeScript code จาก schema
interface SchemaProperty {
  name: string;
  type: 'string' | 'number' | 'boolean' | 'Date' | string;
  optional: boolean;
  readonly: boolean;
}

interface Schema {
  name: string;
  properties: SchemaProperty[];
  methods?: SchemaMethod[];
}

interface SchemaMethod {
  name: string;
  parameters: Array<{ name: string; type: string }>;
  returnType: string;
  isAsync: boolean;
}

class TypeScriptCodeGenerator {
  private factory = ts.factory;
  private printer = ts.createPrinter({ newLine: ts.NewLineKind.LineFeed });
  private resultFile = ts.createSourceFile('output.ts', '', ts.ScriptTarget.Latest);

  generateInterface(schema: Schema): string {
    const members = schema.properties.map(prop => {
      const typeNode = this.createTypeNode(prop.type);
      
      return this.factory.createPropertySignature(
        prop.readonly
          ? [this.factory.createModifier(ts.SyntaxKind.ReadonlyKeyword)]
          : undefined,
        this.factory.createIdentifier(prop.name),
        prop.optional ? this.factory.createToken(ts.SyntaxKind.QuestionToken) : undefined,
        typeNode
      );
    });
    
    const interfaceDecl = this.factory.createInterfaceDeclaration(
      [this.factory.createModifier(ts.SyntaxKind.ExportKeyword)],
      this.factory.createIdentifier(schema.name),
      undefined,
      undefined,
      members
    );
    
    return this.printer.printNode(
      ts.EmitHint.Unspecified,
      interfaceDecl,
      this.resultFile
    );
  }

  generateClass(schema: Schema): string {
    const members: ts.ClassElement[] = [];
    
    // Constructor
    const constructorParams = schema.properties.map(prop => {
      const modifiers = [
        this.factory.createModifier(
          prop.readonly 
            ? ts.SyntaxKind.ReadonlyKeyword 
            : ts.SyntaxKind.PublicKeyword
        ),
      ];
      
      return this.factory.createParameterDeclaration(
        modifiers,
        undefined,
        this.factory.createIdentifier(prop.name),
        prop.optional ? this.factory.createToken(ts.SyntaxKind.QuestionToken) : undefined,
        this.createTypeNode(prop.type)
      );
    });
    
    const constructor = this.factory.createConstructorDeclaration(
      undefined,
      constructorParams,
      this.factory.createBlock([])
    );
    
    members.push(constructor);
    
    // Methods
    if (schema.methods) {
      schema.methods.forEach(method => {
        const params = method.parameters.map(p =>
          this.factory.createParameterDeclaration(
            undefined,
            undefined,
            this.factory.createIdentifier(p.name),
            undefined,
            this.createTypeNode(p.type)
          )
        );
        
        const methodDecl = this.factory.createMethodDeclaration(
          method.isAsync
            ? [this.factory.createModifier(ts.SyntaxKind.AsyncKeyword)]
            : undefined,
          undefined,
          this.factory.createIdentifier(method.name),
          undefined,
          undefined,
          params,
          this.createTypeNode(method.isAsync ? `Promise<${method.returnType}>` : method.returnType),
          this.factory.createBlock([
            this.factory.createThrowStatement(
              this.factory.createNewExpression(
                this.factory.createIdentifier('Error'),
                undefined,
                [this.factory.createStringLiteral('Not implemented')]
              )
            )
          ])
        );
        
        members.push(methodDecl);
      });
    }
    
    const classDecl = this.factory.createClassDeclaration(
      [this.factory.createModifier(ts.SyntaxKind.ExportKeyword)],
      this.factory.createIdentifier(schema.name),
      undefined,
      undefined,
      members
    );
    
    return this.printer.printNode(
      ts.EmitHint.Unspecified,
      classDecl,
      this.resultFile
    );
  }

  private createTypeNode(type: string): ts.TypeNode {
    switch (type) {
      case 'string':
        return this.factory.createKeywordTypeNode(ts.SyntaxKind.StringKeyword);
      case 'number':
        return this.factory.createKeywordTypeNode(ts.SyntaxKind.NumberKeyword);
      case 'boolean':
        return this.factory.createKeywordTypeNode(ts.SyntaxKind.BooleanKeyword);
      case 'void':
        return this.factory.createKeywordTypeNode(ts.SyntaxKind.VoidKeyword);
      default:
        // สำหรับ type references เช่น Date, Promise<T>
        return this.factory.createTypeReferenceNode(type);
    }
  }
}

// ตัวอย่างการใช้งาน
const generator = new TypeScriptCodeGenerator();

const userSchema: Schema = {
  name: 'User',
  properties: [
    { name: 'id', type: 'number', optional: false, readonly: true },
    { name: 'username', type: 'string', optional: false, readonly: false },
    { name: 'email', type: 'string', optional: false, readonly: false },
    { name: 'bio', type: 'string', optional: true, readonly: false },
    { name: 'createdAt', type: 'Date', optional: false, readonly: true },
  ],
  methods: [
    {
      name: 'update',
      parameters: [{ name: 'data', type: 'Partial<User>' }],
      returnType: 'User',
      isAsync: true,
    },
  ],
};

console.log('Generated Interface:');
console.log(generator.generateInterface(userSchema));

console.log('\nGenerated Class:');
console.log(generator.generateClass(userSchema));
```

---

## 9. Linting กับ Compiler API

### 9.1 Custom Linter

```typescript
import ts from 'typescript';

// ✅ Custom lint rules
interface LintRule {
  name: string;
  description: string;
  check(node: ts.Node, checker: ts.TypeChecker, sourceFile: ts.SourceFile): LintError[];
}

interface LintError {
  message: string;
  line: number;
  column: number;
  severity: 'error' | 'warning' | 'info';
  fix?: string;
}

// Rule: ห้ามใช้ var
const noVarRule: LintRule = {
  name: 'no-var',
  description: 'ห้ามใช้ var declaration',
  check(node, checker, sourceFile) {
    if (
      ts.isVariableStatement(node) &&
      node.declarationList.flags & ts.NodeFlags.None &&
      !(node.declarationList.flags & ts.NodeFlags.Let) &&
      !(node.declarationList.flags & ts.NodeFlags.Const)
    ) {
      const pos = sourceFile.getLineAndCharacterOfPosition(node.getStart());
      return [{
        message: 'ใช้ const หรือ let แทน var',
        line: pos.line + 1,
        column: pos.character + 1,
        severity: 'error',
        fix: 'เปลี่ยน var เป็น const หรือ let',
      }];
    }
    return [];
  },
};

// Rule: ต้องมี explicit return type สำหรับ exported functions
const requireExplicitReturnTypeRule: LintRule = {
  name: 'require-explicit-return-type',
  description: 'exported functions ต้องมี explicit return type',
  check(node, checker, sourceFile) {
    if (
      ts.isFunctionDeclaration(node) &&
      node.name &&
      !node.type && // ไม่มี return type
      node.modifiers?.some(m => m.kind === ts.SyntaxKind.ExportKeyword)
    ) {
      const pos = sourceFile.getLineAndCharacterOfPosition(node.getStart());
      return [{
        message: `Function '${node.name.text}' ต้องมี explicit return type`,
        line: pos.line + 1,
        column: pos.character + 1,
        severity: 'warning',
      }];
    }
    return [];
  },
};

// Rule: ห้ามใช้ any
const noExplicitAnyRule: LintRule = {
  name: 'no-explicit-any',
  description: 'ห้ามใช้ any type',
  check(node, checker, sourceFile) {
    if (
      ts.isTypeReferenceNode(node) &&
      ts.isIdentifier(node.typeName) &&
      node.typeName.text === 'any'
    ) {
      const pos = sourceFile.getLineAndCharacterOfPosition(node.getStart());
      return [{
        message: 'ห้ามใช้ any type ใช้ unknown หรือ specific type แทน',
        line: pos.line + 1,
        column: pos.character + 1,
        severity: 'warning',
      }];
    }
    return [];
  },
};

class TypeScriptLinter {
  private rules: LintRule[] = [];

  addRule(rule: LintRule): void {
    this.rules.push(rule);
  }

  lint(program: ts.Program): Map<string, LintError[]> {
    const checker = program.getTypeChecker();
    const results = new Map<string, LintError[]>();
    
    for (const sourceFile of program.getSourceFiles()) {
      if (sourceFile.isDeclarationFile) continue;
      
      const errors: LintError[] = [];
      
      const visit = (node: ts.Node): void => {
        for (const rule of this.rules) {
          const ruleErrors = rule.check(node, checker, sourceFile);
          errors.push(...ruleErrors);
        }
        ts.forEachChild(node, visit);
      };
      
      visit(sourceFile);
      
      if (errors.length > 0) {
        results.set(sourceFile.fileName, errors);
      }
    }
    
    return results;
  }

  formatResults(results: Map<string, LintError[]>): string {
    const lines: string[] = [];
    let errorCount = 0;
    let warningCount = 0;
    
    for (const [fileName, errors] of results) {
      if (errors.length > 0) {
        lines.push(`\n${fileName}`);
        
        errors.forEach(error => {
          const icon = error.severity === 'error' ? '✗' : 
                      error.severity === 'warning' ? '⚠' : 'ℹ';
          lines.push(
            `  ${icon} ${error.line}:${error.column}  ${error.message}`
          );
          
          if (error.severity === 'error') errorCount++;
          else if (error.severity === 'warning') warningCount++;
        });
      }
    }
    
    if (lines.length === 0) {
      lines.push('✓ No lint errors found!');
    } else {
      lines.push(`\n${errorCount} errors, ${warningCount} warnings`);
    }
    
    return lines.join('\n');
  }
}

// ใช้งาน
const linter = new TypeScriptLinter();
linter.addRule(noVarRule);
linter.addRule(requireExplicitReturnTypeRule);
linter.addRule(noExplicitAnyRule);
```

---

## 10. Building a Code Analyzer (สร้าง Code Analyzer)

### 10.1 Complexity Analyzer

```typescript
import ts from 'typescript';

// ✅ Cyclomatic Complexity Analyzer
interface ComplexityReport {
  functionName: string;
  complexity: number;
  line: number;
  file: string;
  risk: 'low' | 'medium' | 'high' | 'very-high';
}

class ComplexityAnalyzer {
  analyzeFile(sourceFile: ts.SourceFile): ComplexityReport[] {
    const reports: ComplexityReport[] = [];
    
    const visit = (node: ts.Node): void => {
      if (
        ts.isFunctionDeclaration(node) ||
        ts.isMethodDeclaration(node) ||
        ts.isArrowFunction(node)
      ) {
        const complexity = this.calculateComplexity(node);
        const name = this.getFunctionName(node);
        const position = sourceFile.getLineAndCharacterOfPosition(node.getStart());
        
        reports.push({
          functionName: name,
          complexity,
          line: position.line + 1,
          file: sourceFile.fileName,
          risk: this.getRiskLevel(complexity),
        });
      }
      
      ts.forEachChild(node, visit);
    };
    
    visit(sourceFile);
    return reports;
  }

  private calculateComplexity(node: ts.Node): number {
    let complexity = 1; // เริ่มที่ 1 เสมอ
    
    const visit = (child: ts.Node): void => {
      switch (child.kind) {
        case ts.SyntaxKind.IfStatement:
        case ts.SyntaxKind.WhileStatement:
        case ts.SyntaxKind.DoStatement:
        case ts.SyntaxKind.ForStatement:
        case ts.SyntaxKind.ForInStatement:
        case ts.SyntaxKind.ForOfStatement:
        case ts.SyntaxKind.CaseClause:
        case ts.SyntaxKind.CatchClause:
        case ts.SyntaxKind.ConditionalExpression: // ternary
          complexity++;
          break;
        case ts.SyntaxKind.BinaryExpression:
          const binaryExpr = child as ts.BinaryExpression;
          if (
            binaryExpr.operatorToken.kind === ts.SyntaxKind.AmpersandAmpersandToken ||
            binaryExpr.operatorToken.kind === ts.SyntaxKind.BarBarToken ||
            binaryExpr.operatorToken.kind === ts.SyntaxKind.QuestionQuestionToken
          ) {
            complexity++;
          }
          break;
      }
      
      ts.forEachChild(child, visit);
    };
    
    ts.forEachChild(node, visit);
    return complexity;
  }

  private getFunctionName(node: ts.Node): string {
    if (ts.isFunctionDeclaration(node) && node.name) {
      return node.name.text;
    }
    if (ts.isMethodDeclaration(node) && ts.isIdentifier(node.name)) {
      return node.name.text;
    }
    if (ts.isArrowFunction(node)) {
      const parent = node.parent;
      if (ts.isVariableDeclaration(parent) && ts.isIdentifier(parent.name)) {
        return parent.name.text;
      }
    }
    return '<anonymous>';
  }

  private getRiskLevel(complexity: number): 'low' | 'medium' | 'high' | 'very-high' {
    if (complexity <= 5) return 'low';
    if (complexity <= 10) return 'medium';
    if (complexity <= 20) return 'high';
    return 'very-high';
  }
}
```

---

## 11. ts-morph Library

### 11.1 การใช้งาน ts-morph

```typescript
import { Project, SyntaxKind, SourceFile, ClassDeclaration } from 'ts-morph';

// ✅ ts-morph ทำให้การทำงานกับ TypeScript Compiler API ง่ายขึ้นมาก
const project = new Project({
  tsConfigFilePath: './tsconfig.json',
  skipAddingFilesFromTsConfig: false,
});

// ✅ อ่านและแก้ไขโค้ด
function addMissingReturnTypes(): void {
  const sourceFiles = project.getSourceFiles();
  
  for (const sourceFile of sourceFiles) {
    const functions = sourceFile.getFunctions();
    
    functions.forEach(func => {
      if (!func.getReturnTypeNode()) {
        // TypeScript จะ infer return type
        const returnType = func.getReturnType();
        func.setReturnType(returnType.getText());
      }
    });
  }
  
  project.saveSync();
}

// ✅ Rename ทุก occurrence ของ symbol
function renameSymbol(
  sourceFile: SourceFile,
  oldName: string,
  newName: string
): void {
  // หา identifier ทั้งหมด
  const identifiers = sourceFile.getDescendantsOfKind(SyntaxKind.Identifier);
  
  identifiers
    .filter(id => id.getText() === oldName)
    .forEach(id => id.replaceWithText(newName));
  
  sourceFile.saveSync();
}

// ✅ Generate CRUD repository จาก Entity
function generateRepository(entityClass: ClassDeclaration): string {
  const className = entityClass.getName() ?? 'Unknown';
  const properties = entityClass.getProperties();
  
  const idProp = properties.find(p => p.getName() === 'id');
  const idType = idProp?.getType().getText() ?? 'number';
  
  return `
import { Repository, DataSource } from 'typeorm';
import { ${className} } from './${className.toLowerCase()}.entity';

export class ${className}Repository {
  private repository: Repository<${className}>;

  constructor(dataSource: DataSource) {
    this.repository = dataSource.getRepository(${className});
  }

  async findAll(): Promise<${className}[]> {
    return this.repository.find();
  }

  async findById(id: ${idType}): Promise<${className} | null> {
    return this.repository.findOne({ where: { id } });
  }

  async create(data: Omit<${className}, 'id' | 'createdAt' | 'updatedAt'>): Promise<${className}> {
    const entity = this.repository.create(data);
    return this.repository.save(entity);
  }

  async update(id: ${idType}, data: Partial<${className}>): Promise<${className} | null> {
    await this.repository.update(id, data);
    return this.findById(id);
  }

  async delete(id: ${idType}): Promise<boolean> {
    const result = await this.repository.delete(id);
    return (result.affected ?? 0) > 0;
  }
}
`;
}

// ✅ สร้าง DTO จาก Entity
function generateDTO(entityClass: ClassDeclaration, type: 'create' | 'update'): string {
  const className = entityClass.getName() ?? 'Unknown';
  const properties = entityClass
    .getProperties()
    .filter(p => !['id', 'createdAt', 'updatedAt'].includes(p.getName()));
  
  const propDefs = properties.map(prop => {
    const propType = prop.getType().getText();
    const optional = type === 'update' ? '?' : '';
    return `  ${prop.getName()}${optional}: ${propType};`;
  });
  
  const dtoName = `${type === 'create' ? 'Create' : 'Update'}${className}Dto`;
  
  return `
export interface ${dtoName} {
${propDefs.join('\n')}
}
`;
}
```

---

## 12. Practical Tools ที่สร้างได้

### 12.1 API Documentation Generator

```typescript
import ts from 'typescript';

// ✅ สร้าง API documentation จาก TypeScript source
interface APIDoc {
  name: string;
  description: string;
  parameters: ParamDoc[];
  returnType: string;
  examples: string[];
  throws: string[];
}

interface ParamDoc {
  name: string;
  type: string;
  description: string;
  required: boolean;
}

function extractJSDocComment(node: ts.Node, sourceFile: ts.SourceFile): string {
  const nodeStart = node.getFullStart();
  const fullText = sourceFile.getFullText();
  const textBeforeNode = fullText.substring(0, nodeStart);
  
  // Heuristic: หา JSDoc comment ก่อน node
  const jsdocMatch = textBeforeNode.match(/\/\*\*([\s\S]*?)\*\/\s*$/);
  if (!jsdocMatch) return '';
  
  return jsdocMatch[1]
    .split('\n')
    .map(line => line.replace(/^\s*\*\s?/, '').trim())
    .filter(Boolean)
    .join(' ');
}

class APIDocumentationGenerator {
  generate(program: ts.Program): APIDoc[] {
    const checker = program.getTypeChecker();
    const docs: APIDoc[] = [];
    
    for (const sourceFile of program.getSourceFiles()) {
      if (sourceFile.isDeclarationFile) continue;
      
      const visit = (node: ts.Node): void => {
        if (
          ts.isFunctionDeclaration(node) &&
          node.name &&
          node.modifiers?.some(m => m.kind === ts.SyntaxKind.ExportKeyword)
        ) {
          const doc = this.extractFunctionDoc(node, checker, sourceFile);
          docs.push(doc);
        }
        ts.forEachChild(node, visit);
      };
      
      visit(sourceFile);
    }
    
    return docs;
  }

  private extractFunctionDoc(
    node: ts.FunctionDeclaration,
    checker: ts.TypeChecker,
    sourceFile: ts.SourceFile
  ): APIDoc {
    const name = node.name!.text;
    const description = extractJSDocComment(node, sourceFile);
    
    const parameters: ParamDoc[] = node.parameters.map(param => {
      const paramType = checker.getTypeAtLocation(param);
      return {
        name: ts.isIdentifier(param.name) ? param.name.text : 'unknown',
        type: checker.typeToString(paramType),
        description: '',
        required: !param.questionToken && !param.initializer,
      };
    });
    
    const signature = checker.getSignatureFromDeclaration(node);
    const returnType = signature
      ? checker.typeToString(checker.getReturnTypeOfSignature(signature))
      : 'void';
    
    return {
      name,
      description,
      parameters,
      returnType,
      examples: [],
      throws: [],
    };
  }

  toMarkdown(docs: APIDoc[]): string {
    return docs.map(doc => {
      const params = doc.parameters.map(p =>
        `| \`${p.name}\` | \`${p.type}\` | ${p.required ? 'Required' : 'Optional'} | ${p.description} |`
      ).join('\n');
      
      return `
### \`${doc.name}\`

${doc.description}

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
${params}

**Returns:** \`${doc.returnType}\`
`.trim();
    }).join('\n\n---\n\n');
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **TypeScript Compiler API Overview** - ส่วนประกอบสำคัญของ Compiler API
2. **Creating a TypeScript Program** - สร้าง Program จากไฟล์และ tsconfig
3. **AST Traversal** - การเดิน AST, Visitor pattern
4. **Node Types** - ประเภท Node ต่างๆ และวิธีสร้าง
5. **Type Checker** - การ extract type information
6. **Symbol Resolution** - การวิเคราะห์ symbol usage
7. **Custom Transformers** - การเขียน transformer เพื่อแปลงโค้ด
8. **Code Generation** - สร้างโค้ด TypeScript จาก schema
9. **Linting** - เขียน custom lint rules
10. **Code Analyzer** - วิเคราะห์ complexity
11. **ts-morph** - Library ที่ทำให้ง่ายขึ้น
12. **Practical Tools** - Documentation generator

TypeScript Compiler API เปิดโอกาสให้สร้างเครื่องมือที่ทรงพลังสำหรับ developer productivity

---

*หัวข้อถัดไป: Part 51 - Building TypeScript Libraries*
