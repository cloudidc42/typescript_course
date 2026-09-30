# Part 83: TypeScript Compiler Internals

## บทนำ

การเข้าใจ TypeScript Compiler Internals ช่วยให้เราเขียน TypeScript ได้มีประสิทธิภาพมากขึ้น เข้าใจ error messages ได้ดีขึ้น และสามารถสร้าง tools เช่น linters, code generators, และ transformers ได้

---

## 1. TypeScript Architecture Overview

TypeScript Compiler (tsc) ทำงานผ่านหลายขั้นตอน:

```
Source Files (.ts)
      ↓
   Scanner (Lexer)
      ↓
   Parser → AST (Abstract Syntax Tree)
      ↓
   Binder → Symbol Table
      ↓
   Type Checker
      ↓
   Emitter → JavaScript Output (.js)
              Declaration Files (.d.ts)
              Source Maps (.js.map)
```

### โครงสร้างของ TypeScript Compiler Package

```
typescript/
├── lib/
│   ├── typescript.js    # Compiler API
│   ├── typescript.d.ts  # Type definitions
│   └── lib.*.d.ts       # Library type definitions
└── bin/
    └── tsc              # CLI executable
```

---

## 2. Scanner และ Lexer

Scanner (Lexer) เป็นขั้นตอนแรกที่แปลง source code เป็น tokens

```typescript
import * as ts from "typescript";

// ตัวอย่างการใช้ Scanner API
const sourceText = `
const greeting: string = "Hello, TypeScript!";
function add(a: number, b: number): number {
    return a + b;
}
`;

const scanner = ts.createScanner(
    ts.ScriptTarget.ES2020,
    false, // skipTrivia
    ts.LanguageVariant.Standard,
    sourceText
);

// วน scan tokens
let token = scanner.scan();
while (token !== ts.SyntaxKind.EndOfFileToken) {
    const tokenText = scanner.getTokenText();
    const tokenValue = scanner.getTokenValue();
    
    console.log({
        kind: ts.SyntaxKind[token],
        text: tokenText,
        start: scanner.getTokenStart(),
        end: scanner.getTokenEnd(),
    });
    
    token = scanner.scan();
}
```

### Token Types

```typescript
// Token ที่สำคัญใน TypeScript
// Literal tokens
ts.SyntaxKind.StringLiteral       // "hello"
ts.SyntaxKind.NumericLiteral      // 42
ts.SyntaxKind.TrueKeyword         // true
ts.SyntaxKind.FalseKeyword        // false
ts.SyntaxKind.NullKeyword         // null

// Keywords
ts.SyntaxKind.ConstKeyword        // const
ts.SyntaxKind.LetKeyword          // let
ts.SyntaxKind.FunctionKeyword     // function
ts.SyntaxKind.ClassKeyword        // class
ts.SyntaxKind.InterfaceKeyword    // interface
ts.SyntaxKind.TypeKeyword         // type
ts.SyntaxKind.ImportKeyword       // import
ts.SyntaxKind.ExportKeyword       // export
ts.SyntaxKind.ReturnKeyword       // return

// Operators
ts.SyntaxKind.EqualsToken         // =
ts.SyntaxKind.PlusToken           // +
ts.SyntaxKind.MinusToken          // -
ts.SyntaxKind.AsteriskToken       // *
ts.SyntaxKind.SlashToken          // /
ts.SyntaxKind.DotToken            // .
ts.SyntaxKind.ColonToken          // :
ts.SyntaxKind.SemicolonToken      // ;
ts.SyntaxKind.OpenParenToken      // (
ts.SyntaxKind.CloseParenToken     // )
ts.SyntaxKind.OpenBraceToken      // {
ts.SyntaxKind.CloseBraceToken     // }
```

---

## 3. Parser และ AST

Parser รับ tokens จาก Scanner และสร้าง Abstract Syntax Tree (AST)

```typescript
import * as ts from "typescript";

// สร้าง AST จาก source text
const sourceFile = ts.createSourceFile(
    "example.ts",
    `
    interface User {
        name: string;
        age: number;
    }
    
    function greet(user: User): string {
        return \`Hello, \${user.name}!\`;
    }
    `,
    ts.ScriptTarget.ES2020,
    true // setParentNodes
);

