---
layout: post
title:  "TypesScript 类型编程 🌚"
date:   2021-07-16 03:16:37 +0800
categories: tech
---

TypesScript 类型编程 🌚

# 认识 TypeScript

TypeScript 是微软开发的一款开源编程语言。本质上是向 JavaScript 增加静态类型系统，因此 TypeScript 编译后可跑在支持 JavaScript 的环境中，比如 Node.js 和浏览器。它在设计的时候优先考虑 ECMA-262 规范，保证是 JavaScript 的严格超集，意味着现有 JavaScript 可以不加改变的放到 TypeScript 中。

> JavaScript 直接放到 TypeScript 中会丢失大部分的静态类型检查能力

如图所示：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/17272633906500.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)





以下是 TypeScript 官方文档例子：

```typescript
// 获取长度
function len(s: string): number;
function len(arr: any[]): number;
function len(x: any) {
  return x.length;
}

const sl = len('1921-2021')
const al = len([1921, 2021])
```


重载函数的额外类型声明在编译生成 JavaScript 后会丢失：

```typescript
function len(x) {
    return x.length;
}
const sl = len('1921-2021');
const al = len([1921, 2021]);
```


# 什么是类型编程？

在以往的认知中，TypeScript 是一门给 JavaScript 加上了类型标注的语言，静态类型检查将很多因类型错误导致的 BUG 扼杀在编译时。但最近流行一个和 TypeScript 有关的名词“TS 体操”，TS 即 TypeScript，“体操”是 TC 的戏称，意思是 Turing Completeness（图灵完备）。**一般泛指以一个 TS 类型作为输入，通过写类型编程输出另外一个新类型实践**。

下面是类型编程的一些应用：

1. 驼峰化类型

在业务中，Server 给到的数据一般是下划线命名风格，为了避免破坏驼峰变量命名规范，一般会用 camelize 函数做一次数据转换；但转换出来的类型通常都会设置成 any，通过 TypeScript 类型编程，可以将下划线命名专程驼峰命名。

下图中 user_email 被转换成了 userEmail，同时保留了其原本的类型 string。仔细看图中的 userEmail 类型提示指向了 user_email。甚至可以通过 userEmail 点击找到类型来源 user_email。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/17272641316479.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)



2. 提取 URL 参数

在 Node.js 监听路由场景中，一般会通过特定语法定义 URL 参数，比如 /user/:userId；但 userId 拿不到变量提示，通过 TypeScript 类型编程，可以将 userId 变量及其类型取出，提升编码体验。

下图 userId 的类型提示来自在 URL 路径中定义的 <userId:string>。不光是 URL 路径，查询参数的类型数据也能被提取出来。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/17272641408664.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


# 类型编程工具

TypeScript 为 JavaScript 套了一门类型语言，在使用语言编程之前，需要了解其“语法”。

## 类型变量(泛型)

在 C++ 语言中里有模板的概念，在使用类或函数时可将模板实例化成真正的代码，大大提高了代码的复用性；模板机制可能唯一的不足在于每一次实例化都会在编译时生成一份独立的代码拷贝，虽然减少了代码量，却撑大了编译输出文件的体积。

在 TypeScript 中，提供了和模板类似的概念 - 泛型，和模板不同，泛型不会影响代码编译输出，只是作为承载类型的变量。利用类型变量泛型，可以将身处不同位置的类型“绑定”起来。比如：函数参数、函数内变量以及函数返回值等等。

> 这种绑定关系类似于在 JavaScript 里声明了一个变量，一次修改变(泛)量(型)多处生效。

以 len 函数为例，没有泛型时前面只能使用函数重载实现有限度的代码复用，利用泛型可以节约重载代码：

```typescript
// 函数返回值
function printLen<Type>(x: Type): Type {
  console.log('[length]', x.length);
  return x; // 返回类型和泛型变量绑定
}

// 指定泛型变量的具体类型
printLen <number[]>([20, 21]);    // 函数签名: (x: number[]) => number[];
printLen <string>('something'); // 函数签名: (x: string) => string;

// 泛型自动推导
printLen(['听说最近用其他打车软件打车很便宜']); // 函数签名: (x: string[]) => string;
```


例子中的 Type 不仅增加了 printLen 函数的使用场景，还绑定了参数 x 和返回值类型。但 Type 作为任意类型却不一定包含 length 属性。好在 TypeScript 提供了泛型约束的机制，可以指定赋给泛型的类型必须满足什么条件（子类型）：

```typescript
type PrintLenType<Type extends { length: number }> = (x: Type> => Type;

// 或者这样写
type PrintLenType = <Type extends { length: number }>(x: Type) => Type;
```


extends 提供了一种约束视角，意思是说 Type 可以被当作 { length: number } 类型使用。如果赋值泛型不满足此条件则会在编译时报错。

## keyof 运算符

就像 Flutter 中一切都是 Widget 一样，JavaScript 全都是对象。TypeScript 提供了一种机制，获取对象属性的联合类型。

以 Object.keys 类型为例:

```typescript
type ObjectType = {
  // extends object保证不允许传入null|undefined等非法类型
  keys: <Type extends object>(o: Type) => keyof Type;
}

Object.keys({ name: string, id: string }) // 返回联合类型 => 'name' | 'id'
```


联合类型(union type) 类似于一种看待类型的方式。当提到一个类型是联合类型的时候，就像是在说这个类型可能是几个类型中之一。一般产生在类型生成、定义的阶段，而不是真正使用的时候。举个例子：

```typescript
type I0 = keyof ({
    name: string;
    id: number;
} | {
    name: string;
}); // => { name: string }

```

