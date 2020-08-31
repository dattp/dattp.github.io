---
layout: post
title: typescript đầu tiên
subtitle: Mình đã bắt đầu code typescript như nào?
published: true
bigimg: /img/path.jpg
tags: [nodejs, ts, typescript]
---

|Nodejs|typescript|

----------

> Mình mới tìm hiểu về typescript được 1 tuần, mọi thứ chỉ là quan điểm và sở thích mang tính cá nhân.

### 1. Tại sao lại là typescript?
* Từ một người code java chuyển sang javascript, mình nhận thấy JS là một ngôn ngữ rất linh hoạt. Từ cách khai báo biến, khai báo hàm, đến các phép tính của nó mà đến bây giờ mình cũng chưa biết 😅.
* Chắc hẳn, nhiều bạn cũng đã từng thấy hoặc từng code những đoạn code như này.
```js
	const getUserById = async(id) => {
		try {
			//TODO
			res.send({error: null, data: user})      (1)
		} catch(error) {
			// TODO
		}
	}
```

* `response` trả về được định nghĩ ngay trong hàm theo format `{error; data}`. Và mình không thích điều này.
* Mình thích tư tưởng của OOP nhiều hơn. Đặc biệt là Java. Đối với mình Java vẫn là best 🤣.
* Và đó là lý do mình mất 1 tuần để code thử typescript.

### 2. Typescript có gì?
* Hướng đối tượng. Chính là cái mình đang cần. Mọi thứ cần theo một khuôn mẫu cụ thể.
* JS rất linh hoạt, nhưng với một developer hơi gà như mình, việc kiểm soát đôi khi là khó khăn 😄.
* Mình thích sự kiểm soát rõ ràng. Mọi biến được khai báo có kiểu dữ liệu cụ thể. Mỗi thuộc tính, hàm phải thuộc đối tượng.
* Vì typescript là kiểu OOP nên code sẽ dễ maintain, mở rộng... Và có thể áp dụng nguyên lý DI/IoC để giảm sự phụ thuộc giữa các class với nhau.

### 3. Bắt đầu bằng TODO nào!
* Sau khi đã tìm hiểu qua một vài khái niệm cơ bản, mình bắt đầu code thử 1 project nhỏ.
*  Cách setup 1 project typescript bạn có thể tìm thấy dễ dàng, nên trong bài viết này mình không đề cập đến. Mình sẽ đi vào cụ thể những điều mà mình kiểm soát rõ ràng trên typescript.

#### 3.1 Class và interface.
* Chắc chắn phải có interface rồi. Để có 1 `giao diện` cụ thể cho các class implement theo. Đồng thời để áp dụng nguyên lý DI - giảm sự phụ thuộc.
* overview qua project. Chúng ta sẽ có mô hình như sau: 
* ![Cấu trúc dự án](/img/typescript-todo.png 
"Cấu trúc dự án")
<h4 style="text-align: center;">Cấu trúc project</h4>

* Chúng ta sẽ quan tâm đến 
	* `routes`:  class - định nghĩa các paths.
	* `controllers`: class - định nghĩa các function xử lý request.
	* `services`: class - định nghĩa các function tương tác với DB.
	* `entities`: class - định nghĩa các đối tượng.
	* `responses`: class - định nghĩa các dạng format trả về cho client.
* Luồng cơ bản chúng ta sẽ đi từ `routes` - nơi nhận request từ client, gửi sang cho `controllers` xử lý. `controllers` có thể sử dụng dữ liệu trong DB thông qua `services`.
* Rõ ràng nhận ra `routes` đang bị phụ thuộc vào `controllers` và `controllers` đang phụ thuộc vào `services`.
* Định nghĩ các interface tương ứng: 
	* Interface todo - service:

```typescript
import ITodo from "../../entities/interfaces/i.todo.entity";

interface ITodoService {
  getListTodo(): Promise<ITodo[]>
  getTodo(id: number): Promise<ITodo | null>
  insert(todo: ITodo): Promise<ITodo | null>
}

export = ITodoService 
```

* Chúng ta có 3 hàm, return về dạng `Promise` của đối tượng `Todo` thông qua interface `ITodo`.
	* Interface  todo - controllers

```typescript
import { Request, Response, NextFunction } from 'express'

interface ITodoController {
  getTodos(req: Request, res: Response): Promise<any>
  getTodoDetail(req: Request, res: Response): Promise<any>
  insertTodo(req: Request, res: Response): Promise<any>
}

export = ITodoController
```

* Đối với class todo - controller sẽ cần implement interface tương ứng như sau: 

```typescript
import { Request, Response } from 'express'

import ITodoService from '../services/interfaces/i.todo.service'
import ITodoController from './interfaces/i.todo.controller'
import ErrorNotFound from '../responses/errors/notfound.exception'
import ErrorMissingParam from '../responses/errors/missingparam.exception'
import ServerError from '../responses/errors/servererror.exception'
import ResponseSuccess from '../responses/success.response'

class TodoController implements ITodoController {
  public static todoService: ITodoService

  constructor(service: ITodoService) {
    TodoController.todoService = service
  }

  public async getTodos(req: Request, res: Response): Promise<any> {
   //TODO
  }

  public async getTodoDetail(req: Request, res: Response): Promise<any> {
    const id = parseInt(req.params.id, 10)
    if (!id || isNaN(id)) {
      // return res.status(406).send('ko')
      return new ErrorNotFound(`todo id ${id} not found`, res)
    }
    try {
      const todo = await TodoController.todoService.getTodo(id)
      if (!todo) {
        return new ErrorNotFound(`todo id ${id} not found`, res)
      }
      return new ResponseSuccess(todo, res)
    } catch (error) {
      return new ServerError(error.message, res)
    }
  }

  public async insertTodo(req: Request, res: Response): Promise<any> {
   //TODO
  }
}

export { TodoController }

```