// Traverse AST
function visit(node: ts.Node, depth: number = 0): void {
    const indent = "  ".repeat(depth);
    const kind = ts.SyntaxKind[node.kind];
    
    let text = "";
    if (ts.isIdentifier(node)) {
        text = ` "${node.text}"`;
    } else if (ts.isStringLiteral(node)) {
        text = ` "${node.text}"`;
    }
    
    console.log(`${indent}${kind}${text}`);
    
    ts.forEachChild(node, child => visit(child, depth + 1));
}

visit(sourceFile);
```

### AST Node Types ที่สำคัญ

```typescript
// Declaration nodes
ts.isVariableDeclaration(node)    // const x = 1
ts.isFunctionDeclaration(node)    // function foo() {}
ts.isClassDeclaration(node)       // class Foo {}
ts.isInterfaceDeclaration(node)   // interface Foo {}
ts.isTypeAliasDeclaration(node)   // type Foo = ...
ts.isImportDeclaration(node)      // import ... from "..."
ts.isExportDeclaration(node)      // export ...

// Expression nodes
ts.isCallExpression(node)         // foo()
ts.isArrowFunction(node)          // () => {}
ts.isBinaryExpression(node)       // a + b
ts.isPropertyAccessExpression(node) // obj.prop
ts.isObjectLiteralExpression(node)  // { key: value }
ts.isArrayLiteralExpression(node)   // [1, 2, 3]

// Statement nodes
ts.isIfStatement(node)            // if (...) {}
ts.isForStatement(node)           // for (...) {}
ts.isReturnStatement(node)        // return ...
ts.isExpressionStatement(node)    // foo();

// Type nodes
ts.isTypeReferenceNode(node)      // SomeType
ts.isUnionTypeNode(node)          // A | B
ts.isIntersectionTypeNode(node)   // A & B
ts.isArrayTypeNode(node)          // A[]
```

### สร้าง AST Visitor

```typescript
// Generic AST visitor
function createVisitor<T>(
    sourceFile: ts.SourceFile,
    predicate: (node: ts.Node) => node is ts.Node & T
): T[] {
    const results: T[] = [];
    
    function visit(node: ts.Node) {
        if (predicate(node)) {
            results.push(node as any);
        }
        ts.forEachChild(node, visit);
    }
    
    visit(sourceFile);
    return results;
}

// ตัวอย่าง: หา function declarations ทั้งหมด
const functions = createVisitor(sourceFile, ts.isFunctionDeclaration);
console.log(functions.map(f => f.name?.text));

// ตัวอย่าง: หา interface declarations ทั้งหมด
const interfaces = createVisitor(sourceFile, ts.isInterfaceDeclaration);
console.log(interfaces.map(i => i.name.text));
```

---

## 4. Binder

Binder สร้าง Symbol Table ที่เชื่อมโยง identifiers กับ declarations

```typescript
// Symbol ใน TypeScript คือ entity ที่มีชื่อ
// แต่ละ symbol มี:
// - name: ชื่อของ symbol
// - declarations: ที่ที่ถูก declare
// - flags: ประเภทของ symbol
// - members: สำหรับ class/interface

// Symbol Flags
ts.SymbolFlags.Variable      // let/const/var
ts.SymbolFlags.Function      // function
ts.SymbolFlags.Class         // class
ts.SymbolFlags.Interface     // interface
ts.SymbolFlags.TypeAlias     // type
ts.SymbolFlags.Enum          // enum
ts.SymbolFlags.Property      // property ของ object/class
ts.SymbolFlags.Method        // method ของ class
ts.SymbolFlags.Constructor   // constructor ของ class

// การใช้ Symbol ผ่าน TypeChecker
const program = ts.createProgram(["example.ts"], {});
const checker = program.getTypeChecker();
const sf = program.getSourceFile("example.ts")!;

