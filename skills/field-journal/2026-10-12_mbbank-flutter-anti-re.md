# MB Bank (com.mbbank / com.mobile) Flutter anti-RE note

> Bằng chứng thực tế từ `libapp.so` (95 MB) + `libflutter.so` (11 MB) của MB Bank, phân tích bằng Blutter + pyelftools 2026-10-12.
> Kết quả: Blutter `Invalid vm isolate snapshot seen`.

## Tóm tắt

MB Bank dùng Flutter **AOT arm64**, nhưng `libapp.so` được **pack/protect (MBCP — Mobile Bank Cyber Protection)** khiến bộ phân tích tĩnh Dart AOT (Blutter) không đọc được snapshot. Đây là chống-RE chủ đạo ở tầng Dart.

## Cơ chế / dấu hiệu (bằng chứng)

1. **Symbol giả `_kDartVmSnapshotData`**
   - Symbol tồn tại đúng tên nhưng `st_value = 832`, `st_size = 16176` (quá nhỏ so với snapshot thật).
   - Bytes tại `st_value+20` là nhị phân ngẫu nhiên (`0xf4...`), **không phải chuỗi ASCII** → Blutter `UnicodeDecodeError`.
   - Mục đích: đánh lừa tool mong đợi symbol chuẩn, đẩy sang vùng data nhiễu.

2. **Snapshot hash / version bị xóa**
   - Quét toàn bộ file: **không có** 40-hex snapshot hash nào (thường `[a-f0-9]{40}` trong `.rodata` của snapshot hợp lệ).
   - Blutter cần hash để map đúng commit Dart VM → không có hash = không build/sử dụng đúng bản.

3. **`_kDartVmIsolateSnapshotData` bị xóa**
   - Symbol isolate snapshot không tồn tại → layout snapshot không theo chuẩn.

4. **`libflutter.so` vẫn "sạch" về engine-id**
   - Vẫn còn 2 engine id + chuỗi `3.11.3 (stable)` trong `.rodata` → version detect được (Dart 3.11.3), nhưng snapshot bên `libapp.so` đã bị mã hóa nên vô dụng.

5. **Runtime check khi chạy phân tích**
   - Blutter build xong `blutter_dartvm3.11.3_android_arm64.exe` rồi phân tích, nhận `exception: Invalid vm isolate snapshot seen`.
   - Chứng tỏ isolate snapshot bị mã hóa/khác layout, không chỉ là thiếu symbol.

## Hàm ý cho hướng tiếp cận

- **Tĩnh (Blutter/static) không đi tiếp được** với file gốc MB Bank → phải **unpack / dump snapshot đã giải mã ở runtime**:
  - Frida hook `libflutter.so` / engine để bắt vùng snapshot sau khi giải mã (`_kDartVmSnapshotData` thật được dựng runtime).
  - Hoặc dùng DeeDump / similar runtime snapshot dump tool cho Flutter AOT.
- Nếu tìm được bản API/libapp **không bảo vệ** (version cũ, apk không MBCP) thì Blutter chạy ngay.
- Không phải lỗi công cụ: Blutter build + chạy thành công; file bị bảo vệ.

## Artefacts

- `D:\Tools\blutter\bin\blutter_dartvm3.11.3_android_arm64.exe` (build xong)
- Input tạm: `%TEMP%\blutter-mb\libapp.so` + `libflutter.so` (tách từ `com.mbbank.biz.prod-1.0.213-base.apk`)
