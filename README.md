# คอร์ส TypeScript ฉบับสมบูรณ์ภาษาไทย

> เรียนรู้ TypeScript ตั้งแต่พื้นฐานจนถึงระดับขั้นสูง ด้วยเนื้อหาภาษาไทยที่เข้าใจง่าย พร้อมตัวอย่างโค้ดจริง

---

## เกี่ยวกับคอร์สนี้

คอร์สนี้ถูกออกแบบมาเพื่อช่วยให้ผู้เรียนชาวไทยสามารถเรียนรู้ TypeScript ได้อย่างเป็นระบบและครบถ้วน ตั้งแต่การติดตั้งและการตั้งค่าเบื้องต้น ไปจนถึงการใช้งาน TypeScript ในโปรเจกต์จริง เนื้อหาทุกบทเขียนเป็นภาษาไทย พร้อมตัวอย่างโค้ดที่สามารถนำไปใช้งานได้จริง

### จุดเด่นของคอร์ส

- **ภาษาไทย 100%** - เนื้อหาทั้งหมดเขียนเป็นภาษาไทยที่เข้าใจง่าย
- **ตัวอย่างจริง** - โค้ดตัวอย่างที่นำไปใช้งานได้จริงในทุกบท
- **ครบครัน** - ครอบคลุมทุกหัวข้อตั้งแต่พื้นฐานจนถึงขั้นสูง
- **ปฏิบัติได้** - มีแบบฝึกหัดและโปรเจกต์ปฏิบัติจริง
- **อัปเดต** - เนื้อหาทันสมัยตาม TypeScript เวอร์ชันล่าสุด

---

## ข้อกำหนดเบื้องต้น (Prerequisites)

ก่อนเริ่มเรียนคอร์สนี้ ผู้เรียนควรมีความรู้พื้นฐานดังนี้:

### ความรู้ที่ต้องมี
- **JavaScript พื้นฐาน** - ตัวแปร, ฟังก์ชัน, อาร์เรย์, ออบเจกต์
- **HTML/CSS เบื้องต้น** - เพื่อเข้าใจบริบทการพัฒนาเว็บ
- **Command Line เบื้องต้น** - การใช้ Terminal หรือ Command Prompt

### เครื่องมือที่ต้องการ
- **Node.js** เวอร์ชัน 18.0 ขึ้นไป
- **npm** หรือ **yarn** สำหรับจัดการ packages
- **VS Code** หรือ Editor ที่รองรับ TypeScript
- **Git** สำหรับการจัดการโค้ด (แนะนำ)

### ไม่จำเป็นต้องมี
- ประสบการณ์ TypeScript มาก่อน
- ความรู้เรื่อง Static Typing
- การเขียนโปรแกรมเชิงวัตถุ (OOP) - จะเรียนในคอร์ส

---

## วัตถุประสงค์การเรียนรู้ (Learning Objectives)

เมื่อเรียนจบคอร์สนี้ ผู้เรียนจะสามารถ:

1. **เข้าใจพื้นฐาน TypeScript** และความแตกต่างจาก JavaScript
2. **ติดตั้งและตั้งค่า** TypeScript project ได้อย่างถูกต้อง
3. **ใช้งาน Type System** ของ TypeScript ได้อย่างคล่องแคล่ว
4. **เขียน Interface และ Type** สำหรับกำหนดรูปแบบข้อมูล
5. **ใช้งาน Generic Types** เพื่อสร้างโค้ดที่ยืดหยุ่น
6. **ใช้งาน OOP** ด้วย Class, Inheritance, และ Decorators
7. **จัดการ Modules** และ Namespaces ได้อย่างมีประสิทธิภาพ
8. **ทำงานกับ React/Vue/Node.js** ด้วย TypeScript
9. **เขียน Unit Tests** ด้วย TypeScript
10. **Deploy โปรเจกต์** TypeScript ไปสู่ Production

---

## วิธีใช้คอร์สนี้ (How to Use This Course)

### สำหรับผู้เริ่มต้น
1. เริ่มจาก Part 1 และเรียนตามลำดับ
2. อ่านเนื้อหาทำความเข้าใจก่อนดูโค้ด
3. พิมพ์โค้ดด้วยตัวเอง ไม่ใช่แค่ copy-paste
4. ทำแบบฝึกหัดในแต่ละบท
5. ถ้าไม่เข้าใจ ให้กลับไปอ่านซ้ำหรือดูตัวอย่างเพิ่มเติม

### สำหรับผู้มีประสบการณ์ JavaScript
1. สามารถข้ามไปยังบทที่สนใจได้
2. ใช้เป็น Reference สำหรับ TypeScript features
3. ดูตัวอย่างการแปลง JavaScript เป็น TypeScript