// หา symbols ใน scope
function findSymbol(name: string): ts.Symbol | undefined {
    return checker.resolveName(
        name,
        sf.endOfFileToken,
        ts.SymbolFlags.All,
        false
    ) ?? undefined;
}
```

---

## 5. Type Checker

Type Checker เป็นส่วนที่ตรวจสอบ type correctness

```typescript
import * as ts from "typescript";
import * as fs from "fs";

// สร้าง Program (entry point สำหรับ TypeScript Compiler)
function createProgram(files: string[]): ts.Program {
    const options: ts.CompilerOptions = {
        target: ts.ScriptTarget.ES2020,
        module: ts.ModuleKind.CommonJS,
        strict: true,
    };
    
    return ts.createProgram(files, options);
}

// ตรวจสอบ types ของ expressions
function analyzeTypes(sourceFile: ts.SourceFile, program: ts.Program): void {
    const checker = program.getTypeChecker();
    
    function visit(node: ts.Node) {
        if (ts.isVariableDeclaration(node) && node.initializer) {
            const type = checker.getTypeAtLocation(node.initializer);
            const typeName = checker.typeToString(type);
            const varName = ts.isIdentifier(node.name) ? node.name.text : "?";
            
            console.log(`${varName}: ${typeName}`);
        }
        
        ts.forEachChild(node, visit);
    }
    
    visit(sourceFile);
}

// ตัวอย่าง: สร้าง diagnostic messages
function getDiagnostics(files: string[]): ts.Diagnostic[] {
    const program = createProgram(files);
    
    return [
        ...program.getSyntacticDiagnostics(),
        ...program.getSemanticDiagnostics(),
        ...program.getDeclarationDiagnostics(),
    ];
}

function formatDiagnostic(diagnostic: ts.Diagnostic): string {
    if (diagnostic.file) {
        const { line, character } = ts.getLineAndCharacterOfPosition(
            diagnostic.file,
            diagnostic.start!
        );
        const message = ts.flattenDiagnosticMessageText(
            diagnostic.messageText,
            "\n"
        );
        return `${diagnostic.file.fileName} (${line + 1},${character + 1}): ${message}`;
    }
    return ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n");
}
```

### Type Information APIs

```typescript
// TypeChecker APIs ที่สำคัญ
const checker = program.getTypeChecker();

// 1. getTypeAtLocation - ดึง type ของ node
const type = checker.getTypeAtLocation(node);

// 2. typeToString - แปลง type เป็น string
const typeStr = checker.typeToString(type);

// 3. getSymbolAtLocation - ดึง symbol ของ node
const symbol = checker.getSymbolAtLocation(node);

// 4. getTypeOfSymbolAtLocation - ดึง type ของ symbol
const symType = checker.getTypeOfSymbolAtLocation(symbol!, node);

// 5. getSignaturesOfType - ดึง call signatures
const signatures = checker.getSignaturesOfType(type, ts.SignatureKind.Call);

// 6. getReturnTypeOfSignature - ดึง return type
const returnType = checker.getReturnTypeOfSignature(signatures[0]);

// 7. getPropertiesOfType - ดึง properties
const props = checker.getPropertiesOfType(type);

// ตัวอย่าง: วิเคราะห์ interface properties
function analyzeInterface(node: ts.InterfaceDeclaration, checker: ts.TypeChecker): void {
    const type = checker.getTypeAtLocation(node);
    const props = checker.getPropertiesOfType(type);
    
    console.log(`Interface: ${node.name.text}`);
    props.forEach(prop => {
        const propType = checker.getTypeOfSymbolAtLocation(prop, node);
        const optional = (prop.flags & ts.SymbolFlags.Optional) !== 0;
        console.log(`  ${prop.name}${optional ? "?" : ""}: ${checker.typeToString(propType)}`);
    });
}
```

---

## 6. Emitter

Emitter แปลง AST เป็น JavaScript code

```typescript
// Emit ผ่าน Program
const program = ts.createProgram(["src/index.ts"], {
    target: ts.ScriptTarget.ES2020,
    module: ts.ModuleKind.CommonJS,
    outDir: "dist",
    declaration: true,
    sourceMap: true,
});

