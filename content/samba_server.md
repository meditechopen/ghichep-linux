# 21. Samba server and Windows file sharing

Samba là một hệ thống xử lí mã nguồn mở sử dụng giao thức SMB/CIFS. Nó cho phép các máy chủ  Microsoft Windows®, Linux, UNIX, và các hệ điều hành khác kết nối với nhau. Sử dụng Samba, các máy chủ Linux/Unix có thể trở thành Windows server đối với các máy Windows clients.

Với Samba, người quản trị có thể

1. Phục vụ cây thư mục và dịch vụ in cho các máy clients chạy hđh Linux, UNIX, và Windows.
2. Hỗ trợ duyệt mạng có hoặc không có NetBIOS
3. Xác thực đăng nhập domain Windows
4. Cung cấp giải pháp WINS name server.

Samba là sự kết hợp của smb, nmb, và winbind services.

`smbd` server cung cấp dịch vụ in và chia sẻ dữ liệu cho các máy Windows clients. Nó có trách nhiệm xác thực người dùng, khóa tài nguyên và chia sẻ dữ liệu thông qua giao thức SMB. Port mặc định mà server dùng cho SMB là các cổng TCP 139 và 445.

`nmbd` server có thể hiểu và trả lời các requests từ SMB. Port mặc định của dịch vụ này là UDP 137.

Dịch vụ `winbindd` sẽ giải quyết thông tin người dùng tới từ những server chạy Windows. Điều này sẽ làm cho thông tin của các Windows clients có thể được "hiểu" bởi các máy chủ chạy Linux và UNIX. Cả `winbindd` và `smbd` đều đi kèm trong các bản phân phối của Samba. Tuy vậy chúng được kiểm soát riêng biệt với nhau.

## Cài đặt Samba server 

Chúng ta sẽ cài đặt Samba server để biến Linux trở thành file sharing server đối với các Windows clients. Quá trình này bao gồm việc cài đặt Samba package, khởi động dịch vụ `smbd` và `nmbd`.

``` sh
# yum install samba
# systemctl enable smb
# systemctl enable nmb
# systemctl start smb
# systemctl start nmb
```

Samba sử dụng `/etc/samba/smb.conf` làm file cấu hình

``` sh
# mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
# vi /etc/samba/smb.conf

# =============== Global configuration ===============
[global]
; Windows workgroup name and server description
workgroup = WORKGROUP
server string = My SMB Server %v
; NetBIOS name as the Linux machine will appear in Windows clients
netbios name = MYSMBSERVER
; interfaces where the service is listening: localhost and ens32 interfaces
interfaces = lo ens32
; users passwords database backend and location
passdb backend = smbpasswd
smb passwd file = /etc/samba/smbpasswd
; permitted hosts to use the Samba server: localhost and all host belonging to 10.10.10.0/24 subnet
hosts allow = 127. 10.10.10.
; protocol version
max protocol = SMB3
; type of security
security = user
; no printing services
printing = bsd
printcap name = /dev/null

# =============== Shares configuration ===============
[share1]
comment = Private Documents
; path of files to share
path = /samba/admin/data
; users admitted to use the file sharing service
valid users = admin
invalid users = user2 user3
; no guest user is admitted
guest ok = no
; make the share writable as Samba make it as readonly by default
writable = yes
; make the share visible as shared folder
browsable = yes

[share2]
comment = Public Documents
path = /samba/user2/data
valid users = user2 admin
guest ok = no
writable = yes
browsable = yes

[share3]
comment = Public Documents
path = /samba/user3/data
valid users = user3 admin
guest ok = no
writable = yes
browsable = yes
```

Có thể check file cấu hình của Samba bằng câu lệnh `testparm`

``` sh
# testparm /etc/samba/smb.conf
Load smb config files from /etc/samba/smb.conf
rlimit_max: increasing rlimit_max (1024) to minimum Windows limit (16384)
Processing section "[homes]"
Processing section "[admin]"
Processing section "[guest]"
Loaded services file OK.
Server role: ROLE_STANDALONE
Press enter to see a dump of your service definitions
```

## Truy cập người dùng

Nhiều người dùng có thể được khai báo để truy cập vào cùng một chia sẻ. Trong trường hợp phía trên, share1 chỉ cho phép admin truy cập. share2 có thể truy cập bởi user2 và admin. 

Lưu ý:  Kết nối chia sẻ bởi cùng Windows client cần phải dùng cùng  user name. Trong trường hợp trên, Window client có thể truy cập tới toàn bộ các chia sẻ dưới danh nghĩa "admin", tuy nhiên nó lại không thể truy cập share2 dưới danh nghĩa "user2" và share3 dưới danh nghĩa "user3". Nếu Windows client cần thiết phải truy cập với nhiều users khác nhau, người dùng sẽ phải logout và login lại với user khác. Bởi vì Windows sẽ cache lại người dùng đăng nhập nên buộc phải đăng xuất bằng lệnh `net use * /delete`.

``` sh
Microsoft Windows [Versione 10.0.10240]
(c) 2015 Microsoft Corporation. Tutti i diritti sono riservati.
C:\Users\Adriano>net use * /delete
Connessioni remote presenti:
                    \\10.10.10.12\IPC$
Continuando si annulleranno le connessioni.
Continuare questa operazione? (S/N) [N]: S
Esecuzione comando riuscita.
```

Samba sử dụng nhiều hình thức bảo mật khác nhau. Trong trường hợp phía trên, phương thức sử dụng là mặc định (user level). Với phương thức này, mỗi chia sẻ được gán truy cập với những user cụ thể. Khi user gửi yêu cầu kết nối để chia sẻ, Samba sẽ xác thức bằng username  đã được khai báo trong file cấu hình và password trong database.

