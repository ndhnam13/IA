Trươc khi thực hiện load driver, mã độc kiểm tra process avp.exe có tồn tại không bằng cách gọi hàm `ZwQuerySystemInformation()` để truy cập vào `_SYSTEM_PROCESS_INFORMATION`, hàm trả về PID của process nếu tồn tại 

<img width="457" height="253" alt="image" src="https://github.com/user-attachments/assets/b460bfa3-7112-4a1e-8db2-f1f75078b716" />

<img width="937" height="608" alt="image" src="https://github.com/user-attachments/assets/90095761-711f-43e8-b8db-b53cf1906df0" />

Nếu process không tồn tại thì mã độc mới tiếp tục thực hiện load driver

Khởi tạo một vtable chứa các hàm phục vụ việc thực thi BYOVD

<img width="971" height="216" alt="image" src="https://github.com/user-attachments/assets/2359b78a-f8ed-4b7f-a171-7423613e4113" />

`fn_open_device_or_install` gọi resolve và gọi `ZwOpenFile()` để lấy handle của của `\Device\UPnP Control Point`. Nếu device này chưa tồn tại trên máy thì mã độc sẽ gọi `fn_install_driver()` để drop và cài đặt một driver không độc hại trên máy người

Trước hết, mã độc decrypt, decompress một rootkit nằm trong section `.data`. Data được decrypt bằng cách XOR với 4 byte `ddd195c6` rồi thực hiện thuật toán decompress lzo1x. Rootkit là một driver không có digital certificate và các đoạn mã cũng bị obfuscate bằng dạng CFF tương tự như của các stage ghost emperor trước 

Rootkit này sẽ được cài đặt sau khi mã độc lợi dụng một driver khác leo quyền để cài đặt driver không có digital certificate. Sau khi đã decrypt rootkit, mã độc bắt đầu thực hiện các kỹ thuật BYOVD

Đầu tiên, RTCore64.sys dược drop vào `%TEMP%` với tên ngẫu nhiên
