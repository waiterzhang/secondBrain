# 描述UI

## 将Props传递给组件

react组件使用props进行通信，实际上就是参数传递。props可以传递任何参数，包括对象、数组和函数。

**1、将参数传递给html的标签：**

```javascript
function Avatar() {
  return (
    <img
      className="avatar"
      src="https://i.imgur.com/1bX5QH6.jpg"
      alt="Lin Lanying"
      width={100}
      height={100}
    />
  );
}

export default function Profile() {
  return (
    <Avatar />
  );
}
```

* 参数的书写方式和原生html差不多；
* class是关键字，所以替换成了className；

**2、将参数传递给react的组件：**

```javascript
export default function Profile() {
  return (
    <Avatar
      person={{ name: 'Lin Lanying', imageId: '1bX5QH6' }}
      size={100}
    />
  );
}
```

* 双重{}的内层{}可以理解为一个对象，外层{}就是js的语法块声明；

**3、子组件读取参数**

```javascript
function Avatar({ person, size = 100 }) {
  // ...
}
```

* 注意：person和size都被包括在{}中。实际上这是在解构props。
* size可以给默认值，当且仅当父组件没有给子组件传递该值的时候默认值才会生效。
* 有时候多个参数传递的时候，代码不够简洁。事实上，组件间**props是组件的唯一参数**。子组件在接受的时候使用{}解构了props：  

  ```javascript
  export default function Father(props){
      const person={
          name:"zhangweite",
          age:15
      }
    
      return (
          <Son name = {person.name} age={person.age}></Son>
      );
  }
  function Son({name, age}) {
      return (
      <p>
          姓名：{name}<br></br>
          年龄：{age}
      </p>
      );
  }
  ```

  上面说props是组件的唯一真实参数，传递的时候可以简写成这样（所有的父组件入参都被包装到props了）：

  ```javascript
  export default function Father(props){
      const person={
          name:"zhangweite",
          age:15
      }
    
      return (
          <Son name = {person.name} age={person.age}></Son>
      );
  }
  function Son(props) {
      return (
      <p>
          姓名：{props.name}<br></br>
          年龄：{props.age}
      </p>
      );
  }
  ```

理解两个点即可：

* props是组件间唯一参数；
* 解构。

**4、将Jsx作为子组件传递**

上面学习了参数传递的方式。出了变量、函数、方法可以被传递以外，组件也是可以被传递的：

```javascript
export default function Father(props){
    const person={
        name:"zhangweite",
        age:15
    }
  
    return (
        <Card children={<Son name = {person.name} age={person.age}></Son>}>
          
        </Card>
    );
}
function Son(props) {
    return (
    <p>
        姓名：{props.name}<br></br>
        年龄：{props.age}
    </p>
    );
}

function Card({children}) {
    const style={
        border:"solid",
    }
    return (
        <div style={{
            backgroundColor: 'gray',
        }} >
            信息展示：
            {children}
        </div>
    );
}
```

组件的单个标签内部是里写组件不是很美观，可以改成这样：

```javascript
    return (
        <Card >
            <Son name = {person.name} age={person.age}></Son>
        </Card>
    );
```

这样也是将组件作为参数传递。

**5、props的不变性**

```javascript
export default function Clock({ color, time }) {
  return (
    <h1 style={{ color: color }}>
      {time}
    </h1>
  );
}
```

* 一个组件可能会随着时间的推移收到不同的props。Props并不总是静态的，然而Props却是不可变得。当一个props需要改变的时候，它会请求它的父组件<u>传递不同的prop</u>s，原来的props会被丢弃，直指回收。
* 现在想想为什么会有{value，setValue}，实际上就是在传递不同的props。
* 不变性带来了安全的好处。

## 保持组件纯粹

要点：

* 组件作为纯函数使用，纯函数的意思是说当输入保持不变的时候，输出应该是一致的。幂等。
* 这样可以更加安全，少埋bug。因为有时候如果状态改变了，但是渲染的顺序是没办法保证的，所以就会出现令人困惑的bug。所以不要读写全局的变量。

**1、不要去直接修改外部的值**

```javascript
export default function Clock({ time }) {
  let hours = time.getHours();
  if (hours >= 0 && hours <= 6) {
    document.getElementById('time').className = 'night';
  } else {
    document.getElementById('time').className = 'day';
  }
  return (
    <h1 id="time">
      {time.toLocaleTimeString()}
    </h1>
  );
}
```

直接返回不同的class即可；

```javascript
export default function Clock({ time }) {
  let hours = time.getHours();
  let className;
  if (hours >= 0 && hours <= 6) {
    className = 'night';
  } else {
    className = 'day';
  }
  return (
    <h1 className={className}>
      {time.toLocaleTimeString()}
    </h1>
  );
}

```

**2、不要去直接读“全局”的值**

currentPerson的存在使得这三个函数都变得不纯粹。如果要传递，使用props。

```javascript
import Panel from './Panel.js';
import { getImageUrl } from './utils.js';

let currentPerson;

export default function Profile({ person }) {
  currentPerson = person;
  return (
    <Panel>
      <Header />
      <Avatar />
    </Panel>
  )
}

function Header() {
  return <h1>{currentPerson.name}</h1>;
}

function Avatar() {
  return (
    <img
      className="avatar"
      src={getImageUrl(currentPerson)}
      alt={currentPerson.name}
      width={50}
      height={50}
    />
  );
}

```

‍
