AuditFlow AI – Streamlit 系統技術規格（繁體中文）
1. 系統概要
AuditFlow AI 是一套以 Streamlit 實作的瀏覽器端「代理式（Agentic）」應用程式，用於協助產製符合 ISO 13485 / GMP 的醫療器材品質稽核報告。

系統特點：

透過 agents.yaml 定義的 多階段代理流程（例：Layout Mapper → CAR Writer → Polisher）。
支援多家大型語言模型供應商：
Google Gemini（gemini-2.5-flash, gemini-2.5-flash-lite）
OpenAI（gpt-5-nano, gpt-4o-mini, gpt-4.1-mini）
Anthropic（如 claude-3.5-haiku, claude-3.5-sonnet）
xAI / Grok（grok-4-fast-reasoning, grok-3-mini）
Smart Replacement 模組：高精度範本套版與變更標示。
全流程 Human-in-the-loop（HITL）：每一代理輸出皆可人工編輯後再傳遞至下一步。
WOW UI：
20 種花卉主題（顏色主題）。
深色 / 淺色模式切換。
介面中英 / 繁中雙語。
幸運轉盤隨機選擇花卉主題。
儀表板與日誌：顯示延遲（latency）、token 使用量與事件紀錄。
強化錯誤處理：
對 Gemini 無文字輸出時不再崩潰，改回傳可讀錯誤訊息。
Streamlit 狀態同步修正，確保代理輸出能正確顯示在文字編輯框中。
2. 系統架構
2.1 技術棧
程式語言：Python 3.x
Web 框架：Streamlit
部署目標：
Hugging Face Spaces
Streamlit Cloud（streamlit.io）
LLM 供應商與 SDK：
Google Gemini：google-generativeai
OpenAI：openai
Anthropic：anthropic
xAI / Grok：xai-sdk
設定與序列化：
agents.yaml：管線定義檔
pyyaml：YAML 解析
視覺化：
Plotly Express（plotly.express）：互動圖表
環境變數 / 機密：
GEMINI_API_KEY
OPENAI_API_KEY
ANTHROPIC_API_KEY
XAI_API_KEY
若環境變數不存在，可由 UI 輸入；若存在則優先採用且不在 UI 顯示。
2.2 高階元件
Streamlit UI

側邊欄：
API 金鑰設定
語言與主題切換
agents.yaml 匯入 / 匯出與重載
三個主要分頁（Tabs）：
Pipeline：代理流程設定與執行
Smart Replace：範本套版與變更標示
Dashboard & Logs：效能與使用量儀表板 + 日誌
全域狀態管理（st.session_state）

pipeline: PipelineState
smart_result: str
smart_template: str, smart_list: str
manual_api_keys：使用者手動輸入的 API 金鑰
UI 狀態：theme_name, dark_mode, ui_language
log_entries：事件日誌
每個代理輸出對應的 widget 狀態（如 out_<agent.id>）
AI 服務層

call_llm 依照 provider 分派到：
call_gemini
call_openai
call_anthropic
call_grok
統一回傳結構：
text
latency_ms
input_tokens
output_tokens
管線設定

agents.yaml 中定義：
預設 template（稽核報告模板）
預設 observations（原始觀察紀錄）
有序的 agents 陣列（每個元素對應一個 AgentConfig）
UI 可將目前狀態匯出成 YAML，也可由 YAML 匯入變更。
3. 資料流程
3.1 整體執行流程
初始化

main() 啟動後呼叫 init_session_state()：
嘗試讀取 agents.yaml → 轉成 PipelineState，若失敗或不存在則使用空白默認管線。
Smart Replace 的 smart_template 與 smart_list 初始值來自 pipeline 的 template / observations。
設定 UI 語言預設為繁體中文、主題為 Sakura、深色模式預設為關閉。
初始化日誌與手動 API 金鑰欄位。
使用者輸入

