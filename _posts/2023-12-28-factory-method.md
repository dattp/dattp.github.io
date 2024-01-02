---

layout: post

title: Factory Method Design Pattern trong thực tế với TS

published: true

tags: [factory method,  design pattern, typescript]

---

|factory method|design pattern|typescript|

---
  
Về lý thuyết của factory method design pattern thì mình không nhắc lại. Ở đây, mình chỉ viết ra cách mình đã áp dụng factory method trong dự án mà mình đã thực hiện.

### 1. Bài toán từ requirement và cách triển khai đơn giản

Mình nhận được yêu cầu viết 1 module gọi api để lấy dữ liệu thông tin (gì đó) từ các thành phố khác nhau:

* Module này sẽ được gọi ở nhiều logic khác.
* Mỗi thành phố là 1 domain, 1 server khác nhau. Ví dụ: Đà Nẵng - danangcity.com, Hà Nội - hanoi.com
* Các server có cấu hình khác nhau nên cần có retry khác nhau (quy định bởi server) khi gọi api failed.
* Ban đầu có 3 thành phố cần lấy data, nhưng sau đó có thể mở rộng thêm.

Với yêu cầu này thì đơn giản là dựa vào mã thành phố truyền vào và if-else để call đúng domain cho của thành phố đó.
```ts
let domain = ''
let retryCount = 0
if (cityCode === 'DN') {
	domain = DANANG_DOMAIN
	retryCount = DANANG_RETRY
}
...
// khởi tạo axios để call API
// xử lý request failed để re-try
```

* Issue xảy ra khi ở các code logic cần lấy data về `City` thì phải viết lại đoạn code trên. Hoặc nếu clean hơn thì sẽ viết 1 hàm riêng để xử lý việc lấy data:

```ts
function getInfoCityByCode(cityCode: string): Promise<ICity> {
    let domain = ''
    let retryCount = 0
    if (cityCode === 'DN') {
        domain = DANANG_DOMAIN
        retryCount = DANANG_RETRY
    }
    ...
    // khởi tạo axios để call API
    // xử lý request failed để re-try
}
```

* Tuy nhiên với yêu cầu sau này có thể bổ sung thêm nhiều thành phố khác, và việc xử lý retry hoặc data các thành phố có thể khác nhau. Nên cách tiếp cận như trên vẫn chưa tối ưu và mình cần tìm một cách khác để có thể xử lý được với trường hợp như này. Vì đây là vấn đề khởi tạo object, constructor nên mình nghĩ ngay đến `factory method` thuộc nhóm creational.

### 2. Thi triển factory method design pattern

* Đoạn code được mình viết lại

```ts
interface ICityInfo {
    retryNumber: number;
    requester: Requester;
    getInfo(): Promise<ICity>;
}

class DANANG implements ICityInfo {
    retryNumber = 5;
    requester: Requester = new Requester('domain');
    getInfo(): Promise<ICity> {
        // call request
        // handle retry if request failed
    }
}

class HANOI implements ICityInfo {
    retryNumber = 5;
    requester: Requester = new Requester('domain');
    getInfo(): Promise<ICity> {}
}

// export class
export class CityService {
   private initCityRequest(cityCode: string): ICityInfo {
        if (cityCode === 'DN') return new DANANG();
		...
    }

    public async getInfoCityByCode(cityCode: string): Promise<ICity> {
        const city = this.initCityRequest(cityCode);
        return city.getInfo();
    }
}
```

* Các class sẽ được tách ra thành các file riêng. Sau này khi code mở rộng thêm các thành phố khác chỉ cần clone ra và bổ sung thêm vào hàm `initCityRequest` là được.
* Ở các logic code sẽ chỉ cần gọi đến hàm `getInfoCityByCode` để lấy ra data.

### 3. Áp dụng với convention của project thực tế

* Các project mình đang tham gia được viết bẳng typescript và vẫn giữ tư tưởng hướng event, nên đa phần các function khai báo sẽ thuộc class, tức là luôn đi kèm từ khoá `static`.
* Như vậy khi call function ở những chỗ khác mình sẽ không cần tạo constructor của class nữa
* Ví dụ:

```ts
class DANANG **implements ICityInfo** {
    static retryNumber = 5;
    static requester: Requester = new Requester('domain');
    static getInfo(): Promise<ICity> {
        // call request
        // handle retry if request failed
    }
}
```

* Nhưng nếu sử dụng `static` như trên thì class sẽ không implement được từ interface `ICityInfo`. Do vậy các class City sẽ không có 1 kiểu chung.
* Tuy rằng hơi cố chấp, nhưng mình vẫn muốn tìm ra giải pháp để xử lý. Vậy thì nếu không thể implemnt từ 1 interface thì mình sẽ add những class đó vào trong 1 collection có type là interface.

```ts
const MapCodeCity: Map<string, ICityInfo> = new Map([
    ['DN', DANANG],
    ['HN', HANOI],
]);
//
async getInfoCityByCode(cityCode: string): Promise<ICity> {
        const city = MapCodeCity.get(cityCode)
        if (city) return city.getInfo();
        throw Error
    }
```

* Như vậy, bây giờ ở các logic khác cần lấy thông tin về `City` sẽ chỉ cần gọi function `getInfoCityByCode` và truyền vào `cityCode`.

### 4. Tổng kết
* Tuy rằng việc áp dụng `factory method` sẽ làm cho code trở nên phức tạp hơn. Tuy nhiên, về lâu dài, khi đã dự đoán được khả năng mở rộng của business, thì sẽ đem lại nhiều lợi ích.
* Giờ đây, khi khai báo thêm 1 thành phố mới, sẽ chỉ cần bổ sung thêm class, và khai báo thêm vào `MapCodeCity`. Không động chạm gì đến những logic khác, đảm bảo code được stable.
* Thực ra thì mọi thứ hoàn toàn có thể dừng lại ở phần `2`. Tuy nhiên, do convention của project đang tham gia nên mình cũng muốn tìm ra các giải pháp để có thể xử lý.
* Mình nhận ra 1 điều, thay vì cố gắng phải implement class vào 1 interface ngay từ đầu, thì có thể "ép" class đó vào trong 1 type. Như vậy, nếu 1 class sinh ra hoặc thay đổi về property, mà không đúng type thì sẽ bị lỗi.
