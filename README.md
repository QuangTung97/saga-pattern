# Synchronous Saga Pattern sử dụng API

Trong bài này chỉ là một trong những cách thức implement Saga Pattern sử dụng
Database và API đơn giản. Có thể implement mà không cần bất kì một library hay
một framework nào. Tuy nhiên chỉ giới hạn trong bài toán Saga thông qua gọi Synchronous API.

### Phân loại lỗi trong quá trình gọi API

Khi một service này gọi service kia trong kiến trúc Microservices, một API có thể gặp
nhiều loại mã lỗi, ví dụ: lỗi mạng, lỗi invalid input, lỗi không truy cập DB,...

Do những API trong một Saga thường luôn là các API thực hiện thay đổi dữ liệu, nên mình cũng không xét đến
những API chỉ đọc (readonly) trong bài này.

Để thuận tiện cho việc mô tả thuật toán Saga, mình sẽ chia các lỗi ra thành 2 loại lỗi sau khi gọi một API:

* Loại lỗi **persistent**: là loại lỗi trả về khi gọi API mà bên gọi (**caller**) có thể chắc chắn rằng bên đầu
  nhận (**callee**) không thực hiện ghi dữ liệu thành công vào DB hoặc ghi vào DB nhưng đã bị rollback. Điển hình
  của lỗi này ví dụ như: sai input, lỗi resource không tồn tại, lỗi không thỏa mãn
  một precondition nào đó trong luồng logic, lỗi deadlock detected và transaction bị rollback.

* Loại lỗi **transient**: là những loại lỗi còn lại, ví dụ lỗi mạng,
  lỗi không kết nối được database, lỗi request timeout,...

Có nhiều người thường bị nhầm lẫn lỗi mạng hay lỗi kết nối vào database là đầu nhận (**callee**) đã không
nhận được request nhưng điều đó là sai lầm vì đầu nhận (**callee**) hoàn toàn có thể
ghi nhận request và trong quá trình response service bị sập, network bị timeout do thời gian xử lý lâu.
Vì vậy trong hệ phân tán nói chung, nếu gặp một lỗi **transient** thì bên **caller** sẽ không được phép
giả định rằng **callee** đã không nhận được request.

### Vấn đề về thứ tự của các API trong network
Mình đã từng gặp rất nhiều trường hợp hiểu sai rằng khi gọi bằng API thì thứ tự sẽ được đảm bảo,
điển hình như ví dụ sau:

1) Một biến version lưu trong db có giá trị ban đầu là 0.
2) Một thread thực hiện update version lên = 1 và sau khi quá trình lưu db 
thành công và gọi sang remote service khác với request có version = 1
3) Trong quá trình thread trước đó đang gọi remote service, một thread khác thực hiện update version lên = 2
và sau đó cũng thực hiện gọi sang cùng remote service với version = 2

Có người sẽ nghĩ rằng remote service sẽ chắc chắn cũng ghi nhận version = 2 nhưng mà giả định đó là sai.
Vì khi thread đầu tiên gọi remote service có thể request đó bị delay ở network, có thể quá trình
load balance làm request đầu tiên đến được một instance mà đang chạy GC lâu, hoặc tương tự.
Điều đó có thể dẫn đến request thứ 2 lại được xử lý trước request thứ nhất.
Khi mà xử lý API để đảm bảo consistency thì hệ thống cũng phải tính đến những trường hợp này.

### Mô hình bài toán
Trước khi đi đến thuật toán Saga tổng quát, mình sẽ giới thiệu thuật toán đảm bảo tính
chất Atomicity cho 2 service đơn giản (và có phần không đúng với thực tế :v) sau:
* **Order Service**: nơi nhận request tạo order từ người dùng và gọi lên Inventory Service
để trừ số lượng tồn dư của sản phẩm.
* **Inventory Service**: đầu nhận request từ phía Order Service, quản lý số lượng tồn dư.

Đầu Order Service sẽ có một API là Create Order API, đầu Inventory Service sẽ có 2 API là
Reserve Product Quantity và Cancel Reserve Product Quantity. 

Một request của người dùng sẽ đồng thời tạo Order trên Order Service
và thực hiện update trên Inventory Service. Tính chất Atomicity mà thuật toán Saga đạt được
là phải đảm bảo:
* Hoặc là cả 2 service đều ghi nhận thành công
* Hoặc là cả 2 service đều không ghi nhận dữ liệu hoặc ghi nhận dữ liệu với trạng thái là Cancelled.

### Thuật toán thứ nhất (thuật toán sai)
Quá trình Create Order API xử lý như sau:
1) Gọi Reserve Product Quantity sang bên Inventory Service.
2) Nếu lỗi trả về là lỗi **persistent** => trả về cho người dùng lỗi không tạo được order.
3) Nếu lỗi trả về là lỗi **transient** => thực hiện gọi API Cancel.
4) Nếu gọi inventory thành công => thực hiện logic tạo Order.
5) Nếu logic tạo order thành công, không có validation nào fail => commit và trả về success cho người dùng.
6) Nếu logic tạo order không thành công => không commit và thực hiện gọi API Cancel của Inventory.

Có thể thấy những vấn đề hiển nhiên của hướng giải quyết này:
* Nếu ở bước 4 trước khi tạo Order mà instance (pod trên K8s) không thể tiếp tục do bị kill sẽ dẫn
đến quantity đã bị reserve nhưng order không được tạo
* Ở bước 3 nếu lỗi **transient** thì khả năng cao là API Cancel
gọi ngay sau đó cũng không thể gọi được => dẫn đến quantity cũng đã có thể bị reserve nhưng không thể
cancel được quá trình reserve.
* Ở bước 6 tương tự như ở bước 4, nếu trong quá trình này instance bị kill cũng sẽ dẫn đến inconsistency.