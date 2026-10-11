# Kế hoạch riêng: Reverse Orchestration (đánh giá + so sánh với local hiện tại)

Ngày: 2026-10-11 · Trạng thái: **plan đánh giá — CHƯA quyết định thực thi**

> File này tách riêng khỏi `REA-additions-plan.md`. Lý do: orchestration **thay đổi kiến
> trúc routing SSoT + behavior-chain** của local, không phải "lớp bổ sung reversible".
> Dưới đây là đề xuất + **so sánh cụ thể với local hiện tại** để bạn quyết.

## 0. Ý tưởng orchestration là gì

REA = 1 MCP duy nhất nối agent → nhiều engine (Hopper/Ghidra/IDA + static JS + JADX +
PTY). Agent hỏi **một chỗ**, REA tự chọn engine + chuẩn hóa output + kèm evidence/unknowns.

Orchestration trong local = lớp facade đứng trước các engine hiện có.

```
        Agent (1 lời hỏi)
             │
  ┌──────────▼───────────┐
  │   ORCHESTRATOR MCP   │   ← lớp đề xuất (mới)
  │  chọn engine          │
  │  chuẩn hóa output     │
  │  evidence + unknowns  │
  └──┬──────┬──────┬──────┘
     ▼      ▼      ▼
  IDA   Ghidra  JADX ... (engine/skill hiện có, giữ nguyên)
```

## 1. Cơ chế local HIỆN TẠI (so sánh nền)

### 1a. Routing SSoT: `skills/config/routing.json`
- Một câu hỏi/đoạn text → **một PRIMARY** (1 skill) theo scoring (meta dòng 6):
  keyword rules → candidate set → chọn `priority` cao nhất → PRIMARY; không khớp → fallback R0.
- **Chỉ sửa routing ở file này** (meta dòng 4), kèm `verify-routing-coherence.ps1`.
- Đây là **routing tĩnh theo chủ đề/keyword**, KHÔNG phải "chọn engine theo file/extension".

### 1b. Behavior-chain: `RULES.md` hot path
- Agent bắt buộc chạy `master-route` → PRIMARY → `case-init`/scope.md → ACT.
- Mỗi PRIMARY là 1 skill (VD `ida-reverse/`, `ghidra-reverse/`, `apk-reverse/`...) —
  skill tự lo tool/engine/port riêng.

### 1c. Engine/port hiện tại (từ RULES.md "MCP Service Management")
| Service | Port | Engine |
|---|---|---|
| idapro | 13337-13350 | IDA Pro |
| anything-analyzer | 23816 | browser + HTTP capture |
| jshookmcp | stdio | JS hook/CDP |
| ghidra | 8765 | Ghidra |
| burpsuite | 9876 | Burp |

### 1d. Điểm mạnh local hiện tại
- **Router chủ đề rõ ràng** (46 route R1-R46), tách chuyên sâu (pwn, EDR, APK, .NET...).
- **Kỷ luật SSoT** mạnh: 1 file routing, có verify/tests.
- **Scope/authorization gate** (case-init) chặt — phù hợp pentest/RE có trách nhiệm.
- Mỗi skill đã có tool-index + bootstrap riêng, test routing.

## 2. So sánh: Orchestration vs local hiện tại

| Tiêu chí | Local hiện tại (routing SSoT) | Đề xuất Orchestration |
|---|---|---|
| **Đầu vào routing** | text quét keyword → PRIMARY | file/URL/APK/extension + câu hỏi → engine |
| **Số interface agent học** | nhiều MCP/port/skill tùy target | 1 MCP facade |
| **Ai chọn engine** | agent + master-route | orchestrator lúc chạy |
| **Output chuẩn** | mỗi skill tự trả dạng riêng | schema chuẩn `{function,addr,disasm,dec,calls,refs,unknowns}` |
| **Evidence+unknowns** | chuẩn ops/evidence (per skill) | áp 1 lần cho mọi engine |
| **Fallback engine** | thủ công (RULES: IDA↔Ghidra↔r2) | tự động |
| **Scope/auth gate** | case-init, hard | CẦN giữ nguyên, không bỏ qua |
| **SSoT routing** | 1 file routing.json | rủi ro tạo **router thứ 2 song song** |

## 3. Mâu thuẫn / rủi ro với local (quan trọng nhất)