使用者可編輯：
稽核模板 Markdown。
原始觀察紀錄文字。
針對每個代理，可配置：
Provider（gemini / openai / anthropic / grok）
Model（依 provider 顯示不同型號清單）
Max tokens
Temperature
User Prompt
System Prompt Suffix
代理執行（單一步驟）

點擊「僅執行此代理」按鈕時：
build_agent_input 組裝該代理的輸入：
System prompt：BASE_SYSTEM_PROMPT_ZH + system_prompt_suffix
User content：包含
[Template]
[Observations]
[Previous Agent Output]（前一代理輸出，若有）
[Agent Task Instructions]（該代理任務說明）
run_agent：
將狀態設為 running。
呼叫 call_llm，將 system/user prompt 傳給對應 provider。
成功：
寫入 state.output = LLM 回傳文字
status = "completed"
記錄延遲與 token 使用量
error = None
失敗：
status = "error"
記錄錯誤訊息
寫入日誌（log_event）
更新 UI 中該代理對應的 st.session_state["out_<agent.id>"]，讓文字框顯示最新輸出。
HITL 人工編輯

使用者可直接在各代理的「代理輸出內容（可編輯）」文字框中修正：
錯誤推論
格式
內容補充
編輯結果會回寫到 pipeline.history[agent.id].output。
之後的代理在 build_agent_input 中，取得「前一代理輸出」時會使用這份已人工校正的內容。
整體管線執行

點擊「執行整體管線」：
run_full_pipeline 依序從第 1 個 agent 執行到最後一個。
每一步的 output 會作為下一步的「Previous Agent Output」。
Smart Replace（智慧套版）

使用者輸入：
Template A（Markdown 結構）→ 已預設為 pipeline 模板
List B（資料或觀察清單）→ 預設為 pipeline 觀察紀錄
選擇 provider / model / max tokens / temperature。
執行時：
建立一個暫時的 AgentConfig（id = smart-replace）。
使用 SMART_REPLACE_SYSTEM_PROMPT 作為 system prompt。
user content 為 Template A + List B。
呼叫 call_llm。
結果：
成功 → st.session_state.smart_result = 回傳文字
若內容為空 → 顯示警告（可能是安全過濾或模型限制）。
失敗 → 顯示錯誤訊息並寫入日誌。
儀表板與日誌

Dashboard：
進度（完成代理數 / 總代理數）。
總延遲、總輸入 token、總輸出 token。
每代理延遲與輸出 token 長條圖（Plotly）。
Logs：
依時間倒序顯示 log_entries。
顏色區分 info / success / error。
4. UI 模組與設計
4.1 側邊欄（Sidebar）
職責：

API 金鑰

四個 password 輸入框：
Gemini API 金鑰
OpenAI API 金鑰
Anthropic API 金鑰
XAI (Grok) API 金鑰
實際使用邏輯：
若對應環境變數存在，優先使用環境變數（不顯示於 UI）。
否則採用使用者在側欄輸入的值。
UI 設定

語言切換：
zh（繁體中文） / en（英文）
主題切換：
從 20 個花卉主題中選擇。
深色模式：
Checkbox 切換深色背景。
幸運花卉轉盤：
隨機選一個主題，並在日誌中記錄。
agents.yaml 管理

重新從檔案系統讀取 agents.yaml（若存在）。
將目前 pipeline 狀態匯出下載（agents.yaml）。
由使用者上傳 YAML 檔：
使用 yaml.safe_load 解析。
若解析失敗，顯示 st.error 並不覆蓋現有管線。
若成功，建立新的 PipelineState。
4.2 Pipeline 分頁
模板與觀察輸入

左側：稽核報告模板（Markdown）
右側：原始觀察紀錄
資料直接更新 pipeline.template 與 pipeline.observations。
代理清單（Agents）

對每個 agent 顯示一個 st.expander：

顯示代理名稱與順序編號。
代理設定區塊

