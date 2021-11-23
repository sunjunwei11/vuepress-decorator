[[toc]]
## 装饰器知识点回顾
装饰器是一种特殊类型的声明，它能够被附加到类声明，方法， 访问符，属性或参数上。 装饰器使用 @expression这种形式，expression求值后必须为一个函数，它会在运行时被调用，被装饰的声明信息做为参数传入。

### 类装饰器

::: tip 类装饰器的参数
类装饰器表达式会在运行时当作函数被调用，类的构造函数作为其唯一的参数。
:::

一个简单的类：
```js
class Onboarding {
  public business = 'onboard';
}
```

需要添加一个sayHello方法：
```js {2,7}
const sayHello: ClassDecorator = target => {
  target.prototype.sayHello = () => {
    console.log('hello');
  }
}

@sayHello
class Onboarding {
  public business = 'onboard';
}
```


::: tip 类装饰器有返回值的情况
如果类装饰器返回一个值，它会使用提供的构造函数来替换类的声明

Vue中的@Component装饰器就是通过返回新的构造函数来替换原先的类
:::
```js {2,7}
const returnHello = (target: any) => {
  return class NewOnboarding {
    public business = 'onboard & address';
  }
}

@returnHello
class Onboarding {
  public business = 'onboard';
}

const onboarding = new Onboarding();
console.log('business: ', onboarding.business); // 输出的是onboard & address
```

### 方法装饰器
::: tip 方法装饰器参数
方法装饰器表达式会在运行时当作函数被调用，传入下列3个参数：

1、对于静态成员来说是类的构造函数，对于实例成员是类的原型对象。

2、成员的名字。

3、成员的属性描述符。
:::

```js {2,3,10,17}
const writeCode: MethodDecorator = (target, propertyKey, descriptor: any) => {
  const originalValue = descriptor.value;
  descriptor.value = () => {
    originalValue();
    console.log('write code');
  }
}

class Onboarding {
  @writeCode
  public doSomthing() {
    console.log('doSomthing')
  }
}

const onboarding = new Onboarding();
onboarding.doSomthing(); // 输出 doSomthing、write code
```

### 属性装饰器
::: tip 属性装饰器参数
方法装饰器表达式会在运行时当作函数被调用，传入下列3个参数：

1、对于静态成员来说是类的构造函数，对于实例成员是类的原型对象。

2、成员的名字
:::

::: warning 注意
属性描述符不会做为参数传入属性装饰器，这与TypeScript是如何初始化属性装饰器的有关。 因为目前没有办法在定义一个原型对象的成员时描述一个实例属性，并且没办法监视或修改一个属性的初始化方法。返回值也会被忽略。**因此，属性描述符只能用来监视类中是否声明了某个名字的属性。**
:::

```js {3,7,10,14}
const collectProperty: PropertyDecorator = (target: any, propertyKey) => {
  target.propertys = target.propertys || [];
  target.propertys.push(propertyKey);
}

class Onboarding {
  @collectProperty
  private business!: string;

  @collectProperty
  private num!: number;
}

console.log((Onboarding.prototype as any).propertys); // 输出 ["business", "num"]
```

### 装饰器工厂函数
返回装饰器的函数，利用闭包，装饰器可以访问外层函数的参数
```js
const sayHello = (str: string): ClassDecorator => {
  return (target => {
    target.prototype.sayHello = () => {
      console.log(`hello ${str}`);
    }
  })
}

@sayHello('shopee')
class Onboarding {
  public business = 'onboard';
}

const onboarding = new Onboarding();
(onboarding as any).sayHello(); // 返回hello shopee
```


### 装饰器的执行顺序

我们常用的装饰器有 类装饰器、方法装饰器、属性装饰器

他们的执行顺序是：
1. 属性装饰器
2. 方法装饰器
3. 类装饰器

如果一个属性有多个装饰器，则优先执行下面的

```js
@classDecorator // // 执行顺序：5
class Onboarding {
  @propertyDecorator1 // 执行顺序：2
  @propertyDecorator2 // 执行顺序：1
  public business = 'onboard';

  @methodDecorator1 // 执行顺序：4
  @methodDecorator2 // 执行顺序：3
  public doSomthing() {
    console.log('doSomthing');
  }
}
```

## Vue里面的常用装饰器及其原理
Vue为了支持TS，引入了装饰器，装饰器都放在`vue-property-decorator`这个包中。
```js
import { Component, Prop, Vue, Watch, Model } from 'vue-property-decorator';
```
### @Component
`@component`的作用可以理解为就是把对应的类构造函数转换为Vue组件的配置对象。

