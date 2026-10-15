# 不踩坑-qwen3.8

[English](./README.en.md) | 中文版

> Qwen 系列的模型很不錯，但格式遵循（format following）不是很準，尤其量化（quantization）壓得越小，格式出問題越明顯。工具呼叫（tool call）失敗就一直重試，還容易冒出奇怪字元，對話忽然中斷。所以要特別改輸出模板（chat template），讓這整個系列回到它該有的表現。

只要是吃 jinja 的地方都能套（LM Studio / llama.cpp / vLLM / SGLang / oMLX），我自己這系列的模型都有上。

## 三個版本怎麼選

| 版本 | 給誰用 | 內容 | 檔案 |
|---|---|---|---|
| v6 | NVFP4 等乖模型 | 繁中一句＋單發限制（SERIAL RULE）＋輸出契約（output-contract） | `v6/` |
| v6.2 | 小量化＋破解（uncensored）模型，預設上線 | v6＋空腦包清理＋未閉合自斷句＋動手前關腦 | `v6.2/` |
| v6.2-alt | 測試參考版 | v6.2＋禁令清單＋收尾自檢等 7 樣未證實的；有人說有效有人說沒效，後果自負 | `v6.2-alt/` |

## 為什麼這樣寫（只講我們改的）

基底是 `froggeric/Qwen-Fixed-Chat-Templates` v22.5（459 行，Apache-2.0，
原作者 Frédéric Guigand），v6 相對上游只有 7 個差異塊（hunk），其餘逐字相同，
見 `v6/v6_vs_upstream.diff`。第一行版號是浮水印（watermark），模型更新會洗掉
客製模板，載入前必驗。

* 單發限制（SERIAL RULE）：一次只准打一通工具電話，防小模型併發寫壞格式。
  實測：原版一次發 2 通 → v6 只發 1 通。
* 繁中規則：`When responding in Chinese, use Traditional Chinese with
  Taiwan conventions.`（原本寫在 serve 端髒檔，從未生效，搬進模板才全員吃到）。
* 輸出契約（output-contract）：要 raw JSON / 純原始碼時不包 markdown。
* v6.2 加三樣：空腦包（empty think wrapper）不渲染、未閉合思考遇到工具呼叫
  自動斷句、動工具前必須關腦（MUST close thinking）。

## 誠實聲明（測過才敢講）

* 簡體滲漏（simplified-character leak）：長文四篇漏一篇，哪個版本都一樣，
  模板治不了，是模型本身的天花板。短任務全淨。
* 思考脹滿（thinking-budget exhaustion）：破解版寫長文會把幾千個 token
  全拿去想、正文一個字沒有，換什麼模板都一樣，要 engine 側限制。
* alt 版 7 樣：單測都沒跑贏基線，僅供實驗。

## 用法

```bash
# vLLM
vllm serve <model> --chat-template ./v6.2/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang（注意：0.5.x 會忽略模型目錄內模板，必須顯式指定）
python -m sglang.launch_server --model-path <model> --chat-template ./v6.2/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v6.2/chat_template.jinja
# LM Studio：Prompt Template 貼上存檔 / oMLX：蓋模型目錄那份重啟
```

乖模型把路徑換成 `./v6/chat_template.jinja` 即可。

## 檔案

* `v6/`：v6 成品＋相對上游差異
* `v6.2/`：v6.2 成品＋相對 v6 差異（`v62_vs_v6.diff`）
* `v6.2-alt/`：參考版成品
* `lab/strict-v6.1.jinja`：v6.1 實驗草稿（已退回，留念）

有更好的寫法也歡迎提出來。