Provider 下拉選單：gemini / openai / anthropic / grok。
Model 下拉選單（依 provider 顯示預設清單）。
Max tokens 數字輸入。
Temperature 滑桿。
User Prompt 多行文字輸入。
System Prompt Suffix 多行文字輸入。
API 金鑰狀態提示

若對應 provider 的 API key 無效或缺漏，顯示 warning。
執行與狀態顯示

「僅執行此代理」按鈕：
呼叫 run_agent，並記錄到日誌。
狀態徽章（Status Badge）：
idle / running / completed / error
顏色與文字樣式經 CSS 美化。
Latency 顯示（若 latency_ms 存在）。
Error 顯示：
若 state.error 有值，以 st.error 顯示錯誤內容。
可編輯輸出區（關鍵修正）

每個 agent 對應一個 output_key = "out_<agent.id>"。
使用 st.session_state[output_key] 作為唯一真實來源：
初次渲染時，若該 key 不存在，將其設為 state.output（可能為空字串）。
若 state.status 為 completed 或 error，在該輪 rerun 之初，強制將 st.session_state[output_key] = state.output，確保新 LLM 結果顯示出來。
st.text_area 只以 key=output_key 綁定，不使用 value= 參數。
使用者編輯文字後，若 new_output != state.output，則同步回寫到 state.output 和 pipeline.history[agent.id]。
下載按鈕：
將 textarea 目前內容輸出為 .md 檔案。
整體管線執行

「執行整體管線」按鈕：
依序執行 pipeline 中所有代理。
4.3 Smart Replace 分頁
左側輸入區

Template A (Markdown 結構) → smart_template。
List B (資料 / 觀察清單) → smart_list。
Provider / Model / Max tokens / Temperature 設定。
若選定 provider 無有效 API key，顯示警告。
執行按鈕

呼叫 run_smart_replace：
使用 SMART_REPLACE_SYSTEM_PROMPT。
新建 AgentConfig(id="smart-replace", ...)，經 call_llm 執行。
成功：
st.session_state.smart_result = res["text"]。
若結果為空，顯示警示（可能安全過濾導致無內容）。
失敗：
錯誤寫入日誌並以 st.error("Smart Replace error: ...") 顯示。
右側結果顯示

兩種 View 模式：
預覽 (HTML)：st.markdown(result, unsafe_allow_html=True)，呈現 <span style="color: coral"> 標註。
原始 Markdown：st.code(result, language="markdown")。
下載按鈕：將 smart_result 另存為 smart_replace.md。
4.4 Dashboard & Logs 分頁
指標顯示

Progress：完成的代理數 / 總代理數。
Total Runtime (ms)：所有代理 latency_ms 加總。
Input Tokens / Output Tokens 加總。
互動圖表

Latency per agent：
x: agent name
y: latency (ms)
Plotly bar chart。
Output tokens per agent：
x: agent name
y: output tokens
Plotly bar chart。
日誌

顯示最近 N 筆 log_entries。
格式：[時間] LEVEL 訊息，色彩區分等級。
5. AI 服務層細節
5.1 通用呼叫介面
call_llm(agent, system_prompt, user_content, api_keys)：

