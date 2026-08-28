# Driver RTCore64.sys
# Kỹ thuật BYOVD trong ghostemperor
Trước khi thực hiện load driver, mã độc kiểm tra process avp.exe có tồn tại không bằng cách gọi hàm `ZwQuerySystemInformation()` để truy cập vào `_SYSTEM_PROCESS_INFORMATION`, hàm trả về PID của process nếu tồn tại 

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

- Gọi `ZwQuerySystemInformation()` với `SYSTEM_INFORMATION_CLASS = 0xB` để truy cập struct [RTL_PROCESS_INFORMATION](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/ntldr/rtl_process_modules.htm) và [RTL_PROCESS_MODULE_INFORMATION](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/ntldr/rtl_process_module_information.htm) lấy thông tin các module được load trong hệ thống. Duyệt và tìm module có đầu `nt` và đuôi `xe` thường là `ntoskrnl.exe` và lưu lại ImageBase của module đó

<img width="818" height="706" alt="image" src="https://github.com/user-attachments/assets/ecb4a7f4-d13d-4f4c-a693-c284ad6675f7" />

- Build đường dẫn `ntoskrnl.exe` bằng `GetSystemWindowsDirectoryW()`, đọc file trên máy vào một buffer

- Gọi `RtlGetVersion` và `RtlGetNtVersionNumbers` để lấy BuildVersion, nếu <= 15000 đặt `PTE_BASE = 0xFFFFF68000000000`. Nếu lớn hơn thực hiện parse export directory của `ntoskrnl.exe` và hash tên export bằng CRC16-CCIT để resolve RVA của hàm `MmGetVirtualForPhysical()`

<img width="510" height="368" alt="image" src="https://github.com/user-attachments/assets/cb715b93-5398-412d-8c66-cd1176fa446a" />

<img width="985" height="355" alt="image" src="https://github.com/user-attachments/assets/9e1d8fe5-1b45-4fad-b12f-80f55efd08b1" />

- Gọi code IOCTL `0x80002048` từ RTCore64.sys để đọc giá trị của `PTE_BASE` từ offset 0x22 của RVA hàm `MmGetVirtualForPhysical()`. [Kĩ thuật tương tự](https://github.com/smallzhong/hide_execute_memory/blob/master/KMDF%20Driver5/memory.c)

<img width="986" height="360" alt="image" src="https://github.com/user-attachments/assets/1d54986d-2f30-431d-b0d3-0c08c7a55d3b" />

- Lấy ImageBase của RTCore64.sys, rồi chuẩn bị trước 2 offset là `0x1310`, đây là RVA của hàm `DispatchDeviceControl()` thực hiện dispatch IOCTL. `0x3060` offset này về sau sẽ được sử dụng làm địa chỉ IAT chứa các kernel API

<img width="633" height="40" alt="image" src="https://github.com/user-attachments/assets/44ce9dd1-0884-466f-99ba-267aa923cd4b" />

- Resolve 17 API từ `ntoskrnl.exe` 

```
ExAllocatePool
IoAllocateMdl
MmBuildMdlForNonPagedPool
MmProbeAndLockPages
MmMapLockedPagesSpecifyCache
IoFreeMdl
IoCreateDriver
MmGetSystemRoutineAddress
RtlFindExportedRoutineByName
RtlInitUnicodeString
RtlInitAnsiString
RtlAnsiStringToUnicodeString
RtlFreeUnicodeString
IofCompleteRequest
MmUnmapIoSpace
ZwUnmapViewOfSection
```

- Sau đó mã độc chuẩn bị 2 buffer:
  + 0x207A byte để chứa hàm thay thế `DispatchDeviceControl()` của `RTCore64.sys` phục vụ việc load rootkit của ghostemperor về sau
  + 0x868 byte để chứa RVA của 17 API vừa được resolve

Sau khi đã setup xong các dữ liệu cần thiết trong userspace, ghostemperor bắt đầu sử dụng `PTE_BASE` để thực hiện patch lại dispatch routine của `RTCore64.sys`:

- Tính toán địa chỉ 2 PTE từ RVA của 2 vùng nhớ đích (RTCore64_Base + 0x1310 và RTCore64_Base + 0x3060) và `PTE_BASE` đã đọc được từ trước
- Gửi IOCTL `0x80002048` của `RTCore64.sys` để đọc giá trị 64-bit của 2 PTE trên từ bộ nhớ, từ đó lấy được physical page frame của 2 vùng nhớ đích

<img width="762" height="361" alt="image" src="https://github.com/user-attachments/assets/48a136fc-917a-49c8-9bf4-ebd66d2eaad0" />

- Gửi IOCTL `0x80002040` (MmMapIoSpace) để map 2 physical frame trên thành con trỏ User-Mode có quyền read/write
- Gửi IOCTL `0x80002044` (MmUnmapIoSpace) để đóng ánh xạ trang vật lý sau khi ghi xong
- Hàm `DispatchDeviceControl()` gốc của `RTCore64.sys` bị thay thế bằng dispatcher của ghostEmperor, có 6 IOCTL (0x220200 - 0x220214)
