# Cache trong moodle 
---
## Tổng quan
3 loại:
- Application cache: Sử dựng phổ biến trong code. Thông tin chủa sẻ giữa tất cả user, giữa các request. Infor lưu trữ vì 1 - 2 lý do.
  - thông tin cân thiết cho các reqset và tiết kiệm tương tác csdl
  - thông tin ít sử dụng nhưng cần nhiều tài nguyên để tạo => thông tin được lưu trong thư mục
- Session cache: Cơ chế cache session mặc định của PHP, sử dụng file mặc đinh thay vì database => cải thiện hiểu năng khi tải db cao
- Request cache: Thông tin lưu trong cache tồn tài trong vòng đời request.

## Các loại cache và multi server

Nếu có nhiều server => App cache cần được chia sẻ giữa các node. Tương tự Session Cache, cần chia sẻ giữa các node trừ khi sử dụng stick session, user luôn truy cập 1 node.

## Cache backends
Nơi dữ liệu được lưu bao gồm: file system, php session, memcache, memory. Mặc định cả file sys, php session, memory được sử dụng bởi moodle.

## Cache stores
- Các loại plugin hỗ trợ trong moodle. Cách để moodle kết nối với các cache backend mở rộng. Mặc định Moodle có thể sử dụng Memcache, Memcached, MongiDB, APCu, Redis.

Mặc định:
- file store: tạo, sử dụng bởi tất cả app cache, lưu trong thư mục moodledata 
- session store: tạo sử dụng bởi tất cả session cache. Lưu trong PHP session, lưu trong db
- static memory lưu instance tạo, sử dụng bởi tất cả request cache type. Data tồn tại trong memory, trong vòng đồi request.


## Cách các thành phần làm việc

Mô tả vài trò nguời quản trị trong tổ chức:
- SA khởi tạo Memcache, XCache, APC v.v cho moodle sử dụng
- Moodle Admin tạo cache store trong moodle, map tới các thành phần
- Dev sử dụng cache, ko quan tâm tới nơi lưu cache.
- Moodle Admin map cache store - cache backend