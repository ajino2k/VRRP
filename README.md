# VRRP

VRRP (Virtual Router Redundancy Protocol) là giao thức được mô tả trong RFC3768, nó cho phép sử dụng chung 1 địa chỉ IP gateway cho một nhóm router. Nếu router chính bị “tèo”, ngay lập tức các con router khác sẽ biết và với một số các nguyên tắc bầu chọn do VRRP quy định, các con còn lại sẽ chọn ra 1 con chính khác nắm giữ địa chỉ IP gateway đã được cấu hình từ trước, và lưu lượng từ người dùng sẽ đi qua con gateway mới này => Đảm bảo dịch vụ của người sử dụng thông suốt, không bị gián đoạn.

<p align="center"><img src="https://github.com/ajino2k/VRRP/blob/master/keepalived-vrrp-network1.png"></p>
 <p align="center"> hình 1: ảnh minh họa </p>

## 1. Một số khái niệm cần nhớ khi làm việc và trao đổi về VRRP

- VRRP Router: Router sử dụng VRRP. Có thể có 1 hay nhiều VRRP Router đồng thời.
- VRRP-ID: Định danh cho các router thuộc cùng 1 nhóm VRRP. Hiểu nôm na là 1 router có thể tham gia nhiều nhóm VRRP (các nhóm hoạt động động lập nhau), và VRRP-ID là tên gọi của từng nhóm.
- Primary IP: Địa chỉ IP thực của Router Interface tham gia vào VRRP. Các gói tin trao đổi giữa các VRRP Router sử dụng địa chỉ thực này.
- VRRP IP: Địa chỉ IP ảo của nhóm VRRP đó (Chính là địa chỉ dùng làm gateway cho các host). Các gói tin trao đổi, làm việc với host đều sử dụng địa chỉ ảo này.
- Virtual MAC: Địa chỉ MAC ảo đi kèm với địa chỉ IP ảo ở trên.  Định dạng của MAC ảo là 00-00-5E-00-02-XX (Với XX là VRID dưới dạng hexa). Các trao đổi về ARP với host đều sử dụng MAC ảo này.
- Virtual Router Master: Là một VRRP Router, gắn với 1 nhóm VRRP-ID cụ thể, và được bầu là Master của nhóm đó. Router này có nhiệm vụ nhận và xử lý các gói tin từ host đi lên.
- Virtual Router Backup: Là một VRRP Router, gắn với 1 nhóm VRRP-ID cụ thể, và đóng vai trò Backup cho con Master ở trên. Nếu con Master tèo, những con Backup này sẽ dựa vào 1 cơ chế bầu chọn và nhảy lên làm Master.
- Preempt Mode: Chế độ này được cấu hình trên mỗi VRRP Router. Khi được cấu hình là True, VRRP Router được quyền tham gia bầu chọn Master. Ngược lại, khi Preempt = False, VRRP sẽ không tham gia làm Master cho dù Priority là cao nhất.
Ok, cơ bản là mấy khái niệm trên. Còn lại chúng ta sẽ tìm hiểu khi nói sâu hơn về VRRP.

## 2. Các timer cho VRRP

- Advertisement Interval (s): Chu kỳ gửi gói tin Advertisement => Bộ định thời Advertisement Timer sẽ được kích hoạt mỗi khi router gửi/nhận một bản tin Advertisement.
- Skew time (s): Sử dụng để tính toán Master Down Interval theo priority.
- Skew_time = ((256 – priority) / 256)
- Master Down Interval (s): Khoảng thời gian để Backup Router nhận ra Master Router gặp sự cố. Hiểu nôm na là cứ 3 lần không nhận được bản tin Advertisement, Backup router sẽ kích hoạt Skew Timer. Skew timer chạy hết mà vẫn không nhận được Advertisement, nó sẽ coi Master bị tèo và bắt đầu quá trình bầu chọn lại Master.
- Master_down_interval = 3 * Advertisement_interval + Skew_time

## 3. Các trạng thái của một VRRP Router

- RFC3768 mô tả 3 trạng thái của VRRP Router: Initialize, Master và Backup.

- Initialize: Router đã được cấu hình VRRP nhưng chưa bật chức năng này lên. Do đó, router không có khả năng xử lý các gói tin VRRP. Khi người quản trị tạo một sự kiện Startup (dựa vào câu lệnh enable), VRRP sẽ chuyển sang trạng thái Backup.
- Backup: Khi ở trạng thái này, VRRP Router có nhiệm vụ chính là giám sát hoạt động của Master Router để phát hiện khi nào con master này gặp sự cố. Nếu gặp sự cố, dựa vào quá trình hoạt động của VRRP, nếu đủ điều kiện, con Backup Router này sẽ nhảy lên làm Master Router. Khi ở trạng thái Backup, router sẽ chỉ nhận các gói tin Advertisement từ Master Router chứ không tham gia vào việc làm gateway trung chuyển gói tin từ các host gửi đến VRRP.
- Master: Khi router ở trạng thái này, nó sẽ định kỳ gửi các gói tin Advertisement theo chu kỳ của Advertisement timer. Master Router sẽ đóng vài trò là một con router với MAC là virtual-MAC, IP là virtual-IP. Mọi gói tin gửi đến địa chỉ virtual-MAC hay virtual-IP đều được con Master Router này xử lý.

## 4. Mô tả hoạt động của VRRP

- Các VRRP Router trong cùng một VRRP Group tiến hành bầu chọn Master sử dụng giá trị priority đã cấu hình cho router đó. Priority có giá trị từ 0 đến 255. Nguyên tắc có bản: Priority cao nhất thì nó là Master, nếu priority bằng nhau thì IP cao hơn là Master.

