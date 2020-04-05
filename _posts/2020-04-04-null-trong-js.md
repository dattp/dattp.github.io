# null trong js và undefined

|JS|null|undefined|

----------

Vào một ngày đẹp trời mình log typeof null nó ra object &#128517; . Toát mồ hôi, và mình đi tìm hiểu xem tại sao nó lại là object nhỉ.

``` js
console.log(typeof null)
>object

console.log(typeof undefined)
>undefined
```
#### 1. null là gì?
hmz!,  vậy ```null``` là gì? Theo như mình hiểu thì nó là  một giá trị rỗng, những vẫn sẽ phải khởi tạo và gán giá trị.
``` js
const a = null
console.log(a)
> null
console.log(typeof a)
> object
```
Như vậy, nó vẫn được coi là một giá trị, vậy JS đã lưu nó như nào?
#### 2. Cách js lưu giá trị.
Theo như những gì mình biết. Trong js có 6 giá trị được coi là ```falsy``` và 5 kiểu  ```primitive``` - kiểu dữ liệu nguyên thuỷ.
```
# 6 falsy
>false
>null
>undefined
>0
>''
>NaN

# 6 primitive
>booleand
>null
>undefined
>number
>string
```
Cả null và undefined đều khá đặc biệt trong js. Để phân biệt các kiểu dữ liệu ```primitive``` khi lưu giá trị của biến sẽ được phân biệt bởi một ``tag``
Mã nguồn của js được xây dựng dựa trên ```C```. Do vậy việc định danh sẽ do thư viện được viết bằng ```C``` đảm nhiệm. Cùng đi sâu vào châm cứu nào.

Mỗi giá trị của biến sẽ được lưu bởi  32 bit, trong đó sẽ dành ra 1-3 bit đầu tiên dùng để làm ```tag``` được phân biệt như sau:
* 000: object
* 1xx: int
* 010: double
* 100: string
* 110: boolean

**  đây là đoạn mã check type trong ```C``` **
```
JS_PUBLIC_API(JSType)
JS_TypeOfValue(JSContext *cx, jsval v)
{
    JSType type = JSTYPE_VOID;
    JSObject *obj;
    JSObjectOps *ops;
    JSClass *clasp;

    CHECK_REQUEST(cx);
    if (JSVAL_IS_VOID(v)) {
	type = JSTYPE_VOID;
    } else if (JSVAL_IS_OBJECT(v)) {                (1)
	obj = JSVAL_TO_OBJECT(v);
	if (obj &&
	    (ops = obj->map->ops,
	     ops == &js_ObjectOps
	     ? (clasp = OBJ_GET_CLASS(cx, obj),
		clasp->call || clasp == &js_FunctionClass)
	     : ops->call != 0)) {
	    type = JSTYPE_FUNCTION;
	} else {
	    type = JSTYPE_OBJECT;                       (2)
	}
    } else if (JSVAL_IS_NUMBER(v)) {
	type = JSTYPE_NUMBER;
    } else if (JSVAL_IS_STRING(v)) {
	type = JSTYPE_STRING;
    } else if (JSVAL_IS_BOOLEAN(v)) {
	type = JSTYPE_BOOLEAN;
    }
    return type;
}
```
Hơi khó hiểu tí nhỉ, mình cũng không hiểu gì đâu &#128514;. Nhưng hãy để ý  có các kiểu dữ liệu được định nghĩa sẵn như: ```JSVAL_IS_VOID, JSVAL_TO_OBJECT, JSVAL_IS_NUMBER, JSVAL_IS_STRING, JSVAL_IS_BOOLEAN``` 
Mình không đi sâu, nhưng cũng mạnh dạn dự đoán dựa vào bit của ```tag``` để khai báo cấu trúc kiểu ```enum``` cho các dữ liệu đó.

####  3 . Vậy tạo sao null lại là object
Dựa vào đoạn code (1) và (2) trong ```C``` ở trên , ta có thể  nhận định ```null``` đã được lưu với ```tag``` là 3 bit 000 ở đầu. Rõ ràng ```NULL``` trong ```C``` ** tương đương với 0 tức là nó trỏ về 0 - vùng địa chỉ không có ý nghĩa.**
Hiển nhiên khi check điền kiện ```if (JSVAL_IS_OBJECT(null)``` sẽ  có giá trị ```true``` và vì nó không phải là ```function``` nên giá trị trả về sẽ là ```type = JSTYPE_OBJECT;``` => đúng object đây rồi &#128514;,

#### 4. Kết luận gì ở đây?
Vậy tại sao lại không định nghĩa ra một kiểu dữ liệu cho ```null``` như ```undefined``` nhỉ. Chỉ cần định nghĩa thêm 1 loại ```JSVAL_IS_NULL``` chẳng hạn &#128518;.
Có lẽ ngay từ bạn đầu JS ra đời, cha đẻ của nó **Crockford** nên làm như vậy, nhưng có thể đến lúc phát hiện ra điều này, thì đã có quá nhiều người sử dụng, các hàm sinh ra cũng đã check ```null``` theo object nên việc sửa lại là điều khá khó khăn. Chính **Crockford** đã chỉ ra: 
>I think it is too late to fix typeof. The change proposed for typeof null will break existing code.

#### 5. Tài liệu tham khảo
* [The history of “typeof null”](https://2ality.com/2013/10/typeof-null.html)
* [Source code js](https://dxr.mozilla.org/classic/source/js/src/jsapi.c#333)

----------
 Chia sẻ là cách ghi nhớ tốt nhất! Cảm ơn tất cả mọi người!
