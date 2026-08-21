# Lab 21 — Evaluation Report

**Họ tên**: Lê Công Dũng  **MSSV**: 2A202601649  **Ngày**: 21/08/2026  
**Tier**: `T4`  **Base model**: `unsloth/Qwen3.5-4B`  **GPU**: `Tesla T4, 14.6 GB`

> Số liệu hiện tại lấy từ `results/` và là smoke n=8. Notebook resume đã được chuẩn
> bị để thay bằng full eval trước khi nộp; tôi không suy diễn số 50 mẫu từ lát cắt này.

## 1. Setup

| | |
|---|---|
| Dataset | 250 ticket CSKH tiếng Việt → JSON triage 4 trường |
| Train / val | 225 / 25, seed 42 |
| `max_length` | 1024 — p95=98, giá trị gợi ý=256 |
| `MASK_MODE` | `assistant-only` |
| Epochs / max_steps | 2 / 30 |

Template **có** giữ khối `<think>`: thẻ mở, nội dung và thẻ đóng đều còn nguyên. Corpus
gốc chỉ chứa JSON nên `valid_trace_rate=0` không phải bằng chứng template làm mất trace.

## 2. Mask proof

| | |
|---|---|
| `supervised_fraction` | 0.4149 (39/94 token) |
| Câu trả lời nằm trong loss | `true` |
| Câu hỏi không nằm trong loss | `true` |

```text
</think>

{"intent": "doi_tra", "urgency": "trung_binh", "product": "balo laptop", "sentiment": "trung_tinh"}<|im_end|>
```

## 3. Ba baseline

| Run | target | regression | format | latency ms |
|---|---:|---:|---:|---:|
| (a) base + naive prompt | 0.0000 | 0.7500 | 0.0000 | 3341.7 |
| (b) base + optimized prompt | 0.6875 | 0.7500 | 1.0000 | 945.6 |
| (c) LoRA fine-tune | 0.9375 | 0.7500 | 1.0000 | 1489.9 |

Baseline (b) mạnh hơn (a): target tăng 0.6875, format tăng từ 0 lên 1 và latency giảm
khoảng 72%. Tôi không sửa `OPTIMIZED_PROMPT`; SHA `719e74d3b6232053` khớp gatekeeper.

## 4. Giải phẫu cấu hình sai

| Run | vị trí | r | trainable | LR | train loss | target | s | VRAM GB |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| `correct` | text-linear | 16 | 32,464,896 | 1e-4 | 0.6268 | 0.9375 | 963.5 | 12.01 |
| `attn_only` | q,v | 283 | 32,456,704 | 1e-4 | 0.5380 | 0.9375 | 786.1 | 12.02 |
| `wrong_lr` | text-linear | 16 | 32,464,896 | 1e-5 | 1.5704 | 0.0000 | 916.9 | 12.01 |
| `qlora` | text-linear | 16 | 32,464,896 | 1e-4 | 0.7058 | 0.8438 | 974.8 | 7.09 |

### 4.1 Placement và rank

`attn_only` hoà `correct` ở target 0.9375 dù train loss thấp hơn, 0.5380 so với
0.6268. Xếp theo loss sẽ gọi `attn_only` là người thắng, còn metric tác vụ chỉ cho thấy
hoà. Kết quả chưa chứng minh text-linear tốt hơn q,v cho tác vụ hẹp này; nó cho thấy
tăng rank lên 283 để cân ngân sách không tự động tạo thêm điểm. Full eval cần thiết để
phân giải trận hoà.

### 4.2 Learning rate

`wrong_lr` dùng 1e-5, nhỏ hơn cấu hình đúng 10 lần. Loss cuối 1.5704, cao hơn nhiều so
với 0.6268, còn target và format bằng 0. Bước cập nhật quá nhỏ trong ngân sách 30 step.
Nếu chỉ nhìn vài step đầu, tôi có thể đổ lỗi cho dữ liệu, rank hoặc mask dù biến duy
nhất trong phép đối chứng là learning rate.

### 4.3 QLoRA

QLoRA dùng 7.09 GB thay vì 12.01 GB: tiết kiệm 4.92 GB, khoảng 41.0%. Đổi lại target
giảm từ 0.9375 xuống 0.8438, latency tăng lên 1687.2 ms và train loss tăng. Lát cắt này
ủng hộ tránh QLoRA khi LoRA 16-bit vẫn vừa VRAM, nhưng lợi ích bộ nhớ là đáng kể.

