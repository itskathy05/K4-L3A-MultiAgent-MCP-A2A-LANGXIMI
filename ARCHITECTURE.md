# L3A — Tài liệu kiến trúc

Tài liệu này mô tả triển khai hiện tại trong `src/student_agent/`. Đầu vào là case L3A; đầu ra là JSON theo contract L3A v2 cùng trace sự kiện. Tài liệu không chứa API key hoặc suy luận nội bộ của agent.

## 1. Luồng xử lý

`day09 run` đọc `case-set.json` và từng `inputs/<case_id>.json`, mở một phiên MCP và xử lý các case tuần tự. Với mỗi case, CLI ghi `case_received`, gọi `solve_case`, kiểm tra output, ghi `outputs/<case_id>.json`, rồi ghi `case_finalized`. Trace của lượt chạy nằm tại `traces/trace.jsonl`.

```text
Input → Coordinator → Order/Item, Payment, Shipment, Policy
                         ↓                ↓
                    MCP Gateway → EvidenceLedger
                                      ↓
                              Facts → Decision → Verifier → Output
                                      └──────── TraceWriter ────────┘
```

Coordinator phát hiện tool MCP, tạo `EvidenceLedger` riêng cho case và giao việc dựa trên topic của claim cùng dữ kiện đã lấy được. `facts.py` chuẩn hóa dữ liệu; `decision.py` chọn issue, action và số tiền; verifier kiểm tra kết quả trước khi CLI ghi file. Các agent là vai trò Python trong cùng tiến trình; không dùng LLM hay dịch vụ A2A bên ngoài.

## 2. Vai trò và quyền truy vấn

| Vai trò | Trách nhiệm | Tool được phép gọi |
| --- | --- | --- |
| Coordinator | Phát hiện tool, điều phối, kiểm tra handoff | Không gọi tool lấy evidence |
| Order/Item | Xác minh đơn, mặt hàng và seller khi cần | `get_order`, `get_order_items`, `get_sellers`, `get_product_context` |
| Payment | Xác minh thanh toán và hoàn tiền | `get_order_payments`, `get_payment_timeline`, `get_refund_timeline` |
| Shipment | Xác minh giao hàng và bàn giao | `get_shipment_summary` |
| Policy | Lấy policy theo `policy_version` | `get_policy` |
| Verifier | Kiểm tra output và evidence refs | Không gọi MCP |

`get_product_context` được cấp quyền nhưng luồng hiện tại không gọi. Gateway có thể phát hiện `get_customer_history`, nhưng không có specialist nào được cấp quyền gọi. `get_order` và `get_order_payments` phải có trong tool discovery; thiếu một trong hai thì dừng.

- Claim về thanh toán hoặc trạng thái đơn `unavailable` dẫn tới truy vấn item; claim về thanh toán dẫn tới truy vấn payment timeline.
- Claim về giao chậm dẫn tới truy vấn shipment; claim về refund dẫn tới truy vấn refund timeline.
- Khi dữ kiện cho thấy issue có thể cần xử lý, coordinator truy vấn policy và refund timeline nếu chưa có. Với issue liên quan seller mà chưa có seller ID, coordinator truy vấn seller.

## 3. Handoff và trace

`Handoff` nội bộ gồm `case_id`, sender, recipient, intent, evidence refs, findings và status. Coordinator ghi `task_assigned`; specialist ghi `tool_result_consumed` khi dùng evidence và `handoff` sau khi hoàn tất. Policy agent còn ghi `policy_decided`; verifier ghi `verification_completed` khi kiểm tra thành công. Trace chỉ ghi sự kiện, mã quyết định, tên tool và evidence refs; findings không được ghi nguyên văn vào trace.

Các task chạy tuần tự, mỗi task tối đa 180 giây. Một MCP call tối đa 45 giây; lỗi tạm thời đủ điều kiện được thử thêm một lần sau 0,25 giây. Không có vòng lặp giao việc hay giao việc đệ quy. Coordinator kiểm tra `case_id` của mọi handoff và bảo đảm các ref trong handoff thuộc ledger của case đó.

## 4. Vòng đời evidence

