---
title: "SOLID Principles With React - OCP"
date: 2022-06-23T02:27:06+08:00
draft: false
---

Hi 大家好我是 Curt 家人

# 系列相關文章

- [SOLID Principles With React](/posts/solid-principles-with-react/)
- [Single responsibility principle (SRP) - 單一職責原則](/posts/solid-principles-with-react-srp/)
- Liskov substitution principle (LSP) - 里氏替換原則
- Interface segregation principle (ISP) - 介面隔離原則
- Dependency inversion principle (DIP) - 依賴反轉原則

# OCP

SOLID 的第二個原則 OCP，中文又稱作開放封閉原則，意思是說

我允許我的 class、module、function 給你擴充，但不允許你修改它

定義上也是這麼說

> software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification

我們先來釐清什麼是擴充、什麼是修改

- 擴充：在原本的功能上擴充一些東西，但別人想用原本的功能也不影響，是可以選擇的。
- 修改：將核心的邏輯、功能換掉，所有人都會用到同一個功能。

舉個生活中的例子，
EX: 機車是用來交通的工具，如果有導航需求自己可以加裝手機架，給其他人騎的時候，如果他覺得不需要可以隨時拆掉，不影響原本的功用。

因此當有一個需求會影響這個 class 時，需要思考一下它是 `擴充` 還是 `修改`，

- 如果是擴充：那不應該修改這個檔案的 code，而是進行擴充
- 如果是修改：那就直接改檔案把。

我們來延續 SRP 中物件導向的例子

## OCP 物件導向的範例

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

const studentA = { name: "小劉", grade: 20 };
const studentB = { name: "小紅", grade: 30 };
const studentC = { name: "小白", grade: 40 };

const studens = [studentA, studentB, studentC];
const studentsGrades = new StudentsGrades(studens);
studentsGrades.sum();
```

前面有提到，全班成績的加總算法可能會更動，像是客家人身份要多 10 分，那我們思考一下這樣算是 `擴充` 還是 `修改`？

答案是我覺得兩個都有可能，首先要問一下 user

1. 以後都固定這樣算嗎？ （修改）
2. 還是要提供兩種算法，讓大家自由決定要拿原始的加總成績還是加權後的加總成績 （擴充）

假如最後確認的需求是擴充，但我卻直接修改 SumStudentsGrades 會變怎樣呢？我們來看看

```js
class SumStudentsGrades {
  constructor(students = [], sumFormulaType) {
    this.students = students;
    this.sumFormulaType = sumFormulaType;
  }

  sum() {
    if (this.sumFormulaType === "hakkaPlus10") {
      return this.students.reduce((acc, student) => {
        const additionalScore = student.isHakka ? 10 : 0;
        return acc + student.grade + additionalScore;
      }, 0);
    } else {
      return this.students.reduce((acc, student) => {
        return acc + student.grade;
      }, 0);
    }
  }
}
```

在使用 SumStudentsGrades 時，我們多傳入一個參數 `hakkaPlus10`，用它來決定我們的加總算法

使用時變這樣

```js
//...
const sumFormulaType = "hakkaPlus10";

const studentsGrades = new SumStudentsGrades(studens, sumFormulaType);

// 最後結果: 100
studentsGrades.sum();
```

看起來沒問題，不過以後想要擴充其他功能，EX: 原住民加 20 分，那我不就要增加一個 `if` 在 `SumStudentsGrades`？ 所以違反 OCP

那可以怎麼改呢？
首先將 sum 的邏輯 `委派` 給其他 class，原本的 class 只需要呼叫他的 function 就好

```js
class SumStudentsGrades {
  constructor(students = [], sumGrades) {
    this.students = students;
    this.sumGrades = sumGrades;
  }

  sum() {
    return sumGrades.sum(this.students);
  }
}
```

接著建立負責處理 sum 的 class：SumGrades，

以及需要擴充的 class：SumStudentsGradesWithHakkaPlus10、sumStudentsGradesOriginalWay

```js
class SumGrades {
  sum() {
    throw "please implement by your self";
  }
}

class SumStudentsGradesWithHakkaPlus10 extends SumGrades {
  sum(students) {
    return students.reduce((acc, student) => {
      const additionalScore = student.isHakka ? 10 : 0;
      return acc + student.grade + additionalScore;
    }, 0);
  }
}