装饰器的写法
```js
@Component({})
export default class HelloWorld extends Vue {
  @Prop({ default: '' }) 
  private msg!: string;

  private num1 = 1;
  private num2 = 2;
  get sum() {
    return this.num1 + this.num2;
  }

  @Watch('sum', {
    immediate: true
  })
  private watchSum(oldSum: number, newSum: number) {
    console.log('sum: ', oldSum, newSum);
  }

  private created() {
    console.log('created');
  }

  private sayHello() {
    console.log('hello');
  }
}
```

最终通过`@Component`会被转化为类似下面这样的结构：
```js
export default Vue.extend({
  props: {
    msg: {
      default: ''
    }
  },
  data() {
    return {
      num1: 1,
      num2: 2,
    }
  },
  computed: {
    sum() {
      return this.num1 + this.num2;
    }
  },
  watch: {
    sum: function(oldSum, newSum) {
      console.log('sum: ', oldSum, newSum);
    }
  },
  created() {
    console.log('created');
  }
  methods: {
    sayHello: function() {
      console.log('hello');
    }
  }
});
```

类装饰器可以返回一个新的构造函数替换掉原先的构造函数，所以Component装饰器的大致原理就是
```js
const Component: ClassDecorator = target => {
  const res = {};
  // 对target进行处理
  // 从target上拿到data、computed、methods、生命周期相关的配置，添加到res中
  // ...
  // ...
  // ...
  return Vue.extend(res);
}
```

源码中的处理：

首先要处理是否是装饰器工厂的两种情况
```js
function Component(options) {
    // options是函数，即参数已经是一个类构造函数，则说明不是装饰器工厂，使用方式为@Component
    if (typeof options === 'function') {
        return componentFactory(options);
    }
    // 否则认为是装饰器工厂函数，使用方式是@Component({...})
    // 需要返回最终使用的类装饰器
    return function (Component) {
        // 在装饰器里再返回新的构造函数，替换原先的构造函数
        return componentFactory(Component, options);
    };
}
```

componentFactory的实现
```js
// 第一个参数是类构造函数
// 第二个参数是我们给装饰器工厂传的参数
const componentFactory: ClassDecorator = (Component, options = {}) {
  // 首先得到构造函数的原型对象
  const proto = Component.prototype;
  // 遍历原型对象，将原型对象上定义的生命周期钩子、普通函数、data、computed添加到options上
  Object.getOwnPropertyNames(proto).forEach(function (key) {
      if (key === 'constructor') {
          return;
      }
      // hooks created、mounted这些钩子函数
      if ($internalHooks.indexOf(key) > -1) {
          options[key] = proto[key];
          return;
      }
      const descriptor = Object.getOwnPropertyDescriptor(proto, key);
      if (descriptor.value !== void 0) {
          // 普通函数添加到options.methods中
          if (typeof descriptor.value === 'function') {
              (options.methods || (options.methods = {}))[key] = descriptor.value;
          }
          else {
              // 其它的认为都是可响应属性，添加到options.data中
              (options.mixins || (options.mixins = [])).push({
                  data() {
                      return { [key]: descriptor.value };
                  }
              });
          }
      }
      // get和set认为是computed属性，添加到options.computed中
      else if (descriptor.get || descriptor.set) {
          (options.computed || (options.computed = {}))[key] = {
              get: descriptor.get,
              set: descriptor.set
          };
      }
  });

  // 对prop、watch、Model 统一处理...
  const decorators = Component.__decorators__;
  decorators.forEach(fn => fn(options));

  // 最终返回Vue构造函数
  return Vue.extend(options);
}
```

### @Prop
`Prop`是一个装饰器工厂

使用
```js
@Component({})
export default class HelloWorld extends Vue {
  @Prop({ default: '' }) 
  private msg!: string;
}
```

核心原理
```js
// options是我们给@Prop传的参数
const Prop = (options: any): PropertyDecorator => {
  // 返回一个属性装饰器
  // 第一个参数target是类构造函数的原型对象
  // 第二个参数是要属性的名称
  return (target, key) => {
    const Ctor = target.constructor;
    Ctor.__decorators__ = Ctor.__decorators__ || [];

    // 这个vueOptions是Component装饰器里最终会得到的配置对象
    // 在Component装饰器里会调用Ctor.__decorators__里的所有函数，并且把这个配置对象传进去
    Ctor.__decorators__.push(vueOptions => vueOptions.props[key] = options);
  }
}
```

::: tip 提示
因为装饰器的执行顺序是先执行属性装饰器，再执行类装饰器，所以Prop的装饰器先执行，会在类的构造函数的__decorators__属性上添加一个方法。
类装饰器执行的时候会调用这些方法，并且传入一个配置对象作为参数。
:::