Gateway chỉ gọi tool đã phát hiện và luôn gửi `case_id`. MCP response được kiểm tra theo schema public. Ledger kiểm tra thêm `domain` mong đợi của tool, lưu `evidence_ref`, `result_hash`, tham số truy vấn, dữ liệu và warnings. Truy vấn trùng trong cùng case dùng lại record; cache không chia sẻ giữa các case.

Khi specialist tiêu thụ record, ledger xác nhận record thuộc case hiện tại và phát một sự kiện `tool_result_consumed` cho ref đó. Output chỉ trích dẫn ref đã tiêu thụ trong case; mỗi ref của claim phải có trong danh sách ref cấp cao của output. Nội dung khách hàng khai báo chỉ dùng để chọn hướng điều tra, không được coi là bằng chứng.

## 5. Xử lý lỗi và thiếu dữ liệu

| Tình huống | Cách xử lý hiện tại |
| --- | --- |
| Timeout, lỗi kết nối/transport, `MCPError` hoặc lỗi thực thi tool dạng chung | Thử lại tối đa một lần; hết lượt thì ghi lỗi nguồn trong ledger |
| Tool tùy chọn chưa được phát hiện hoặc response sai schema/domain | Ghi lỗi nguồn, không tự tạo evidence |
| Hết 180 giây cho một task | Tạo handoff với status `TASK_TIMEOUT` và không có refs |
| Không lấy được bất kỳ evidence nào | Dừng case bằng lỗi; không tạo output thiếu căn cứ |
| Thiếu nguồn cần cho claim hoặc issue | Có thể chọn `insufficient_evidence` và `needs_investigation`; không suy ra khoản hoàn tiền khi thiếu policy hoặc refund timeline |
| MCP trả order ID khác ID được claim | Dừng case bằng lỗi |
| Output không qua verifier hoặc schema | Dừng case trước khi ghi file output và `case_finalized` |

Payment và item có tổng khác nhau được đưa vào `data_conflicts` với mã `PAYMENT_TOTAL_DIFFERS_FROM_ITEM_TOTAL`. Nếu một case lỗi, `day09 run` dừng; CLI không tự bỏ qua để chạy tiếp.

## 6. Quyết định và kiểm tra đầu ra

`facts.py` rút trích trạng thái đơn, item, seller, thanh toán, refund và mốc giao hàng từ evidence. `decision.py` dùng quy tắc cố định để xác định issue được hỗ trợ, ưu tiên topic khách hàng nêu nếu có chứng cứ, rồi chọn `case_status`, confidence, bên chịu trách nhiệm và action. Refund mới chỉ được đề xuất khi có payment, policy và refund timeline; số tiền đã hoàn thành được trừ khỏi khoản còn lại. Tiền được tính bằng `Decimal` theo cent BRL trước khi đưa vào JSON.

Verifier kiểm tra schema L3A v2, `case_id`, danh sách ref duy nhất, ref đã được tiêu thụ trong case, liên kết ref của claim, ID của tối đa 5 claim được đánh giá, tổng các dòng hoàn tiền và tính duy nhất của entity ID. Với `no_action`, output không được có refund hoặc action. CLI kiểm tra lại schema và `case_id` trước khi ghi file. Các kiểm tra này xác nhận cấu trúc và liên kết cơ bản; scorer vẫn đánh giá tính đúng đắn nghiệp vụ.

## 7. Chạy lại và đóng gói

Yêu cầu Python 3.11 trở lên; dependency và khoảng phiên bản nằm trong `pyproject.toml`. Không có model, seed ngẫu nhiên hoặc cấu hình sampling. Các case và specialist chạy tuần tự qua một phiên MCP; kết quả còn phụ thuộc dữ liệu gateway trả về lúc chạy. Endpoint và Team API Key được đọc từ `.env`, không đưa vào tài liệu hoặc artifact.

```bash
pytest -q
day09 validate-inputs
day09 mcp-tools
day09 run
day09 validate
day09 package --output dist/submission.zip
```

`day09 run` xóa output JSON và trace cũ trước lượt chạy. `day09 validate` yêu cầu đủ output cho toàn bộ case set và trace hợp lệ. ZIP nộp bài chỉ chứa `manifest.json`, `trace.jsonl` và `outputs/<case_id>.json`; `ARCHITECTURE.md` được duy trì trong repo, không nằm trong ZIP.
