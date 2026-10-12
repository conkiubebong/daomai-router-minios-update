# DaoMaiOS Router — Hướng dẫn cài đặt & sử dụng

DaoMaiOS Router là hệ điều hành router live (chạy trực tiếp từ USB, nạp vào RAM)
dựa trên Debian bookworm, dùng để biến một máy tính/mini PC thường thành router
NAT/proxy nhiều WAN, quản lý qua giao diện web.

Repo này (`daomai-router-minios-update`) là **kênh cập nhật** — nơi mọi bản ISO
và bản cập nhật được phát hành (xem [Releases](../../releases)). Repo chứa mã
nguồn nằm ở [daomai-router-minios](https://github.com/conkiubebong/daomai-router-minios)
(có thể riêng tư), còn repo này luôn công khai để router đang chạy ngoài thực tế
có thể tự kiểm tra và cài bản mới.

## 1. Cài đặt

### Cần chuẩn bị

- Một máy tính/mini PC x86_64 (không cần ổ cứng cài hệ điều hành riêng —
  DaoMaiOS chạy live thẳng từ USB, nạp toàn bộ vào RAM khi boot).
- Ít nhất 2 cổng mạng: một cổng ra Internet (WAN — DHCP nhà mạng, IP tĩnh, hoặc
  PPPoE), một cổng cắm switch/AP cho các thiết bị dùng chung (LAN). Có thể dùng
  nhiều cổng WAN hơn nếu cần nhiều đường mạng PPPoE hoặc load-balancing.
- 1 USB (khuyến nghị 4GB trở lên) để ghi ISO.
- Công cụ ghi ISO ra USB: [Rufus](https://rufus.ie) (Windows) hoặc
  [balenaEtcher](https://etcher.balena.io) (Windows/macOS/Linux), hoặc lệnh
  `dd` trên Linux/macOS.

### Ghi ISO ra USB

1. Tải file `daomaios_<version>.iso` mới nhất ở [Releases](../../releases).
2. Dùng Rufus/balenaEtcher chọn file ISO này và ghi vào USB (chế độ ghi ảnh
   đĩa/DD image — không phải copy file thường).
3. Cắm USB vào máy sẽ dùng làm router, vào BIOS/UEFI chọn boot từ USB đó.

ISO tự boot thẳng vào router, không có menu chọn (direct auto-boot).

### Sau khi boot lần đầu

1. Cắm dây mạng WAN (ra Internet) vào một cổng, cắm switch/AP cho thiết bị
   dùng vào cổng còn lại.
2. **Không rút USB ra** — agent tự tạo một phân vùng `DAOMAI-PERSIST` ngay trên
   USB đó để lưu database (`router.db`) bền vững qua các lần khởi động lại; rút
   USB ra sẽ mất khả năng lưu cấu hình lâu dài (xem mục [Dữ liệu khi chạy
   RAM](#5-dữ-liệu-khi-chạy-ram)).
3. Từ một máy khác trong cùng mạng LAN, mở trình duyệt vào:
   ```text
   http://20.20.0.1:3
   ```
   (`20.20.0.1` là gateway LAN mặc định; nếu đã đổi thì dùng IP LAN hiện tại
   của router). Đăng nhập bằng tài khoản mặc định:
   ```text
   Tài khoản: root
   Mật khẩu:  daomai
   ```
   Đổi mật khẩu ngay sau lần đăng nhập đầu: nút **Đổi mật khẩu** trên thanh
   trên cùng, chọn **Mật khẩu admin** (dùng cho Web Admin và SSH, là mật khẩu
   root Linux thật).
4. Vào tab **Internet**, gán vai trò cho từng cổng mạng vật lý: cổng nào là
   WAN (DHCP/IP tĩnh/PPPoE), cổng nào là LAN (`br-phone`, để thiết bị lấy DHCP
   từ router). DaoMaiOS không tự đoán vai trò cổng — phải chọn tay lần đầu.
5. SSH chạy ở **cổng 2** và **mặc định tắt** (đóng cả trong LAN). Muốn dùng
   thì vào tab **Internet → Web từ xa**, tick **Bật SSH** (không chọn WAN = chỉ
   dùng trong LAN), rồi:
   ```bash
   ssh -p 2 root@20.20.0.1
   ```

## 2. Hai giao diện quản lý

DaoMaiOS có hai giao diện dùng cho hai đối tượng khác nhau:

| Giao diện | Cổng | Dùng cho | Cần đăng nhập |
|---|---|---|---|
| **Web Admin** | `3` | Người quản trị router | Có (tài khoản root) |
| **Mobile Self-Service** | `5` (gõ thẳng `http://20.20.0.1` là vào, cổng 80 trên IP router tự chuyển về đây) | Người dùng thiết bị tự chỉnh proxy/thông tin của chính mình | Không — nhận diện theo IP/MAC thiết bị |

Các cổng khác của router:

| Dịch vụ | Cổng | Mặc định |
|---|---|---|
| SSH | `2` | Tắt |
| API công khai (link "API Xoay", API cho tool) | `4` | Tắt |
| Trang quảng cáo (mode brand) | `6` | — |

Bật/tắt SSH và API công khai, hoặc mở chúng (và Web Admin) ra Internet, ở tab
**Internet → Web từ xa**.

> **Đổi cổng từ bản 0.4.174.** Trước đó: Web Admin `18080`, API công khai
> `18081`, SSH `22`. Cập nhật lên bản mới router tự chuyển cổng. Từ 0.4.175
> cổng `18080` và `18081` **không còn được dùng**: bookmark, tool, link "API
> Xoay" phải đổi sang cổng `3` (web) và `4` (API). Cổng ra WAN đã lưu là
> `18080`/`18081` được tự đổi thành `3`/`4`; cổng ngoài khác (kể cả SSH `22`)
> được giữ nguyên. Từ 0.4.176 trang mobile chuyển `18082` → `5` và trang brand
> `18083` → `6`; điện thoại vẫn chỉ cần gõ `http://20.20.0.1`.

Với Mobile Self-Service, chỉ cần mở trình duyệt trên điện thoại/máy đang nối
vào LAN router và gõ thẳng `http://20.20.0.1` — không cần gõ số cổng, không cần
tài khoản.

## 3. Tính năng Web Admin

### Dashboard
- Băng thông tải xuống/tải lên tổng và theo từng egress (biểu đồ realtime).
- Số client online/tổng, số egress, số proxy.
- CPU, RAM, dung lượng ổ đĩa (đồng hồ tròn %), tốc độ đọc/ghi ổ đĩa.
- Đếm nhanh: số interface, egress, phiên PPPoE, proxy, nhóm, client, NAT port,
  luật bypass.

### Internet
- **Chính sách cho thiết bị mới**: bật/tắt tự cấp Internet, chọn egress mặc
  định, giới hạn băng thông mặc định cho thiết bị vừa kết nối.
- **DHCP**: dải IP cấp phát, gateway/CIDR mạng LAN, DNS, thời gian thuê IP,
  timeout tự xoá IP tĩnh không hoạt động (kèm nút xoá ngay).
- **Web từ xa** (Web Admin cổng 3, API công khai cổng 4, SSH cổng 2):
  - Web Admin: trong LAN luôn mở; chọn đầu ra WAN để mở thêm ra Internet.
  - SSH và API công khai: mặc định tắt. Bỏ tick = đóng hẳn (cả LAN). Tick +
    không chọn WAN = chỉ trong LAN. Tick + chọn WAN = mở thêm ra Internet.
  - Mở ra WAN thì chỉ đúng cổng ngoài đã đặt dùng được; đổi cổng thì cổng cũ
    đóng ngay.
- **Nhóm cân bằng tải (LB Group)**: gộp nhiều egress WAN thành một nhóm, gán
  cho client để tự chuyển đường khi một egress rớt.
- **DNS Proxy**: định danh một "đích ra" (theo egress hoặc giữ nội bộ —
  `No_Nat`) kèm tài khoản, DDNS hostname, URL cập nhật DDNS, nút cập nhật IP
  ngay.
- **Từng cổng mạng vật lý**: trạng thái online/offline, bật/tắt PPPoE mà không
  xoá tài khoản, nút **"Xoay IP"** (đổi IP công khai bằng cách quay lại pppd),
  tốc độ cổng, VLAN, và nếu có proxy host trên egress đó thì hiện sẵn dòng
  HTTP/SOCKS5/API rotate để copy.
- Nút dò lại toàn bộ cổng mạng, nút xuất danh sách proxy đang host trên PPPoE
  ra file .txt (HTTP/SOCKS5/rotate link).

### Clients
- Bảng quản lý toàn bộ thiết bị: MAC, tên, IP (tĩnh/động), nhóm, proxy, loại
  proxy, vùng (region), chặn WebRTC, chặn scan cổng, CPU/RAM/uptime của tiến
  trình proxy, giới hạn băng thông, NAT port, egress/internet, chế độ, trạng
  thái online.
- **Ẩn cột** và **ẩn nhóm** để gọn bảng khi có nhiều thiết bị (lưu lựa chọn ở
  trình duyệt).
- Sửa trực tiếp từng ô, bộ lọc theo tên/nhóm/egress/proxy/trạng thái/chế
  độ/băng thông...
- Chọn nhiều dòng (kéo chọn hoặc chuột phải) để sửa hàng loạt: nhóm, egress,
  proxy (đơn hoặc dùng chung một upstream), loại proxy, băng thông, NAT port,
  WebRTC/chặn scan, chế độ.
- **Xoá nhiều thiết bị** một lần: xoá luôn mọi thứ gắn với thiết bị đó (luật
  firewall, giới hạn băng thông, IP đặt trước và lease DHCP, NAT port, proxy
  riêng, kết nối đang mở), không chỉ xoá dòng trong bảng.
- Proxy IPv6: router gửi **tên miền** lên proxy (thay vì IP), nên proxy chỉ ra
  IPv6 vẫn chạy được với các trang có IPv6.
- Luật bypass domain/IP theo từng client hoặc áp dụng toàn hệ thống.
- Cảnh báo khi nhiều client đang dùng trùng một proxy.
- Tự làm mới danh sách mỗi vài giây, báo (toast) khi có thiết bị mới hoặc đổi
  trạng thái online/offline.

- **Proxy type "auto"** (mặc định cho máy mới): không cần biết proxy là loại
  gì, router tự dò **socks5 → http → socks4 → https**, mỗi proxy chỉ một kết
  nối thử mỗi lúc, dừng ngay khi nhận ra. Đã dò xong thì ô loại hiện
  "auto (socks5)". Khách chọn loại cụ thể thì router dùng đúng loại đó. Đổi
  loại mặc định cho máy mới ở tab **Internet → IP mới**.

### Proxy Pool
- Kho proxy dùng chung cho chế độ **proxy auto**: dán danh sách proxy (mọi
  định dạng phân tách phổ biến), hệ thống tự tách host/port/user/pass, bỏ
  trùng.
- Giới hạn số thiết bị dùng chung một proxy trong pool, tự chuyển thiết bị
  sang proxy khác còn sống nếu proxy hiện tại chết quá N giây.
- Danh sách proxy trong pool: trạng thái sống/chết, vùng, thiết bị nào đang
  dùng, lọc theo loại/trạng thái/vùng, xoá hàng loạt.
- Tự kiểm tra sức khoẻ proxy định kỳ.

### Logs
- Lịch sử 200 lệnh gần nhất agent đã thực thi (thời gian, loại lệnh, mã trả
  về, có phải dry-run hay không).
- Nút kiểm tra nhanh sức khoẻ: interface, egress, proxy, client.

### Backup
- Tạo bản sao lưu database ngay lập tức (.zip).
- Tải lên bản sao lưu cũ để khôi phục, tải xuống bản sao lưu đã lưu.

### Info
- Phiên bản hiện tại, hostname, uptime, repo cập nhật đang dùng.
- Nút khởi động lại (agent tự đồng bộ DB trước khi tắt).
- Xem [mục Cập nhật](#4-cập-nhật-hệ-thống) bên dưới.

### API Docs
- Trang Swagger UI tự sinh từ API thật của agent, có nút "Try it out" dùng
  luôn token đã đăng nhập — tiện để dò/test API mà không cần công cụ ngoài.

## Tính năng Mobile Self-Service

Không cần đăng nhập — nhận diện theo IP/MAC của thiết bị đang truy cập.

- Xem thông tin của chính thiết bị: IP công khai, MAC, tên, nhóm, vùng proxy.
- Thiết bị đang bị chặn hoặc ở mode brand cần **mật khẩu web mobile** để mở
  khoá. Admin đặt mật khẩu này ở nút **Đổi mật khẩu → Mật khẩu web mobile**
  (khác mật khẩu admin, không vào được trang admin). Chưa đặt thì tạm dùng mật
  khẩu admin.
- Sửa **dòng proxy** của chính mình (host:port hoặc host:port:user:pass), chọn
  loại proxy (**auto** = router tự dò, hoặc http/https/socks4/socks5), chọn ký
  tự phân tách tuỳ ý.
- Nếu ở chế độ **proxy auto**: ô proxy tự khoá (không sửa tay được), thay vào
  đó có **nút xoay proxy** — bấm để đổi sang proxy khác trong pool, tuân theo
  đúng luật giới hạn số thiết bị/proxy như trên Web Admin (dùng chung một cơ
  chế với admin).
- Chọn chế độ (direct/proxy/mixed/blocked/proxy auto), egress/internet, nhóm,
  chặn scan cổng, giới hạn băng thông — trong mục "Tuỳ chọn khác" có thể thu
  gọn.
- Xem CPU/RAM/uptime của proxy đang chạy cho thiết bị mình (nếu có), xem danh
  sách NAT port đã được admin cấu hình cho thiết bị (chỉ xem, không sửa).
- Nút xoá proxy đang gán.
- Chuyển ngôn ngữ Việt/Anh, tự lưu theo trình duyệt điện thoại.
- Nếu thiết bị chưa có trong hệ thống, hiển thị thông báo liên hệ admin thay
  vì form chỉnh sửa.

## 4. Cập nhật hệ thống

Tab **Info** trong Web Admin có hai kiểu cập nhật, cả hai đều lấy từ cùng một
GitHub Release trên repo này:

- **Update zip** — cập nhật nóng, không cần khởi động lại: tải file `.zip`,
  backup, thay `daomai-agent` + giao diện web, kiểm tra agent và web khớp
  nhau, rồi **lưu luôn bản cài xuống USB/SSD**. Khi khởi động lại, router tự
  lắp lại bản này (nếu nó mới hơn bản trong ISO), nên cập nhật zip **sống sót
  qua reboot**. Lỗi ở bất kỳ bước nào thì tự quay về bản cũ. Cũng có thể tải
  lên file `.zip` từ máy (nút cài từ file).
- **Update ISO** — ghi đè cả ổ boot bằng `daomaios_<version>.iso` mới, kiểm
  tra checksum SHA256 trước khi ghi, backup rồi khôi phục lại database và
  phân vùng persist, sau đó tự khởi động lại. Cách này **sống sót qua reboot**
  nhưng rủi ro hơn nếu mất điện giữa lúc ghi đĩa — không rút điện router trong
  lúc cập nhật ISO.

Cả hai kiểu cập nhật đều không đụng tới database người dùng
(`/var/lib/daomai/router.db`).

Từ 0.4.174, giao diện web (trang admin, trang mobile, trang brand, ảnh) nằm
ngay trong file `daomai-agent`, nên chương trình và giao diện luôn khớp nhau.

## 5. Dữ liệu khi chạy RAM

DaoMaiOS chạy live `toram`: hệ điều hành nạp hết vào RAM khi boot, không có
overlay persistence toàn hệ thống. Chỉ riêng database router được agent tự
lưu định kỳ xuống USB/SSD, tại phân vùng `DAOMAI-PERSIST` (tự tạo ngay trên
USB đã boot, không cần thao tác gì thêm). Khi khởi động lại, agent tự khôi
phục database từ phân vùng này vào RAM trước khi chạy — vì vậy cấu hình, danh
sách client, proxy... không mất khi tắt/mở lại máy, miễn là USB vẫn còn cắm.

---

## Phụ lục — kỹ thuật cho ai quản lý bản phát hành

Mỗi [Release](../../releases) ở đây mang hai đường cập nhật độc lập cho cùng
một phiên bản:

- **Agent/web zip** — asset `.zip` chứa:
  ```text
  daomai-agent      # binary linux/amd64 (đã kèm sẵn web UI bên trong)
  web/
    index.html      # bản web trên đĩa: router chạy bản cũ (< 0.4.174) cần
                    # thư mục này để cài; bản mới dùng để tự kiểm tra
  ```
  Tên asset phải là `daomai-router-minios-update-v<version>.zip`.
  Không cần file `VERSION` trong zip — phiên bản lấy từ tag của Release (ví dụ
  `v0.2.0`), agent tự ghi vào `/etc/daomai/VERSION` sau khi áp dụng.

- **Full ISO** — asset gồm:
  - `daomaios_<version>.iso` — đúng tên như script build ở repo nguồn.
  - `daomaios_<version>.iso.sha256` (hoặc `.sha256`) — checksum bắt buộc phải
    có kèm, agent từ chối ghi ISO xuống đĩa nếu thiếu asset này hoặc checksum
    không khớp.

`update.json` ở repo này chỉ là con trỏ tham khảo dạng máy đọc được (mirror
của GitHub Release) — bản thân `daomai-agent` đọc trực tiếp GitHub Releases
API, không đọc file này. `CHANGELOG.md` ghi lại thay đổi từng phiên bản dạng
người đọc được.