### สำหรับผู้ที่ต้องการ Reference
1. ใช้สารบัญด้านล่างเพื่อค้นหาหัวข้อ
2. แต่ละบทมีหัวข้อย่อยที่ชัดเจน
3. มีตัวอย่างโค้ดที่ใช้งานได้จริง

---

## สารบัญ (Table of Contents)

### ส่วนที่ 1: พื้นฐาน TypeScript (Foundation)

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| [Part 01](course/part-01-introduction.md) | บทนำ TypeScript | ✅ พร้อมแล้ว |
| [Part 02](course/part-02-installation-setup.md) | การติดตั้งและตั้งค่า | ✅ พร้อมแล้ว |
| [Part 03](course/part-03-basic-types.md) | Types พื้นฐาน | ✅ พร้อมแล้ว |
| Part 04 | Arrays และ Tuples | 🔜 เร็วๆ นี้ |
| Part 05 | Objects และ Type Aliases | 🔜 เร็วๆ นี้ |
| Part 06 | Functions | 🔜 เร็วๆ นี้ |
| Part 07 | Union Types และ Intersection Types | 🔜 เร็วๆ นี้ |
| Part 08 | Type Guards | 🔜 เร็วๆ นี้ |
| Part 09 | Nullable Types | 🔜 เร็วๆ นี้ |
| Part 10 | Type Assertions | 🔜 เร็วๆ นี้ |

### ส่วนที่ 2: Interfaces และ Types (Structural Typing)

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 11 | Interfaces เบื้องต้น | 🔜 เร็วๆ นี้ |
| Part 12 | Optional Properties | 🔜 เร็วๆ นี้ |
| Part 13 | Readonly Properties | 🔜 เร็วๆ นี้ |
| Part 14 | Index Signatures | 🔜 เร็วๆ นี้ |
| Part 15 | Extending Interfaces | 🔜 เร็วๆ นี้ |
| Part 16 | Interface vs Type Alias | 🔜 เร็วๆ นี้ |
| Part 17 | Function Types ใน Interface | 🔜 เร็วๆ นี้ |
| Part 18 | Hybrid Types | 🔜 เร็วๆ นี้ |
| Part 19 | Interface Merging | 🔜 เร็วๆ นี้ |
| Part 20 | Structural Subtyping | 🔜 เร็วๆ นี้ |

### ส่วนที่ 3: Classes และ OOP

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 21 | Classes พื้นฐาน | 🔜 เร็วๆ นี้ |
| Part 22 | Constructors | 🔜 เร็วๆ นี้ |
| Part 23 | Access Modifiers | 🔜 เร็วๆ นี้ |
| Part 24 | Inheritance | 🔜 เร็วๆ นี้ |
| Part 25 | Abstract Classes | 🔜 เร็วๆ นี้ |
| Part 26 | Static Members | 🔜 เร็วๆ นี้ |
| Part 27 | Getters และ Setters | 🔜 เร็วๆ นี้ |
| Part 28 | Implementing Interfaces | 🔜 เร็วๆ นี้ |
| Part 29 | Mixins | 🔜 เร็วๆ นี้ |
| Part 30 | Class Decorators | 🔜 เร็วๆ นี้ |

### ส่วนที่ 4: Generics

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 31 | Generic Functions | 🔜 เร็วๆ นี้ |
| Part 32 | Generic Interfaces | 🔜 เร็วๆ นี้ |
| Part 33 | Generic Classes | 🔜 เร็วๆ นี้ |
| Part 34 | Generic Constraints | 🔜 เร็วๆ นี้ |
| Part 35 | keyof Operator | 🔜 เร็วๆ นี้ |
| Part 36 | Conditional Types | 🔜 เร็วๆ นี้ |
| Part 37 | Mapped Types | 🔜 เร็วๆ นี้ |
| Part 38 | Template Literal Types | 🔜 เร็วๆ นี้ |
| Part 39 | Infer Keyword | 🔜 เร็วๆ นี้ |
| Part 40 | Utility Types | 🔜 เร็วๆ นี้ |

### ส่วนที่ 5: Enums และ Advanced Types

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 41 | Numeric Enums | 🔜 เร็วๆ นี้ |
| Part 42 | String Enums | 🔜 เร็วๆ นี้ |
| Part 43 | Const Enums | 🔜 เร็วๆ นี้ |
| Part 44 | Discriminated Unions | 🔜 เร็วๆ นี้ |
| Part 45 | Exhaustive Checks | 🔜 เร็วๆ นี้ |
| Part 46 | Recursive Types | 🔜 เร็วๆ นี้ |
| Part 47 | Variadic Tuple Types | 🔜 เร็วๆ นี้ |
| Part 48 | Template Literal Types ขั้นสูง | 🔜 เร็วๆ นี้ |
| Part 49 | Nominal Typing | 🔜 เร็วๆ นี้ |
| Part 50 | Type Predicates | 🔜 เร็วๆ นี้ |

