---
title: "SOLID Principles With React - SRP"
date: 2022-06-23T02:25:32+08:00
draft: false
---

Hi 大家好我是 Curt 家人

# 系列相關文章

- [SOLID Principles With React](/posts/solid-principles-with-react/)
- [Open–closed principle (OCP) - 開放封閉原則](/posts/solid-principles-with-react-ocp/)
- Liskov substitution principle (LSP) - 里氏替換原則
- Interface segregation principle (ISP) - 介面隔離原則
- Dependency inversion principle (DIP) - 依賴反轉原則

# SRP

首先來看第一個規則 SRP，中文又叫單一職責原則，
其實從字面上可以很簡單的猜到他要表達的意思，
也就是說每個 module、class 或 function 只需要負責做一件事情就好，不要將太多無關或複雜的邏輯放在裡面。

比如說湯匙就是拿來挖食物的工具，但卻設計成可以挖食物又可以拿來當作吸管喝飲料的東西，
這樣一來如果要將吸管的功能改成可以彎折的效果，那就要考慮到會不會影響湯匙的功能，進而增加他們之間的耦合度。

我們來看一下定義是怎麼描述的

> A class should have only one reason to change - Robert C. Martin

中文意思是 `每個類別只能有一個可以修改它的理由`

也就是說，如果某個類別出現了兩個理由要修改它，是不是就代表它做了兩件事情？ 因此不符合 SRP。

除此之外，`封裝` 也是 SRP 中很重要的事情，如果將功能拆成了好幾個元件，卻不知道怎麼用，那不如不要拆。

因此我們要能將元件的實作細節給隱藏起來，並提供一個很好的介面來讓使用者使用。

接著我們來看一些範例吧！

## 物件導向的範例

情境：需要一個類別來計算班上所有學生的分數總和，並且將結果輸出給使用者。

首先我們來建立一些學生的資料

```js
const studentA = { name: "小劉", grade: 20 };
const studentB = { name: "小紅", grade: 30 };
const studentC = { name: "小白", grade: 40 };

const studens = [studentA, studentB, studentC];
```

接著將學生們的成績傳入 StudentsGrades class，由它幫我完成加總以及顯示結果。

```js
const studentsGrades = new StudentsGrades(studens);
studentsGrades.printReport();
```

開始實作 StudentsGrades class

```js
class StudentsGrades {
  constructor(students = []) {
    this.students = students;
  }

  sum() {
    return this.students.reduce((acc, student) => {
      return acc + student.grade;
    }, 0);
  }

  printReport() {
    return console.log(`Their total grades are ${this.sum()}`);
  }
}
```

最後結果

```
"Their total grades are 90"
```

看完以上的範例我們可以發現，StudentsGrades 類別除了負責加總（sum）之外，還實作了 printReport，是不是就一個 class 負責了兩件事情？ 也就會有兩個原因造成 StudentsGrades 需要修改。

- 如果需要更改輸出的格式，那我們要修改 StudentsGrades 的 printReport
- 如果需要更改加總的規則，例如客家人學生額外加 10 分，那我們要修改 StudentsGrades 的 sum

因此違反了 SRP，那可以怎麼改呢？
就是將這兩個邏輯拆成不同 class

sum 的部分變成 SumStudentsGrades

```js
class SumStudentsGrades {
  constructor(students = []) {
    this.students = students;
  }

  sum() {
    return this.students.reduce((acc, student) => {
      return acc + student.grade;
    }, 0);
  }
}
```

printReport 部分變成 StudentReportPrinter

```js
class StudentReportPrinter {
  constructor(sumStudentsGrades) {
    if (!sumStudentsGrades instanceof SumStudentsGrades) {
      throw "not instance of SumStudentsGrades";
    }
    this.sumStudentsGrades = sumStudentsGrades;
  }

  printReport() {
    return console.log(
      `Their total grades are ${this.sumStudentsGrades.sum()}`
    );
  }
}
```

使用時變成這樣

```js
const sumStudentsGrades = new SumStudentsGrades(studens);
const printer = new StudentReportPrinter(sumStudentsGrades);

printer.printReport();
```

這樣一來他們就負責各自的職責，每個 class 也只存在一個修改的理由。

## React 的範例

情境：從後端 fetch 使用者列表資料並顯示在畫面上

