---

layout: post

title: Singleton pattern và sai lầm "oái oăm" trong hệ thống scale services

published: true

tags: [singleton pattern,  design pattern, typescript]

---

|singleton pattern|design pattern|typescript|

---
  
Singleton pattern và sai lầm "oái oăm" trong hệ thống scale services và phụ thuộc vào centralized database. Có những bug người khác nhìn vào là thấy ngay, mà sao người viết ra lại mất nhiều thời gian để debug.

### 1. Bối cảnh của bài toán

Quay trở lại với bài viết [áp dụng factory method design pattern](https://dattp.github.io/2023-12-28-factory-method/) để gọi api lấy thông tin về các thành phố. Mình cần khởi tạo 1 requester để call API và sử dụng thư viện `axios`.
Service bên thứ 3 yêu cầu 1 token gắn kèm vào header trên mỗi request. Token được lấy qua api `/api/token` và có expire time là 30 phút.
Ban đầu để ""đơn giản" nên mình lưu token trên vào redis và đặt expire time là 28 phút.
Mình áp dụng singleton cho `requester` và đoạn code như sau:
```ts
export class CityService {
    private static requester: Requester;
    
	private static async getRequester(): Promise<Requester> {
		let token = (await RedisAdapter.get(CITY_TOKEN_KEY)) as string;
(1)     if (!token) {
	        token = await CityService.getTokenCT();
	        CityService.requester = new Requester(CITY_DOMAIN, CITY_APIKEY, { token });
	    }
	        
	    if (!CityService.requester) {
	        CityService.requester = new Requester(CITY_DOMAIN, CITY_APIKEY, { token });
	    }
	        
	    return CityService.requester;
	}
}
```

Thoạt nhìn đoạn code không có vấn đề gì. Chạy trên môi trường localhost và development vẫn work. Mình bắt đầu đẩy code lên môi trường staging chạy docker swarm với 3 node trên 3 server khác nhau. Và vấn đề bắt đầu phát sinh.

### 2. Scale service và sai lầm
* Môi trường staging chạy docker swarm với 3 node. Service được deploy replicate mode với 3 instance chạy đều trên 3 node của swarm.
* Service start được 1 lúc thì có request báo lỗi 401. Mà không phải lúc nào cũng bị, đặt log và debug thì chưa phát hiện ra ngay điều gì. Mà không thể tái hiện trên localhost và development. Do development cũng chạy swarm nhưng chỉ có 1 node.
* Sau đó mình thấy tỉ lệ error/total request ~ 1/3 và bắt đầu nghi ngờ do chế độ replicate 3 instance trên staging.
*  Khi token bị expire điều kiện (1) được thoả mãn, nếu khi đó chỉ có 1 request thì sẽ chỉ có 1 instance detect được và lấy lại token để store vào redis, `requester` của instance đó cũng sẽ được khởi tạo mới.
* Request tiếp theo đến thì token mới đã set vào redis và không thảo mãn điều kiện (1) dẫn đến `requester` không được tạo mới và `CityService.requester` vẫn đang cầm token cũ bị expirte nên xuất hiện error 401.
* Mình đã mất rất nhiều thời gian để phát hiện ra được điều này, cho đến khi mình phải log ra hẳn requester. Thật là hồ đồ khi mắc phải sai lầm như thế :(

### 3. Sai thì sửa

* Biết sai chỗ nào rồi thì sửa nhanh thôi. Theo đúng logic, cứ lấy token ra thì gán lại vào header.

```ts
export class CityService {
    private static requester: Requester = new Requester(CITY_DOMAIN, );
    
	private static async getRequester(): Promise<Requester> {
		let token = (await RedisAdapter.get(CITY_TOKEN_KEY)) as string;
(1)     if (!token) {
	        token = await CityService.getTokenCT();
	        CityService.requester = new Requester(CITY_DOMAIN, CITY_APIKEY, { token });
	    }
	        
	    if (!CityService.requester) {
	        CityService.requester = new Requester(CITY_DOMAIN, CITY_APIKEY, { token });
	    }
	        
	    return CityService.requester;
	}
}
```
* Chỉ cần thêm 1 dòng code đã giải quyết 1 bug "oái ăm" mà mình đã mắc phải.

### 4. Tổng kết
* Đoạn code ví dụ trong bài mình đã lược bỏ nhiều logic để nó đơn giản nhất có thể.
* Mình dùng singleon để khởi tạo duy nhất một `requester`, nhưng lưu ý ở đây là `requester` sẽ được tạo mới hoặc được  sau 1 thời gian nhất định. Và đó cũng dẫn đến sai lầm của mình.
* Đoạn code "đơn giản" ban đầu hoá ra lại phức tạp. Việc sử dụng redis lưu chung 1 biến vô tình lại thành bug. Không hiểu sao lúc đó mình lại code như vậy, rồi đoạn code làm lại thì đơn giản hơn nhiều.
* Luôn phải hiểu những dòng code mình viết ra được chạy ở đâu, mỗi môi trường chạy khác nhau như nào. Process start, environment đầu vào, và communicate với các service khác ra sao.