依 agent.provider 分派至不同實作。
驗證對應 API key 是否存在，若缺省則丟出具體錯誤（例如「Gemini API key missing.」）。
回傳：
{
  "text": <str>,          # 模型輸出文字
  "latency_ms": <float>,  # 執行時間（毫秒）
  "input_tokens": <int or None>,
  "output_tokens": <int or None>,
}
5.2 Gemini
SDK：google-generativeai
使用方式：
genai.GenerativeModel(model_name=model, system_instruction=system_prompt)
generate_content(user_content, generation_config={...})
resp 解析：
使用 _extract_gemini_text(resp) 檢出所有 candidates -> content -> parts -> text。
若無任何文字部分，會根據 finish_reason 組成描述性錯誤訊息並 raise RuntimeError，避免出現「quick accessor requires a valid Part」錯誤。
Token 計數：
從 resp.usage_metadata.prompt_token_count / candidates_token_count 取得（若存在）。
5.3 OpenAI
SDK：openai （OpenAI client）
呼叫模式：
client.chat.completions.create(model=..., messages=[..., ...])
取值：
resp.choices[0].message.content 為文字輸出。
resp.usage.prompt_tokens / resp.usage.completion_tokens。
5.4 Anthropic
SDK：anthropic
呼叫模式：
client.messages.create(model=..., system=..., messages=[{"role":"user","content":...}])
取值：
將 resp.content 中 type == "text" 的 block.text 串接為完整輸出。
Token 使用量來自 resp.usage.input_tokens / resp.usage.output_tokens。
5.5 Grok (xAI)
SDK：xai-sdk
呼叫模式：
client = XAIClient(api_key, timeout=3600)
chat = client.chat.create(model=model)
chat.append(xai_system(system_prompt))
chat.append(xai_user(user_content))
response = chat.sample(options={"temperature": ..., "max_output_tokens": ...})
輸出：
response.content 作為文字內容。
若 response.usage 存在則讀取 input / output tokens。
6. agents.yaml 設定
6.1 結構範例
pipeline:
  template: ""
  observations: ""
  agents:
    - id: layout-mapper
      name: "Layout Mapper"
      provider: "gemini"
      model: "gemini-2.5-flash"
      max_tokens: 4096
      temperature: 0.1
      user_prompt: |
        ...
      system_prompt_suffix: |
        ...
    - id: car-writer
      ...
    - id: polish
      ...
6.2 載入與錯誤處理
load_agents_from_yaml("agents.yaml")：
若檔案不存在、為空或 YAML 格式錯誤，將回傳空白 PipelineState，不使應用崩潰。
僅當 data 為 dict 且有 pipeline / agents 時才轉換為 AgentConfig。
上傳 YAML：
使用者透過 UI 上傳。
yaml.safe_load 解析，錯誤則透過 st.error 通知。
6.3 匯出
export_agents_to_yaml(pipeline)：
將目前 pipeline.template / observations / agents 序列化成 YAML。
可從側邊欄下載成 agents.yaml 以版本控管或部署之用。
7. 錯誤處理與使用者體驗保護
Gemini 無文字輸出：

使用 _extract_gemini_text 避免直接呼叫 resp.text。
若仍無任何文字部件，產生帶有 finish_reason 的錯誤訊息。
在 UI：
Pipeline：上述錯誤會顯示在代理卡片中（st.error）。
Smart Replace：以 st.error("Smart Replace error: ...") 顯示詳細訊息。
Streamlit widget 狀態同步：

使用 st.session_state[output_key] 作為唯一來源。
解決過去「LLM 有輸出，但 textarea 一直是空白」的問題。
API 金鑰缺失：

在代理卡片與 Smart Replace 中，若對應 provider 的 API key 不存在，顯示警告訊息。
YAML 解析錯誤：

不會導致整個 app 失敗啟動。
錯誤訊息印出在後端 log，前端必要時顯示 st.error。
8. 主題與在地化（Theming & Localization）
花卉主題（20 種）：

每一主題定義：
primary
secondary
accent
透過動態注入 CSS 設定：
--color-primary
--color-secondary
--color-accent
--bg-color
--text-color
深色 / 淺色模式：

控制背景與文字顏色。
介面語言（Localization）：

以 UI_LABELS 對應 zh / en。
所有主要文案、標題、按鈕皆可根據語言切換顯示。
幸運花卉轉盤：

隨機選擇一個花卉主題。
在日誌中記錄該次選擇。
9. 安全與隱私
API 金鑰

優先使用環境變數（伺服端設定），不在前端顯示。
使用者手動輸入金鑰僅儲存在當前 st.session_state，不會持久化。
前端執行

