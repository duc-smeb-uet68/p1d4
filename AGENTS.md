# Hướng dẫn cho RO — Day 04 Lab v2: Research Agent Tool Eval

## Danh xưng và cách phối hợp

- Chủ sở hữu/người ra quyết định của dự án là **SI**. Khi trao đổi trực tiếp, hãy gọi người dùng là **SI**.
- Agent tự xưng là **RO**. Viết ngắn gọn, nêu kết quả trước, rồi nêu evidence hoặc rủi ro cần thiết.
- Ưu tiên yêu cầu trực tiếp của SI, đồng thời tuân thủ các chỉ dẫn hệ thống/developer áp dụng cho phiên làm việc.
- Không tự ý thay đổi phạm vi bài, tạo dữ liệu/chạy giả để có metric đẹp, hoặc coi một run không có evidence là hoàn thành.

## Môi trường đã sẵn sàng

Môi trường Python đúng cho **toàn bộ project** đã được cài tại:

```text
D:\duc\home\vinAI\phase1\Day04-C401-Prompt-Engineering-Tool-Calling-Labs-student-k3\.venv
```

- Tin cậy môi trường này: **không kiểm tra sự tồn tại**, không tạo lại virtualenv, không cài lại dependency nền, và không sửa/xóa `.venv`.
- Khi cần chạy Python trên Windows, dùng trực tiếp `D:\duc\home\vinAI\phase1\Day04-C401-Prompt-Engineering-Tool-Calling-Labs-student-k3\.venv\Scripts\python.exe` từ thư mục `starter_v0/`.
- `starter_v0/.env` là nơi giữ key. Không commit, in ra terminal, đưa vào prompt, transcript, ảnh chụp, report, hoặc UI. Không đưa token Telegram vào command line hay raw exception.

## Sản phẩm đang xây

Đây không phải bài làm chatbot tổng quát. Sản phẩm là một **research agent chạy tool thật**: nhận nhu cầu nghiên cứu, chọn đúng tool và args, lấy dữ liệu thật, rồi trả lời/tổng hợp có trace và log.

Phạm vi sản phẩm chính:

- tin tức/web hiện tại;
- bài đăng X/Twitter theo tài khoản hoặc chủ đề;
- đọc một URL cụ thể;
- tổng hợp các item đã thu thập thành digest;
- hỏi lại khi thiếu thông tin và xác nhận trước action có side effect.

Các capability `policy`, `papers`, `paper_text`, và `send` là built-in optional/advanced. Chúng có thể hữu ích cho demo hoặc extension nhưng **không** thay thế tool mới bắt buộc của team và không tự tạo điểm bonus.

Các yêu cầu ngoài phạm vi research/news (ví dụ giải tích hoặc viết hàm Python trong fixed eval) phải trả lời không dùng tool, từ chối/định hướng phù hợp. Không đoán bừa URL, tài khoản, hay thông tin còn thiếu.

## Kiến trúc và pipeline kỹ thuật

```text
User request/history
  -> artifacts/system_prompt.md + artifacts/tools.yaml
  -> provider structured tool call
  -> tools/__init__.py :: TOOL_FUNCTIONS
  -> tools/<tool_name>/tool.py chạy tool thật
  -> tool result / error / trace
  -> chat.py tool loop hoặc run_eval.py JSON evidence
```

- `artifacts/system_prompt.md` và `artifacts/tools.yaml` là interface model nhìn thấy; đây là hai artifact chủ yếu để tối ưu routing.
- `tools/__init__.py` là registry. Tên khóa trong `TOOL_FUNCTIONS` phải khớp tên model thấy trong `tools.yaml`.
- `agent.py` chỉ thực hiện một lượt provider + tool calls, phù hợp eval. `chat.py` có `run_model_tool_loop`, chạy nhiều round (mặc định tối đa 4), đưa tool results trở lại model, dừng khi `clarify` trả `awaiting_user`, và lưu transcript.
- UI phải tái sử dụng `run_model_tool_loop` của `chat.py`, cùng prompt/declarations với eval; không viết một agent loop khác dẫn tới hành vi UI và eval lệch nhau.
- `versioning.py` hash prompt/tool declaration thành `artifact_version`. Run JSON và transcript phải giữ hash/version này làm evidence.
- `run_eval.py` chấm **tên tool và subset args**, không chấm chất lượng văn xuôi. Với case cần tool, grader ép tool call; với `no_tool`, agent không được gọi tool. Multi-turn chỉ chấm user turn cuối, còn các turn trước là context để carry/correct intent.

