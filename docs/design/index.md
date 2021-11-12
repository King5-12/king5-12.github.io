---
title: 设计模式
nav:
  title: 设计
---

## 工厂模式

### 所需条件

工厂类=>对应不同的工厂的共同属性及方法并需要有一个方法对应产品接口。

产品类型接口=>对应不同产品的共同属性及方法。

```js
//通用工厂类
//需要返回一个产品接口的方法
abstract class Creator {
    public abstract factoryMethod(): Product;
    public someOperation(): string {
        const product = this.factoryMethod();
        return `Creator: The same creator's code has just worked with ${product.operation()}`;
    }
}
//工厂1
class ConcreteCreator1 extends Creator {
    public factoryMethod(): Product {
        return new ConcreteProduct1();
    }
}
//工厂2
class ConcreteCreator2 extends Creator {
    public factoryMethod(): Product {
        return new ConcreteProduct2();
    }
}
//产品接口
interface Product {
    operation(): string;
}
//生成的产品1
class ConcreteProduct1 implements Product {
    public operation(): string {
        return '{Result of the ConcreteProduct1}';
    }
}
//生成的产品2
class ConcreteProduct2 implements Product {
    public operation(): string {
        return '{Result of the ConcreteProduct2}';
    }
}
function clientCode(creator: Creator) {
    console.log('Client: I\'m not aware of the creator\'s class, but it still works.');
    console.log(creator.someOperation());
}
console.log('App: Launched with the ConcreteCreator1.');
clientCode(new ConcreteCreator1());
console.log('');
console.log('App: Launched with the ConcreteCreator2.');
clientCode(new ConcreteCreator2());
```

## 抽象工厂

### 所需条件

工厂接口分别对应生成不同产品的函数。

为不同产品都编写对应的接口。

工厂类对应某一套风格的产品函数。

不同风格的不同产品都需要自己的类型。

### 优点

- 能确保同一工厂生产的产品相互匹配。
- 可以避免客户端和具体产品代码耦合。
- 单一职责原则。可以将产品生成代码抽取到同一位置，便于维护。
- 开闭原则。向应用程序中引入新产品变体时，无需修改客户端代码。

### 缺点

- 引入众多接口和类，代码更复杂。

```js
//工厂接口分别对应产品A和产品B
interface AbstractFactory {
    createProductA(): AbstractProductA;
    createProductB(): AbstractProductB;
}
//工厂1类，生产产品A和产品B的1风格
class ConcreteFactory1 implements AbstractFactory {
    public createProductA(): AbstractProductA {
        return new ConcreteProductA1();
    }

    public createProductB(): AbstractProductB {
        return new ConcreteProductB1();
    }
}

//工厂2类，生产产品A和产品B的2风格
class ConcreteFactory2 implements AbstractFactory {
    public createProductA(): AbstractProductA {
        return new ConcreteProductA2();
    }

    public createProductB(): AbstractProductB {
        return new ConcreteProductB2();
    }
}
//产品A的接口
interface AbstractProductA {
    usefulFunctionA(): string;
}

//1风格的产品A
class ConcreteProductA1 implements AbstractProductA {
    public usefulFunctionA(): string {
        return 'The result of the product A1.';
    }
}
//2风格的产品A
class ConcreteProductA2 implements AbstractProductA {
    public usefulFunctionA(): string {
        return 'The result of the product A2.';
    }
}
//产品B的接口
interface AbstractProductB {
    usefulFunctionB(): string;
    anotherUsefulFunctionB(collaborator: AbstractProductA): string;
}
//风格1的产品B
class ConcreteProductB1 implements AbstractProductB {
    public usefulFunctionB(): string {
        return 'The result of the product B1.';
    }
    public anotherUsefulFunctionB(collaborator: AbstractProductA): string {
        const result = collaborator.usefulFunctionA();
        return `The result of the B1 collaborating with the (${result})`;
    }
}
//风格2的产品B
class ConcreteProductB2 implements AbstractProductB {
    public usefulFunctionB(): string {
        return 'The result of the product B2.';
    }
    public anotherUsefulFunctionB(collaborator: AbstractProductA): string {
        const result = collaborator.usefulFunctionA();
        return `The result of the B2 collaborating with the (${result})`;
    }
}

function clientCode(factory: AbstractFactory) {
    const productA = factory.createProductA();
    const productB = factory.createProductB();
    console.log(productB.usefulFunctionB());
    console.log(productB.anotherUsefulFunctionB(productA));
}
console.log('Client: Testing client code with the first factory type...');
clientCode(new ConcreteFactory1());
console.log('');
console.log('Client: Testing the same client code with the second factory type...');
clientCode(new ConcreteFactory2());
```

## 生成器

### 所需条件

生成器通用函数接口。

多个具体生成器类（分别生成不同规格的产品）

产生不同生成器的主管类。

产品类。

### 优点

- 单一职责原则。可以将复杂的构造代码分离出来
- 可以避免客户端和具体产品代码耦合你可以分步创建对象， 暂缓创建步骤或递归运行创建步骤。
- 你可以分步创建对象， 暂缓创建步骤或递归运行创建步骤。

```js
//生成器的通用接口
interface Builder {
    producePartA(): void;
    producePartB(): void;
    producePartC(): void;
}
//生成器1，可以构建产品的a、b、c等部分
class ConcreteBuilder1 implements Builder {
    private product: Product1;

    constructor() {
        this.reset();
    }

    public reset(): void {
        this.product = new Product1();
    }

    public producePartA(): void {
        this.product.parts.push('PartA1');
    }

    public producePartB(): void {
        this.product.parts.push('PartB1');
    }

    public producePartC(): void {
        this.product.parts.push('PartC1');
    }

    public getProduct(): Product1 {
        const result = this.product;
        this.reset();
        return result;
    }
}
//产品1
class Product1 {
    public parts: string[] = [];

    public listParts(): void {
        console.log(`Product parts: ${this.parts.join(', ')}\n`);
    }
}
//主管类，可以返回具体某个生成器以及控制生成器构建产品各部分的顺序
class Director {
    private builder: Builder;

    public setBuilder(builder: Builder): void {
        this.builder = builder;
    }

    public buildMinimalViableProduct(): void {
        this.builder.producePartA();
    }

    public buildFullFeaturedProduct(): void {
        this.builder.producePartA();
        this.builder.producePartB();
        this.builder.producePartC();
    }
}

function clientCode(director: Director) {
    const builder = new ConcreteBuilder1();
    director.setBuilder(builder);

    console.log('Standard basic product:');
    director.buildMinimalViableProduct();
    builder.getProduct().listParts();

    console.log('Standard full featured product:');
    director.buildFullFeaturedProduct();
    builder.getProduct().listParts();

    console.log('Custom product:');
    builder.producePartA();
    builder.producePartC();
    builder.getProduct().listParts();
}

const director = new Director();
clientCode(director);
```
