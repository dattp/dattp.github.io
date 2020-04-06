---
layout: post
title: null và undefined trong js
subtitle: Cùng tìm hiểu xem null và undefined trong js có gì hay ho
published: true
tags: [nodejs, js, 'null', undefined]
---

|JS|null|undefined|

----------

Trong các kiểu dữ liệu của JS, có lẽ null và undefined là 2 giá trị cũng như 2 kiểu giá trị khá là đặc biệt và thú vị &#128516;

### 1. undefined

`undefined` có nghĩa là không xác định. Cái gì không xác định? - thường dùng để nói về giá trị. Giá trị của biến, giá trị của thuộc tính, giá trị của hàm... đã được khai báo nhưng chưa có giá trị.
```js
  let a
  console.log(a)
  >undefined

  *************************

  const object ={a: 'a'}
  console.log(object.b)
  >undefined

  *************************

  function f() {return}
  console.log(f())
  >undefined

```

### 2. null

`null` là gì? Theo như mình hiểu thì nó là  một giá trị rỗng, những vẫn sẽ phải khởi tạo và gán giá trị.
```js
const a = null
console.log(a)
> null

```
Mình đã có bài viết tìm hiểu kỹ hơn về null [**tại đây**](https://dattp.github.io/2020-04-04-null-trong-js/)

### 3. null và undefined

Cả `null` và `undefined` đều là giá trị `primitive` và `falsy` trong JS nhưng lại có sự khác biệt nhau ra khó ràng.

```js
  null == undefined         // true
  null === undefined        //false

  ***********************************

  console.log(typeof undefined)
  > undefined
  console.log(typeof null)
  > object
```
 Kiểu dữ liệu của `undefined` là `undefined` nhưng của `null` lại là `object`. Đó là lý do lại sao phép so sánh `===` lại là `false`

Chúng ta sẽ có rất nhiều phép tính thú vị giữa `null` và `undefined`, nhưng mình không nhớ &#128517;

Cả `null` và `undefined` đều là những giá trị ít mong muốn gặp phải. Nhất là với trường hợp bắt giá trị từ client gửi lên server. Thường để kiểm soát 2 giá trị này, mình sẽ gán giá trị mặc định khi khai báo biến:
```js
  let variable = null
  const userName = variable || 'No Name'
  console.log(userName)
  > No Name

  *****************************************

  let variable
  const userName = variable || 'No Name'
  console.log(userName)
  > No Name
```
Kể cả khi `variable` là `null` hay `undefined` thì biến sẽ gán với giá trị default mà mình định nghĩa.

--------------------------------

** Lý thuyết chỉ là tĩnh, con người phải vận dụng lý thuyết vào thực tế mới là động. Nhưng phải lấy tĩnh chế động. **