### Quy tắc routing quan trọng

| Ý định | Tool | Convention cần giữ |
|---|---|---|
| Bài mới của một tài khoản | `timeline` | `screenname` là handle, không có `@`; map tên nổi tiếng khi biết chắc |
| Mọi người bàn gì trên X/Twitter | `social_search` | `search_type=Top` cho “top/phổ biến”; mặc định `Latest` |
| Tin mới/broad web research | `lookup` | tin hôm nay: `topic=news`, `timeframe=day`; tuần này: `timeframe=week` |
| Đã có URL cụ thể | `fetch` | đọc chính URL đã cung cấp, không thay bằng web search |
| Đã có các item cần biên tập | `format` | chỉ format dữ liệu đã thu thập, không fetch thêm |
| Thiếu handle/URL hoặc cần confirm action | `clarify` | thiếu thông tin: `response_type=text`; action: `response_type=yes_no` |
| Gửi/post Telegram | `send` | chỉ sau xác nhận rõ ở hội thoại hiện tại; `confirmed=true` mới có side effect |

Tool policy nội bộ dùng `policy`; paper/preprint dùng `papers`; chỉ khi có arXiv ID/URL cụ thể và cần đọc nội dung mới dùng `paper_text`. Tweet chỉ là tín hiệu Tier 3, không được trình bày là fact đã xác nhận nếu thiếu nguồn Tier 1/2.

## Workflow bắt buộc, theo evidence

1. **Preflight rồi baseline v0.** Provider preflight và smoke-test các core API/team tool thực sự dùng là gate của lab (đây là kiểm tra capability/API, không phải kiểm tra `.venv`). Sau đó chạy fixed base eval bằng provider/API thật. Đọc run JSON, đặc biệt `summary`, `results[*].result.failures`, `observed_mismatch`, actual tool calls và tool results.
2. **Đặt một hypothesis.** Xác định một nguyên nhân routing/args/boundary cụ thể; sửa đúng `system_prompt.md` hoặc `tools.yaml` để kiểm chứng. Không “tune” nhiều thứ mơ hồ trong cùng một vòng.
3. **Lặp v1, v2, v3.** Mỗi version phải là cải tiến thật, có run riêng và so sánh trước/sau. Ghi đầy đủ `artifacts/version_log.csv` (version, artifact/hash, lý do, hypothesis, metric, run file).
4. **Thêm tool mới của team.** Tool mới đầu tiên là điều kiện core: có `tools/<tool_name>/TOOL.md` với frontmatter contract, `tool.py`, registry trong `tools/__init__.py`, declaration trong `tools.yaml`, và smoke test trực tiếp an toàn. Tool action phải có confirmation boundary.
5. **Viết group eval.** `data/eval_group.json` phải có **đúng 10** case do team tự viết: 5 single-turn dùng `query`, 5 multi-turn dùng `turns`. Mỗi case có `id`, `phase: "B"`, `failure_type` hợp lệ, `expect` (`tool_calls` hoặc `no_tool`) và `metadata.what_it_tests`. Case mẫu trong `samples/` chỉ là schema, không tính vào 10 case.
6. **Chạy group eval, chat và UI.** Chạy group suite bằng v3; tạo live transcript với ít nhất một research request bình thường, một flow thiếu thông tin rồi bổ sung, và một action cần confirm. UI cần hiển thị request/response, từng round, tool name + args + result/error, transcript/run/artifact version.
7. **Hoàn thiện report và demo.** `artifacts/REPORT.md` Phần A là giới thiệu/demonstration và phải xong trước 11:30 theo brief; Phần B phải lấy bảng version, failure analysis, 10 team cases, live evidence và reflection từ log thật. Rehearse 3–5 scenario, có fallback run/transcript và kiểm tra không lộ secrets.

Ví dụ lệnh (chọn cùng một provider/model đã cấu hình cho toàn bộ flow):

```powershell
Set-Location starter_v0
$LabPython = 'D:\duc\home\vinAI\phase1\Day04-C401-Prompt-Engineering-Tool-Calling-Labs-student-k3\.venv\Scripts\python.exe'
& $LabPython run_eval.py --provider <provider> --version v0 --suite base --eval-cases data/eval_base.json
& $LabPython run_eval.py --provider <provider> --version v3 --suite group --eval-cases data/eval_group.json
& $LabPython chat.py --provider <provider> --version v3
```

Nếu provider/API lỗi, giữ run JSON như evidence và phân biệt nó với lỗi routing. Không che lỗi bằng fake result hoặc sửa log.