## 5. Phán quyết

**Cổng hồi quy smoke**: `PASSED`  
`target Δ = +0.250` · `regression Δ = +0.000` · `valid_trace_rate = 0.00`

Trên 8 mẫu, LoRA vượt baseline tối ưu 0.25 target và giữ regression 0.75. Format của
cả hai bằng 1, nên lợi thế không đến từ cách chấm đầu ra sai định dạng. LoRA chậm hơn
baseline (b) khoảng 57.6%, vì vậy mức tăng target có chi phí phục vụ. Tám mẫu chưa đủ
cho phán quyết nộp bài: vài lỗi trường đơn lẻ có thể làm delta đổi đáng kể. Trace rate
bằng 0 cũng không được hiểu là reasoning collapse vì corpus không có trace. Kết luận
hợp lệ hiện tại là pipeline hoạt động và cần full eval, chưa phải quyết định deploy.

## 6. Định tính

| # | Ticket rút gọn | Nhãn đúng | Fine-tune | Nhận xét |
|---|---|---|---|---|
| 1 | Chuột không dây, trả lại, gấp | doi_tra / cao / chuột không dây / tich_cuc | đúng 4/4 | ✅ tốt |
| 2 | Đèn LED vỡ, gấp | san_pham_loi / cao / đèn bàn LED / trung_tinh | đúng 4/4 | ✅ tốt |
| 3 | Bình giữ nhiệt, chưa thấy tiền | hoan_tien / trung_binh / bình giữ nhiệt / tich_cuc | đúng 3/4 | ❌ sai sentiment |
| 4 | Nồi chiên thiếu phụ kiện | san_pham_loi / trung_binh / nồi chiên không dầu / tich_cuc | đúng 3/4 | ❌ sai sentiment |
| 5 | Balo đổi size, hỏi cho biết | doi_tra / thap / balo laptop / tieu_cuc | đúng 4/4 | ✅ tốt |

Hai ca chưa hoàn hảo đều đúng intent, urgency và product nhưng sai sentiment. Model học
tốt marker nghiệp vụ trực tiếp, còn sắc thái cảm xúc dễ bị nhiễu bởi diễn đạt ngắn.
Artefact hiện không lưu prediction từng mẫu của baseline (b), nên tôi không dựng dữ
liệu không tồn tại; lần full eval nên lưu thêm prediction nếu muốn so từng ca.

## 7. Kết luận và điều học được

Tôi chưa deploy bản fine-tune từ kết quả hiện tại. Smoke cho tín hiệu tốt: target
0.9375 vượt baseline 0.6875, không giảm regression và giữ format 1. Tuy nhiên n=8 quá
nhỏ, latency cao hơn và lỗi tập trung ở sentiment. Cần chạy đủ 50 target và 15
regression, rồi chỉ deploy nếu cổng vẫn PASSED và không có nhóm lỗi hệ thống.

Đòn bẩy rõ nhất là mask và learning rate. Mask đúng bảo đảm model chỉ học đáp án; nếu
prompt lọt vào loss thì mọi so sánh phía sau mất ý nghĩa. LR 1e-5 làm `wrong_lr` rơi
xuống target 0 dù biến khác giữ nguyên. Placement chưa tạo khác biệt trên lát cắt:
`attn_only` hoà `correct` dù loss thấp hơn. QLoRA thể hiện đánh đổi thật giữa bộ nhớ và
chất lượng. Dữ liệu quyết định khả năng sửa sentiment; tăng rank không bổ sung tín hiệu
mà corpus chưa cung cấp.

Ba điều tôi học được:

1. Phải giải mã loss mask; loss đẹp không chứng minh prompt không lọt vào loss.
2. Baseline prompt tối ưu là đối thủ thật: target tăng từ 0 lên 0.6875 và còn nhanh hơn.
3. Train loss không xếp hạng chất lượng tác vụ: `attn_only` loss tốt hơn nhưng chỉ hoà target.

Nếu có thêm hai giờ, tôi sẽ chạy full eval, phân tầng lỗi theo trường và bổ sung mẫu
sentiment khó sau khi giữ riêng một tập đánh giá mới.

## Phụ lục — thưởng

- [ ] B1 NB6 merge + hot-swap
- [ ] B2 dataset miền riêng
- [ ] B3 reasoning-trace collapse
- [ ] B4 quét rank có kiểm soát
- [ ] B5 HuggingFace Hub