// Emit files
const emitResult = program.emit();

// ตรวจสอบ errors
if (emitResult.emitSkipped) {
    const diagnostics = ts.getPreEmitDiagnostics(program);
    diagnostics.forEach(d => console.error(formatDiagnostic(d)));
}

// Custom Transformer
function createAddConsoleLogTransformer(): ts.TransformerFactory<ts.SourceFile> {
    return (context: ts.TransformationContext): ts.Transformer<ts.SourceFile> => {
        return (sourceFile: ts.SourceFile): ts.SourceFile => {
            function visitNode(node: ts.Node): ts.Node {
                if (ts.isFunctionDeclaration(node) && node.body) {
                    // เพิ่ม console.log ในทุก function
                    const logStatement = ts.factory.createExpressionStatement(
                        ts.factory.createCallExpression(
                            ts.factory.createPropertyAccessExpression(
                                ts.factory.createIdentifier("console"),
                                "log"
                            ),
                            undefined,
                            [ts.factory.createStringLiteral(
                                `Function ${node.name?.text} called`
                            )]
                        )
                    );
                    
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
                
                return ts.visitEachChild(node, visitNode, context);
            }
            
            return ts.visitNode(sourceFile, visitNode) as ts.SourceFile;
        };
    };
}

// ใช้ transformer กับ emit
const transformers: ts.CustomTransformers = {
    before: [createAddConsoleLogTransformer()],
};

program.emit(undefined, undefined, undefined, false, transformers);
```

---

## 7. Language Service

Language Service ให้บริการ IDE features เช่น autocomplete, go-to-definition, rename

```typescript
import * as ts from "typescript";

// สร้าง Language Service Host
class SimpleLanguageServiceHost implements ts.LanguageServiceHost {
    private files: Map<string, string> = new Map();

    addFile(name: string, content: string): void {
        this.files.set(name, content);
    }

    getCompilationSettings(): ts.CompilerOptions {
        return {
            target: ts.ScriptTarget.ES2020,
            strict: true,
        };
    }

    getScriptFileNames(): string[] {
        return Array.from(this.files.keys());
    }

    getScriptVersion(fileName: string): string {
        return "1";
    }

    getScriptSnapshot(fileName: string): ts.IScriptSnapshot | undefined {
        const content = this.files.get(fileName);
        if (content === undefined) return undefined;
        return ts.ScriptSnapshot.fromString(content);
    }

    getCurrentDirectory(): string {
        return process.cwd();
    }

    getDefaultLibFileName(options: ts.CompilerOptions): string {
        return ts.getDefaultLibFilePath(options);
    }

    fileExists(path: string): boolean {
        return this.files.has(path);
    }

    readFile(path: string): string | undefined {
        return this.files.get(path);
    }

    readDirectory(): string[] {
        return [];
    }
}

// ใช้ Language Service
const host = new SimpleLanguageServiceHost();
host.addFile("example.ts", `
interface User {
    name: string;
    age: number;
}

const user: User = {
    name: "Alice",
    age: 30,
};

console.log(user.
`);

const service = ts.createLanguageService(host);

// Completions
const completions = service.getCompletionsAtPosition(
    "example.ts",
    host.getScriptSnapshot("example.ts")!.getLength() - 1,
    undefined
);

console.log("Completions:");
completions?.entries.slice(0, 5).forEach(entry => {
    console.log(`  ${entry.name}: ${entry.kind}`);
});

// Quick Info (hover)
const quickInfo = service.getQuickInfoAtPosition("example.ts", 50);
console.log("\nQuick Info:");
console.log(quickInfo?.displayParts?.map(p => p.text).join(""));
```

### Language Service Features

```typescript
// Go to Definition
const definitions = service.getDefinitionAtPosition("example.ts", position);
definitions?.forEach(def => {
    console.log(`Definition at ${def.fileName}:${def.textSpan.start}`);
});

// Find References
const refs = service.getReferencesAtPosition("example.ts", position);
refs?.forEach(ref => {
    console.log(`Reference at ${ref.fileName}:${ref.textSpan.start}`);
});

// Rename
const renameInfo = service.getRenameInfo("example.ts", position, { allowRenameOfImportPath: false });
if (renameInfo.canRename) {
    const renames = service.findRenameLocations("example.ts", position, false, false);
    console.log(`Found ${renames?.length} rename locations`);
}

// Diagnostics
const syntacticDiags = service.getSyntacticDiagnostics("example.ts");
const semanticDiags = service.getSemanticDiagnostics("example.ts");
const suggestionDiags = service.getSuggestionDiagnostics("example.ts");

// Code Fixes
const fixes = service.getCodeFixesAtPosition(
    "example.ts",
    start,
    end,
    [semanticDiags[0].code],
    {}
);

// Format
const edits = service.getFormattingEditsForDocument("example.ts", {
    indentSize: 4,
    tabSize: 4,
    convertTabsToSpaces: true,
    indentStyle: ts.IndentStyle.Smart,
});
```

---

## 8. Watch Mode

Watch Mode ตรวจสอบการเปลี่ยนแปลงของ files และ recompile

```typescript
// Programmatic Watch Mode
function watchFiles(files: string[]): void {
    const host = ts.createWatchCompilerHost(
        files,
        {
            target: ts.ScriptTarget.ES2020,
            module: ts.ModuleKind.CommonJS,
        },
        ts.sys,
        ts.createEmitAndSemanticDiagnosticsBuilderProgram,
        // Report diagnostics
        (diagnostic) => {
            console.error(ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n"));
        },
        // Report watch status
        (diagnostic) => {
            console.log(ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n"));
        }
    );

    // Override afterProgramCreate
    const origAfterProgramCreate = host.afterProgramCreate;
    host.afterProgramCreate = (program) => {
        console.log("Program updated!");
        origAfterProgramCreate?.(program);
    };

    ts.createWatchProgram(host);
}

// watchFiles(["src/index.ts"]);
```

---

## 9. Incremental Compilation

Incremental Compilation ลดเวลา build โดย cache ผลลัพธ์ก่อนหน้า

```typescript
// tsconfig.json สำหรับ incremental compilation
const incrementalConfig: ts.CompilerOptions = {
    target: ts.ScriptTarget.ES2020,
    incremental: true,
    tsBuildInfoFile: ".tsbuildinfo",
    outDir: "dist",
};

// Incremental build
function incrementalBuild(configPath: string): void {
    const configFile = ts.readConfigFile(configPath, ts.sys.readFile);
    const parsedConfig = ts.parseJsonConfigFileContent(
        configFile.config,
        ts.sys,
        "."
    );

    // สร้าง incremental program
    const program = ts.createIncrementalProgram({
        rootNames: parsedConfig.fileNames,
        options: {
            ...parsedConfig.options,
            incremental: true,
        },
    });

    const emitResult = program.emit();
    
    console.log(`Emit skipped: ${emitResult.emitSkipped}`);
    console.log(`Diagnostics: ${emitResult.diagnostics.length}`);
}

// .tsbuildinfo format (simplified)
interface BuildInfo {
    program: {
        fileInfos: Record<string, { version: string; signature: string }>;
        options: ts.CompilerOptions;
        referencedMap: Record<string, string[]>;
        exportedModulesMap: Record<string, string[]>;
    };
    version: string;
}
```

---

## 10. Project References Internals

Project References ช่วยจัดการ multi-project TypeScript setups

```typescript
// tsconfig.json with project references
/*
{
    "compilerOptions": {
        "composite": true,
        "declaration": true,
        "declarationMap": true,
        "outDir": "dist"
    },
    "references": [
        { "path": "../shared" },
        { "path": "../utils" }
    ]
}
*/

// Build solution ที่มี project references
function buildSolution(configPath: string): void {
    const buildHost = ts.createSolutionBuilderHost(
        ts.sys,
        ts.createEmitAndSemanticDiagnosticsBuilderProgram,
        (diag) => console.error(ts.flattenDiagnosticMessageText(diag.messageText, "\n")),
        (diag) => console.log(ts.flattenDiagnosticMessageText(diag.messageText, "\n"))
    );

    const builder = ts.createSolutionBuilder(buildHost, [configPath], {});
    const exitCode = builder.build();
    
    console.log(`Build exit code: ${exitCode}`);
}

// Watch mode สำหรับ project references
function watchSolution(configPath: string): void {
    const host = ts.createSolutionBuilderWithWatchHost(
        ts.sys,
        ts.createEmitAndSemanticDiagnosticsBuilderProgram,
        (diag) => console.error(ts.flattenDiagnosticMessageText(diag.messageText, "\n")),
        (diag) => console.log(ts.flattenDiagnosticMessageText(diag.messageText, "\n")),
        (diag) => console.log(ts.flattenDiagnosticMessageText(diag.messageText, "\n"))
    );

    ts.createSolutionBuilderWithWatch(host, [configPath], {});
}
```

---

## 11. Source Maps

Source Maps ช่วยให้ debug code ต้นฉบับใน browser/Node.js

```typescript
// CompilerOptions สำหรับ source maps
const sourceMapOptions: ts.CompilerOptions = {
    sourceMap: true,           // .js.map
    inlineSourceMap: false,    // ฝัง source map ใน .js
    declarationMap: true,      // .d.ts.map
    inlineSources: false,      // ฝัง source ใน source map
};

// อ่าน source map
import * as sourceMap from "source-map";
import * as fs from "fs";

async function readSourceMap(jsFile: string): Promise<void> {
    const mapFile = jsFile + ".map";
    const rawMap = JSON.parse(fs.readFileSync(mapFile, "utf-8"));
    
    const consumer = await new sourceMap.SourceMapConsumer(rawMap);
    
    // แปลง position จาก generated เป็น original
    const originalPos = consumer.originalPositionFor({
        line: 10,
        column: 5,
    });
    
    console.log(`Original position:`, originalPos);
    // { source: 'src/example.ts', line: 5, column: 2, name: 'myFunction' }
    
    consumer.destroy();
}

// Source map format
interface SourceMapFile {
    version: 3;
    file: string;           // output file name
    sourceRoot: string;     // base URL for sources
    sources: string[];      // original source files
    sourcesContent: string[] | null; // original source contents
    names: string[];        // original identifiers
    mappings: string;       // VLQ encoded mappings
}
```

---

## 12. Declaration Files Generation

TypeScript สร้าง `.d.ts` files สำหรับ type information

```typescript
// สร้าง declaration files
const declProgram = ts.createProgram(["src/index.ts"], {
    declaration: true,
    emitDeclarationOnly: true,
    outDir: "types",
});

// emit แค่ declaration files
declProgram.emit(
    undefined,
    (fileName, text) => {
        if (fileName.endsWith(".d.ts")) {
            console.log(`Generated: ${fileName}`);
            // fs.writeFileSync(fileName, text);
        }
    },
    undefined,
    true // emitOnlyDtsFiles
);

// ตัวอย่าง .d.ts file structure
/*
// Generated by TypeScript 5.x

export interface User {
    name: string;
    age: number;
}

export declare function greet(user: User): string;

export declare class UserService {
    private users;
    addUser(user: User): void;
    getUser(name: string): User | undefined;
}
*/

// สร้าง .d.ts ด้วยตนเอง (custom declaration)
function generateDeclaration(sourceFile: ts.SourceFile, checker: ts.TypeChecker): string {
    const printer = ts.createPrinter({ newLine: ts.NewLineKind.LineFeed });
    const declarations: ts.Node[] = [];
    
    function visit(node: ts.Node) {
        if (ts.isFunctionDeclaration(node)) {
            // สร้าง declare function
            const funcDecl = ts.factory.createFunctionDeclaration(
                [ts.factory.createModifier(ts.SyntaxKind.DeclareKeyword)],
                undefined,
                node.name,
                node.typeParameters,
                node.parameters,
                node.type,
                undefined // ไม่มี body
            );
            declarations.push(funcDecl);
        }
        ts.forEachChild(node, visit);
    }
    
    visit(sourceFile);
    
    const resultFile = ts.createSourceFile(
        "declarations.d.ts",
        "",
        ts.ScriptTarget.Latest,
        false,
        ts.ScriptKind.DTS
    );
    
    return declarations
        .map(decl => printer.printNode(ts.EmitHint.Unspecified, decl, resultFile))
        .join("\n");
}
```

---

## 13. Understanding Error Messages

TypeScript error messages สามารถเข้าใจได้เมื่อรู้โครงสร้างของมัน

```typescript
// Error code categories
// TS1xxx: Syntax errors
// TS2xxx: Semantic/Type errors
// TS4xxx: Declaration emit errors
// TS5xxx: Compiler option errors
// TS6xxx: Build errors
// TS7xxx: Implicit any errors

// ตัวอย่าง errors ที่พบบ่อย
/*
TS2322: Type 'X' is not assignable to type 'Y'
TS2345: Argument of type 'X' is not assignable to parameter of type 'Y'
TS2339: Property 'X' does not exist on type 'Y'
TS2304: Cannot find name 'X'
TS7006: Parameter 'X' implicitly has an 'any' type
TS2531: Object is possibly 'null'
TS2532: Object is possibly 'undefined'
TS2551: Property 'X' does not exist on type 'Y'. Did you mean 'Z'?
*/

// สร้าง error formatter
function formatDiagnosticWithContext(
    diagnostic: ts.Diagnostic,
    program: ts.Program
): string {
    const category = ts.DiagnosticCategory[diagnostic.category];
    const code = `TS${diagnostic.code}`;
    const message = ts.flattenDiagnosticMessageText(diagnostic.messageText, "\n");
    
    if (!diagnostic.file) {
        return `${category} ${code}: ${message}`;
    }
    
    const sourceFile = diagnostic.file;
    const { line, character } = ts.getLineAndCharacterOfPosition(
        sourceFile,
        diagnostic.start!
    );
    
    const lineText = sourceFile.text.split("\n")[line];
    const pointer = " ".repeat(character) + "^".repeat(diagnostic.length || 1);
    
    return [
        `${sourceFile.fileName}:${line + 1}:${character + 1}`,
        `${category} ${code}: ${message}`,
        "",
        `  ${lineText}`,
        `  ${pointer}`,
    ].join("\n");
}

// ตัวอย่าง error chains
/*
Type '{ a: string; }' is not assignable to type '{ a: number; }'.
  Types of property 'a' are incompatible.
    Type 'string' is not assignable to type 'number'.
*/

function parseErrorChain(messageText: ts.DiagnosticMessageChain | string): string[] {
    if (typeof messageText === "string") return [messageText];
    
    const result: string[] = [messageText.messageText];
    if (messageText.next) {
        messageText.next.forEach(chain => {
            result.push(...parseErrorChain(chain).map(msg => `  ${msg}`));
        });
    }
    return result;
}
```

---

## 14. Custom TypeScript Plugin

```typescript
// สร้าง TypeScript Language Service Plugin
// Plugin นี้เพิ่ม completion items สำหรับ Thai language

interface PluginConfig {
    name: string;
    addThaiKeywords?: boolean;
}

function createPlugin(modules: {
    typescript: typeof ts;
}): ts.server.PluginModule {
    const { typescript } = modules;
    
    function create(info: ts.server.PluginCreateInfo): ts.LanguageService {
        const { languageService } = info;
        
        // Override getCompletionsAtPosition
        const originalGetCompletions = languageService.getCompletionsAtPosition.bind(languageService);
        
        languageService.getCompletionsAtPosition = (
            fileName: string,
            position: number,
            options: ts.GetCompletionsAtPositionOptions | undefined
        ) => {
            const original = originalGetCompletions(fileName, position, options);
            
            if (!original) return original;
            
            // เพิ่ม custom completions
            const customEntries: ts.CompletionEntry[] = [
                {
                    name: "console.log", // shorthand
                    kind: typescript.ScriptElementKind.functionElement,
                    sortText: "00",
                    insertText: "console.log($1)",
                    isSnippet: true,
                    labelDetails: { detail: "(snippet)", description: "Log to console" },
                },
            ];
            
            return {
                ...original,
                entries: [...customEntries, ...original.entries],
            };
        };
        
        return languageService;
    }
    
    return { create };
}

// export = createPlugin; // สำหรับ module exports
```

---

## 15. Code Generation ด้วย TypeScript Factory API

```typescript
// TypeScript Factory API ช่วยสร้าง AST nodes ใหม่
const factory = ts.factory;

// สร้าง interface declaration
function createInterface(
    name: string,
    properties: Array<{ name: string; type: string; optional?: boolean }>
): ts.InterfaceDeclaration {
    return factory.createInterfaceDeclaration(
        [factory.createModifier(ts.SyntaxKind.ExportKeyword)],
        name,
        undefined,
        undefined,
        properties.map(prop => 
            factory.createPropertySignature(
                undefined,
                prop.name,
                prop.optional ? factory.createToken(ts.SyntaxKind.QuestionToken) : undefined,
                factory.createTypeReferenceNode(prop.type)
            )
        )
    );
}

// สร้าง function declaration
function createFunction(
    name: string,
    params: Array<{ name: string; type: string }>,
    returnType: string,
    body: ts.Statement[]
): ts.FunctionDeclaration {
    return factory.createFunctionDeclaration(
        [factory.createModifier(ts.SyntaxKind.ExportKeyword)],
        undefined,
        name,
        undefined,
        params.map(p => 
            factory.createParameterDeclaration(
                undefined,
                undefined,
                p.name,
                undefined,
                factory.createTypeReferenceNode(p.type)
            )
        ),
        factory.createTypeReferenceNode(returnType),
        factory.createBlock(body, true)
    );
}

// Print generated code
function printNode(node: ts.Node): string {
    const resultFile = ts.createSourceFile(
        "result.ts",
        "",
        ts.ScriptTarget.Latest,
        false,
        ts.ScriptKind.TS
    );
    
    const printer = ts.createPrinter({ newLine: ts.NewLineKind.LineFeed });
    return printer.printNode(ts.EmitHint.Unspecified, node, resultFile);
}

// ตัวอย่างการใช้งาน
const userInterface = createInterface("User", [
    { name: "id", type: "string" },
    { name: "name", type: "string" },
    { name: "age", type: "number", optional: true },
]);

console.log(printNode(userInterface));
/*
export interface User {
    id: string;
    name: string;
    age?: number;
}
*/

const greetFunction = createFunction(
    "greet",
    [{ name: "user", type: "User" }],
    "string",
    [
        factory.createReturnStatement(
            factory.createTemplateExpression(
                factory.createTemplateHead("Hello, "),
                [factory.createTemplateSpan(
                    factory.createPropertyAccessExpression(
                        factory.createIdentifier("user"),
                        "name"
                    ),
                    factory.createTemplateTail("!")
                )]
            )
        )
    ]
);

console.log(printNode(greetFunction));
/*
export function greet(user: User): string {
    return `Hello, ${user.name}!`;
}
*/
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ TypeScript Compiler Internals:

1. **Architecture** - โครงสร้างของ TypeScript Compiler
2. **Scanner** - การแปลง source code เป็น tokens
3. **Parser & AST** - การสร้าง Abstract Syntax Tree
4. **Binder** - การสร้าง Symbol Table
5. **Type Checker** - การตรวจสอบ type correctness
6. **Emitter** - การสร้าง JavaScript output
7. **Language Service** - IDE features เช่น autocomplete
8. **Watch Mode** - การตรวจสอบการเปลี่ยนแปลง
9. **Incremental Compilation** - การ cache ผลลัพธ์
10. **Project References** - การจัดการ multi-project
11. **Source Maps** - การ debug code ต้นฉบับ
12. **Declaration Files** - การสร้าง .d.ts files
13. **Error Messages** - การเข้าใจ error messages
14. **Plugins** - การสร้าง Language Service Plugins
15. **Code Generation** - การสร้าง code ด้วย Factory API

ความรู้เหล่านี้จะช่วยให้คุณสร้าง TypeScript tools ที่ทรงพลังได้
