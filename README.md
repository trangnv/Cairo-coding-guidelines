# Hướng dẫn lập trình với Cairo

*Bài viết gốc [Cairo coding guidelines](https://medium.com/nethermind-eth/cairo-coding-guidelines-74eb6f4ee264)*

*Viết bởi [Massil Achab](https://twitter.com/machilassab) và [Caleb Zacher](https://twitter.com/kalzakdev).*

*Xin cảm ơn [Mathieu Saugier](https://twitter.com/eniwhere_), [Soufiane Hajazi](https://twitter.com/quixotic_eth), [Krzysztof Szubiczuk](https://twitter.com/kaliberpoziomka) và [Edgar Barrantes](https://twitter.com/EdgarBarrantes) vì các ý kiến đóng góp và chỉnh sửa*

## Lập trình với Cairo
Lập trình với Cairo có thể khá khó khăn bởi 2 lý do: lập trình viên có xu hướng sử dụng các khuôn mẫu (patterns) code từ Solidity; Cairo thay đổi cú pháp (syntax), API liên tục, mới nhất là cập nhật từ 0.9.0 lên 0.10.0. Mặc dù vậy, công nghệ zk-rollup đã và sẽ phát triển, với số lượng lập trình viên gia nhập hệ sinh thái StarkNet và học Cairo tăng lên hàng tuần. Tại Nethermind, chúng tôi đã và đang làm việc với Cairo thông qua hàng loạt các dự án và vì thế muốn chia sẻ những khuôn mẫu code, mẹo, khuyến nghị mà chúng tôi dùng để chuẩn hoá Cairo code.

## Lời mở đầu
Khi làm việc trong 1 nhóm, viết code theo 1 bộ quy tắc chuẩn là điều rất quan trọng và thường được sử dụng để duy trì sự nhất quán cho toàn codebase. Điều này yêu cầu tất cả các thành viên trong nhóm phải tuân theo bộ quy tắc để viết và phê duyệt code. 1 điều có lợi nữa với 1 codebase nhất quán là các đoạn code sẽ rất dễ đọc, đặc biệt quan trọng cho quá trình audit, Codebase càng dễ đọc càng giúp cho auditor hiểu được dự án của bạn hơn, cho phép họ tập trung nhiều hơn vào các khía cạnh an ninh quan trọng.

## Code
### Composition thay vì Inheritance
Khác với Solidity, trong Cairo bạn không thể dùng inheritance. Thay vào đó, bạn có thể dùng composition - OpenZeppelin gọi nó là “Extensibility pattern” - định nghĩa 1 class bằng cách sử dụng lại code của các classes khác. Chúng tôi đề xuất cấu trúc của các files trong contracts như sau:

- Cho mỗi contract, chúng tôi  tạo các files khác nhau như sau:
    - file thư viện (hoặc file logic), với tên `my_contract_library.cairo`, chứa các logic code của hợp đồng. Cụ thể bao gồm: (i) hàm internal và external được gói trong 1 namespace (ii) storage variables và events được định nghĩa bên ngoài namespace
    - file contract, tên `my_contract.cairo`, sẽ gồm các hàm external so với các files thư viện. Ví dụ, 1 hợp đồng cToken viết từ 1 hợp đồng ERC20  chứa đựng cả tất cả các hàm trong `c_token_library.cairo` and `erc20_library.cairo`.

- Đối với abstract contract:
    - file thư viện, đặt tên theo quy ước giống như trên. Trong thực tế abstract contract không nên chứa các hàm mà chỉ cung cấp code cho các contracts khác.

### Quy ước đặt tên
Đa phần, không phải tất cả chúng tôi sẽ theo các khuyến nghị từ OpenZeppelin, bạn có thể theo dõi bài viết của họ về [Extensibility pattern](https://github.com/OpenZeppelin/cairo-contracts/blob/main/docs/modules/ROOT/pages/extensibility.adoc)
Cụ thể:
- Sử dụng `snake_case` cho tên file, hàm (trong các file thư viện) và biến số (variables)

```rust
// filenames
// example 1: contracts/pool_config.cairo
// example 2: tests/test_pool_config.cairo
// functions
func config_of{...}(user : felt) -> (config : MyConfig) {
    ...
}
// variables 
let (user_config) = config_of(user);
let user_balance = user_config.balance;
```

- Sử dụng `snake_case` cho hàm view và external (hàm trong file contract)

```rust
// view function
@view
func get_name{...}() -> felt {
    ...
}
// external function
@external
func set_name{...}(name : felt) {
    ...
}
```

- Sử dụng `UPPER_SNAKE_CASE` cho hằng số

```rust
const CAIRO_FIELD_ORDER = 2 ** 251 + 17 * 2 ** 192 + 1;
```

- `PascalCase` cho struct, namespace, interface và event

```rust
// struct
struct MyStruct {
    ...
}
// namespace
namespace MyNamespace {
    ...
}
// namespace (interface)
@contract_interface
namespace IPool {
    ...
}
// event
@event
func AssetUpdated(asset : felt) {
}
```

- `PascalCase_snake_case` cho storage variables, trong đó phần PascalCase là tên của namespace

```rust
// storage variable (defined for namespace MyNamespace)
@storage_var
func MyNamespace_user_config(user : felt) -> (config : MyConfig) {
}
```

### Thứ tự import
Về việc import trong Cairo, chúng tôi phân ra thành 3 mục riêng biệt, sắp xếp theo thứ tự alphabet đối với từng mục. Việc này giúp cho việc đọc code dễ dàng hơn. 
3 mục đó cụ thể là:
- Starkware imports
- Thư viện bên ngoài (thường là các cairo contracts từ OpenZeppelin)
- Import các code tự viết

### Error messages - Báo lỗi
Khi sử dụng `with_attr error_message(...)` , phải đảm bảo chỉ có 1 biểu thức (expression) trong statement được lỗi (fail). Nếu không, biểu thức bị lỗi và lỗi báo có thể không trùng khớp, sẽ rất khó để debug.

### Calldata
Với Cairo, để input 1 mảng các phần tử felt (array of felt) đến 1 hàm,   cách thức / khuôn mẫu  phổ biến thường được sử dụng là dùng 1 felt pointer và độ dài array. Chúng tôi khuyến nghị gói (encapsulate) array này trong 1 struct, ví dụ `MyStruct` và dùng `MyStruct.SIZE` cho độ dài array.
Trái lại, nếu bạn hardcode độ dài array, ví dụ `IProxy.initialize(pool, 2, new (name, value))` - có thể bạn muốn thêm 1 phần tử vào array, nhưng nhiều khả năng bạn sẽ quên tăng độ dài array và điều này sẽ gây lỗi. Sử dụng struct là cách để có thể kiểm soát độ dài array 1 cách linh hoạt thay vì cố định nó.

```rust
// using array as calldata
IProxy.initialize(pool, 2, cast(new (name, value), felt*));
// using struct as calldata
struct InitializeInput {
    name : felt,
    value : felt,
}
IProxy.initialize(pool, InitializeInput.SIZE, new InitializeInput(2, 22));
```

### Guards
Chúng tôi khuyến nghị sử dụng `guards` trong file contract. Trong Solidity, chúng ta thường định nghĩa chúng bằng modifiers. 

Ví dụ:

```rust
// in my_contract_library.cairo
namespace MyContract {
    func set_name{...}(name : felt) {
        MyContract_name.write(name);
    }
}
// in my_contract.cairo
@external
func set_name{...}(name : felt) {
    Ownable.assert_only_owner();
    MyContract.set_name(name);
}
```

### Booleans
Do Cairo không có 1 dữ liệu riêng cho booleans, chúng ta sử dụng felt. Có thể rất dễ nhầm lẫn cho người mới bắt đầu khi đọc đoạn code sau: assert bool_1 + bool_2 + bool_3 = 3.
Thay vào đó, chúng tôi khuyến nghị sử dụng thư viện riêng để so sánh và thực hiện các phép toán trên các biến boolean, ví dụ như [BoolCmp](https://github.com/aave-starknet-project/aave-starknet-core/blob/main/contracts/protocol/libraries/helpers/bool_cmp.cairo), để code dễ hiểu hơn.

### Vòng lặp - Loops
Hiện tại Cairo không hỗ trợ vòng lặp (phiên bản 1.0 sẽ có hỗ trợ). 1 cách để thay thế là dùng hàm đệ quy (recursion) với đầu vào là array. Cho mỗi vòng lặp, chúng tôi khuyến nghị định nghĩa 1 hàm internal với tên bắt đầu bằng _inner hoặc _loop
Ví dụ:

```rust
func sum_array(array_len : felt, array : felt*) -> felt {
    let sum = 0;
    let (res) = _sum_array_inner{array_len=array_len, array=array, sum=sum}(0);
    return res;
}
func _sum_array_inner{array_len : felt, array : felt*, sum : felt}(current_index : felt) -> felt {
    if (current_index - array_len == 0) {
        return (sum);
    }
    let sum = sum + array[current_index];
    return _sum_array_inner(current_index + 1);
}
```

### Storage variables - Biến số
Tên của các Storage variable các tên có thể gây ra mâu thuẫn lẫn nhau trong các phiên bản Cairo trước 0.10.0. Hiện tại lỗi đó sẽ hiện ra ngay khi contract được compiled. Chúng tôi khuyến nghị đặt tên storage variable bắt đầu với tên của namespace tương ứng để đụng hàng

Ví dụ:

```rust
// in my_contract_library.cairo
@storage_var
func MyContract_name() {
}
namespace MyContract {
    ...
}
```

### Asserts
Bạn có thể xác nhận nhiều phép so sánh cùng 1 lúc sử dụng structs. Ví dụ với 1 hàm simple_func đơn giản là output ra 3 giá trị a, b, c. Bạn có thể sử dụng đoạn code sau:

```rust
// without struct
let (res : SimpleFuncResult) = simple_func();
assert res.a = a_true;
assert res.b = b_true;
assert res.c = c_true;
// with struct
let (res) = simple_func();
assert res = SimpleFuncResult(a_true, b_true, c_true);
```

## Toán
Thư viện chuẩn của Cairo hiện nay có thể gây ra khá nhiều nhầm lẫn khi so sánh các giá trị felt, ví dụ như lỗi [này](https://github.com/OpenZeppelin/cairo-contracts/issues/433). Chúng tôi đã viết 2 thư viện để đảm bảo an toàn và chính xác khi sử dụng felt có dấu và không dấu: [SafeCmp](https://github.com/aave-starknet-project/aave-starknet-core/blob/main/contracts/protocol/libraries/math/safe_cmp.cairo) cho việc so sánh, [FeltMath](https://github.com/aave-starknet-project/aave-starknet-core/blob/main/contracts/protocol/libraries/math/felt_math.cairo) cho các phép tính toán

### Unsigned felts - felt không dấu
Số nguyên trong Cairo nằm trong giới hạn P = 2^251 + 17 * 2^192 + 1. Cụ thể, số nguyên có thể có giá trị từ 0 đến P-1 và sau đó trở lại là 0
Vì thế bạn có thể gán giá trị của 1 felt không dấu trong phạm vi từ 0, 1, … tới P-2, P-1. Nếu kết quả phép toán lớn hơn P-1 nó sẽ bị lỗi overflow, ngược lại nếu kết quả nhỏ hơn 0, sẽ bị lỗi underflow

### Signed felts - felt có dấu
Giá trị lớn nhất của felt có dấu sẽ là  (P-1)/2, giá trị nhỏ nhất sẽ là -(P-1)/2

2 thư viện toán đã nêu ở trên xây dựng trên các phân tích này


## Ví dụ về cấu trúc của contract

### File thư viện
`my_contract_library.cairo` 

```rust
// imports
from starkware.cairo.common.uint256 import Uint256
// storage variables
@storage_var
func MyContract_name() {
}
...
// events
@event
func Initialized() {
}
...
namespace MyContract {
    // internal functions
    func _first_func{...}(...) {
        ...
    }
    // getter functions
    func second_func{...}(...) {
        ...
    }
    // setter functions
    func third_func{...}(...) {
        ...
    }
    // more complex functions writing data
    func fourth_func{...}(...) {
        ...
    }
}
```

### File contract 
`my_contract.cairo`

```rust
// imports
from openzeppelin.access.ownable.library import Ownable
from contracts.my_contract_library import MyContract
// view functions
@view
func get_name{..}() -> felt {
    return MyContract.get_name();
}
...
// external functions
@external
func set_name{..}(name : felt) {
    Ownable.assert_only_owner();
    return MyContract.set_name();
}
...
```

**Disclaimer - Tuyên bố miễn trừ trách nhiệm**: Xin lưu ý rằng các khuyến nghị trên là từ kinh nghiệm làm việc với Cairo tại Nethermind. Chúng tôi không tuyên bố rằng đó là cách duy nhất để giải quyết vấn đề. Chúng tôi đơn giản là chia sẻ các bài học để bạn có thể làm theo, nếu bạn muốn. Không có bất kỳ 1 đảm bảo nào về chất lượng mã code nếu bạn làm theo các khuyến nghị của chúng tôi.








