# Kế hoạch: bổ sung từ REA (morluto/rea) vào local repo reverse-skills

Ngày: 2026-10-11 · Trạng thái: **bản đã rà soát mâu thuẫn (v2)**

> Chiều áp dụng: **REA → local repo** (local là repo mình sở hữu).
> Bản v2 chỉ giữ các mục **ít mâu thuẫn** với local hiện tại. Orchestration được tách
> sang file riêng `REA-orchestration-plan.md` (vì nó thay đổi kiến trúc routing SSoT).

## Quy tắc rà soát (bản v2)

Bỏ/sửa các mục mâu thuẫn với local:
- ❌ **P0-1 orchestration** → tách sang plan riêng (mâu thuẫn thiết kế behavior-chain + SSoT).
- ⚠️ **P1-1** → local ĐÃ có field `confidence` bắt buộc trong Finding
  (`ops/evidence-finding-path.md`). Bản v2 chỉ bổ sung `unknowns` + `limitations`, KHÔNG thêm
  confidence. Đã sửa.
- ✅ Giữ nguyên nhận định khớp với local.

---

## P0-A. Skill mới: Code Reconstruction + Verify (disasm → C/C++ → byte-match)

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
1. Chọn hàm đích, lấy disasm + pseudocode từ engine.
2. Viết lại bằng C/C++ tường minh (không dựa hằng số ma thuật chưa rõ).
3. Recompile cùng compiler/opt-level (REA highlight: phải khớp để byte-match).
4. So sánh byte output / chạy case set (VD 3205 case x86) để xác minh hành vi.
5. Ghi kết quả vào evidence; làm rõ unknowns nếu lệch.

**Yêu cầu hoà vào local (bắt buộc, tránh mâu thuẫn):**
- Đăng ký **route mới trong `routing.json`** (không chỉ thêm SKILL.md) — theo đúng SSoT.
- Chạy regression bắt buộc: `test-routing.ps1`, `verify-routing-coherence.ps1`, `smoke.ps1`.
- Bind vào `ops/evidence-finding-path.md` (finding phải kèm evidence, confidence sẵn có).

**Cái hay:** biến "hiểu được code" thành "tái dựng được, chạy đúng, byte gần khớp" —
đỉnh cao của RE khả chứng minh. Khác với "đọc hiểu" của các skill hiện có.

---

## P1-A. Bổ sung field `unknowns` / `limitations` vào evidence format

**Gap (đã rà):** local có `confidence` field bắt buộc trong Finding
(`evidence-finding-path.md` dòng 54). Cái thiếu là thông tin **còn dang dở** trả về
cùng finding như REA.

**Bổ sung** (CHỈ 2 field, KHÔNG đụng confidence):
- `unknowns` — những gì chưa xác định (địa chỉ, phần branch, hằng số).
- `limitations` — giới hạn phương pháp (static thiếu runtime, bỏ sót đường nào).

Agent nhận được "bài toán còn dang dở" ngay, biết hỏi/của tiếp theo — giảm vòng lặp
mò mẫm. Giữ nguyên `confidence` hiện có, chỉ thêm 2 field phụ trợ.

---

## P1-B. Hướng Hopper + 16-bit DOS trong ghidra/ida-reverse

**Gap:** REA hỗ trợ **Hopper, Ghidra, IDA** đa provider, kể cả **Windows Ghidra
experimental** và **16-bit DOS**. Local `ida-reverse/` nghiêng IDA; `ghidra-reverse/`
chưa có mục 16-bit x86/DOS.

**Bổ sung:**
- Thêm mục **16-bit x86/DOS** vào `ghidra-reverse/` (REA TH04 là case chuẩn: PC-98
  16-bit, phục hồi góc tính toán, so C++ với compiler lịch sử). Dùng làm case-study
  seed cho cả `code-reconstruction-verify/`.
- Thêm mục **Hopper** như engine option trong `ghidra-reverse/` + `TOOLS.md` (chỉ khi
  có máy/máy ảo macOS — Hopper là macOS-only; ghi rõ trong doc).

---

## P1-C. Website static analysis: source map + route → function map

**Gap:** REA có row "Websites" — đọc page structure, scripts, network, source maps,
map function như code. Local `browser-automation/` + `explore-web/` làm walk/click/traffic
nhưng **thiếu góc "đọc script + source map + route rồi map function như source-code"**
kiểu REA.

**Bổ sung:** vào `js-reverse/` (hoặc `browser-extension-reverse/`) thêm hướng
**static website analysis**: trích source map → khôi phục module/function → lập route
map → ánh xạ nút/traffic với hàm thật.

---

## P2-A. .NET assembly CIL-diff nối vào binary-diff / patch-diff

**Gap:** REA so sánh 2 build .NET (build comparison). Local `dotnet-reverse/` +
`binary-diff/` (R15) + `patch-diff-exploit/` (R16) tồn tại riêng, chưa có case
.NET/CIL-level diff rõ.

**Bổ sung:** thêm hướng **.NET assembly diff** (metadata + CIL instruction diff) vào
`binary-diff/` hoặc `patch-diff-exploit/`. CIL ổn định hơn x86 → dễ diff và migrate
symbol. Cần đăng ký keyword ghép nếu muốn routing bắt được (qua routing.json + benchmark).

---

## P2-B. Go/Rust runtime recovery chủ động nhận diện ngôn ngữ

**Gap:** REA native row trả pseudocode chung. Local `go-rust-reverse/` **đã mạnh**
(GoReSym/pclntab/redress cho Go; panic strings / `rust_begin_unwind` / tokio state
machine cho Rust) và đã có route R33 — chỉ là đưa thành bước **bắt buộc nhận diện
runtime** trước khi decompile trong quy trình native.

**Bổ sung (nhỏ):** trong `go-rust-reverse/`, thêm bước nhận diện runtime ngôn ngữ
ngay từ đầu (file/strings/runtime traits) → chọn đúng plugin/engine → trả về symbol +
string xref. Đây là thứ local đã có, chỉ củng cố thứ tự quy trình.

---

## Ma trận tổng — REA → local additions (bản v2)

| Ưu tiên | Bổ sung | Loại | Mâu thuẫn local |
|---|---|---|---|
| P0 | **Code Reconstruction + Verify** (disasm→C/C++→recompile→byte-match) | skill mới + routing | thấp |
| P1 | **unknowns / limitations** field (KHÔNG đụng confidence có sẵn) | ops/doc | đã rà, thấp |
| P1 | **16-bit DOS + Hopper** trong ghidra/ida-reverse | skill/doc | thấp |
| P1 | **Website source-map/route static analysis** | skill bổ sung | thấp |
| P2 | **.NET CIL-diff** nối binary-diff/patch-diff | routing/doc | thấp |
| P2 | **Go/Rust runtime detection** củng cố quy trình | routing/doc | thấp |

> Orchestration → xem riêng `REA-orchestration-plan.md`.

## Ghi chú triển khai

- Tất cả là **bổ sung**, giữ nguyên skill/routing hiện có → reversible.
- Mỗi skill mới / thay đổi routing **bắt buộc** đăng ký `routing.json` + chạy
  `test-routing.ps1` / `verify-routing-coherence.ps1` / `smoke.ps1`.
- Mỗi mục khi làm cần disclosure + approval trước side effect (theo RULES.md); bản này
  mới là kế hoạch, chưa thực thi.

## Bước kế tiếp (chờ quyết định)

- [ ] Ưu tiên làm P0-A (reconstruction-verify skill + routing) trước
- [ ] Xem riêng `REA-orchestration-plan.md` nếu muốn đánh giá hướng orchestration