> 在 TypeScript 里许多类型运算符都是站在联合类型的角度进行类型运算

## typeof 运算符

TypeScript 允许使用 typeof 从 JavaScript 变量中提取类型。

```typescript
// 返回类型 => 'name' | 'id'
const keys = getObjectKeys({ name: string, id: number });

// 作为类型运算时typeof用于提取类型
type KeysType = typeof keys;

// 在JavaScript作为判断运行时类型的语法
console.log(typeof keys); // => 'object'

// 试图提取未赋值的泛型变量会得到unknown
type getObjectKeysReturnType = ReturnType<typeof getObjectKeys>; // unknown
```


> JavaScript 变量不能直接参与类型运算，得先使用 typeof 转换成对应类型

## 索引类型

有时候需要从某个复杂类型中提取出想要的类型，TypeScript 支持索引类型。通过数组下标的方式，只不过下标里面传递的是类型，而非具体值：

```typescript
type Person = { age: number; name: string; alive: boolean };
type I1 = Person['age' | 'name']; // => number | string

const sequence = [0, 1, 1, 2, 3];
type SequenceItem = (typeof sequence)[number]; // => number

// TypeScript中字面量本身也代表各自的类型
// 如: 数字1(number), 布尔false(boolean)等等
type SequenceItem1 = (typeof sequence)[0]; // => number

```

## 映射类型

映射类型可以对类型里的每个属性做修饰，或变换。比如添加 readonly、optional 等等。

以 TypeScript 内建类型 Required 为例:

```typescript
type Required<T> = {
    [P in keyof T]-?: T[P];
};

type Person = {
    name: string;
    wealth?: number;
}

type I2 = Required<Person>; // => { name: string; wealth: number }

```

## 条件类型

条件类型是类型编程中的关键一环，它允许对泛型变量赋值时能针对性的做出反应。
同样以 TypeScript 内建类型 NonNullable 举例：

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;

// 去掉null类型
type T3 = NonNullable<number | null>; // => number
```

同 keyof，索引类型，映射类型一样，条件类型也站在联合类型的角度计算类型。联合类型中每一项都会经过一次“条件运算”：

```typescript
const T4 = number | string;

// 因为extends any，never永远不会触达
type ToArray<T> = T extends any ? T[] : never;

type T4Array = ToArray<T4>; // => number[] | string[]
```

## infer 推断

在这之前，TypeScript 仅支持提取属性名类型(keyof)，属性类型(索引)，infer 把类型提取能力提升到任何类型，任意位置：

```typescript
// 通过infer R提取函数返回值类型
type ReturnType<T extends (...args: any) => any>
    = T extends (...args: any) => infer R ? R : any;
    
type I5 = ReturnType<typeof Date.now>; // => number

// 提取Promise值类型(甚至能拿到泛型变量的类型)
type ThenArg<T> = T extends PromiseLike<infer U> ? U : T;

let futureVal = Promise.resolve(1917);
type I6 = ThenArg<typeof futureVal>; // => number
```

## 模板字面量类型

类型属姓名本质仍然是字符串，TypeScript 提供了一套计算字符串类型的模式，大大丰富了类型计算的能力：


```typescript
type EventName = 'click' | 'scroll' | 'mousedown';

// => 'onClick' | 'onScroll' | 'onMousedown'
type EvenHandler = `on${Capitalize<EventName>}`;
```

TypeScript 内置了 Capitalize、Uppercase及其对应的相反操作类型。

# 实现 Camelize

Camelize 类型可以把下划线风格的属性名转成驼峰命名。要做到这一点得分两步走：

1. 利用模板字面量把下划线转成驼峰 CamelizeString
2. 使用映射类型把单个属性名的变更应用到整个对象类型

## CamelizeString

首先以 user_name 为例，需要去掉下划线并将 name 大写成 Name(利用 Capitalize)；如果不是下划线则返回原类型：


```typescript
type I7<T> = 
    T extends `${infer Left}_${infer Right}` ?
        `${Left}${Capitalize<Right>}` : T;
        
type I8 = I7<'user_name'>; // => 'userName'
```


多个下划线只需要递归的应用上面的类型即可：

```typescript
type CamelizeString<T extends keyof any> = T extends string
  ? Uncapitalize<
      T extends `${infer Left}_${infer Right}`
        ? `${Capitalize<Left>}${Capitalize<CamelizeString<Right>>}`
        : `${Capitalize<T>}`
    >
  : T;
  
  type I9 = CamelizeString<'user_last_name'>; // => 'userLastName'
```


## Camelize

只需要利用类型映射，把变更应用到每一个属性名上，每个属性都递归应用 Camelize 即可：

```typescript
type Camelize<T extends object> = {
      // as语句变更属性到驼峰命名
      [key in keyof T as CamelizeString<key>]:
          T[key] extends object ? Camelize<T[key]> : T[key];
};
```

一些比较特殊的类型（比如数组），需要单独处理：

```typescript
type Camelize<T extends object | object[]> = T extends object[]
  ? Camelize<T[number]>[]
  : {
      [key in keyof T as CamelizeString<key>]:
          T[key] extends object ? Camelize<T[key]> : T[key];
    };
```

拓展思考

1. 如何把驼峰类型转回下划线命名呢？
2. 开头的提取 URL 变量类型如何实现？

参考链接

- [深入typescript类型系统(二): 泛型和类型元编程](https://zhuanlan.zhihu.com/p/96046788)
- [读懂类型体操：TypeScript 类型元编程基础入门](https://zhuanlan.zhihu.com/p/384172236)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/handbook/)
- [cchudant/type-challenges](https://github.com/cchudant/type-challenges/)