Samba sử dụng nhiều database backends để lưu trữ password người dùng. Các đơn giản nhất đó là lưu trữ password trong file `smbpasswd` giống như `/etc/passwd`. Thường thì file này sẽ được lưu tại `/var/lib/samba/private/smbpasswd`.

Thêm user và set password trong database

``` sh
# smbpasswd -a admin
New SMB password:
Retype new SMB password:
#
```

Câu lệnh `pdbedit` sẽ hiển thị danh sách user trong database

``` sh
# pdbedit -L
admin:1000:
user1:1001:
user2:1002:
user3:1003:
```

Với smbpassword database, Samba user phải là một user trong hệ thống Linux. Để bảo mật máy chủ, bạn nên hủy bỏ quyền đăng nhập từ những user này

``` sh
# useradd -d /samba/share user1
# usermod -s /bin/false user1
# cat /etc/passwd | grep user1
user1:x:1003:1002::/samba/share:/sbin/nologin
#
# ssh user1@localhost
user1@localhost's password:
Last login: Tue Sep 15 11:50:08 2015
This account is currently not available.
Connection to localhost closed.
#
# sftp user1@localhost
user1@localhost's password:
subsystem request failed on channel 0
Couldn't read packet: Connection reset by peer
```

## Quyền truy cập file và các thuộc tính

Ở ví dụ bên trên, Linux files sẽ được chia sẻ cho các máy Windows clients. Vì hai hệ điều hành này có quyền truy cập file và thuộc tính khác nhau nên Samba sẽ có cơ chế để mapping cả 2 lại.

Linux file có 3 chế độ là read, write và execute dành cho 3 nhóm đối tượng owner (u), group (g) và others (o). Trong khi đó, Windows có 4 chế độ đó là read-only, system, hidden, và archive:

1. Read-only: Nội dung file chỉ có thể được đọc
2. System: File này có mục đích cụ thể tùy theo yêu cầu của hệ điều hành
3. Hidden: File này sẽ bị đánh dấu cho ẩn đi đối với user
4. Archive: File này đã bị sửa đổi kể từ lần cuối cùng nó được backup.

Không có bất cứ bit nào chỉ ra rằng file này có thể được thực thi (execute) bởi Windows xác định điều này ở phần mở rộng của file. Windows files được lưu trữ trên Samba có những thuộc tính riêng cần được lưu giữ. Samba sẽ lưu giữ những bits này bằng cách tái sử dụng lại các bit cho phép thực thi của file. Tuy nhiên điều này cũng mang lại ảnh hưởng: nếu Windows user lưu trữ file ở Samba thì ở phía Linux, một vài bit thực thi sẽ được thiết lập.

Các tùy chọn của Samba quyết định việc mapping

``` sh
[share]
...
	store dos attributes = yes
	map archive = yes ;default is yes
	map system = yes  ;default is no
	map hidden = yes  ;default is no
```

Ba tùy chọn cuối cùng "map" achive, system và hidden với owner, group và others. Trong ví dụ phía trên, các tùy chọn được sử dụng theo cơ chế per-share. Chúng trở thành mặc định cho tất cả các chia sẻ. Tùy chọn thứ nhất cũng đảm bảo rằng Samba không thực hiện bất cứ thay đổi nào trên các bits chứa quyền truy cập của Windows.

Lưu ý các tùy chọn trên có thể được dùng nếu hệ thống file Linux hỗ trợ các tham số mở rộng và các tham số này đã được kích hoạt, thường là qua tùy chọn mount `user_xattr` trong file `/etc/fstab`. Không giống như `ext3` và `ext4`, hệ thống file `xfs` mặc định đã kích hoạt tùy chọn `user_xattr`.

Samba có tùy chọn `create mask` và `directory mask` giúp đỡ cho việc tạo mới file và thư mục. Những file và thư mục mới được tạo ra sẽ được khai báo quyền truy cập. Ở phía Linux, người dùng có thể kiểm soát quyền truy cập của file hoặc thư mục khi nó được tạo ra. Ở phía Windows, người dùng cũng có thể vô hiệu các tham số hóa read-only, archive, system, và hidden .

``` sh
[share]
...
	store dos attributes = yes
	map archive = yes            ;default is yes
	map system = yes             ;default is no
	map hidden = yes             ;default is no
	create mask = 0744           ;default is 0744
	directory mask = 0755        ;default is 0755
```

File và thư mục mới được tạo ở phía Linux:

``` sh
# ll /samba/share/user1
total 0
-rwxr--r-- 1 user1 samba 0 Sep 15 13:00 mydocument.txt
drwxr-xr-x 2 user1 samba 6 Sep 15 13:00 myfolder
```

Có thể "ép" các bits khác nhau theo tùy chọn `force create mode` và `force directory mode` . Với tùy chọn `create mask` và `create directory mask`, người quản trị có thể cho phép người dùng thiết lập các bits chứa quyền truy cập. Bên cạnh đó `force create mode` và `force directory mode` sẽ "ép" một số bits cụ thể kể cả khi nó không được yêu cầu bởi user.

Đồng thời, người dùng có thể "ép" thuộc tính của các file được tạo từ phía Windows bằng 2 tùy chọn `force user` và `force group`

``` sh
[share]
...
	store dos attributes = yes
	map archive = yes            ;default is yes
	map system = yes             ;default is no
	map hidden = yes             ;default is no
	create mask = 0744           ;default is 0744
	directory mask = 0755        ;default is 0755
	force create mode = 0000     ;default is 0000
	force directory mode = 0000  ;default is 0000
	force user = user1
	force group samba
```