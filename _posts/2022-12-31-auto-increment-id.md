---

layout: post

title: Tận dụng sức mạnh của id tự tăng trong Database

published: true

tags: [id, index, database]

---

|nodejs|id|index|database|

---
  

### 1. Auto increment ID
* Hay gọi một cách đơn giản là id tự tăng. Ý chỉ cột id được sinh ra tự động và có giá trị  tăng dần theo số bản ghi được insert.
* Trong những DB SQL như mysql, sql server thì nó thường có kiểu number. Và được set là primary key để tiết kiệm được nhiều tài nguyên.
* `_id` trong mongo không phải là AI, tuy nhiên nó được generate 1 phần theo timestamp. Nên mình sẽ đề cập chung trong những trường hợp được sử dụng. 
* Đặc điểm của cột `id` này thường được làm định danh cho giá trị của model. Được đánh index mặc định nên truy vấn rất nhanh.
* Việc để `id` là number tự tăng cũng mang đến nhiều rủi ro về leak data. Do vậy cần có những giải pháp để `che giấu` nó, tuy nhiên, trong bài viết này chỉ đề cập đến những case được mình sử dụng để tận dụng "sức mạnh" mà nó đem lại.

### 2. Các trường hợp sử dụng
#### 2.1 Như 1 cursor
* Cụ thể mình đã tận dụng id trong các trường hợp muốn jump đến 1 đối tượng cụ thể trong 1 list.
* Trong màn hình chat có tin nhắn được pin. Làm thế nào khi user bấm vào tin nhắn được pin mà api trả về đúng tin nhắn đó + các tin nhắn trước và sau nó.
* Tương tự như các trường hợp khi bấm vào tin nhắn replied làm sao để nhảy về được tin nhắn gốc hay là khi bấm vào kết quả search tin nhắn.
* Trong trường hợp có notification ở comment khi user bấm vào, làm sao để trả về đúng comment đó + thêm các comment khác.
* Đối với id là AI trong việc xử lý các trường hợp sẽ thuận tiện hơn. API fetch list tin nhắn với center là `message_id` + 10 tin nhắn cũ và 10 tin nhắn mới. 
* Lúc này, để lấy 10 tin nhắn cũ và 10 tin nhắn mới, mình sẽ tận dụng cái `message_id` như 1 cursor để so sánh dựa vào tính AI.
* Để lấy 10 tin nhắn mới hơn. Điều kiện so sánh là `id > message_id` + `sort` theo `asc` và `limit = 10`.
* Để lấy 10 tin nhắn cũ hơn. Điều kiện so sánh là `id < message_id` + `sort` theo `desc` và `limit = 10`.
* Khi đó api sẽ trả về 1 list gồm 21 messages + 2 cursors là `next` và `prev` để handle action khi user scroll tiếp theo. 
* Nếu user muốn xem tiếp tin nhắn, api fetch sẽ xử lý lấy list theo `next` hoặc `prev` cursor theo logic tương tự.

#### 2.2. Pagination trên table dữ liệu lớn.
* Hiệu quả khi skip lượng dữ liệu lớn.
* Có lẽ mọi người đã biết trường hợp này nhiều hơn, và mình cũng thế :v.
* Đa phần chúng ta sẽ phân trang theo kiểu `skip + limit` hoặc `page_number + limit` theo từng nghiệp vụ. 
* Nhưng nếu là kiểu scroll như lướt newsfeed hay ứng dụng tin nhắn mà lượng dữ liệu lớn thì phân trang theo kiểu trên sẽ có nhiều vấn đề về performance.
* Ví dụ khi mình truy vấn trên mongo

```
// query
db.getCollection(collection_name).find(filter)
				.skip(1000000)
				.limit(100)
				.sort({_id: -1})
				.explain('executionStats')
								
-   "executionStats":
    -   "executionSuccess":true,
    -   "nReturned":100,
    -   "executionTimeMillis":596,
    -   "totalKeysExamined":1000100,
    -   "totalDocsExamined":100,
    -   "executionStages":
        -   "stage":"LIMIT",
        -   "nReturned":100,
        -   "executionTimeMillisEstimate":550
```

