Trươc khi thực hiện load driver, mã độc kiểm tra process avp.exe có tồn tại không bằng cách gọi hàm `ZwQuerySystemInformation()` để truy cập vào `_SYSTEM_PROCESS_INFORMATION`, hàm trả về PID của process nếu tồn tại 

<img width="457" height="253" alt="image" src="https://github.com/user-attachments/assets/b460bfa3-7112-4a1e-8db2-f1f75078b716" />

<img width="937" height="608" alt="image" src="https://github.com/user-attachments/assets/90095761-711f-43e8-b8db-b53cf1906df0" />

Nếu process không tồn tại thì mã độc mới tiếp tục thực hiện load driver