class sumStudentsGradesOriginalWay extends SumGrades {
  sum(students) {
    return students.reduce((acc, student) => {
      return acc + student.grade;
    }, 0);
  }
}
```

這裡需要獨立 SumGrades class 的原因，其實是為了確保他們有 sum 這個 funtion，必免 SumStudentsGrades 在呼叫時發生錯誤，同時也強迫每個擴充的 SumGrades 子類別實作自己的 sum funtion。

使用方式就變成

```js
// or new sumStudentsGradesOriginalWay()
const sumGrades = new SumStudentsGradesWithHakkaPlus10();
const studentsGrades = new SumStudentsGrades(studens, sumGrades);
studentsGrades.sum();
```

當需要不同的加總規則時，我們只要新增一個 class 檔案並且繼承 SumGrades 就好，而不必去動到 SumStudentsGrades 類別
這樣一來就符合了 OCP。

## React 的範例

情境： 有個呈現所有使用者列表的頁面，上面的內容需要根據 user 的性別（gender），來提供不同的 layout

先來一些 user 資料

```js
const data = [
  { name: "小黑", gender: "boy" },
  { name: "小白", gender: "boy" },
  { name: "小紅", gender: "girl" },
];
```

接者用 map 來 render 這些資料

```react
const UserListPage = () => {
  const [userData] = useState(data);
  return (
    <div>
      {userData.map((user, index) => {
        return <UserDetail user={user} key={index} />;
      })}
    </div>
  );
};
```

列表邏輯處理好後，來看看 UserDetail 有什麼規則

業主說：

- 如果是男生，外框用藍色的，大頭貼用正方形。
- 如果是女生，外框用紅色的，大頭貼用圓形。

OK，沒什麼問題直接來做吧。

```react
const UserDetail = ({ user }) => {
  return (
    <div
      style={{
        // 男生藍色、女生紅色
        border: user.gender === "boy" ? "1px solid blue" : "1px solid red",
        marginBottom: "10px",
        padding: "10px 0",
      }}
    >
      <div>
        <img
          style={{
            // 男生正方形、女生圓形
            borderRadius: user.gender === "boy" ? "0" : "50%",
          }}
          src="https://via.placeholder.com/30x30"
          alt=""
        />
      </div>
      <div>{user.name}</div>
      <div>{user.gender}</div>
    </div>
  );
};
```

看起來大致上好了，
而且老實說我覺得這也是在 React 上很正常的設計方式

不過可以回想一下前面說的 `擴充` 概念：

> 在原本的功能上擴充一些東西，但別人想用原本的功能也不影響，是可以選擇的。

我們可以想像說，原本的 app 只有男生的樣式，但後來想要擴充女生的樣式，而男生的樣式也要保留下來 （雖然實務上，這個 case 應該是兩者同時發生拉）

如果以這個邏輯去想，當我要擴充女生樣式的功能時，是不是只能去 `修改` UserDetail 檔案（使用 if 判斷），
而不是用 `擴充` 方式多出一個檔案？ 好像就違反了 OCP，那該怎麼做？

很簡單就是將他們拆成兩個 component

Boy 變成 UserDetailBoy

```react
const UserDetailBoy = ({ user }) => {
  return (
    <div
      style={{
        border: "1px solid blue",
        marginBottom: "10px",
        padding: "10px 0"
      }}
    >
      <div>
        <img src="https://via.placeholder.com/30x30" alt="" />
      </div>
      <div>{user.name}</div>
      <div>{user.gender}</div>
    </div>
  );
};
```

Girl 變成 UserDetailGirl

```react
const UserDetailGirl = ({ user }) => {
  return (
    <div
      style={{
        border: "1px solid red",
        marginBottom: "10px",
        padding: "10px 0"
      }}
    >
      <div>
        <img
          style={{ borderRadius: "50%" }}
          src="https://via.placeholder.com/30x30"
          alt=""
        />
      </div>
      <div>{user.name}</div>
      <div>{user.gender}</div>
    </div>
  );
};
```

原本的 UserDetail 也修改一下

```react
const UserDetail = ({ user }) => {
  const components = {
    boy: <UserDetailBoy />,
    girl: <UserDetailGirl />
  };
  return components[user.gender];
};
```

OK 完成了，我們 code 看起來變得又臭又長，別著急請聽我娓娓道來。

假如不這麼做，如果未來多了 `中性` 的性別樣式，是不是又要在 UserDetail 再加上一些 if 來呈現？

而且除了性別之外

- 或許還會有 VIP 使用者，他的樣式要跟別人不同，藉此凸顯尊貴感。
- 或許還會有 Admin 使用者，業主說需要隱藏起來。
- ......等

我們可以想像 UserDetail 內會充滿著各種身份邏輯判斷，
所以如果拆成新的寫法，就可以不用考慮那些 if else，直接新增一個 component 來擴充，是不是蠻不錯的？

事實上還可以再思考一個情境，如果 app 想要開發一個女性使用者專區，裡面的列表只需要呈現女性，
而當你準備使用 UserDetail 時，你發現裡面 style 上很多 if else，你會不會用的很不安心？結果最後決定重寫一個 UserDetailGirl，
然後就發現這不是 OCP 嗎？

## References

- [Open-closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)
- [Applying the Open-Closed Principle To Write Clean React Components](https://betterprogramming.pub/applying-the-open-closed-principle-to-write-clean-react-components-4e4514963e40)
- [深入淺出開放封閉原則 Open-Closed Principle](https://www.jyt0532.com/2020/03/19/ocp/)
