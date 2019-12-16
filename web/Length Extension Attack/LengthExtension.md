Hash length extension attack - tấn công mở rộng độ dài hash là kỹ thuật tấn công dựa trên lỗ hổng của thuật toán hash.<br>
Hash length extension attack nhắm vào kỹ thuật MAC (Message Authenticate Code), bằng cách sử dụng kỹ thuật này, 
attacker có thể truy cập được vào hệ thống dựa trên các cookie public.

## Bối cảnh

Trang web bug.io có 1 chức năng X cần sử dụng dữ liệu được lưu trong cookie **data** của user.<br>
Để đảm bảo khi sử dụng chức năng đó user gửi lên đúng dữ liệu đã được set trong cookie, 
người lập trình web đã gửi thêm 1 cookie **signature** cho user. Cookie **signature** đó được tính như sau:<br>
```shell
signature = Hash(salt||data) (với '||' là phép nối 2 xâu secret và data)
```
Khi user sử dụng chức năng X, server sẽ đọc **data** và **signature** từ cookie của user, 
đọc chuỗi **salt** được lưu bí mật trên server, không public cho bất cứ ai. 
Sau đó tiến hành so sánh **Hash(salt||data) == signature**, đếu đúng thì mới cho phép user sử dụng chức năng.

Cách kiểm tra này được cho là bảo mật vì:
- Chuỗi **salt** được lưu trên server, không public.
- Các thuật toán hash đều là thuật toán mã hóa 1 chiều, 
các cách tấn công hash hiện nay đều là tấn công từ điển dựa trên wordlist với dữ liệu lớn.
- Không thể tìm đc **salt** dựa vào **signature** và **data**.

Tuy nhiên cách kiểm tra này có thể bị vượt qua bằng cách sử dụng kỹ thuật Hash Length Extension Attack.

## Cách tấn công

Từ cookie, chúng ta biết được **Hash(salt||data) = signature**, 
và có thể tính được **signature1 = Hash(salt||data||data1)** mà không cần biết **salt** là gì, 
nhưng cần phải biết độ dài salt.

Tool: [HashPump](https://github.com/bwall/HashPump)

Giả sử  với **salt** là secret (tất nhiên user không biết) thì cookie gửi về gồm:
- data = 0
- signature = 175ec77a7f63082590987d0d8b051aff

Ta có thể đánh lừa server để sử dụng được chức năng X với data = 2 
bằng cách gửi request với cookie mới gồm:
- data = data1
- signature = signature1

Chắc chắn là attacker không biết **salt** nên sẽ không thể tính được signature1 theo cách thông thường, 
tuy nhiên attacker có thể đoán độ dài **salt** = 6.

Tool HashPump có thể giúp chúng ta tính được signature1 trong 1 nốt nhạc nếu biết độ dài của salt, 
chỉ cần chạy lệnh như sau:
```shell
hashpump -s <signature trong cookie> -k <độ dài salt> -d <data trong cookie> -a <data1 muốn thêm vào>

Trong trường hợp này sẽ là:

hashpump -s 175ec77a7f63082590987d0d8b051aff -k 6 -d 0 -a 1
```
Kết quả:
```shell
7455d9a543c94add622ce190d032bd70
0\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x008\x00\x00\x00\x00\x00\x00\x001
```
Dòng đầu tiên trong output trả về là signature1, dòng thứ 2 là data1. 
Trước khi gửi request lên server thì sẽ cần encode một chút: đổi hết '\x' thành '%'

Giờ attacker chỉ cần thay đổi giá trị cookie trong request gửi lên server thành:
- data: 0%80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%008%00%00%00%00%00%00%001
- signature: 7455d9a543c94add622ce190d032bd70
