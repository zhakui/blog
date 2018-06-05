---
title: JavaScript继承方式
date: 2013-06-20 11:43:05
categories:
   前端
tags: 
  - JavaScript
---
# JavaScript对象概念
JavaScript的所有数据都可以看作对象，在ECMAScript中，对象是一个无序属性集，这里的“属性”可以是基本值、对象或者函数。JavaScript的面相对象编程和Java、c#的面向对象编程不太一样。在java、c#中面向对象的概念是：
1.类：类是同一类实物的抽象，例如定义一个Person代表人类，类是一种类型，它不代表某个具体的人。
2.实例：实例是根据类创建的对象，例如，根据Person类可以创建出xiaoming、xiaojun等多个实例，每个实例表示一个具体的人，他们全都属于Person类型。
不过，在JavaScript中是不区分类和实例的概念，而是通过原形（prototype）来实现面向对象。原型是指创建的具体的人，而并没有Person类型可用。
```js
var Person = {
    name: 'xiao',
    height: 1.7,
    run: function () {
        console.log(this.name + ' is running');
    }
};
```
上面的Person对象有名字、身高，人也有这两个属性，于是我们就用它来创建一个具体的人小李：
```js
var xiaoli = {
    name: XiaoLi'
};
xiaoli.__proto__ = Person;
```
把person这个对象赋值给__proto__属性，这样xiaoli就继承了run这个方法了。但是，在实际编写JavaScript代码时是不可以直接给__proto__属性赋值的，需要其他方式实现继承。

# 原型链继承
这是实现继承最简单的方式了

## 实现方式：
```js
function Animal(name){
	//属性
	this.name = name || "animal";
	this.features = [];
	//实例方法
	this.sleep = function(){
		console.log(this.name + 'is sleeping!');
	}
}
Animal.prototype.eat = function(){
	console.log(this.name + "is eating milk!");
}

function Cat(){
}
Cat.prototype = New Animal();

var blackCat = new Cat();
var whiteCat = new Cat();

blackCat.name = 'Black Cat';
blackCat.features.push('eat');

//针对父类实例值类型成员的更改，不影响
console.log(blackCat.name); // "Black Cat"
console.log(whiteCat.name); // "animal"
//针对父类实例引用类型成员的更改，会通过影响其他子类实例
console.log(blackCat.features); // ['eat']
console.log(whiteCat.features); // ['eat']
```

## 特点
1. 非常纯粹的继承关系，实例是子类的实例，也是父类的实例
2. 父类新增原型方法/原型属性，子类都能访问到
3. 简单，易于实现

## 缺点
1. 要想为子类新增属性和方法，必须要在new Animal()这样的语句之后执行，不能放到构造器中
2. 无法实现多继承
3. 来自原型对象的引用属性是所有实例共享的
> blackCat.features.push，首先找tom对象的实例属性（找不到），
> 那么去原型对象中找，也就是Animal的实例。发现有，那么就直接在这个对象的
> features属性中插入值。
> 在console.log(whiteCat.features); 的时候。whiteCat实例上也没有，那么去原型上找。
> 原型上有就直接返回，但是这个原型对象中features属性值已经变化了。
4. 创建子类实例时，无法向父类构造函数传参

# 借用构造函数
原型链的范式确实简单，可是存在2个致命缺点，所以就出现了借用构造函数方式

## 实现方式：
```js
function Cat(name){
	Animal.call(this,name);
}

var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // false
console.log(cat instanceof Cat); // true
```

## 特点
1. 解决了子类实例共享父类引用属性的问题
2. 创建子类实例时，可以向父类传递参数
3. 可以实现多继承（call多个父类对象）

## 确定
1. 实例并不是父类的实例，只是子类的实例
2. 只能继承父类的实例属性和方法，不能继承原型属性/方法
3. 无法实现函数复用，每个子类都有父类实例函数的副本，影响性能

# 组合继承
通过调用父类构造，继承父类的属性并保留传参的优点，然后通过将父类实例作为子类原型，实现函数复用

## 实现方式：
```js
function Cat(name){
  Animal.call(this,name);
}
Cat.prototype = new Animal();

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); // true
```

## 特点
1. 弥补了方式2的缺陷，可以继承实例属性/方法，也可以继承原型属性/方法
2. 既是子类的实例，也是父类的实例
3. 不存在引用属性共享问题
4. 可传参
5. 函数可复用

## 缺点
1. 调用了两次父类构造函数，生成了两份实例（子类实例将子类原型上的那份屏蔽了），浪费了点内存

# 寄生组合继承
通过寄生方式，砍掉父类的实例属性，这样，在调用两次父类的构造的时候，就不会初始化两次实例方法/属性，避免的组合继承的缺点

## 实现方式：
```js
function Cat(name){
  Animal.call(this);
  this.name = name || 'Tom';
}
(function(){
  // 创建一个没有实例方法的类
  var Super = function(){};
  Super.prototype = Animal.prototype;
  //将实例作为子类的原型
  Cat.prototype = new Super();
})();

// Test Code
var cat = new Cat();
console.log(cat.name);
console.log(cat.sleep());
console.log(cat instanceof Animal); // true
console.log(cat instanceof Cat); //true
```

## 特点
1. 前面的缺点都没了

## 缺点
1. 实现较为复杂