## Bảo toàn fixed eval và đồng bộ tool

`data/eval_base.json` là fixed eval. Không sửa query, expected arguments, expected behavior hay “đáp án” để tăng điểm. Ngoại lệ duy nhất là đổi field **tên tool** khi team rename tool; khi đó phải đồng bộ tất cả nơi sau:

1. `artifacts/system_prompt.md`
2. `artifacts/tools.yaml`
3. `tools/<tool_name>/TOOL.md`
4. `tools/__init__.py`
5. `data/eval_base.json`
6. `data/eval_research_extension.json`
7. `data/eval_group.json` nếu case có nhắc tool đó
8. `artifacts/REPORT.md` và nội dung demo/poster

Sau rename, để mọi expected tool đều vừa được declare trong YAML vừa có implementation trong registry; nếu không `run_eval.py` sẽ fail validation hoặc chấm sai routing.

`data/eval_research_extension.json` chỉ chạy nếu team thực sự dùng optional built-ins. Nếu để optional declaration trong `tools.yaml`, model vẫn nhìn thấy chúng và gọi nhầm vẫn làm hỏng core routing; có thể isolate bằng cách bỏ declaration optional không dùng, nhưng chỉ bật lại trước khi chạy extension.

## Rubric/gate cần tối ưu

Repo không cung cấp bảng điểm số cụ thể. Vì vậy RO không được bịa phần trăm hay tuyên bố “đạt điểm X”. Hãy coi các điều kiện sau là rubric/gate có thể chứng minh bằng artifact:

| Hạng mục | Evidence/điều kiện core |
|---|---|
| Agent chạy thật | provider preflight + smoke-test core API/team tool liên quan, ít nhất 5 tool trong `tools.yaml`, base eval chạy được |
| Tối ưu có phương pháp | baseline `v0` + ba vòng `v1`/`v2`/`v3` khác nhau, version log và run JSON thật |
| Tool do team xây | ít nhất 1 tool mới có contract, registry, declaration và smoke-test evidence |
| Eval do team tự thiết kế | đúng 10 case: 5 single-turn + 5 multi-turn, schema/failure type hợp lệ |
| UI/demonstration | UI là core, không phải bonus; trace + version + transcript hiển thị được, 3–5 scenario đã rehearse |
| Báo cáo/nộp bài | `REPORT.md` A+B, runs, transcripts, implementation/UI, log version; không có `.env`, key, `.venv`, cache/build output |

Gate hợp lệ của mỗi suite khi báo metric:

- `summary.provider_error_cases == 0`;
- `summary.measured_cases == summary.total_cases`;
- mọi `tool_results` có `error` phải được review thủ công. Routing PASS không tự chứng minh API/tool execution đã đúng.

Bonus chỉ xuất hiện sau khi UI core hoàn thành **và** team tự viết thêm **hơn 3 tool mới** (tức ít nhất bốn tool mới tổng cộng). UI riêng lẻ và optional built-ins có sẵn không tính bonus.

## Trạng thái scaffold ban đầu và các file trọng tâm

Starter được cố ý để prompt/tool declaration mơ hồ; system prompt ban đầu còn khuyến khích đoán dữ liệu thiếu và tự gửi action, tức là hành vi cần được sửa chứ không phải chuẩn để giữ lại. `data/eval_group.json` đang trống có chủ đích, `version_log.csv` mới có header, và starter chưa có `app.py`.

| Đường dẫn | Vai trò |
|---|---|
| `starter_v0/artifacts/system_prompt.md` | Quy tắc quyết định/routing của agent |
| `starter_v0/artifacts/tools.yaml` | Tool name, description, JSON schema model nhìn thấy |
| `starter_v0/tools/` | Contract và implementation của từng tool |
| `starter_v0/run_eval.py` | Chạy/grade eval và ghi run JSON |
| `starter_v0/chat.py` | Tool loop multi-round và transcript |
| `starter_v0/data/eval_base.json` | Fixed core eval — bảo toàn nội dung |
| `starter_v0/data/eval_group.json` | 10 test case team tự viết |
| `starter_v0/artifacts/version_log.csv` | Evidence hypothesis/metric theo version |
| `starter_v0/artifacts/REPORT.md` | Bằng chứng demo và bài nộp |
| `TOOL-SETUP.md` | Cấu hình API, smoke test, UI/deploy safety |

Trước khi bàn giao, RO cần đối chiếu final checklist với artifact thật, giữ thay đổi của SI không liên quan, và báo rõ phần nào đã verify/chưa verify.
