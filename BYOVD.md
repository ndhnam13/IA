Trươc khi thực hiện load driver, mã độc kiểm tra process avp.exe có tồn tại không bằng cách gọi hàm `ZwQuerySystemInformation()` để truy cập vào `_SYSTEM_PROCESS_INFORMATION`, hàm trả về PID của process nếu tồn tại 

<img width="457" height="253" alt="image" src="https://github.com/user-attachments/assets/b460bfa3-7112-4a1e-8db2-f1f75078b716" />

<img width="937" height="608" alt="image" src="https://github.com/user-attachments/assets/90095761-711f-43e8-b8db-b53cf1906df0" />

Nếu process không tồn tại thì mã độc mới tiếp tục thực hiện load driver

Khởi tạo một vtable chứa các hàm phục vụ việc thực thi BYOVD

<img width="971" height="216" alt="image" src="https://github.com/user-attachments/assets/2359b78a-f8ed-4b7f-a171-7423613e4113" />

`fn_open_device_or_install` gọi resolve và gọi `ZwOpenFile()` để lấy handle của của `\Device\UPnP Control Point`. Nếu device này chưa tồn tại trên máy thì mã độc sẽ gọi `fn_install_driver()` để drop và cài đặt một driver không độc hại trên máy người

Trước hết, mã độc decrypt, decompress một rootkit nằm trong section `.data`. Data được decrypt bằng cách XOR với 4 byte `ddd195c6` rồi thực hiện thuật toán decompress lzo1x. Rootkit là một driver không có digital certificate và các đoạn mã cũng bị obfuscate bằng dạng CFF tương tự như của các stage ghost emperor trước 

Rootkit này sẽ được cài đặt sau khi mã độc lợi dụng một driver khác leo quyền để cài đặt driver không có digital certificate. Sau khi đã decrypt rootkit, mã độc bắt đầu thực hiện các kỹ thuật BYOVD

Đầu tiên, drop driver RTCore64.sys trong section `.data` (0x34C8 byte) của DLL vào `%TEMP%` với tên được tạo ngẫu nhiên (`w%08x.sys`)

<img width="952" height="416" alt="image" src="https://github.com/user-attachments/assets/48ee445b-e715-4650-bf09-b609bef820f6" />

<img width="805" height="565" alt="image" src="https://github.com/user-attachments/assets/1286850a-a129-48e8-90da-0b86c65ceb48" />

Nếu file được tạo thành công, ghost emperor tạo một registry key `HKLM\SYSTEM\CurrentControlSet\Services\Enrollments` với data là đường dẫn đến file driver được drop trong `%TEMP%`

Sau đó gọi `AdjustTokenPrivileges()` để đặt quyền `SeLoadDriverPrivilege` cho chính nó và resolve `ZwLoadDriver()` để load registry key trên vào hệ thống và lấy handle của `\Device\RTCore64`

<img width="956" height="272" alt="image" src="https://github.com/user-attachments/assets/6f9a1900-32f1-4e2a-abdb-1c94e833a652" />

<img width="788" height="588" alt="image" src="https://github.com/user-attachments/assets/509171ca-1a2f-46c1-98d9-f8d1e56fa5c0" />

Sau khi RTCore64 được load thành công, mã độc thực hiện patch kernel để load rootkit

- Gọi `ZwQuerySystemInformation`, tìm process `ntoskrnl.exe` extract địa chỉ kernel base
- Build `ntoskrnl.exe` path bằng `GetSystemWindowsDirectoryW()`, đọc file trên máy vào một buffer
- Gọi `RtlGetVersion` và `RtlGetNtVersionNumbers` để lấy BuildVersion, nếu <= 15000 0xFFFFF68000000000. Nếu lớn hơn thực hiện parse `ntoskrnl.exe` trong buffer vừa được allocate để tìm `MmPteBase/MiGetPteAddress` và gọi code IOCTL `0x80002048` từ RTCore64.sys để đọc giá trị của `PTE_BASE`
- `PTE_BASE` được sử dụng để mã độc có thể 