- Nếu VRRP Router được cấu hình Virtual IP = Primary IP, nó sẽ có priority = 255 và trở thành Master luôn. Và tất nhiên, nếu VRRP Router được cấu hình priority = 255 thì mặc nhiên nó sẽ sử dụng Primary IP làm Virtual IP.

- Nếu VRRP Router có priority < 255 thì ban đầu nó sẽ chuyển về trạng thái Backup. Sau đó nó khởi động bộ định thời Master Down Timer. Khi bộ định thời Master Down Timer chạy hết, nó sẽ nhảy từ Backup lên làm Master.

Sau khi lên làm Master, các router bắt đầu nhận bản tin Advertisement từ router khác thuộc cùng VRRP Group cũng như gửi cho các router khác bản tin Advertisement chưa các tham số VRRP của nó. Router sẽ so sánh priority của mình với priority nhận được từ bản tin Advertisement, nếu cao hơn và router đang được cấu hình Preempt Mode = True, nó sẽ giữ nguyên trạng thái Master. Nếu không đạt đủ các điều kiện trên, nó sẽ nhảy xuống làm Backup.

- Trong trường hợp priority bằng nhau, nó sẽ so sánh địa chỉ IP của interface được cấu hình VRRP vơi src-IP của gói tin Advertisement, nếu cao hơn thì giữ nguyên trạng thái Master, thấp hơn thì xuống làm Backup.

Sau khi bầu chọn xong Master, thằng nào là Master Router sẽ gửi trả lời bản tin ARP Request cho host sử dụng Virtual MAC.

- Master Router định kỳ gửi các bản tin Advertisement cho tất cả Virtual Router Backup để thông báo về trạng thái hoạt động của mình. Backup dựa vào các bộ timer của mình để xác định Master có gặp sự cố hay không. Nếu có sự cố, các VRRP Router còn lại trong VRRP Group sẽ tiến hành bầu chọn lại VRRP Master Router theo thứ tự như trên.

- Nếu Master chết (gọi là Router A), xong Backup router khác lên thay (gọi là Router B), xong Router A sống lại…Nếu 
- Router A có Primary IP = Physical IP => Nó sẽ được khôi phục làm Master. Nếu không, nó sẽ giữ ở trạng thái Backup và khôi phục lại Priority cũ.



## VRRP vitual router redundancy protocol
HSRP	|VRRP
----|----
Chuẩn của cisco ,1994|	IETF,RFC 3768||
16 grousp Max	|255 groups Max||
1 active,1 standby,several candidates	|1 master, tất cả con Router còn several backups||
Tạo ra 1 IP  ảo,1 MAC  ảo	|Tạo ra 1 IP ảo,1 MAC ảo||
224.0.0.2 |	224.0.0.18||
Can track interface or object	|Can track only objects||
Default times: hello 3s ,hold  10s	|Hello : 1s hold time  3s||


### 1.Hoạt động vrrp:
## 1.1 Cách bầu chọn Router Master :
+ Dựa vào priority cao nhất : range 1-254
+ Nếu chỉ  số priority bằng nhau thì xét đến địa chỉ IP cao nhất mà cổng đang tham gia  vrrp
+ Tất cả router còn lại thì làm backup

## 1.2 Nhiệm vụ của Master :
+ Trả lời arp request
+ Forward dử liệu
+ Gửi gói hello
+ Trả lời Ip gateway
## 1.3 Nhiệm vụ của backup:
+ lắng nghe gói tin hello của Master.
+ Dựa vào master down để con backkup nào lên thay con master khi con master chết

### 2.Cấu hình VRRP

```yum install gcc  kernel-headers kernel-devel```
```yum install keepalived```

## 2.1 Cấu hình keepalived

```echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf```
```sysctl -p```

# VIP PMT MASTER

```! Configuration File for keepalived

global_defs {
   notification_email {
     cuongnp2@localhost
   
     }
   notification_email_from gt.monitor@localhost
   smtp_server 10.17.55.xxx # default port 25
   smtp_connect_timeout 30
}

    vrrp_script chk_nginx {
    script "/usr/local/nagios/libexec/check_tcp -H 127.0.0.1 -p 80"
    interval 2 # check every 2 seconds
    weight 2   # add 2 points of prio if OK
} 

vrrp_instance VI_1 {
    state MASTER
    smtp_alert 
    interface p2p1
    virtual_router_id 50
    priority 101      # 101 on master, 100 on backup
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        103.73.xxx.xx/29 p2p1
    }
    track_script {
        chk_nginx
    }
}

vrrp_instance VI_2 {
    state MASTER
    smtp_alert 
    interface p2p2
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.17.55.xxx/24 p2p2
    }
    track_script {
        chk_nginx
    }
}


# VIP PMT BACKUP

! Configuration File for keepalived

global_defs {
   notification_email {
     cuongnp2@localhost
   
     }
   notification_email_from gt.monitor@localhost
   smtp_server 10.17.55.xxx
   smtp_connect_timeout 30
}

    vrrp_script chk_nginx {
    script "/usr/local/nagios/libexec/check_tcp -H 127.0.0.1 -p 80"
    interval 2
    weight 2
} 

vrrp_instance VI_1 {
    state BACKUP
    smtp_alert 
    interface p2p1
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        103.73.xxx.xxx/29 p2p1
    }
    track_script {
        chk_nginx
    }
}

vrrp_instance VI_2 {
    state BACKUP
    smtp_alert 
    interface p2p2
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.17.55.xxx/24 p2p2
    }
    track_script {
        chk_nginx
    }
}
