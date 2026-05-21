---
title: 前端面试题-TypeScript
date: 2026-05-21 15:00:00
tags: [前端, 面试, TypeScript]
categories: 前端面试
---

**内置工具类型**
    
| 工具类型 | 作用 | 等价写法 |
|----------|------|----------|
| `Partial<T>` | 所有属性变为可选 | `{ [P in keyof T]?: T[P] }` |
| `Required<T>` | 所有属性变为必填 | `{ [P in keyof T]-?: T[P] }` |
| `Readonly<T>` | 所有属性变为只读 | `{ readonly [P in keyof T]: T[P] }` |
| `Pick<T, K>` | 选取 T 中指定属性 K | `{ [P in K]: T[P] }` |
| `Omit<T, K>` | 排除 T 中指定属性 K | `Pick<T, Exclude<keyof T, K>>` |
| `Record<K, T>` | 创建键为 K 类型，值为 T 类型的对象 | `{ [P in K]: T }` |
| `Exclude<T, U>` | 从联合类型 T 中排除 U 类型 | `T extends U ? never : T` |
| `Extract<T, U>` | 从联合类型 T 中提取 U 类型 | `T extends U ? T : never` |

**any和unknown的主要区别是什么**

  - any禁用所有类型检查，相当于退化到JS；unknown要求在使用前显式断言或类型检查，确保操作的安全性。


**为什么使用元组而不是普通数组**

  - 元组精确维护元素的位置和类型，例如[string, number]确保第一个元素是字符串，第二个是数字，防止越界访问和类型错位。


**void和undefined在函数返回值中的区别？**


  - void表示函数没有有效返回值（可返回undefined和null），而undefined要求必须显式返回undefined值 

**双重断言为什么危险**

  - as any as T 绕过类型系统检查，可能导致运行时错误。应优先使用类型守卫或正确的类型声明

**协变和逆变的实践意义？**

  - 协变允许子类型赋值给父类型，逆变允许父类型赋值给子类型。

**type和interface的区别？**

| 特性        | `interface`                  | `type`                       |
| --------- | ---------------------------- | ---------------------------- |
| 定义对象      | ✅ 可以                         | ✅ 可以                         |
| 定义基本类型别名  | ❌ 不能                         | ✅ 可以，例如 `type Name = string` |
| 定义联合/交叉类型 | ❌ 不支持直接                      | ✅ 支持，例如 `type AorB = A 或 B` |
| 扩展        | ✅ `extends` 可以扩展其他 interface | ✅ 可以通过交叉类型 `&` 扩展            |
| 合并声明      | ✅ 同名 interface 会自动合并         | ❌ type 不支持同名合并，会报错           |

**如何选择使用type还是interface？**

  - 定义对象类型时优先使用interface，定义基本类型别名、联合/交叉类型时使用type。

**readonly和const有什么区别？**

  - readonly用于类型层面，表示属性不可修改，但对象本身可变；const用于值层面，表示变量不可重新赋值，但对象属性可修改。

**何时需要编写 .d.ts文件？**

  - 当引入没有类型声明的第三方库时。
  - 扩展已有模块类型
  - 声明全局变量/类型。优先使用@types包，其次考虑自定义声明文件。

**keyof typeof组合使用有什么价值？**

  - keyof typeof可以获取对象的键名类型，常用于限制函数参数为对象的某个属性。例如：

  ```ts
  const Colors = {
    Red: 'red',
    Green: 'green',
    Blue: 'blue',
  } as const;
  type Color = keyof typeof Colors; // 'Red' | 'Green' | 'Blue'
  ```

**satisfies和as断言有什么区别？**

  - as：强制类型转换（可能掩盖错误）
  - satisfies：验证类型兼容性（不改变类型推断）

**declare**

  - 声明全局变量
  - 声明模块
  - 声明函数/类类型