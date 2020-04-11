---
layout: post
title: xử lý request trong nodejs
subtitle: Ai cũng có thể biết về event loop trong js, nhưng event loop trong server side nodejs là như nào!
published: true
tags: [nodejs, js, event loop, request]
---

|Nodejs|event loop |request|

----------

> Bài viết mang cái nhìn cá nhân. Các kiến thức ngoài phần tham khảo ở các tài liệu khác còn lại do mình tự suy luận, nên có thể mắc những sai sót nhất định. Hy vọng có thể nhận được sự góp ý của mọi người để bài viết được hoàn chỉnh cũng như mình có thể bổ sung thêm phần kiến thức.


Như chúng ta đã biết về `single-thread`, `non-blocking`, `asynchronous`, `concurrent` là các từ để miêu tả về JS. `call stack`, `event loop`, `callback queue` là những gì có trong JS. Nodejs được xây dựng bởi JS, hiển nhiên nó có tất cả những thứ trên. Trong bài viết này, chúng ta sẽ đi tìm hiểu về cách thức các thành phần trên hoạt động trong Nodejs

### 1. Cùng đi qua một vài vấn đề nào?


 **V8 engine**

Bao gồm `call stack` và `memory heap`
- `memory heap`: vùng nhớ được dùng để lưu kết quả được tính toán ở các hàm trong `call stack`. Cũng như trong `C++` nó có thể được cấp phát tĩnh hoặc cấp phát động.

- `call stack`: Hoạt động theo đúng nghĩa stack LIFO - Last In First Out. Khi chương trình thực thi đến hàm nào, thì hàm đó sẽ đẩy vào trong `call stack`. Và nó chỉ được lấy ra khi đã hoàn thành và return.

Phần **V8 engine** này các bạn có thể đọc và tìm hiểu ở bất kỳ tài liệu nào, rất cụ thể và chi tiết.

**Web APIs**

Không phải là thành phần của JS, nó là tiện ích của Browser, chứa các hàm như setTimeout(), fs.readFile(), emitter...
Không phải bất cứ câu lệnh nào cũng được thực thi ngay trên `call stack`, ví dụ như những hàm trên sẽ được đưa sang Web APIs khi gọi đến.

**Callback Queue** 

Nơi chứa các hàm ở **Web APIs** xuống. Nó sẽ ở đây chờ đến khi nào `call stack` rỗng để nhảy lên và thực thi tiếp các câu lệnh bên trong hàm.

**Event Loop** 

Là thành phần của *`libuv`* nó chính là vòng lặp để xử lý các sự kiện. Các hàm trong **Callback Queue** nhờ vòng lặp này sẽ được bế lên `call stack` khi rỗng để tiếp tục thực thi.


OK, có lẽ cơ bản như thế là đủ. Các bạn hoàn toàn có thể tìm trên mạng rất nhiều bài nói chi tiết về các vấn đề trên.

**Tham khảo thêm**:

