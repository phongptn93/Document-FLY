Mục lục

1. [Mô hình hoạt động](#tong-quan)
2. [Vì sao không dùng OpenSSH](#vi-sao)
3. [Chuẩn bị](#chuan-bi)
4. [Cài tự động bằng Claude](#tu-dong)
5. [Cài đặt từng bước](#cai-dat)
6. [Dùng hằng ngày](#hang-ngay)
7. [Herdr cơ bản](#herdr)
8. [Làm việc trên ổ Windows](#luu-y)
9. [Xử lý sự cố](#su-co)
10. [Bảo mật](#bao-mat)

ssh \<linux-user>@\<ten-may>-wsl

# Điều khiển Claude Code trên máy công ty từ điện thoại

Hướng dẫn dựng lối vào từ xa cho máy dev Windows: SSH từ iPhone hoặc laptop vào máy, mở herdr và tiếp tục các phiên Claude Code đang chạy, trên đúng thư mục dự án và đúng tài khoản Windows của bạn.

**Áp dụng cho** Windows 10/11, máy join Azure AD (Entra ID) **Thời gian** khoảng 30 phút **Công cụ** WSL, Tailscale, Termius, herdr

## Mô hình hoạt động

Điện thoại và máy dev cùng nằm trong một mạng riêng Tailscale. Bạn SSH vào một Ubuntu chạy trong WSL. Từ đó, `herdr` và `claude` gọi thẳng bản Windows, nên chạy dưới tài khoản Windows của bạn và dùng chung cấu hình, memory, phiên đăng nhập với lúc ngồi trước máy.

**Điện thoại**Termius + app Tailscale

**Tailscale**Mạng riêng, chỉ thiết bị của bạn

**WSL Ubuntu**Tailscale SSH, không cần sshd hay key

**herdr.exe, claude.exe**Chạy dưới tài khoản Windows, mở code ở `/mnt/d/…`

Chương trình Windows gọi từ WSL chạy dưới chính tài khoản đã đăng nhập Windows, nên không cần tạo tài khoản phụ.

## Vì sao không dùng OpenSSH của Windows

Cách hiển nhiên là bật OpenSSH Server có sẵn trong Windows. Trên máy join Azure AD, cách này không đăng nhập được bằng tài khoản công ty. Chúng tôi đã thử cả bản 9.5 có sẵn và bản 10.0 mới nhất:

| Cách đăng nhập | Kết quả |
| --- | --- |
| Key, user `azuread\ten` | Key được chấp nhận rồi treo khi mở phiên. Sau đó mọi lần đăng nhập vào user này đều treo cho tới khi **khởi động lại máy**. |
| Mật khẩu, user `azuread\ten.ho@congty.vn` | Bị từ chối dù mật khẩu đúng. Windows báo lỗi xác định tài khoản Azure AD (`0xC004844C`). |
| User `ten.ho@congty.vn` | sshd báo user không tồn tại. |
| Tạo user Windows cục bộ riêng | Đăng nhập được, nhưng là profile khác: không thấy các phiên Claude và herdr đang chạy, phải đăng nhập lại mọi thứ. |

**Không thêm SSH key cho tài khoản Azure AD**

Nếu đã lỡ bật OpenSSH của Windows, đừng bỏ key vào `C:\ProgramData\ssh\administrators_authorized_keys`. File này áp dụng cho mọi tài khoản admin, kể cả tài khoản Azure AD. Một lần đăng nhập bằng key là đủ làm máy treo SSH tới lần khởi động lại.

## Chuẩn bị

- Máy dev Windows 10 (22H2) hoặc Windows 11, có quyền admin để bấm UAC.
- Một tài khoản Tailscale. Đăng nhập **cùng một tài khoản** trên máy dev và điện thoại.
- Điện thoại cài **Tailscale** và **Termius** (hoặc app SSH khác).
- Claude Code và herdr đã cài trên Windows, dùng bình thường khi ngồi trước máy.

Trong hướng dẫn, thay các chỗ đánh dấu bằng giá trị của bạn:

| Chỗ đánh dấu | Ý nghĩa | Ví dụ |
| --- | --- | --- |
| `<ten-may>` | Tên ngắn cho máy, dùng làm tên trong Tailscale | `an-nguyen` |
| `<linux-user>` | User Linux bạn tạo trong Ubuntu | `an` |
| `<WINDOWS_USER>` | Tên thư mục của bạn trong `C:\Users\` | `AnNguyen` |

## Cài tự động bằng Claude

Cách nhanh nhất: mở Claude Code **trên chính máy cần cài** (terminal Windows, hoặc VS Code), dán prompt dưới đây, điền 3 dòng đầu. Claude tự làm các bước ở mục sau, kiểm tra kết quả từng bước, và chỉ dừng lại khi cần bạn thao tác.

**Bạn vẫn phải tự làm 3 việc**

- Bấm **Yes** ở hộp thoại UAC khi Claude báo (cập nhật WSL, tạo scheduled task).
- Mở link đăng nhập Tailscale mà Claude đưa ra, để thêm máy WSL vào tailnet.
- Trên điện thoại: cài Tailscale, Termius, và nhập thông tin host mà Claude báo ở cuối.

Prompt cho Claude Code

```
Hãy dựng lối vào SSH từ xa cho máy Windows này theo mô hình:
điện thoại → Tailscale → WSL Ubuntu (Tailscale SSH) → herdr.exe / claude.exe bản Windows.
Làm từng bước, in kết quả kiểm tra sau mỗi bước, dừng lại hỏi tôi khi cần tôi thao tác.

THÔNG TIN (để trống thì dùng mặc định trong ngoặc):
- Thư mục dự án mở sẵn khi đăng nhập: ______ (bắt buộc, hỏi tôi nếu trống)
- Tên máy trong Tailscale: ______ (tên máy Windows viết thường + "-wsl")
- User Linux: ______ (tên user Windows viết thường, bỏ ký tự đặc biệt)

QUY TẮC BẮT BUỘC:
1. KHÔNG bật hay cấu hình OpenSSH Server của Windows. KHÔNG thêm key vào
   C:\ProgramData\ssh\administrators_authorized_keys. Máy join Azure AD:
   đăng nhập SSH bằng key làm treo dịch vụ đăng nhập Windows tới khi khởi động lại,
   đăng nhập bằng mật khẩu thì bị từ chối (0xC004844C).
2. Lệnh cần quyền admin: chạy qua Start-Process -Verb RunAs, báo tôi trước để bấm UAC,
   gom nhiều thao tác admin vào một lần.
3. Không truyền script nhiều dòng qua `wsl -- bash -c "..."` từ PowerShell (hỏng quoting).
   Ghi script ra file với xuống dòng LF, rồi chạy: wsl -d Ubuntu -u root -- bash /mnt/c/...
4. Không đoán. Mỗi bước phải có lệnh kiểm tra và in kết quả.

CÁC BƯỚC:
A. Hiện trạng: wsl --version, wsl -l -v, tailscale status (Windows),
   Get-Command herdr, Get-Command claude -All.
   Tailscale trên Windows chưa cài hoặc chưa đăng nhập: dừng, nhờ tôi cài từ
   https://tailscale.com/download. herdr chưa cài: cài bằng
   powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"
B. WSL: wsl --update (admin). Chưa có distro Ubuntu: wsl --install -d Ubuntu --no-launch,
   rồi ubuntu.exe install --root. Gặp Wsl/Service/E_UNEXPECTED: wsl --shutdown,
   Restart-Service WslService, wsl --update, thử lại.
   Xác nhận: wsl -d Ubuntu -u root -- whoami  ==> root
C. Trong Ubuntu (root): useradd -m -s /bin/bash -G sudo <user>; passwd -l <user>;
   /etc/sudoers.d/<user> với "<user> ALL=(ALL) NOPASSWD:ALL" (chmod 440);
   /etc/wsl.conf gồm [boot] systemd=true và [user] default=<user>.
   wsl --terminate Ubuntu, rồi xác nhận systemctl is-system-running (running hoặc degraded).
D. Tailscale trong WSL:
   curl -fsSL https://tailscale.com/install.sh | sh
   systemctl enable --now tailscaled
   nohup tailscale up --ssh --hostname=<ten-may> > /tmp/ts-up.log 2>&1 &
   Đọc link https://login.tailscale.com/a/... trong /tmp/ts-up.log, đưa tôi mở,
   chờ tôi xác nhận. Kiểm tra: tailscale status thấy máy mới; ghi lại tailscale ip -4.
E. Giữ WSL chạy: Register-ScheduledTask tên 'WSL Ubuntu keepalive (Tailscale SSH)',
   trigger AtLogOn cho user hiện tại, action
   conhost.exe --headless wsl.exe -d Ubuntu --exec sleep infinity,
   ExecutionTimeLimit = 0, AllowStartIfOnBatteries. Rồi Start-ScheduledTask.
F. ~/.bashrc của user Linux: xoá khối cũ nếu có, thêm khối mở đầu bằng dòng
   "# --- Remote dev via Tailscale SSH ---" gồm:
   - Nếu không có cmd.exe trong PATH: nạp PATH Windows bằng
     $(cd /mnt/c && /mnt/c/Windows/System32/cmd.exe /d /c 'echo %PATH%' | tr -d '\r'),
     tách theo ';', wslpath -u từng mục, chỉ thêm thư mục tồn tại.
     (Phiên Tailscale SSH do systemd tạo, không qua wsl.exe nên thiếu PATH Windows.)
   - alias herdr='herdr.exe'
   - alias claude trỏ tới bản Windows: Claude cài bằng npm thì là
     /mnt/c/Users/<WinUser>/AppData/Roaming/npm/node_modules/@anthropic-ai/claude-code/bin/claude.exe
     (đọc trường "bin" trong package.json để xác nhận); có claude.exe trên PATH thì
     alias claude='claude.exe'.
   - Shell tương tác đang đứng ở $HOME thì cd tới thư mục dự án (dạng /mnt/d/...).
G. Kiểm tra cuối, chạy từ Windows qua đúng đường Tailscale SSH:
   ssh -tt -o BatchMode=yes -o StrictHostKeyChecking=no <user>@<ip-wsl> "bash -li <file-test>"
   File test (LF) bắt đầu bằng shopt -s expand_aliases, rồi in: PWD,
   cmd.exe /d /c whoami (phải là tài khoản Windows của tôi), command -v herdr.exe,
   claude --version, herdr status.
   Nếu ra "Tailscale SSH requires an additional check" kèm link: đưa link cho tôi mở, chạy lại.
H. Báo cáo: bảng từng bước và kết quả; thông tin nhập Termius (Hostname, Port 22,
   Username, không key và không mật khẩu). Nhắc tôi bật app Tailscale trên điện thoại
   cùng tài khoản, và chế độ "check" sẽ định kỳ hỏi mở link xác nhận.
```

Nếu Claude dừng giữa chừng hoặc gặp lỗi lạ, đối chiếu với các bước tay ở mục dưới và bảng [Xử lý sự cố](#su-co).

## Cài đặt từng bước

Các bước dưới đây là những gì prompt ở trên tự làm. Dùng khi muốn tự làm tay, hoặc để hiểu Claude đang làm gì.

1. Trên Windows và điện thoại

   ### Cài Tailscale và đăng nhập

   Tải Tailscale cho Windows tại [tailscale.com/download](https://tailscale.com/download), cài app Tailscale trên điện thoại, đăng nhập cùng một tài khoản ở cả hai nơi. Bật Tailscale trên điện thoại.
2. PowerShell trên Windows

   ### Cập nhật WSL và cài Ubuntu

   Cập nhật WSL trước. Bản cũ (ví dụ 2.3.x) có thể báo `Wsl/Service/E_UNEXPECTED` khi mở Ubuntu.

   PowerShell

   ```
   wsl --update
   wsl --install -d Ubuntu
   ```

   Ubuntu mở ra và hỏi tạo user. Nhập `<linux-user>` và một mật khẩu (dùng khi chạy `sudo`).
3. Trong Ubuntu

   ### Bật systemd

   Tailscale trong WSL cần systemd để tự chạy.

   bash · Ubuntu

   ```
   sudo tee /etc/wsl.conf >/dev/null <<'EOF'
   [boot]
   systemd=true

   [user]
   default=<linux-user>
   EOF
   ```

   Khởi động lại WSL từ PowerShell, rồi mở lại Ubuntu:

   PowerShell

   ```
   wsl --shutdown
   wsl -d Ubuntu
   ```
4. Trong Ubuntu

   ### Cài Tailscale trong WSL và bật Tailscale SSH

   WSL trên Windows 10 dùng mạng riêng (NAT), nên Tailscale phải chạy **bên trong** Ubuntu. WSL sẽ là một máy riêng trong tailnet, tên `<ten-may>-wsl`.

   bash · Ubuntu

   ```
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo systemctl enable --now tailscaled
   sudo tailscale up --ssh --hostname=<ten-may>-wsl
   ```

   Lệnh cuối in ra một link `https://login.tailscale.com/a/…`. Mở link, đăng nhập đúng tài khoản Tailscale ở bước 1. Kiểm tra:

   bash · Ubuntu

   ```
   tailscale status   # phải thấy <ten-may>-wsl, máy Windows và điện thoại
   ```
5. PowerShell trên Windows

   ### Giữ WSL chạy nền sau khi đăng nhập

   WSL tự tắt khi không có gì chạy, và khi đó máy `<ten-may>-wsl` biến khỏi tailnet. Tạo một scheduled task chạy ẩn mỗi khi bạn đăng nhập Windows:

   PowerShell

   ```
   $a = New-ScheduledTaskAction -Execute "$env:WINDIR\System32\conhost.exe" -Argument '--headless wsl.exe -d Ubuntu --exec sleep infinity'
   $t = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
   $s = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit ([TimeSpan]::Zero) -StartWhenAvailable
   Register-ScheduledTask -TaskName 'WSL Ubuntu keepalive (Tailscale SSH)' -Action $a -Trigger $t -Settings $s -Force
   Start-ScheduledTask -TaskName 'WSL Ubuntu keepalive (Tailscale SSH)'
   ```

   **Chỉ dùng được khi đã đăng nhập Windows**

   Máy vừa khởi động mà chưa ai đăng nhập thì WSL chưa chạy. Khóa màn hình (Win + L) thì vẫn dùng được.
6. Trong Ubuntu

   ### Nối herdr và Claude bản Windows vào shell

   Phiên Tailscale SSH không có biến PATH của Windows, nên gõ `herdr` sẽ báo `command not found`. Đoạn dưới nạp lại PATH của Windows mỗi lần đăng nhập, trỏ `herdr` và `claude` sang bản Windows, và mở sẵn thư mục dự án.

   bash · Ubuntu

   ```
   cat >> ~/.bashrc <<'EOF'

   # --- Remote dev via Tailscale SSH ---
   # Tailscale SSH sessions are spawned by systemd, not wsl.exe, so the Windows PATH is missing.
   if ! command -v cmd.exe >/dev/null 2>&1 && [ -x /mnt/c/Windows/System32/cmd.exe ]; then
     _winpath=$(cd /mnt/c && /mnt/c/Windows/System32/cmd.exe /d /c 'echo %PATH%' 2>/dev/null | tr -d '\r')
     IFS=';' read -ra _parts <<< "$_winpath"
     for _p in "${_parts[@]}"; do
       [ -n "$_p" ] && _u=$(wslpath -u "$_p" 2>/dev/null) && [ -d "$_u" ] && PATH="$PATH:$_u"
     done
     export PATH
     unset _winpath _parts _p _u
   fi
   # Windows herdr/claude: run as your Windows account, share its config and running sessions
   alias herdr='herdr.exe'
   alias claude='/mnt/c/Users/<WINDOWS_USER>/AppData/Roaming/npm/node_modules/@anthropic-ai/claude-code/bin/claude.exe'
   # Start in the main working folder on interactive login
   if [[ $- == *i* ]] && [[ "$PWD" == "$HOME" ]]; then cd /mnt/d/duong/dan/du-an; fi
   EOF
   ```
   - Thay `<WINDOWS_USER>` và đường dẫn dự án trong dòng `cd`. Ổ `D:\` trong WSL là `/mnt/d/`.
   - Đường dẫn `claude` ở trên dành cho Claude Code **cài bằng npm**. Nếu bạn cài bằng bộ cài riêng của Claude (có file `%USERPROFILE%\.local\bin\claude.exe`), đổi dòng đó thành `alias claude='claude.exe'`.
   - Không biết cài kiểu nào: mở PowerShell, gõ `Get-Command claude -All`.
7. Trang quản trị Tailscale, tùy chọn

   ### Bỏ bước xác nhận lại định kỳ

   Mặc định Tailscale SSH ở chế độ `check`: định kỳ (thường 12 giờ) Termius hiện "Complete additional check" kèm link, bạn mở link để xác nhận là vào được. Nếu tailnet chỉ có thiết bị của riêng bạn, có thể tắt bước này:

   Vào [login.tailscale.com/admin/acls](https://login.tailscale.com/admin/acls), trong khối `"ssh"` đổi `"action": "check"` thành `"action": "accept"`, rồi Save.

   **Tailnet dùng chung của công ty**

   Nếu team dùng chung một tailnet thì để admin quyết định chính sách này, và giữ `autogroup:self` để mỗi người chỉ vào được máy của mình.
8. Trên điện thoại

   ### Thêm host trong Termius

   | Trường | Giá trị |
   | --- | --- |
   | Hostname | `<ten-may>-wsl`, hoặc IP `100.x.x.x` lấy từ `tailscale ip -4` |
   | Port | `22` |
   | Username | `<linux-user>` |
   | Password / Key | Để trống. Tailscale xác thực theo thiết bị. |

   Kết nối. Nếu hiện "Complete additional check", chạm vào link và đăng nhập Tailscale.

   **Kiểm tra nhanh sau khi vào**

   Gõ `cmd.exe /c whoami`: kết quả phải là tài khoản Windows của bạn (ví dụ `azuread\annguyen`). Gõ `claude --version` và `herdr status` để chắc cả hai đã chạy được.

## Dùng hằng ngày

Termius → \<ten-may>-wsl

```
herdr        # mở herdr, hoặc vào lại phiên đang chạy
claude       # gõ trong một pane: herdr tự nhận ra agent
```

Xong việc, rời đi mà **vẫn giữ Claude chạy**: bấm Ctrl+b, thả tay, bấm q. Đóng hẳn Termius cũng được. Lần sau SSH vào, gõ `herdr` là thấy lại đúng trạng thái cũ.

Ngồi lại trước máy thì mở terminal Windows, gõ `herdr`: cùng một server, cùng các phiên.

## Herdr cơ bản

Herdr giống tmux nhưng làm riêng cho AI agent. Một server chạy nền giữ các terminal, nên ngắt kết nối không làm dừng Claude. Sidebar cho biết agent nào đang chạy, xong, hay đang chờ bạn trả lời.

### Khái niệm

| Khái niệm | Ý nghĩa |
| --- | --- |
| Workspace | Mỗi dự án hoặc mỗi việc một workspace. |
| Tab | Các màn hình khác nhau trong một workspace, ví dụ `agents`, `logs`. |
| Pane | Một terminal thật. Có thể chia đôi, vẫn chạy khi bạn ngắt kết nối. |
| Trạng thái agent | `working` đang chạy · `blocked` **đang chờ bạn** · `done` xong · `idle` rảnh |

### Phím tắt

Mọi lệnh bắt đầu bằng prefix Ctrl+b: bấm, **thả tay**, rồi bấm phím lệnh. Bảng dưới là phím mặc định của herdr 0.9.5.

| Việc | Phím |
| --- | --- |
| Xem toàn bộ phím tắt | Ctrl+brồi? |
| Rời đi, giữ phiên chạy | Ctrl+brồiq |
| Tab mới | Ctrl+brồic |
| Tab sau / trước | Ctrl+brồin / p |
| Nhảy tới tab 1–9 | Ctrl+brồi1…9 |
| Chia pane sang phải / xuống dưới | Ctrl+brồiv / - |
| Chuyển pane | Ctrl+brồiTab, hoặc h j k l |
| Phóng to / thu nhỏ pane | Ctrl+brồiz |
| Đóng pane | Ctrl+brồix |
| Chọn workspace | Ctrl+brồiw |
| Workspace mới | Ctrl+brồiShift+n |
| Ẩn / hiện sidebar | Ctrl+brồib |
| Nhảy tới agent đang có thông báo | Ctrl+brồio |

### Dán ảnh cho Claude

Khi ngồi trước máy Windows, Claude chạy trong herdr đọc thẳng clipboard Windows (cùng máy, cùng tài khoản), nên không cần cài thêm gì.

| Bước | Thao tác |
| --- | --- |
| Chụp hoặc copy ảnh | Win+Shift+S để chụp một vùng, hoặc copy ảnh từ trình duyệt, Teams… |
| Chọn pane đang chạy `claude` | Click vào pane đó |
| Dán | Alt+V. Ô nhập hiện `[Image #1]` là được. |

- Không dùng Ctrl+V: trên Windows phím này bị terminal giữ để dán chữ, clipboard chứa ảnh thì không ra gì.
- Cách thay thế: kéo thả file ảnh từ Explorer vào cửa sổ terminal (dán đường dẫn), hoặc gõ đường dẫn trong câu hỏi, ví dụ `xem ảnh D:\tmp\loi.png`.
- Trong herdr mà Alt+V không ra gì, trong khi ngoài herdr lại được: kiểm tra `%APPDATA%\herdr\config.toml` có phím tắt nào gán `alt+v` không.

**Từ điện thoại qua Termius thì không dán ảnh được**

SSH qua terminal chỉ truyền chữ. Tính năng chuyển ảnh của herdr chỉ có khi máy bên kia cũng cài herdr và kết nối bằng `herdr --remote`. Cần gửi ảnh từ điện thoại thì lưu ảnh vào thư mục đồng bộ (OneDrive…) trên máy dev rồi đưa đường dẫn cho Claude.

### Trên điện thoại

- Phím Ctrl nằm trên thanh phím phụ phía trên bàn phím Termius.
- Màn hình hẹp: hạn chế chia pane, nên để **mỗi agent một tab** và dùng z để phóng to pane đang xem.
- Ẩn sidebar cho rộng chỗ, mở lại khi cần xem agent nào đang `blocked`.

### Lệnh ngoài giao diện

bash

```
herdr status                      # server có đang chạy không
herdr agent list                  # các agent và trạng thái
herdr integration install claude  # tùy chọn: khôi phục phiên Claude
herdr server stop                 # tắt hẳn server, MẤT các phiên đang chạy
```

Cấu hình phím tắt và giao diện ở `%APPDATA%\herdr\config.toml`. Sửa xong chạy `herdr server reload-config`. Tài liệu đầy đủ: [herdr.dev/docs](https://herdr.dev/docs/).

## Làm việc trên ổ Windows

Code vẫn nằm trên ổ Windows và mở được từ WSL qua `/mnt/d/…`. File tạo hoặc sửa từ WSL vẫn thuộc tài khoản Windows của bạn. Một vài điểm cần nhớ:

| Việc | Nên làm |
| --- | --- |
| `npm`, `node`, `dotnet` | Dùng bản Windows: `npm.cmd`, `node.exe`, `dotnet.exe`, hoặc `cmd.exe /c npm run …`. Bản Linux sẽ hỏng với `node_modules` và `bin/obj` build trên Windows. |
| `git` | Dùng `git.exe`. `git` bản Linux có thể báo hàng loạt file thay đổi do quyền file và ký tự xuống dòng CRLF. |
| Tốc độ | Đọc/ghi ổ Windows từ Linux chậm hơn vài lần. Lệnh nặng (cài package, build) chạy bằng bản `.exe`. |
| Tên thư mục | Linux phân biệt chữ hoa/thường, Windows thì không. Gõ đúng chính tả. |

Thông thường bạn chỉ cần `herdr` và `claude`. Claude chạy bản Windows nên tự dùng đúng công cụ Windows.

## Xử lý sự cố

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `herdr: command not found` | Phiên SSH thiếu PATH Windows | Làm lại bước 6. Đăng xuất rồi SSH vào lại. |
| Termius đứng ở "Complete additional check" | Tailscale SSH đang ở chế độ `check` | Chạm vào link trong khung, đăng nhập Tailscale. Hoặc làm bước 7. |
| Termius không thấy `<ten-may>-wsl` | WSL không chạy, hoặc app Tailscale trên điện thoại đang tắt | Bật Tailscale trên điện thoại. Kiểm tra task ở bước 5 đang `Running`, máy đang đăng nhập Windows. |
| `Wsl/Service/E_UNEXPECTED` | WSL bản cũ | `wsl --update` (PowerShell admin), rồi `wsl --shutdown`. |
| `herdr status` báo `not running` | Server herdr đã tắt, thường sau khi khởi động lại máy | Gõ `herdr` để tạo server mới. Các phiên cũ không khôi phục được. |
| `Failed to start the systemd user session for 'root'` | Cảnh báo khi chạy `wsl -u root` | Không ảnh hưởng, bỏ qua. |
| Chương trình Windows báo "UNC paths are not supported" | Đang đứng trong thư mục Linux (`~`) | `cd` sang `/mnt/c` hoặc `/mnt/d/…` trước khi chạy `.exe`. |

## Bảo mật

- Tailscale SSH chỉ nhận kết nối từ thiết bị trong tailnet đã đăng nhập tài khoản của bạn. Không có cổng nào mở ra Internet hay mạng LAN.
- Không cần bật OpenSSH Server của Windows. Nếu đã bật, dừng nó (`Stop-Service sshd`, đặt Manual), hoặc giới hạn rule firewall cổng 22 chỉ cho dải Tailscale `100.64.0.0/10`.
- Ai vào được WSL là chạy được chương trình dưới tài khoản Windows của bạn. Bảo vệ tài khoản Tailscale bằng xác thực hai lớp, và gỡ thiết bị cũ khỏi tailnet khi đổi điện thoại.
- Mất điện thoại: vào [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines), xóa thiết bị đó ngay.

Viết dựa trên lần dựng thực tế trên Windows 10 22H2 (máy join Azure AD), WSL 3.0.3, Ubuntu 26.04, Tailscale 1.104, herdr 0.9.5, Claude Code 2.1. Phím tắt và lệnh có thể đổi theo phiên bản.