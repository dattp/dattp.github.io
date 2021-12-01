---
layout: post
title: Circular và exports, module.exports trong nodejs
published: true
tags: [nodejs, circular, exports, module.exports]
---

|Nodejs|circular|exports|module.exports|

---

> Và sai lầm của tuổi trẻ được thể hiện từ quá khứ đến hiện tại và ngược lại, lẫn lộn khó hiểu.

### 1. Tại sao lại có sai lầm?

- Code nodejs cũng được 1 thời gian tương đối, nhưng mình sử dụng js viết code như java (trước kia đã dùng) nên thực sự mình không hiểu quá nhiều và sâu về js 😓
- Từ những ngày đầu code nodejs, khi mà code vẫn còn non và gà, mình cũng đã dính lỗi require lẫn nhau giữa các file.
- Khi start server và run thử, thì nó bắn ra những lỗi kiểu như `cái gì đó is not a function`
- Mình ngồi debug mòn đít mà chưa rõ nguyên nhân vì sao. Chỉ biết search gg và tìm cách fix, nhưng chưa rõ ngọn ngành 😔
- Ví dụ đơn giản (không phải trường hợp thực tế)

```
// file a.js
const b = require('./b');

function getModuleB() {
  return b.getModuleA();
}

function getModuleA() {
  return "moduleA";
}

module.exports = {
  getModuleB: getModuleB,
  getModuleA: getModuleA
}
```

---

```
// file b.js
const a = require('./a');

function getModuleA() {
  return a.getModuleA();
}

module.exports = {
  getModuleA: getModuleA
}
```

- Chạy code ở file `index.js`

```
// file index.js
const a = require('./a');
console.log(a.getModuleB());

// Lỗi
// TypeError: a.getModuleA is not a function
//    at Object.getModuleA (/Users/datpt/Desktop/nodejs/download-file/b.js:6:12)
// ...
```

- Cùng đi tìm hiểu vì sao đoạn code trên bị lỗi và cách xử lý sẽ làm như nào?

### 2. Điểm qua lại một vài kiến thức

#### 2.1 exports và module.exports

- Các bạn có thể tìm hiểu rõ hơn về 2 khái niệm này trên [trang chủ nodejs](https://nodejs.org/api/modules.html#moduleexports)
- Hiểu một cách đơn giản khi một file `.js` được thực thi trên môi trường nodejs, nó sẽ có thêm thành phần là `module`. `exports` là 1 thành phần thuộc `module`

```js
var module = { exports: {} };
var exports = module.exports;

// your code

return module.exports;
```

- Ngoài ra có thêm biến `exports` trỏ cùng vào địa chỉ ô nhớ với `module.exports` và kết thúc file là `return module.exports;`

#### 2.2 Vấn đề là bị làm sao và circular là như nào?

- Như vậy bản chất khi 1 file `.js` được thực thi nó sẽ return 1 đối tượng là `module.exports;`
- Quay lại với ví dụ được nêu ra ở phần 1 chúng ta thấy 1 đặc điểm chung ở 2 file `a.js` và `b.js` đó là đều trỏ `module.exports` vào 1 object mới, không còn là địa chỉ default.
- Khi đoạn code trong `index.js` được thực thi, file `a.js` được require nên sẽ được gọi, trong file `a.js` lại require `b.js`. Lúc này code trong `a.js` chưa được thực thi xong nhưng mặc định đã có `module.exports` được sinh ra ngay từ đầu.
- Trong file `b.js` lại require lại `a.js`. `a.js` đã được require trước đó bởi `index.js` nên `const a = require('./a')` trong `b` sẽ trỏ cùng về 1 địa chỉ mà không thực thi lại những đoạn code trong file `a`.
- Sau khi code trong `b` được thực thi xong thì code trong file `a` mới được thực thi phần còn lại. Nhưng kết thúc file `a` ta lại gán `module.exports= một object mới`, cái mà trước đó trong `b` ghi nhận 1 địa chỉ default của `a` 😓.
- Việc require chéo nhau được gọi là `circular`. Do vậy khi đoạn code `console.log(a.getModuleB())` trong `index` được thực thi ta gọi đến hàm trong `a`, từ `a` gọi qua hàm của `b`, từ `b` lại gọi qua hàm của `a`, nhưng lúc `b` gọi hàm của `a` do `module.exports` của a đã bị thay đổi giá trị default nên trong `b` sẽ không có những hàm đó => `TypeError: a.getModuleA is not a function` là ở đây.

### 3. Giải quyết vấn đề

- Sau khi đã biết rõ nguyên nhân, ta có thể sửa để đoạn code trên có thể thực thi.
- Đơn giản là ta sẽ giữ nguyên địa chỉ mặc định của `module.exports`, không gán nó với một giá trị mới.
- Nếu đã tìm hiểu về `module.exports` và `exports` ta chỉ cần thêm những hàm cần exports vào đối tượng này.

```
// file a.js
exports.getModuleB = getModuleB
exports.getModuleA = getModuleA
// hoặc
module.exports.getModuleB = getModuleB
module.exports.getModuleA = getModuleA
```

- Có 2 cách để viết, nhưng vì `module.exports` và `exports` là như nhau trong trường hợp này nên chọn cách viết ngắn hơn.

```
// file b.js
exports.getModuleA = getModuleA
```

- Khi chạy lại file `index.js` sẽ không còn bị lỗi nữa, nhưng sẽ nhận được warning như này: `Warning: Accessing non-existent property 'Symbol(nodejs.util.inspect.custom)' of module exports inside circular dependency`.
- Node đã cảnh báo cho ta biết về việc các module phụ thuộc chéo nhau. Điều này theo mình là nên tránh.

### 4. Kết luận gì ở đây!

- Rõ ràng vấn đề `circular` nên cần tránh, đoạn code được nêu ra ở phần 1 chỉ là ví dụ đơn giản.
- Có thể fix để đoạn code đó không lỗi nhưng cách đó vẫn bị warning về circular.
- Do cách design code ban đầu có vấn đề, dẫ đến các module require lẫn nhau. Ta nên có design rõ ràng. Như trong ví dụ trên nếu để thiết kế lại ta có thể tạo 1 module require cả `a` và `b` và xử lý riêng, file `index` sẽ require module mới này.

#### Tham khảo:

https://nodejs.org/api/modules.html#cycles
https://nodejs.org/api/modules.html#moduleexports
