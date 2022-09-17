---

layout: post

title: Nếu sử dụng Observer Pattern trong nodejs P1

published: true

tags: [nodejs, observer, design pattern, queue]

---

|design pattern|nodejs|observer|queue|

---
  

### 1. Cũng là học design pattern nhưng mà lạ lắm
* Cũng như bao sinh viên công nghệ khác và đặc biệt là khi học java cũng đã nghe ít nhiều về design pattern.
* Kiểu như nếu biết design pattern, code của bạn sẽ xịn sò hơn. Trình độ của bạn sẽ lên 1 tầm cao mới, các kiểu...
* Đi theo ánh hào quang, mình cũng đã từng thử tìm hiểu, nhưng không. Có những pattern, chỉ đọc thôi cũng không thể hình dung được nó đang implement code như nào 😓
* Và rồi, mình cũng chỉ biết về những lý thuyết cơ bản của Factory Method, Singleton.
* Bẵng đi 1 thời gian mình không code java nữa mà chuyển sang code nodejs. Với js thì nó lại là 1 thứ gì khác hoàn toàn với java. Và mình đã từng nghĩ, design pattern sinh ra là dành cho java và oop 😆

### 2. Tôi code nodejs và cũng không có design pattern nào cả
* Với 1 suy nghĩ non trẻ ấy, mình đã nhao đầu vào code nodejs mà chỉ có function này gọi sang function khác.
* Ngay cả 1 cái Singleton đơn giản là sử dụng biến global để khởi tạo connection vào DB mình cũng không biết mà sử dụng. Hoặc là đi copy code từ example nhưng cũng không biết tên gọi của nó như nào.
> Cho đến khi tôi đọc được template xịn sò của anh **[@Minh Monmen](https://viblo.asia/u/monmen)** mình đã có nhiều hình dung hơn về cách áp dụng design pattern vào trong 1 project nodejs.

### 3. Observer Pattern ví dụ
* Để nói về lý thuyết thì có thể tìm thấy ở bất kỳ đâu.
![Observer Pattern](/img/observer-pattern.png "Observer Pattern")

* Đoạn code ví dụ lý thuyết cho nodejs.
* Bài toán đặt ra là khi có hành động `create user`, ta cần gửi dữ liệu `user` đó sang 3 hệ thống khác `(service A, service B, service C)` thông qua restAPI.
* Theo cách code thông thường nào đó.

```js
// Main service
// connect to serviceA
// connect to serviceB
// connect to serviceC

export function createUser(user) {
	// todo something to create user
	sendToServiceA()
	sendToServiceB()
	sendToServiceC()
}
// handle service exit
// disconnect to serviceA
// disconnect to serviceB
// disconnect to serviceC
```

* Nếu áp dụng `observer pattern` ta có thể triển khai như sau:

```js
// observer.js
const subscribers = []
export default {
	notify: (data) =>  subscribers.forEach((sub) => sub(data)),
	subscribe: (func) => subscribers.push(func),
	unsubscribe: (func) => {
		[...subscribers].forEach((sub, index) => {
			if(sub === func) subscribers.splice(index, 1);
		})
	}
}
```

```js
// send-data.js
export function sendToServiceA(data) {
	// TODO - A
}
export function sendToServiceB(data) {
	// TODO - B
}
export function sendToServiceC(data) {
	// TODO - C
}

Observer.subscribe(sendToServiceA)
Observer.subscribe(sendToServiceB)
Observer.subscribe(sendToServiceC)

```

```js
// Main service
export function createUser(user) {
	// todo something to create user
	Observer.notify(user)
}
```


* Các bạn thấy gì không, implement 1 cái `observer pattern` nó còn vất vả hơn cả cách code xôi thịt bình thường.
* Nhưng rõ ràng là code khá tường minh, ở đây `main service` đã không còn phụ thuộc vào các hàm `send to service`. Code chính sẽ gọn gàng hơn.
* Sau này nếu có cần `send user` tới service khác nữa, thì ta không cần sửa code ở `main service`, mà chỉ thêm 1 `subscriber`.

> Nhìn  cứ như kiểu code event driven ấy nhỉ, sử dụng event bus, emit ra 1 event để các handler khác lắng nghe và xử lý event đó.

### 4. Observer Pattern có thể áp dụng trong project

* Một trong các case có thể áp dụng observer pattern đó là handle việc service exit.
* Nếu các bạn đã từng sử dụng `job queue` trong hệ thống, thì code sẽ kiểu như này"

```js
const handleAction_1 =  new  Queue('handle-action-1',  'redis://127.0.0.1:6379');
const handleAction_2 =  new  Queue('handle-action-2',  'redis://127.0.0.1:6379');
```
* Khi service exit, chúng ta sẽ có đoạn code để close queue.
>https://github.com/OptimalBits/bull/blob/develop/REFERENCE.md#queueclose

```js
function handleShutdown(){
	handleAction_1.close().catch((e) => console.error(e));
	handleAction_2.close().catch((e) => console.error(e));
}
process.on('SIGINT', () => {
	handleShutdown()
});
```

* Rõ ràng vấn đề ở đây là khi có 1 queue mới, chúng ta sẽ phải handle các action có liên quan, một trong số đó là phải xử lý queue close.
* Vậy thì mỗi khi 1 queue mới được tạo ra, ta sẽ subscribe queue đó vào 1 observer, để khi service exit ta chỉ cần notify đến cho các queue đó close.

```js
// observer-queue.js
const subscribers = []
export default {
	notify-close: async() => {
		 const promises = subscribers.map((sub) => sub.close().catch((e) => console.error(e))),
		 await Promise.all(promises)
	} 
	subscribe: (queue) => subscribers.push(queue),
	unsubscribe: (queue) => {
		[...subscribers].forEach((sub, index) => {
			if(sub === queue) subscribers.splice(index, 1);
		})
	}
}
```

```js
const handleAction_1 =  new  Queue('handle-action-1',  'redis://127.0.0.1:6379');
const handleAction_2 =  new  Queue('handle-action-2',  'redis://127.0.0.1:6379');

Observer.subscribe(handleAction_1)
Observer.subscribe(handleAction_2)
```

```js

async function handleShutdown(){
	Promise.resolve()
		.then(() => Observer.notify-close())
		.catch((err) => {
			console.error('Error during shutdown', err);
			process.exit(1);
		});
}	
process.on('SIGINT', () => {
	handleShutdown()
});
```

### 5. Nhưng mà
* Đoạn code trên là mình suy nghĩ ra khi vừa đọc lại phần observer pattern. Dựa vào project template và mình nghĩ ra cách implement lại pattern này.
* Mình cũng đã thử và nó work, tuy nhiên để implement thì lại mất thời gian ban đầu hơn, nhưng sau việc thêm queue mới cũng nhàn hơn.
* Nói chung, theo quan điểm cá nhân mình thấy thì các design pattern đều phức tạp hơn nhiều so với cách mà ta vẫn code xôi thịt. 