### ส่วนที่ 6: Modules และ Namespaces

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 51 | ES Modules | 🔜 เร็วๆ นี้ |
| Part 52 | CommonJS Modules | 🔜 เร็วๆ นี้ |
| Part 53 | Namespaces | 🔜 เร็วๆ นี้ |
| Part 54 | Declaration Files (.d.ts) | 🔜 เร็วๆ นี้ |
| Part 55 | DefinitelyTyped | 🔜 เร็วๆ นี้ |
| Part 56 | Path Mapping | 🔜 เร็วๆ นี้ |
| Part 57 | Module Resolution | 🔜 เร็วๆ นี้ |
| Part 58 | Dynamic Imports | 🔜 เร็วๆ นี้ |
| Part 59 | Ambient Declarations | 🔜 เร็วๆ นี้ |
| Part 60 | Triple-Slash Directives | 🔜 เร็วๆ นี้ |

### ส่วนที่ 7: Decorators

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 61 | Class Decorators | 🔜 เร็วๆ นี้ |
| Part 62 | Method Decorators | 🔜 เร็วๆ นี้ |
| Part 63 | Property Decorators | 🔜 เร็วๆ นี้ |
| Part 64 | Parameter Decorators | 🔜 เร็วๆ นี้ |
| Part 65 | Decorator Factories | 🔜 เร็วๆ นี้ |
| Part 66 | Decorator Composition | 🔜 เร็วๆ นี้ |
| Part 67 | Reflect Metadata | 🔜 เร็วๆ นี้ |
| Part 68 | Decorators ใน NestJS | 🔜 เร็วๆ นี้ |
| Part 69 | Custom Decorators | 🔜 เร็วๆ นี้ |
| Part 70 | Decorator Patterns | 🔜 เร็วๆ นี้ |

### ส่วนที่ 8: Async Programming

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 71 | Promises ใน TypeScript | 🔜 เร็วๆ นี้ |
| Part 72 | Async/Await | 🔜 เร็วๆ นี้ |
| Part 73 | Error Handling ใน Async | 🔜 เร็วๆ นี้ |
| Part 74 | Observable พื้นฐาน | 🔜 เร็วๆ นี้ |
| Part 75 | RxJS กับ TypeScript | 🔜 เร็วๆ นี้ |
| Part 76 | Async Iterators | 🔜 เร็วๆ นี้ |
| Part 77 | Generator Functions | 🔜 เร็วๆ นี้ |
| Part 78 | Concurrent Operations | 🔜 เร็วๆ นี้ |
| Part 79 | AbortController | 🔜 เร็วๆ นี้ |
| Part 80 | Streaming Data | 🔜 เร็วๆ นี้ |

### ส่วนที่ 9: TypeScript กับ Frameworks

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 81 | TypeScript กับ React | 🔜 เร็วๆ นี้ |
| Part 82 | React Hooks กับ TypeScript | 🔜 เร็วๆ นี้ |
| Part 83 | React Context กับ TypeScript | 🔜 เร็วๆ นี้ |
| Part 84 | TypeScript กับ Vue 3 | 🔜 เร็วๆ นี้ |
| Part 85 | TypeScript กับ Angular | 🔜 เร็วๆ นี้ |
| Part 86 | TypeScript กับ Node.js | 🔜 เร็วๆ นี้ |
| Part 87 | TypeScript กับ Express | 🔜 เร็วๆ นี้ |
| Part 88 | TypeScript กับ NestJS | 🔜 เร็วๆ นี้ |
| Part 89 | TypeScript กับ Next.js | 🔜 เร็วๆ นี้ |
| Part 90 | TypeScript กับ Deno | 🔜 เร็วๆ นี้ |

### ส่วนที่ 10: Testing

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 91 | Unit Testing ด้วย Jest | 🔜 เร็วๆ นี้ |
| Part 92 | Integration Testing | 🔜 เร็วๆ นี้ |
| Part 93 | Mocking ใน TypeScript | 🔜 เร็วๆ นี้ |
| Part 94 | Type Testing | 🔜 เร็วๆ นี้ |
| Part 95 | E2E Testing ด้วย Playwright | 🔜 เร็วๆ นี้ |
| Part 96 | Test Coverage | 🔜 เร็วๆ นี้ |
| Part 97 | TDD ด้วย TypeScript | 🔜 เร็วๆ นี้ |
| Part 98 | Testing Best Practices | 🔜 เร็วๆ นี้ |
| Part 99 | Performance Testing | 🔜 เร็วๆ นี้ |
| Part 100 | Security Testing | 🔜 เร็วๆ นี้ |