* [What the heck is the event loop anyway? - Philip Roberts - JSConf EU](https://www.youtube.com/watch?v=8aGhZQkoFbQ){:target="_blank"}
* [Link demo](http://latentflip.com/loupe/?code=JC5vbignYnV0dG9uJywgJ2NsaWNrJywgZnVuY3Rpb24gb25DbGljaygpIHsKICAgIHNldFRpbWVvdXQoZnVuY3Rpb24gdGltZXIoKSB7CiAgICAgICAgY29uc29sZS5sb2coJ1lvdSBjbGlja2VkIHRoZSBidXR0b24hJyk7ICAgIAogICAgfSwgMjAwMCk7Cn0pOwoKY29uc29sZS5sb2coIkhpISIpOwoKc2V0VGltZW91dChmdW5jdGlvbiB0aW1lb3V0KCkgewogICAgY29uc29sZS5sb2coIkNsaWNrIHRoZSBidXR0b24hIik7Cn0sIDUwMDApOwoKY29uc29sZS5sb2coIldlbGNvbWUgdG8gbG91cGUuIik7!!!PGJ1dHRvbj5DbGljayBtZSE8L2J1dHRvbj4%3D){:target="_blank"}



### 2. Bức tranh trong môi trường Nodejs?

> Theo dõi video và link demo trên chúng ta thấy khá rõ ràng về cách thức JS thực thi các hàm. Nhưng trong môi trường server Nodejs, chúng ta có các request, response đồng thời thì cơ chết hoạt động sẽ ra sao?


![Cơ chế xử lý request trong Nodejs](/img/flow_event_loop_nodejs.jpg "Cơ chế xử lý request trong Nodejs")
<h4 style="text-align: center;">Cơ chế xử lý request trong Nodejs</h4>
Mình nghĩ đó là những cái basic nhất về cơ chế hoạt động của Nodejs.

**Vì sao JS đơn luồng**

Vì nó chỉ có một `event queue`, một `call stack`, một `event loop`, một `callback queue`. Đối với các ngôn ngữ khác như Java, thì mỗi request được gửi từ client, server sẽ có 1 thread riêng để xử lý request đó. Như vậy các request là song song đồng thời. Nhưng ở Nodejs thì không như vậy, vì nó chỉ có đơn luồng, việc dùng chung các tài nguyên hẳn là sẽ xảy ra các tranh chấp. Nhưng tại sao, ở nhiều bài viết trên mạng, mọi người vẫn nói Nodejs có thể xử lý hàng triệu kết nối cùng lúc &#128519;. Cùng đi tìm hiểu nào.



**Bắt đầu**

- Khi có nhiều request được gửi từ nhiều client trong cùng một thời điểm, các request này sẽ được đưa vào `event queue` hoạt động theo cơ chế FIFO - First In First Out.
- request được lấy ra ở `event queue` sẽ được đưa đến `call stack` để xử lý. Ở đây, trên server sẽ thực thi các hàm để có kết quả trả về cho request đó - giống như thực thi hàm có trong [video tham khảo](https://www.youtube.com/watch?v=8aGhZQkoFbQ){:target="_blank"} ở phần 1.
- Như vậy, tại 1 thời điểm Nodejs chỉ có thể phục vụ được 1 request duy nhất, vậy nếu request này xử lý quá lâu như cần query vào DB, thì hiển nhiên các request khác trong `event queue` sẽ phải đợi và có thể dẫn đến timeout.
- Nếu chỉ có mỗi  **V8 engine** thì sẽ xảy ra vấn đề như vậy. Nên Nodejs cần thêm một số thành phần khác. Đó là `Node APIs`, `libuv`.

**Bước thứ 2**

- Rõ ràng nếu chỉ có **V8 engine** thì nó không thể thực hiện query vào DB được. Vì **V8 engine** được thiết kế ra chỉ để thực hiện các phép toán, thực thi hàm, cung cấp data type, object... và GC(thành phần rọn rác cho bộ nhớ).
- Đa phần khi server nhận được request, sẽ phải query vào trong DB, lấy ra dữ liệu, xử lý dữ liệu đó rồi trả về cho client. Đó gần như là một vòng khép kín.
- Khi trong `call stack` thực thi một hàm của một request, hàm này cần phải query dữ liệu trong DB. Vì query vào DB là hoạt động `I/O` được quy định trong `Node APIs`. Do vậy nó không thể thực thi trên `call stack` được. Giống như `setTimeout()` có trong ví dụ phần 1. Nó sẽ được thực thi ở một nơi *khác*.

**Nơi *khác* là nơi nào?**

- Làm thế nào để đoạn mã trong `call stack` biết được nó có được `Node APIs` đảm nhiệm không?
- Theo mình nghĩ chỗ này sẽ có một `adapter` chuyên check việc hàm đó có hay không trong `Node APIs`. 
- Nếu hàm không thực thi một tác vụ liên quan đến `I/O` thì `adapter` sẽ trả về `false`. Ví dụ, request yêu cầu lấy thời gian của server, khi đó chỉ cần `return new Date()` rồi response về cho client. Như vậy hàm mà có `return new Date()` không liên quan đến `Node APIs` nên nó sẽ được xử lý ngay trong `call stack` với thời gian rất nhanh và trả về cho client luôn.
- Nếu nó thực thi một tác vụ liên quan đến `I/O` thì `adapter` sẽ trả về `true`. Ví dụ như việc query vào DB, sẽ cần một thời gian nhất định. Khi đó `Node APIs` biết mình cần phải xử lý cái hàm đó. Nó sẽ lấy ra khỏi `call stack` để cho `call stack` thực thi các hàm khác. Đồng thời phân cho 1 thread có trong `thread pool` để xử lý tác vụ `I/O` trong hàm vừa lấy ra. Chỗ này là bắt đầu đa luồng rồi nhé lại quay về kiến thức của Java &#128518;
- Luồng này sẽ được xử lý trong `handle thread`, lấy dữ liệu trong DB, sau khi xong sẽ được đẩy xuống `callback queue` chờ ngày được lên `call stack` &#128518;
- Khi đó `event loop` bắt đầu nhiệm vụ. Kiểm tra nếu `call stack` rỗng nó sẽ đưa hàm ở `callback queue` lên và thực thi các câu lệnh tiếp theo. Sau khi xong nó sẽ return về kết quả và response về cho client và câu chuyện đến đây là hết rồi &#128514;.

**Đa luồng vẫn có trong Nodejs**

- Phân tích ở phía trên với chỉ 1 request. Còn nếu nhiều request đồng thời thì Nodejs vẫn cần đa luồng để xử lý các tác vụ `I/O`.
- Khi 1 request cần query vào DB để lấy dữ liệu. Nó sẽ được một thread nào đó đảm nhiệm. Khi có `call stack` rỗng nó sẽ tiếp nhận các request tiếp theo trong `event queue` để xử lý tiếp. Nếu request này tiếp tục chứa tác vụ `I/O` thì sẽ được một thread khác đảm nhiệm tiếp. Đó là lý do chúng ta có `thread pool` - một thành phần của `libuv`.
- Chúng ta có thể lợi dụng điểm này để chạy các tác vụ cùng query 1 lúc trên nhiều thread khác nhau bằng việc sử dụng `Promise.all()`.
- Các hàm có trong `Node APIs` đều được xử lý dựa vào đa luồng có trong `thread pool`. Nếu nói Nodejs có đa luồng không, thì mình nghĩ trong những trường hợp nhất định, Nodejs vẫn có đa luồng.

### 3. Kết luận gì ở đây.

- Để server Nodejs của bạn có thể đáp ứng được nhiều kết nối từ client, hay đảm bảo `call stack` xử lý các hàm nhanh nhất có thể. Luôn luôn có pool trong `thread pool` sẵn sàng đảm nhiệm việc thực thi hàm.
- Bạn có thể hay nghe đến `block luồng chính` có nghĩa là hàm nào đó thực thi quá lâu trong `call stack` dẫn đến các request không được thực thi. Do vậy, khi code cần rất chú ý đến vấn đề này.
- Nếu vừa có dữ liệu ở `callback queue` và có request ở `event queue` trong thời điểm `call stack` rỗng thì sẽ xảy ra tranh chấp tài nguyên. Nodejs đã xử lý như nào? Theo mình nghĩ Nodejs sẽ ưu tiên lấy trong `callback queue` trước để hoàn thành request cho client. Cái này mình cũng chưa thử tìm hiểu.
- Nhưng nếu request của bạn thực thi 1 phép tính quá phức tạp, thì nó sẽ thực hiện ngay trên `call stack` và hiển nhiên các request khác sẽ phải đợi cho đến khi request đang chiếm `call stack` được thực thi xong. Điều là là tối kỵ trong Nodejs. Nếu phải thực hiện một phép tính phức tạp nào đấy. Hãy tìm phương án tối ưu hơn như đẩy sang 1 worker để xử lý.

### 4. Tài liệu tham khảo:

* [Node JS Architecture – Single Threaded Event Loop](https://www.journaldev.com/7462/node-js-architecture-single-threaded-event-loop)
* [JavaScript: Tổng quan về engine, runtime, call stack, single-threaded, concurrency, event loop...](https://blogchanhday.com/p/javascript-tong-quan-ve-engine-runtime-call-stack-va-event-loop/)