* Ta thấy rằng, câu query trên vẫn phải đi scan qua cả `1000000` được skip để lấy ra 100 bản ghi tiếp theo đó. Khi số lượng `skip` tăng lên nhiều, sẽ gây chậm đến query.
* Khi đó, nếu ra sử dụng `id` như 1 cursor như trường hợp trên thì kết quả sẽ tốt hơn rất nhiều.
* Trong TH này thì việc phân trang bắt buộc phải lấy bản ghi lần lượng mà ko thể nhảy page, với cursor là id của item cuối cùng cả response trước đó.

```
// query
db.getCollection(collection_name).find({_id:{$gt:ObjectId('63b024c40000000000000000')}})
				.limit(100)
				.sort({_id: -1})
				.explain('executionStats')

-   "executionStats":
    -   "executionSuccess":true,
    -   "nReturned":100,
    -   "executionTimeMillis":0,
    -   "totalKeysExamined":100,
    -   "totalDocsExamined":100,
    -   "executionStages":
        -   "stage":"LIMIT",
        -   "nReturned":100,
        -   "executionTimeMillisEstimate":0      
```

* Số bản ghi cần scan cũng chỉ là 100 bản ghi, và thời gian exec của query cũng giảm đáng kể.

#### 2.3.  Sharding data.
* Không phải là kỹ thuật sharding data khi lưu vào db. Mà là tận dụng việc id tự tăng để chia data xử lý trên nhiều worker khi cần scan số lượng bản ghi.
* Ví dụ để gửi push notification cho toàn bộ vài  triệu user có trong hệ thống. Ta không thể nào query toàn bộ bảng user vì quá nặng. Hoặc nếu skip, limit để xử lý lần lượt thì quá lâu.
* Lúc này ta sẽ chia việc query users theo batch. Nếu `id` trong mysql được đánh từ 1 đến 8.000.000 ta sẽ chia ra thành các cụm `id_start` và `id_end` cách nhau 500 đơn vị và đẩy vào từng worker để thực hiện query với công thức như sau:
	* id_start = 1 & id_end = 500
	* id_start = 501 & id_end = 1000
	....
* Việc chia id như này giúp các worker làm việc song song và không bị trùng lặp bản ghi. Quá trình thực thi cũng diễn ra nhanh chóng.
* Đối với mongo, thì mình thường chia theo `_id` dựa vào độ mau-thưa của dữ liệu. Ví dụ 1 ngày, có khoảng 500 user được insert, thì mình sẽ chia batch theo từng ngày theo công thức sau:
	* Lấy ra `id_first` là bản ghi đầu tiên trong DB.
	* `id_end` là timestamp của ngày hiện tại + 1.

```
	let cursor = idFirst;
	
	while (cursor <= idEnd) {
	const  idFirstOfBatch  = cursor;
	cursor +=  60  *  60  *  24; // 24 hours
	// add to worker with data body 
		{
			id_fist: idFirstOfBatch,
			id_last: cursor,
			...
		}
	// into worker - convert id to ObjectId
}
```

### 3. Kết bài
* Trên đây là một vài trường hợp điển hình mà mình đã tận dụng được đặc tính auto increment của ID để giúp hệ thống xử lý nhanh hơn.
* Việc hiểu cấu trúc của dữ liệu + xác định rõ nghiệp vụ, không máy móc khi áp dụng, tận dụng được đặc điểm của hệ thống đang có sẽ giúp hệ thống chạy tốt lên rất nhiều.
* Bản thân mình ban đầu cũng áp dụng những cách cơ bản, như sau đó khi hệ thống lớn dần, bắt đầu thấy những hạn chế mình cũng đi tìm hiểu và hỏi các tiền bối đi trước để có được những phương pháp tốt hơn.
* Luôn học hỏi, trau dồi, biến kinh nghiệm thành kiến thức để thúc đẩy bản thân tiến về phía trước.
* Kết thúc một năm 2022.