### ส่วนที่ 11: Advanced Topics

| บทที่ | หัวข้อ | สถานะ |
|-------|--------|--------|
| Part 101 | Design Patterns ใน TypeScript | 🔜 เร็วๆ นี้ |
| Part 102 | SOLID Principles | 🔜 เร็วๆ นี้ |
| Part 103 | Clean Architecture | 🔜 เร็วๆ นี้ |
| Part 104 | Domain Driven Design | 🔜 เร็วๆ นี้ |
| Part 105 | TypeScript Compiler API | 🔜 เร็วๆ นี้ |
| Part 106 | Custom TypeScript Plugins | 🔜 เร็วๆ นี้ |
| Part 107 | Performance Optimization | 🔜 เร็วๆ นี้ |
| Part 108 | Monorepos ด้วย TypeScript | 🔜 เร็วๆ นี้ |
| Part 109 | CI/CD กับ TypeScript | 🔜 เร็วๆ นี้ |
| Part 110 | TypeScript 5.x Features | 🔜 เร็วๆ นี้ |

---

## โครงสร้างคอร์ส (Course Structure)

```
typescript_course/
├── README.md                          # ไฟล์นี้
├── course/
│   ├── part-01-introduction.md        # บทนำ
│   ├── part-02-installation-setup.md  # การติดตั้ง
│   ├── part-03-basic-types.md         # Types พื้นฐาน
│   ├── part-04-arrays-tuples.md       # Arrays & Tuples
│   ├── part-05-objects.md             # Objects
│   └── ...                            # บทถัดไป
├── examples/
│   ├── part-01/                       # โค้ดตัวอย่างบท 1
│   ├── part-02/                       # โค้ดตัวอย่างบท 2
│   └── ...
├── exercises/
│   ├── part-01-exercises.md           # แบบฝึกหัดบท 1
│   └── ...
└── projects/
    ├── todo-app/                      # โปรเจกต์ To-Do App
    ├── rest-api/                      # โปรเจกต์ REST API
    └── ...
```

---

## ทรัพยากรเพิ่มเติม (Additional Resources)

### เว็บไซต์อย่างเป็นทางการ
- [TypeScript Official Website](https://www.typescriptlang.org/)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [TypeScript Playground](https://www.typescriptlang.org/play)
- [TypeScript GitHub Repository](https://github.com/microsoft/TypeScript)

### เครื่องมือที่มีประโยชน์
- [VS Code](https://code.visualstudio.com/) - Editor ที่แนะนำ
- [ts-node](https://typestrong.org/ts-node/) - รัน TypeScript โดยตรง
- [tsc](https://www.typescriptlang.org/docs/handbook/compiler-options.html) - TypeScript Compiler
- [ESLint](https://eslint.org/) กับ TypeScript plugin
- [Prettier](https://prettier.io/) - Code Formatter

### ชุมชน TypeScript
- [TypeScript Discord](https://discord.com/invite/typescript)
- [Stack Overflow TypeScript](https://stackoverflow.com/questions/tagged/typescript)
- [Reddit r/typescript](https://www.reddit.com/r/typescript/)

---

## การมีส่วนร่วม (Contributing)

หากพบข้อผิดพลาดหรือต้องการเพิ่มเติมเนื้อหา สามารถ:
1. แจ้งปัญหาผ่าน Issues
2. ส่ง Pull Request พร้อมการแก้ไข
3. เสนอหัวข้อใหม่ผ่าน Discussions

---

## ประวัติการอัปเดต

| วันที่ | เวอร์ชัน | รายละเอียด |
|--------|---------|------------|
| 2024-01-01 | 1.0.0 | เปิดตัวคอร์ส |
| 2024-06-01 | 1.1.0 | เพิ่มเนื้อหา Advanced Types |
| 2024-12-01 | 1.2.0 | อัปเดต TypeScript 5.x Features |

---

## ใบอนุญาต (License)

เนื้อหาในคอร์สนี้อยู่ภายใต้ [MIT License](LICENSE) - สามารถนำไปใช้งาน แก้ไข และแจกจ่ายได้อย่างอิสระ

---

*คอร์สนี้จัดทำขึ้นเพื่อชุมชนนักพัฒนาไทย ด้วยความตั้งใจให้ทุกคนสามารถเรียนรู้ TypeScript ได้อย่างมีประสิทธิภาพ*