* Mình sẽ lấy hàm `getTodoDetail()` để làm ví dụ chính. Với input là `id` của một `todo` nên khi bắt validate cho param này mình sẽ return ra một Error - đối tượng mình đã định nghĩa trong `responses`. 
* Sau đó mình gọi sang `TodoService` thông qua interface để lấy ra `todo` tương ứng.
* Để ý hàm `constructor(service: ITodoService)` - mình không trực tiếp khởi tạo `new TodoSerivce()` ở đây. Mà nó sẽ truyền vào ở hàm bên ngoài, theo nguyên lý DI.
* Response trả về cho client cũng tuân theo format mình định nghĩa ở class `ResponseSuccess`.
* Như vậy ngay cả khi success hay error thì đều tuân theo 1 format cụ thể, cái đã được định nghĩa từ trước.
* Đôi với class `TodoRoute` chúng ta sẽ định nghĩa như sau: 

```typescript
import ITodoController from '../controllers/interfaces/i.todo.controller'

class TodoRoute {

  private app: any
  private todoController: ITodoController

  constructor(app: any, controller: ITodoController) {
    this.app = app
    this.todoController = controller
    this.routes()
  }

  private routes() {
    this.app.get('/api/gettodos', this.todoController.getTodos)
    this.app.get('/api/gettodo/:id', this.todoController.getTodoDetail)
    this.app.post('/api/inserttodo', this.todoController.insertTodo)
  }
}

export { TodoRoute }

```

* Lúc này `route` sẽ không phụ thuộc trực tiếp vào một controller cụ thể nào nữa. Mà nó sẽ phụ thuộc vào 1 interface.
* Tương tự với `controller` cũng chỉ phụ thuộc vào interface của service.

#### 3.2 Truyền sự phụ thuộc tập trung.

* Khi sự phụ thuộc chỉ còn thông qua các interface, rõ ràng mình cần 1 `nơi` để khởi tạo tập trung các class tương ứng, và mình chọn `app.ts` để làm điều này.
* Chúng ta có đoạn mã khởi tạo sau: 

```typescript
  private _loadRoute() {
    //init component
    // init constructor service todo
    const todoService = new TodoService()
    // init constructor controller todo
    const todoController = new TodoController(todoService)
    // init constructor route
    new TodoRoute(this.app, todoController)
  }
```
* Chúng ta cần khởi tạo `constructor` tương ứng của `service`, truyền nó vào `controller` tương ứng. Tương tự với `route` cũng cần `controller` tương ứng.
* Rõ ràng, nếu sau này có sự thay đổi về `controller` hoặc `service`. Ví dụ `controller` sẽ gọi đến `service` khác thì khi đó ta chỉ cần thay đổi `constructor` của `serivce` ở bên ngoài `app.ts`.

#### 3.3 Response
* Mình cần 1 format chung để gửi về cho client. Do vậy sẽ định nghĩ class Response trong `entities` với 3 thuộc tính: `statusCode`, `data`, `error`.
* Đối với 1 response not found mình sẽ định nghĩa như sau:

```typescript
import { Response } from 'express';

import ResponseException from "./response.exception";
import StatusCode from '../../constants/statuscode.constant';

class ErrorNotFound extends ResponseException {
  constructor(message: string, res: Response) {
    super(StatusCode.NOT_FOUND, message, res)
  }
}

export default ErrorNotFound

```
* `StatusCode` là 1 dạng `enum` định nghĩa các statuscode tương ứng.
* Như vậy với mỗi 1 error nào là not found mình chỉ cần truyền `message` tương ứng. Và nó sẽ return về cho client thông qua class `ResponseException` mà mình đã exends.
* Đối với response là success:

```typescript
import { Response } from 'express';
import ResponseFormat from '../entities/response.entity';
import StatusCode from '../constants/statuscode.constant';

class ResponseSuccess {

  constructor(data: any, res: Response) {
    this._handResponse(data, res)
  }

  private _handResponse(data: any, res: Response): any {
    const output = new ResponseFormat(StatusCode.SUCCESS, data, null)
    res.status(output.getStatus).send(output)
  }

}

export default ResponseSuccess;

```

* Mình sẽ return về `data` tương ứng với mỗi API.

### 4. Kết luận gì ở đây!
* Đối với mình mọi dữ liệu, biến, object ... hay bất kỳ thứ gì tạo ra nên được định nghĩa một cách rõ ràng, giống như cách tạo ra 1 class.
* Cá nhân mình thích sự kiểm soát chặt chẽ từng đoạn code. Có format định nghĩa chung cho từng dữ liệu.
* Typescript hỗ trợ rất tốt điều này. OOP. Triển khai tốt về mặt giảm sự phụ thuộc lẫn nhau giữa các class.
* Bài viết có bố cục hơi lủng củng 😅. Dẫn dắt ý chưa rõ ràng. Nên mọi người có thể tham khảo code project theo repo của mình nha: 
* Source code: [TODO typescript](https://github.com/dattp/todo_typescript)