首先建立 UserListPage component，並用 map 列出每個使用者

```js
import { useEffect, useState } from "react";

const UserListPage = () => {
  const [users, setUsers] = useState([]);
  return (
    <div>
      <h1> User List Page </h1>
      <div>
        {users.map((user) => (
          <div key={user.id}>
            <div>{user.name}</div>
            <div>{user.email}</div>
            <br />
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserListPage;
```

接著在 useEffect 裡面 fetch api 資料

```js
// UserListPage.js
import { useEffect, useState } from "react";

const UserListPage = () => {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("https://jsonplaceholder.typicode.com/users")
      .then((res) => {
        return res.json();
      })
      .then((data) => {
        setUsers(data);
      });
  }, []);

  return (
    <div>
      <h1> User List Page </h1>
      <div>
        {users.map((user) => (
          <div key={user.id}>
            <div>{user.name}</div>
            <div>{user.email}</div>
            <br />
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserListPage;
```

好，現在來思考一下 UserListPage 是不是可以拆一下？

useEffect 裡面的做的事情跟 `網路請求` 有關，應該可以將它獨立出去，寫成一個 useUserData 的 hook。

變成

```js
// useUserData.js
import { useEffect, useState } from "react";

const useUserData = () => {
  const [users, setUsers] = useState([]);

  useEffect(() => {
    fetch("https://jsonplaceholder.typicode.com/users")
      .then((res) => {
        return res.json();
      })
      .then((data) => {
        setUsers(data);
      });
  }, []);

  return { users };
};

export default useUserData;
```

然後 UserListPage 變成

```js
//UserListPage.js
import useUserData from "./useUserData.js";

const UserListPage = () => {
  const { users } = useUserData();

  return (
    <div>
      <h1> User List Page </h1>
      <div>
        {users.map((user) => (
          <div key={user.id}>
            <div>{user.name}</div>
            <div>{user.email}</div>
            <br />
          </div>
        ))}
      </div>
    </div>
  );
};

export default UserListPage;
```

是不是看起來乾淨許多了？

不過仔細想一下，這個 component 叫做 UserListPage，主要負責 `條列` 出每個 user 的資料，所以我只要 focus 在怎麼列出他們就好，至於裡面的內容長什麼樣子，應該不用去關心吧？

因此來試著將 map 裡面的內容在獨立出去，變成 UserDetail component

```js
// UserDetail.js
const UserDetail = ({ user }) => {
  return (
    <div>
      <div>{user.name}</div>
      <div>{user.email}</div>
      <br />
    </div>
  );
};

export default UserDetail;
```

最後 UserListPage 的樣子

```js
import UserDetail from "./UserDetail.js";
import useUserData from "./useUserData.js";

const UserListPage = () => {
  const { users } = useUserData();
  return (
    <div>
      <h1> User List Page </h1>
      <div>
        {users.map((user) => (
          <UserDetail key={user.id} user={user} />
        ))}
      </div>
    </div>
  );
};

export default UserListPage;
```

- 以後網路請求的部分有需要更改 -> useUserDate
- 以後條列的方式要改，EX: 由新到舊 or 舊到新 -> UserListPage
- 以後使用者顯示資料要增減 -> UserDetail

## SRP 總結

以上就是我對 SRP 套用到 React 上的心得，我覺得它算是五個原則裡面相對好懂的規則，卻也非常考驗工程師的內功，以上都是簡單以及特別設計過的情境，但是在現實專案上，需要有長期的實務經驗，才能知道怎麼將大塊的邏輯拆得既合理又實用，
不然最後只是為了拆而拆，搞的專案凌亂又難以理解。

NEXT -> [SOLID Principles With React - OCP](http://localhost:1313/posts/solid-principles-with-react-ocp/)

## References

- [Single-responsibility principle - wiki](https://en.wikipedia.org/wiki/Single-responsibility_principle)
- [SOLID: The First 5 Principles of Object Oriented Design](https://www.digitalocean.com/community/conceptual_articles/s-o-l-i-d-the-first-five-principles-of-object-oriented-design)
- [SOLID Principles in React](https://www.everydayreact.com/articles/solid-principles-in-react)
- [How To Apply SOLID Principles To Clean Your Code in React](https://betterprogramming.pub/how-to-apply-solid-principles-to-clean-your-code-in-react-cdfd5e0a9cea)
