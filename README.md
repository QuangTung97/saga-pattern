# Synchronous Saga Pattern sử dụng API
Trong bài này chỉ là một trong những cách thức implement Saga Pattern sử dụng 
Database và API đơn giản. Có thể implement mà không cần bất kì một library hay
một framework nào. Tuy nhiên chỉ giới hạn trong bài toán Saga thông qua gọi Synchronous API.

### Phân biệt lỗi trong quá trình gọi API
Khi một service này gọi service kia trong kiến trúc Microservices, một API có thể gặp
nhiều loại mã lỗi, ví dụ: lỗi mạng, lỗi invalid input, lỗi không truy cập DB,...

Do những API trong một Saga luôn là các API thực hiện thay đổi dữ liệu, nên mình cũng không xét đến
những API chỉ đọc trong bài này. 


Để thuận tiện cho việc mô tả thuật toán Saga, mình sẽ chia các lỗi ra thành 2 loại lỗi sau khi gọi một API:
* Loại lỗi **persistent**: là loại lỗi trả về khi gọi API mà bên gọi (caller) có thể chắc chắn rằng bên đầu
nhận (callee) không thực hiện ghi dữ liệu thành công vào DB hoặc ghi vào DB nhưng đã bị rollback. Điển hình
của lỗi này ví dụ như: sai input, lỗi resource không tồn tại, lỗi không thỏa mãn
một precondition nào đó trong luồng logic, lỗi deadlock detected và transaction bị rollback.

* Loại lỗi **transient**: là những loại lỗi còn lại, ví dụ lỗi mạng,
lỗi không kết nối được database, lỗi request timeout,...

Có nhiều người thường bị nhầm lẫn lỗi mạng hay lỗi kết nối vào database là đầu nhận (callee) đã không
nhận được request nhưng điều đó là sai lầm vì đầu nhận (callee) hoàn toàn có thể
ghi nhận request và trong quá trình response service bị sập, network bị timeout do thời gian xử lý lâu.
Vì vậy trong hệ phân tán nói chung, nếu gặp một lỗi **transient** thì bên caller sẽ không được phép
giả định rằng callee đã không nhận được request.