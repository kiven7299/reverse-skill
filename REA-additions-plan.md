# Kế hoạch: bổ sung từ REA (morluto/rea) vào local repo reverse-skills

Ngày: 2026-10-11 · Trạng thái: **bản tổng hợp đầy đủ** (đã chốt hướng)

> Chiều áp dụng: **REA → local repo** (local là repo mình sở hữu).
> Bổ sung những methodology/tool REA có mà reverse-skills chưa có, theo ưu tiên P0→P2.

## 0. Tóm tắt REA là gì

REA = 1 MCP duy nhất nối agent → nhiều engine (Hopper/Ghidra/IDA + static JS + JADX +
PTY). Điểm khác biệt cốt lõi:
- **Một interface cho mọi target** (orchestration).
- Mỗi finding kèm **evidence + limitations/unknowns**.
- **Understand → rebuild**: disasm → C/C++ → recompile → verify byte/behavior.

Bảng "What you can analyze" (9 hàng): native binaries, JS/Electron, websites,
network captures (HAR), .NET, Android APK, firmware, packages/resources, process
behavior (PTY).

Những phần dưới là cái **REA có mà local repo đang thiếu → nên bổ sung**.

---

## P0-1. Lớp Reverse Orchestration (1 interface cho mọi engine)

**Vấn đề local hiện tại:** nhiều engine phân mảnh, mỗi cái 1 MCP/port/interface:
`ida-reverse/` (port 13337-13350), `ghidra-reverse/` (8765), `binary-ninja-reverse/`,
`apk-reverse/` (jadx), `dotnet-reverse/` (dnSpyEx), `radare2/` (r2). Agent phải biết
trước target → tự chọn MCP + port + skill.

**Giải pháp:** lớp facade/orchestrator đứng trước các engine, agent hỏi 1 chỗ.

```
        Agent (1 lời hỏi)
             │
  ┌──────────▼───────────┐
  │   ORCHESTRATOR MCP   │   ← lớp mới
  │  chọn engine          │
  │  chuẩn hóa output     │
  │  evidence + unknowns  │
  └──┬──────┬──────┬──────┘
     ▼      ▼      ▼
  IDA   Ghidra  JADX ... (engine/skill hiện có, GIỮ NGUYÊN)
```

**4 chức năng:**
1. **Engine routing theo target** — file/URL/APK/câu hỏi → chọn IDA/Ghidra/JADX/dnSpyEx/
   static-JS/r2; tái dùng `routing.json` làm runtime selector (routing tĩnh → runtime).
2. **Canonical output** — chuẩn hóa pseudocode từ mọi engine thành 1 schema:
   `{ function, address, disasm, decompiled, calls, refs, unknowns }`; agent nhận 1
   định dạng ổn định, giảm context, lập luận nhất quán.
3. **Evidence + unknowns đồng nhất** — chuẩn `ops/evidence-finding-path.md` áp một lần
   cho mọi engine; mỗi finding kèm unknowns → agent biết hỏi bước tiếp.
4. **Fallback/provider thay thế** — engine A fail/thiếu license → tự fallback
   Ghidra/r2 cùng câu hỏi, cùng schema (đúng tinh thần RULES "IDA↔Ghidra↔r2").

**Sản phẩm trong repo:**
| Thành phần | Mô tả | Loại |
|---|---|---|
| `skills/reverse-orchestrator/SKILL.md` | mô tả lớp orchestration, chọn engine | skill |
| Adapter MCP (facade) | 1 MCP gom → engine backends, trả schema chuẩn | code/server |
| Schema chuẩn hóa | `{function, addr, disasm, dec, calls, refs, unknowns}` | spec (ops/) |
| Runtime router | đọc `routing.json` + target → engine selector | code/script |
| Guide "bring your engine" | gắn engine mới (Hopper, BInja…) vào orchestrator | doc |

**Không làm (ngoài phạm vi):** không thay skill chuyên sâu (`pwn-chain`, `edr-bypass-re`
vẫn cần); không khóa vào 1 engine — mở cho nhiều engine qua adapter.

---

## P0-2. Skill mới: Code Reconstruction + Verify (disasm → C/C++ → byte-match)

**Gap:** local repo thiên "đọc hiểu + khai thác"; REA showcase làm hẳn **tái dựng ra
source xác minh được** — DX-Ball (3205 case x86 → 63 hàm C, tái dựng đúng byte),
TH04 DOS (16-bit → C++ so sánh với compiler lịch sử), Notion clipboard.

**Methodology mới nên thêm (`skills/code-reconstruction-verify/`):**

```
disasm (IDA/Ghidra/r2) → pseudocode → viết C/C++ → recompile (cùng compiler/opt-level)
   → diff byte/behavior với bản gốc → verify (thông qua) / chỉnh (fail)
   → loop đến khi match
```

**Các bước:**
1. Chọn hàm đích, lấy disasm + pseudocode từ engine qua orchestrator.
2. Viết lại bằng C/C++ ưu tiên tường minh (không dựa hằng số ma thuật chưa rõ).
3. Recompile cùng compiler/opt-level (REA highlight: phải khớp để byte-match).
4. So sánh byte output / chạy case set (VD 3205 case x86) để xác minh hành vi.
5. Ghi kết quả vào evidence; làm rõ unknowns nếu lệch.