目前設計為前端（Streamlit 後端）直接呼叫外部 LLM API，無額外後端服務。
若未來需要更高安全性，可改為透過中介後端 proxy 隱藏金鑰。
個資（PII）

現階段未內建個資去識別，但架構允許在 call_llm 前新增本地文字前處理（遮蔽姓名、ID 等）。
10. 可擴充性
代理數量與流程：

可在 agents.yaml 中新增 / 刪除 / 調整代理順序。
UI 會自動讀取並顯示。
LLM 供應商擴展：

call_llm 中新增分支與對應 helper 函式，即可接更多 API。
Prompt 工程：

系統 base prompt 與每個 Agent 的 user_prompt / system_prompt_suffix 皆可透過 UI 修改並重新匯出 YAML。
UI 功能延伸：

容易新增新分頁（例如「健康檢查」「PII 工具」「報告審核模式」）整合進現有架構。
20 個後續規劃與需求釐清問題（繁中）
是否需要在 agents.yaml 中加入 版本欄位與變更紀錄（changelog），以便未來對管線設定做版本控管？
是否需要將每次完整管線執行的結果（包含設定與輸出）永久儲存，以便稍後重新開啟與比對？
在醫院或公司環境中，是否需導入 角色權限控管（如稽核員可編輯、主管僅檢視）？
是否有需要加入 PII 自動遮蔽功能（如姓名、拜訪者 ID、病歷號碼），並提供可自訂的遮蔽規則？
是否希望支援 多種預設稽核類型（ISO 13485 / GMP / MDSAP / 供應商稽核），並在 UI 中以下拉選單切換對應的模板與 agents.yaml？
是否有 成本預估需求，希望在儀表板中根據 provider / model / token 使用量估算大致美金成本？
對 agents.yaml 的修改是否需要 嚴格 schema 驗證（例如使用 JSON Schema），並提供「設定檢查」視圖來顯示錯誤細節？
對於長篇報告，是否應提供 章節樹狀編輯介面（左側章節樹，右側編輯內容），取代單一大 textarea？
是否需要在輸出的 Markdown 或額外 JSON 中加入 元資料（如模型提供者、型號、時間戳記、代理 ID），以滿足稽核追溯需求？
若模型出現錯誤或安全過濾，是否希望在 agents.yaml 中可設定 自動 fallback 策略（例如：Gemini 失敗時自動改用 OpenAI）？
是否希望讓每個代理支援 進階 provider 設定（例如：Gemini 的 safety category、OpenAI 的 response_format），且可在 YAML 與 UI 中調整？
是否需要額外一個「系統自測分頁」，用固定測試範本跑一小段 pipeline 以驗證目前所有 provider / model 是否可用？
是否有需求實作 兩個完整管線結果的差異比較（diff） 功能，以方便比較 prompt 或模型升級前後的差異？
對於最終對外送交的正式報告，是否需要一個 「乾淨匯出模式」，自動移除 Smart Replace 中的 <span> 標記並合併所有代理輸出？
是否需要支援 雙語輸出報告（繁中 + 英文對照），例如每一段繁文後附英文翻譯？
是否需要加入專門的 「條文符合性檢查」代理，只檢查 ISO 13485 條文對應與符合性，而不更動報告敘述？
是否有 批次處理需求（如上傳多筆觀察清單，批次產生多份報告），並在 UI 中顯示進度條？
是否希望支援 多模態輸入（例如上傳不符合事項照片），由 Gemini / Grok 等多模態模型解析並自動描述於報告中？
若未來對外提供給多部門 / 多單位使用，是否需要整合 身分驗證機制（如 OAuth、公司 SSO）？
長期來看，是否希望在現有 Streamlit UI 之外，另外提供 REST / GraphQL API 介面，讓其他系統（QMS / DMS）可以程式化呼叫 AuditFlow AI 來產生稽核報告？
