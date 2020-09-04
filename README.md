# The difference between ELF and PE files.
Cách sử dụng .so/.lib file và sự khác nhau giữa file ELF và PE khi liên kết động và biên dịch file thực thi.
-

  Biên dịch file liên kết động trên Linux sẽ tạo ra một file `.so`, file này nên được mang theo khi compiling một file thực thi. Biên dịch một file liên kết động trên Windows sẽ tạo ra file `.dll` và `.lib`, file này cần mang theo khi compiling file thực thi cần dùng `.lib`. Bài viết này sẽ giải thích cho câu hỏi tại sao tệp thực thi cần phải được compiled với file .so/.lib và sự khác nhau giữa file ELF và PE (theo cách tác giả nhìn nhận).

  Ta sẽ lấy ví dụ về việc gọi các hàm trong thư viện liên kết động để giải thích rõ hơn.

# Tại sao chúng ta cần file .so/.lib?
  Giả sử chúng ta có 2 file (`Main.c` và `Lib.c`) như sau:
  ``` c
  /* Main.c */
  int main() {
    foo();
    return 0;
  }

  /* Lib.c */
  #include <stdio.h>
  void foo() {
    printf("this is foo");
  }
  ```
  `Lib.c` được biên dịch vào một thư viện liên kết động của chính nó, còn Main.c sẽ được biên dịch thành một file thực thi. Hàm `main()` trong Lib.c sẽ làm hàm `foo()`, lưu ý đây chỉ là ví dụ mà ta mong đợi. Lúc này khi ta biên dịch file Main.c sẽ không được, do trình biên dịch không biết gì về hàm foo(). Vì vậy, chúng ta cần phải có ít nhất 2 thông tin sau:
  + Hàm được định nghĩa ở đâu: nó vẫn chưa được định nghia trong thư viện liên kết động.
  + Tên thư viện liên kết động là gì: Thư viện liên kết động tương ứng có thể được load khi tệp thực thi chạy.
  Chính vì lẽ đó, các file .so/.lib cần phải được cung cấp khi biên dịch và tạo ra một tệp thực thi có sử dụng thư viện liên kết động. 2 phần tiếp theo (**ELF** và **PE**) sẽ cho chúng ta cách nhìn chi tiết hơn về trình biên dịch:
  ## ELF
    ### Ví dụ:
      `Program.c`
      ``` c
      // Program.c
      int main() {
        foobar(1);
        return 0;
      }
      ```
      `Lib.c`
      ``` c
      // Lib.c 
      #include <stdio.h>
      
      void foobar(int i) {
        printf("result = %d\n", i);
      }
      ```
      
      Tiến hành biên dịch `Lib.c` vào một `shared object` để tạo ra `Lib.so`:
      ```
      gcc -fPIC -shared Lib.c -o Lib.so
      ```
      Biên dịch `Program.c` và tạo ra `Program.o`:
      ```
      gcc -c Program.c
      ```
      Liên kết `Program.o` và `Lib.so` để tạo ra file thực thi:
      ```
      gcc Program.o ./Lib.so -o Program
      ```
      
      Sử dụng `objdump` để phân tích, ta có thể thấy địa chỉ thật sự của hàm `foobar()` không tồn tại. Vậy làm thế nào nó tìm được địa chỉ của hàm foobar khi chương trình chạy? Trong ELF, tất cả các hàm toàn cục và biến toàn cục của `shared object` (files .so) được export theo mặc định và được lưu trong bảng dynamic symbol ở phần `.symbol`. Khi linking, ta có thể nhìn thấy `foobar()` được định nghĩa bởi `shared object` bằng việc nhìn vào bảng dynamic symbol trong `Lib.so`. Tại đây GOT và PLT được tạo trong file thực thi và hàm `foobar()` sẽ được thêm vào đó:
      ``` c
      0000000000000000 <main>:	// Program.o
         0:   55                      push   rbp
         1:   48 89 e5                mov    rbp, rsp
         4:   bf 01 00 00 00          mov    edi, 0x1
         9:   b8 00 00 00 00          mov    eax, 0x0
         e:   e8 00 00 00 00          call  13 <main+0x13>
        13:   b8 00 00 00 00          mov    eax, 0x0
        18:   5d                      pop    rbp
        19:   c3                      ret
      
      
      0000000000400686 <main>:		// Program
        400686:       55                      push   %rbp
        400687:       48 89 e5                mov    %rsp,%rbp
        40068a:       bf 01 00 00 00          mov    $0x1,%edi
        40068f:       b8 00 00 00 00          mov    $0x0,%eax
        400694:       e8 c7 fe ff ff          callq  400560 <foobar@plt>
        400699:       b8 00 00 00 00          mov    $0x0,%eax
        40069e:       5d                      pop    %rbp
        40069f:       c3                      retq
      ```
      
    ### Load dynamic link library
      Làm thế nào để tệp thực thi biết được `stored object` nào được load? Giả sử có nhiều file .so được liên kết với file thực thi, liệu tất cả các file .so đó có cần phải được load lên khi chương trình chạy? Câu trả lời là không, file thực thi sẽ chỉ load các tệp `shared object` cần thiết, cái mà có chứa hàm hoặc biến mà file thực thi cần dùng. Vậy file thực thi ghi những gì? Hãy nhìn vào dynamic section của Program bằng công cụ `readelf`

      ``` 
      Dynamic section at offset 0xe18 contains 25 entries:
        Tag                 Type                         Name/Value
       0x0000000000000001 (NEEDED)             Shared library：[./Lib.so]
       0x0000000000000001 (NEEDED)             Shared library：[libc.so.6]
      ...
      ```
      Lib.so được đặt trong dynamic section, chứa đường dẫn của file `Lib.so`. Để khi thực thi file nó sẽ load `shared object` (ở đây là file `.so`) bằng được dẫn được lưu trong dynamic section. Nếu file `shared object` không tồn tại, nó sẽ báo lỗi.

  ## PE
    ### Ví dụ:
      `Program.c`:
      ``` c
      // Program.c 
      int main() {
        foobar(1);
        return 0;
      }
      ```
      
      `Lib.c`:
      ``` c
      // Lib.c 
      #include <stdio.h>
      #define DllExport __declspec(dllexport)
      
      DllExport void foobar(int i) {
        printf("result = %d\n", i);
      }
      ```
      
      Biên dịch `Lib.c` vào `dll` để tạo `Lib.dll` và `Lib.lib`.
      ``` bash
      cl /LD Lib.c
      ``` 
      Biên dịch `Program.c` thành `Program.obj`.
      ``` bash
      cl /c Program.c
      ``` 
      Liên kết `Program.obj` và `Lib.lib` để tạo ra file thực thi `Program.exe`.
      ``` bash
      link Program.obj Lib.lib
      ```
      
    ### File .lib là gì?
      Trong phần biên dịch, ta thấy `Lib.c` sẽ tạo ra 2 file: `Lib.dll` và `Lib.lib`, tuy nhiên, lúc liên kết với `Program.obj`, ta chỉ sử dụng `Lib.lib`, vậy `Lib.lib` có gì và nó có liên quan đến `Lib.dll` không?
      `Lib.lib` ở đây được gọi là **import Library**, nó không chưa code và data của source (ở đây là `Lib.c`), nó chỉ được sử dụng để mô tả các export symbol của DLL (ở đây là `Lib.dll`) và chứa một phần "stub code". Dưới đây là những gì có bên trong file `Lib.lib`:
      ``` bash
      5 public symbols
      
        172 __IMPORT_DESCRIPTOR_Lib
        38C __NULL_IMPORT_DESCRIPTOR
        4BE Lib_NULL_THUNK_DATA
        608 __imp__foobar
        608 _foobar
      ...
      Version      : 0
      Machine      : 14C (x86)
      TimeDateStamp: 57F89554 Sat Oct  8 14:42:28 2016
      SizeOfData   : 00000010
      DLL name     : Lib.dll
      Symbol name  : _foobar
      Type         : code
      Name type    : no prefix
      Hint         : 0
      Name         : foobar
      ...
      ```
      
      Nội dung trên chứa bản ghi của hàm `foobar()` và tên file DLL `Lib.dll`.
      
      Một điều lưu ý ở đây là ** DLL không export bất cứ symbols nào theo mặc định (ELF export symbols theo mặc định) ** và các **symbols** cần được export cần khai báo rõ ràng, ví dụ như trong file `Lib.c`,  __declspec(dllexport) sẽ được export foobar() exported qua việc khai báo.
      
      Các symbols được export từ DLL sẽ được đặt trong bảng Export Table và sẽ được ghi trong Import library (file `.lib`), nó có thể tìm thấy hàm `foobar()` bằng việc kiểm tra `Lib.lib` khi linking được khai bào bởi DLL. Tại đây IAT (`Import Address Table` - các hàm thư viện nhập mà một chương trình sử dụng đều được đặt trong bảng này) được tạo trong file thực thi và hàm `foobar()` sẽ được thêm vào:
      
      ```
      _main:				//Program.obj
        00000000: 55                 push        ebp
        00000001: 8B EC              mov         ebp,esp
        00000003: 6A 01              push        1
        00000005: E8 00 00 00 00     call        _foobar
        0000000A: 83 C4 04           add         esp,4
        0000000D: 33 C0              xor         eax,eax
        0000000F: 5D                 pop         ebp
        00000010: C3                 ret
        
        
        File Type: EXECUTABLE IMAGE	// Program.exe
      
        00401000: 55                 push        ebp
        00401001: 8B EC              mov         ebp,esp
        00401003: 6A 01              push        1
        00401005: E8 08 00 00 00     call        00401012
        0040100A: 83 C4 04           add         esp,4
        0040100D: 33 C0              xor         eax,eax
        0040100F: 5D                 pop         ebp
        00401010: C3                 ret
        00401011: CC                 int         3
        00401012: FF 25 08 C1 40 00  jmp         dword ptr ds:[0040C108h]
      ```
      
      Khá giống với ELF trên Linux, tuy nhiên nhìn vào code trong file `Program.exe` chúng ta có thể thấy, chương trình thay vì nhảy trực tiếp vào IAT, nó nhảy đến `00401012`, tại đây nó mới bắt đầu nhảy đến IAT `jmp dword ptr ds:[0040C108h]`. Tại sao lại như vậy?
      Bởi vì PE phân biệt giữa lệnh gọi trực tiếp và lệnh gọi gián tiếp, tương ứng với lệnh `call` và lệnh `jmp dword ptr xxxx` (Đoạn này không chắc, nếu sai xin hãy góp ý). Vì vậy ta không chỉ sửa đổi offset address là được, nó cần một vài lệnh khác nữa.
      Vậy lệnh jump trung gian đó được tạo ra từ đâu? Giả sử, chúng ta đều biết rằng linker generally  không tạo ra lệnh, vậy dòng lệnh 00401012 được tạo ra từ đâu? Câu trả lời là từ import library - `Lib.lib` - Lệnh này còn được gọi là "stub code". Theo đó, ở lần gọi hàm `foobar()` đầu tiên sẽ nhảy đến stub code, sau đó nhảy đến `foobar()` trong IAT và cuối cùng là nhảy đến địa chỉ thật của hàm `foobar()`.
        
    ### Load dynamic link library
      ELF sử dụng dynamic link library khi linking, vì vậy tên và đường dẫn của dynamic link library được ghi lại trong file thực thi. Còn PE chỉ sử dụng file import library .lib khi linking, vậy tệp thực thi được ghi trong dynamic link library như thế nào?
      Câu trả lời thật ra đã được đưa ra từ trước, chúng ta cùng xem lại file `Lib.lib`:
      ```
      Version      : 0
      Machine      : 14C (x86)
      TimeDateStamp: 57F89554 Sat Oct  8 14:42:28 2016
      SizeOfData   : 00000010
      DLL name     : Lib.dll
      Symbol name  : _foobar
      Type         : code
      Name type    : no prefix
      Hint         : 0
      Name         : foobar
      ```
      
      Với mỗi symbol được export, import library không chỉ ghi tên của symbol, nó còn ghi thêm cả tên DLL chứa symbol đó (ở đây là `Lib.lib`), ngoài ra còn có `version` và timeDateStamp`. 
      Ngoài ra trong file thực thi, thông tin này còn được ghi lại trong Import Table:
      ```
      Section contains the following imports:
      
        Lib.dll
                    40C108 Import Address Table
                    411190 Import Name Table
                         0 time date stamp
                         0 Index of first forwarder reference
      
                        0 foobar
      
        KERNEL32.dll
                    40C000 Import Address Table
                    411088 Import Name Table
                         0 time date stamp
                         0 Index of first forwarder reference
      ```
            
      `Lib.dll` chứa trong file thực thi, nó sẽ dựa vào đây (Import Table) để tìm kiếm DLL theo tên dll khi chạy. Bạn có thác mắc tại sao nó chỉ chứa tên mà không phải là một đường dẫn chứa file dll như ELF không? PE không ghi path của dll? Đúng vậy, PE sẽ không ghi lại path của dll. Khi tìm kiếm một file dll, hệ điều hành sẽ tìm kiếm file dll theo quy tắc được thiết lập từ trước (tham khảo thêm tại [Search Path Used by Windows to Locate a DLL](https://docs.microsoft.com/en-us/previous-versions/7d83bc18(v=vs.140)?redirectedfrom=MSDN)). Vì vậy mà chỉ có tên file dll được ghi vào tệp thực thi.
      
# Tóm tắt các điểm khác nhau chính giữa file ELF và PE:
  Ở đây tôi sẽ tóm tắt các điểm khác nhau chủ yếu của ELF và PE file theo quan điểm của [tác giả](https://www.polarxiong.com/archives/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E7%BC%96%E8%AF%91%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%E6%97%B6-so-lib%E6%96%87%E4%BB%B6%E7%9A%84%E7%94%A8%E5%A4%84%E4%BB%A5%E5%8F%8AELF%E4%B8%8EPE%E6%96%87%E4%BB%B6%E7%9A%84%E5%8C%BA%E5%88%AB.html) về dynamic link library:
  + Dynamic link library của ELF export tất cả các symbol theo mặc định, còn PE thì ngược lại, PE cần phải được khai báo rõ ràng.
  + Khi relocating, ELF chỉ cần sửa đổi offset address, còn PE có thể cần chèn thêm stub code (Do các lệnh khác nhau mà PE gọi là internal và external functions)
  + Trong file thực thi, ELF sẽ ghi đường dẫn của dynamic link library, và load dynamic link library từ path đã ghi trong runtime, PE thì chỉ ghi lại thông tin như tên của dynamic link library, và tìm kiếm theo được dẫn được thiết lập trước bởi hệ điều hành trong runtime.
    
# Tham khảo: [The use of .so/.lib files and the difference between ELF and PE files when dynamically linking and compiling executable files](https://www.polarxiong.com/archives/%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E7%BC%96%E8%AF%91%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%96%87%E4%BB%B6%E6%97%B6-so-lib%E6%96%87%E4%BB%B6%E7%9A%84%E7%94%A8%E5%A4%84%E4%BB%A5%E5%8F%8AELF%E4%B8%8EPE%E6%96%87%E4%BB%B6%E7%9A%84%E5%8C%BA%E5%88%AB.html)

<p align="right">
  <b><i>DatntSec. Viettel Cyber Security.<i><b>
</p>
    