**M1 — Tạo hệ router thứ 2 (trái SSoT).**
- Orchestrator "chọn engine theo extension" = một router chạy song song với routing.json
  (vốn chọn theo keyword/chủ đề). Hai nguồn quyết định → không còn "single source of truth",
  phá vỡ kỷ luật mà repo đang có. `verify-routing-coherence.ps1` sẽ không bao phủ nó.

**M2 — Đối đầu behavior-chain, không bổ sung.**
- RULES.md bắt agent chạy `master-route → PRIMARY → case-init → ACT`. Orchestrator chèn
  "chọn engine theo file" ở trước — đổi thứ tự tri thức, có nguy cơ bỏ qua PRIMARY skill
  chuyên sâu (VD file .apk đến, orchestrator auto-JADX thay vì đi `apk-reverse/` có ca-depth
  longer hơn). Mâu thuẫn với "không phải lớp bổ sung reversible".

**M3 — Compatibility với scope gate.**
- Nếu orchestrator tự chạy engine/script trên file, phải đi qua case-init + scope.md
  trước. Nếu bỏ qua → vi phạm RULES hard gate. Orchestrator không được phép tự ý ACT.

**M4 — Chuẩn hóa output làm mất chi tiết skill riêng.**
- Canonical schema tốt cho consistency nhưng có thể làm phẳng thông tin chuyên sâu mà
  skill riêng trả (VD `native-instrumentation` cần lib+offset+VA, `dsl-vm-reverse` cần
  opcode table). 1 schema chung khó hàm chứa hết.

## 4. Phương án khả thi (chọn 1)

| Phương án | Mô tả | Mức xung đột |
|---|---|---|
| **A. Không làm orchestration** | Giữ routing SSoT; chỉ thêm P0-A/P1 trong `REA-additions-plan.md` | 0 |
| **B. Engine-fallback trong từng skill** | Mỗi skill tự có "nếu engine X thiếu → dùng Y cùng câu hỏi" (VD ida→ghidra→r2), không có lớp MCP chung | thấp — đúng tinh thần RULES hiện tại |
| **C. Orchestrator MCP riêng biệt** (đề xuất ban đầu) | 1 MCP facade, giữ bên cạnh routing.json, KHÔNG chạm routing SSoT; dùng cho workflow RE thuần, quyết định rõ khi nào mỗi route | trung bình |
| **D. Đổi SSoT sang engine-based** | Viết lại routing.json theo extension/file-based | **cao — phá 46 route hiện có** |

**Khuyến nghị:**
- Trước mắt: phương án **B** — gặt lợi "fallback engine" mà không động SSoT.
- Nếu muốn hẳn 1 interface: phương án **C** — minh bạch tách khỏi routing, ghi rõ biên
  giới. Tránh D (phá vỡ toàn bộ kiến trúc routing).

## 5. Nếu chọn C — phạm vi + điều kiện

- Là **tool/workflow riêng** trong `skills/reverse-orchestrator/`, KHÔNG ghi đè routing.json.
- Giữ hard gate: case-init/scope.md trước khi orchestrator ACT.
- Quy định rõ "khi nào đi orchestrator, khi nào đi skill chuyên sâu" (VD task .NET sâu vẫn
  qua `dotnet-reverse/`).
- Canonical schema là **superset có field mở rộng** (không làm phẳng skill riêng).
- Đăng ký như route kích hoạt thủ công, không phải PRIMARY tự động.

## 6. Kết luận

- Orchestration hấp dẫn (1 interface, chuẩn hóa, fallback) nhưng **mâu thuẫn cốt lõi
  với routing SSoT + behavior-chain local hiện tại** (M1, M2).
- Khuyến nghị hiện tại: **chưa làm orchestration**; ưu tiên phương án **B** (engine-fallback
  trong skill) + các mục ít mâu thuẫn ở `REA-additions-plan.md`.
- Nếu thực sự muốn 1 MCP thống nhất → phương án **C** với biên giới rõ, KHÔNG phải D.

## Bước kế tiếp (chờ quyết định)

- [ ] Xác nhận phương án: A / B / C (khuyến nghị B)
- [ ] Nếu B: liệt kê skill nào cần thêm fallback engine (ida→ghidra→r2, .NET→dnSpyEx...)
- [ ] Nếu C: soạn scope riêng, bảo toàn hard gate