在Component装饰器里会有如下代码：
```js {11-13}
// 第一个参数是类构造函数
// 第二个参数是我们给装饰器工厂传的参数
const componentFactory: ClassDecorator = (Component, options = {}) {
  // 首先得到构造函数的原型对象
  const proto = Component.prototype;
  // 遍历原型对象，将原型对象上定义的生命周期钩子、普通函数、data、computed添加到options上
  Object.getOwnPropertyNames(proto).forEach(function (key) {
    // ...
  });

  // 这里是对props的处理...
  const decorators = Component.__decorators__;
  decorators.forEach(fn => fn(options));

  // 最终返回Vue构造函数
  return Vue.extend(options);
}
```

### @Watch
`Watch`也是一个装饰器工厂，接收要监听的路径以及对应的配置选项，返回一个函数装饰器

使用
```js
@Component({})
export default class HelloWorld extends Vue {
  @Watch('sum', {
    immediate: true
  })
  private watchSum(oldSum: number, newSum: number) {
    console.log('sum: ', oldSum, newSum);
  }
}
```

核心原理
```js
// path是我们要监听的属性
const Watch = (path: string, options: any): MethodDecorator => {
  // 返回一个方法装饰器
  // 第一个参数target是类构造函数的原型对象
  // 第二个参数是要方法的名称
  return (target, methodName) => {
    const Ctor = target.constructor;
    Ctor.__decorators__ = Ctor.__decorators__ || [];

    // 这个vueOptions是Component装饰器里最终会得到的配置对象
    // 在Component装饰器里会调用Ctor.__decorators__里的所有函数，并且把这个配置对象传进去
    Ctor.__decorators__.push(vueOptions => vueOptions.watch[path] = {
      handler: methodName,
      deep: options.deep,
      immediate: options.immediate
    });
  }
}
```

::: tip 提示 handler: methodName
因为`@Watch`装饰的是一个函数，这个函数也会在`@Component`里被处理，会添加到options.methods里面，所以在watch里面的handler属性传对应的函数名就可以了。
:::

### @Emit
`Emit`也是一个装饰器工厂

使用
```js
export default class HelloWorld extends Vue {
  @Emit('sumChange')
  private watchSum(oldSum: number, newSum: number) {
    return newSum;
  }
}
```

原理：先执行原有的函数，在函数执行完之后拿到结果，然后再执行`this.$emit('sumChange', newSum)`

核心原理
```js
/**
 * decorator of an event-emitter function
 * @param  event The name of the event
 * @return MethodDecorator
 */
function Emit(event) {
  return function (_target, propertyKey, descriptor) {
      var original = descriptor.value;
      descriptor.value = function emitter() {
          var returnValue = original.apply(this, args);
          if (isPromise(returnValue)) {
              returnValue.then(this.$emit(event, returnValue));
          }
          else {
              this.$emit(event, returnValue);
          }
          return returnValue;
      };
  };
}
```
### @Model

一个组件上的 v-model 默认会利用名为 value 的 prop 和名为 input 的事件，但是像单选框、复选框等类型的输入控件可能会将 value attribute 用于不同的目的。
model 选项可以用来避免这样的冲突：

常规用法：
```js
Vue.component('base-checkbox', {
  model: {
    prop: 'checked',
    event: 'change'
  },
  props: {
    checked: Boolean
  },
  template: `
    <input
      type="checkbox"
      v-bind:checked="checked"
      v-on:change="$emit('change', $event.target.checked)"
    >
  `
})
```
::: warning 提示
仍然需要在组件的 props 选项里声明 checked 这个 prop。
:::


@Model的使用
```js
export default class HelloWorld extends Vue {
  @Model('add', { default: 0 })
  private num!: number
}
```

相当于
```js
Vue.component('HelloWorld', {
  model: {
    prop: 'num',
    event: 'add'
  },
  props: {
    num: {
      default: 0
    }
  }
})
```

核心原理
```js
/**
 * decorator of model
 * @param  event event name
 * @param options options
 * @return PropertyDecorator
 */
 export function Model(event, options = {}): PropertyDecorator {
  // 返回一个属性装饰器，target是原型对象
  return function (target, key) {
    const Ctor = target.constructor;
    Ctor.__decorators__ = Ctor.__decorators__ || [];

    // 这个vueOptions是Component装饰器里最终会得到的配置对象
    // 在Component装饰器里会调用Ctor.__decorators__里的所有函数，并且把这个配置对象传进去
    Ctor.__decorators__.push(vueOptions => {
      vueOptions.model = {
        prop: key,
        event: event
      }
      vueOptions.props[key] = options;
    });
  };
}
```

