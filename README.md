# Cry-More

#### Đầu tiên mình sẽ giải thích đoạn code trong file server.py:

![image](https://user-images.githubusercontent.com/66832698/140521984-28a3fd47-992a-441d-b9e6-4df7f6e98dbd.png)
 
![image](https://user-images.githubusercontent.com/66832698/140522125-a2a13bd4-90f2-45cf-ae99-35d2bcf25694.png)
Khi ta kết nối đến server, server sẽ gen ra một đoạn key có độ dài ngẫu nhiên từ 8-32 bytes dùng để hash với thông tin thanh toán. Sau đó ta sẽ được cấp một khoản tiền ngẫu nhiên trong khoảng 1 đến 2000. Và giá của flag là 99999 hoàn toàn không đủ tiền để mua.

Tiếp theo sẽ là 2 function quan trọng nhất là order và confirm:
![image](https://user-images.githubusercontent.com/66832698/140522677-044aadc0-3850-4d62-bc3e-5ce584bdee52.png)
![image](https://user-images.githubusercontent.com/66832698/140522720-6d74fd28-7798-4931-a92d-17419397b9a5.png)
![image](https://user-images.githubusercontent.com/66832698/140522881-08933a0e-9f30-4581-a640-7cb4c03a1795.png)

##### Trước tiên sẽ là function order:
Khi ta chọn một item bất kì để thanh toán nó sẽ được lưu lại thông tin gồm tên sản phẩm, giá và thời gian.
Sau đó sẽ được hash với signkey đã được server gen ra ở trên qua câu lệnh sau
**signature = sha512(self.signkey+payment).hexdigest()  (1)**

*Mọi người chú ý thứ tự thông tin được hash*

Sau đó được nối chuỗi với thông tin thanh toán, encode base64 và đẩy về phía người dùng

**payment += b'&sign=%s' % signature.encode() (2)**
**self.request.sendall(b'Your order: ')**
**self.request.sendall(b64encode(payment))**
**self.request.sendall(b'\n')**

Khi mình đọc đến đây lập tức nghĩ ra đây là hash length extension attack vì đoạn hash này thông tin payment nằm ở phía sau của key và người dùng biết được đoạn dữ liệu ở trước đó.

Tiếp theo sẽ là function confirm để xem cách server sẽ giải mã
Server lấy toàn bộ phần hash phía sau &sign gọi là hash1. Tiếp đó server lấy toàn bộ thông tin thanh toán ở phía trước (từ đầu tính đến trước &sign) sau đó hash gọi là hash2. So sánh hash1 với hash2 nếu đúng sẽ thực hiện tiếp quá trình thanh toán, nếu sai sẽ hủy bỏ.

Như các bạn đã biết thì hash là không thể dịch ngược vậy làm sao để ta có thể thay đổi thông tin thanh toán mà không cần biết signkey. Để có thể nắm rõ hơn phần này mình khuyến khích các bạn xem video này trước khi đọc phần sau nếu chưa biết về kỹ thuật hash length extension attack: https://www.youtube.com/watch?v=9yOKVqayixM

Ở phần **(1)** các bạn chú ý vào thứ tự của phần hash, signkey sau đó mới tới thông tin thanh toán. Phần thanh toán và mã hash có chứa signkey ta hoàn toàn biết được do server đã gửi về phía người dùng. Từ đó ta sẽ chèn dữ liệu giả mạo vào để tạo ra block thứ n+1 với nội dung theo ý muốn.

**Khai thác**
Sử dụng netcat để liên lạc với server và chọn bừa một gói bất kỳ để lấy mã hash và thông tin thanh toán:
![image](https://user-images.githubusercontent.com/66832698/140536601-11fdc315-e88e-4db1-aae1-999fc50917d5.png)
decode bash64 ta sẽ được phần thông tin thanh toán:
product=Fowl x 3&price=1&time=1636125992.77&sign=89917b5436c9956a10a59ce089d1e281b146c889c940579adaf8b9e55d133df9134b03453d92dda06fec24c61520a91720989c747a2cfd42a7983d69a4a8c955
payload ta cần:
product=Fowl x 3&price=1&time=1636125992.77[padding]&product=FLAG&sign=xxxxxxxxxxxxxxxxxx
Có thể nhiều bạn sẽ thắc mắc tại sao không thay thế luôn phần **product=Fowl x 3** vì đó là điều không thể. Ta lợi dụng các tính mã hash để tạo ra payload với signkey mới và phần product ở phía sau sẽ được thay thế product=Fowl x 3.
Mình sử dụng tool hashpump để tạo payload: https://github.com/bwall/HashPump
Tuy nhiên ta sẽ phải bruteforce phần padding từ 8-32 bytes
![image](https://user-images.githubusercontent.com/66832698/140536456-a0cd9de1-501c-41a3-b4c7-3095a8ae673d.png)
ở dòng trên sẽ là mã hash ta sẽ đẩy vào signkey, dòng dưới là dữ liệu đã được thay đổi

payload: product=Fowl x 3&price=1&time=1636125992.77\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x98&product=FLAG&sign=f8df4f2491d0f085d6bdc3d40e89bdb3d839cc56bb436ce2e5a2798a0f889a261ad29893dc7908cf75c515b365fe086a3166ee15698996747b7c63d7956ae7a5

encode lại và gửi về server:
![image](https://user-images.githubusercontent.com/66832698/140532546-32dc6029-64cb-40ab-8ebb-3201b44d772b.png)
tăng dần độ dài của khóa để bruteforce cho đến khi thành công.

hơi đen đến payload thứ 27 mới thành công:

![image](https://user-images.githubusercontent.com/66832698/140538151-0db2f290-8d70-4bd2-ad6b-49078f3a24a2.png)
![image](https://user-images.githubusercontent.com/66832698/140538091-717a4cc2-031f-4776-8503-8230b9e5380a.png)