**Cái hay:** biến "hiểu được code" thành "tái dựng được, chạy đúng, byte gần khớp" —
đỉnh cao của RE khả chứng minh. Khác với "đọc hiểu" của các skill hiện có.

---

## P1-1. Field chuẩn "unknowns / confidence-limits" vào evidence format

**Gap:** local có Evidence→Finding→Path (`ops/evidence-finding-path.md`) nhưng chưa
ép mỗi finding kèm **limitations / unknowns** trả về cùng lúc như REA.

**Bổ sung:** thêm field bắt buộc chuẩn vào format finding:
- `unknowns` — những gì chưa xác định (địa chỉ, phần branch, hằng số).
- `confidence` — mức chắc chắn (high/medium/low) + lý do.
- `limitations` — giới hạn phương pháp (static thiếu runtime, bỏ sót đường nào).

Agent nhận được "bài toán còn dang dở" ngay, biết hỏi/của tiếp theo — giảm vòng lặp
mò mẫm. Áp đồng nhất qua orchestrator P0-1 (một lần sửa, mọi engine hưởng).

---

## P1-2. Hướng Hopper + 16-bit DOS trong ghidra/ida-reverse

**Gap:** REA hỗ trợ **Hopper, Ghidra, IDA** đa provider, kể cả **Windows Ghidra
experimental** và **16-bit DOS**. Local nghiêng IDA; Ghidra có nhưng không có mục
Hopper và 16-bit x86/DOS rõ.

**Bổ sung:**
- Thêm mục **Hopper** (engine option) vào orchestration + doc `guide/bring-your-engine`.
- Thêm **16-bit x86/DOS** vào `ghidra-reverse/` (REA TH04 là case chuẩn: PC-98 16-bit,
  phục hồi góc tính toán, so C++ với compiler lịch sử). Dùng làm case-study seed cho
  cả `code-reconstruction-verify/`.

---

## P1-3. Website static analysis: source map + route → function map

**Gap:** REA có row "Websites" — đọc page structure, scripts, network, source maps,
map function như code. Local `browser-automation/` + `explore-web/` làm walk/click/traffic
nhưng **thiếu góc "đọc script + source map + route rồi map function như source-code"**
kiểu REA.

**Bổ sung:** vào `js-reverse/` (hoặc `browser-extension-reverse/`) thêm hướng
**static website analysis**: trích source map → khôi phục module/function → lập route
map → ánh xạ nút/traffic với hàm thật. Tái dùng canonical output của orchestrator.

---

## P2-1. .NET assembly CIL-diff nối vào binary-diff

**Gap:** REA so sánh 2 build .NET (build comparison). Local `dotnet-reverse/` +
`binary-diff/` chưa liệt kê case .NET/CIL-level diff rõ.

**Bổ sung:** nối `patch-diff-exploit/` / `binary-diff/` sang **.NET assembly diff**
(metadata + CIL instruction diff). CIL ổn định hơn x86 → dễ diff và migrate symbol;
tận dụng orchestration chọn dnSpyEx làm backend.

---

## P2-2. Go/Rust runtime recovery làm native row "biết ngôn ngữ"

**Gap:** REA native row trả pseudocode chung, chưa phân loại runtime. Local
`go-rust-reverse/` đã có (GoReSym/pclntab/redress cho Go; panic strings/
`rust_begin_unwind`/tokio state machine cho Rust).

**Bổ sung (nhỏ):** đưa `go-rust-reverse/` thành **backend language-aware** trong
orchestrator — nhận diện Go/Rust trước khi decompile, chọn đúng plugin/engine, trả về
symbol + string xref. Đây là thứ local **đã mạnh**, chỉ cần gắn vào lớp chung.

---

## Ma trận tổng — REA → local additions

| Ưu tiên | Bổ sung | Loại | Trạng thái |
|---|---|---|---|
| P0 | Lớp **Reverse Orchestration** (1 MCP → engine, canonical output, fallback) | skill + code + spec | đề xuất |
| P0 | **Code Reconstruction + Verify** (disasm→C/C++→recompile→byte-match) | skill mới | đề xuất |
| P1 | Field **unknowns/confidence/limitations** vào evidence format | ops/doc | đề xuất |
| P1 | **Hopper + 16-bit DOS** trong ghidra/ida-reverse (+orchestration) | skill/doc | đề xuất |
| P1 | **Website source-map/route static analysis** | skill bổ sung | đề xuất |
| P2 | **.NET CIL-diff** nối vào binary-diff/patch-diff | routing/doc | đề xuất |
| P2 | **Go/Rust language-aware** backend trong orchestrator | routing/doc | đề xuất |

## Ghi chú triển khai

- Tất cả là **lớp bổ sung**, giữ nguyên 100% skill/routing hiện có → reversible.
- P0-1 (orchestration) là nền: các P1/P2 gắn vào qua backend/format chuẩn.
- Mỗi mục khi làm cần disclosure + approval trước side effect (theo RULES.md); mục này
  mới là kế hoạch, chưa thực thi.

## Bước kế tiếp (chờ quyết định)

- [ ] Ưu tiên làm 2 P0 trước (orchestrator skill, reconstruction-verify skill)
- [ ] Tạo `work/<case>/scope.md` nếu chuyển sang thực thi + cần viết code/script
