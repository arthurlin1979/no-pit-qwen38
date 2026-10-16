# 不踩坑-qwen3.8

[English](./README.en.md) | 中文版

Qwen 這系列模型，腦子是真的不錯，但格式遵循（format following）很瞎。
尤其是量化（quantization）壓得越小越離譜：工具呼叫（tool call）失敗就一直重試，
還會冒奇怪字元，對話講到一半直接斷掉。所以我改了它的輸出模板（chat template），
讓它回到該有的樣子。

只要吃 jinja 的地方都能套（LM Studio / llama.cpp / vLLM / SGLang / oMLX），
我自己這系列的模型全上了。

## 三個版本，拿一個就好

| 版本 | 給誰 | 裡面有什麼 | 檔案 |
|---|---|---|---|
| v6 | NVFP4 這種乖模型 | 繁中一句＋單發限制（SERIAL RULE）＋輸出契約（output-contract） | `v6/` |
| v6.2 | 小量化＋破解（uncensored）的，預設拿這個 | v6＋空腦包清理＋未閉合自斷句＋動手前關腦 | `v6.2/` |
| v6.2-alt | 愛實驗的人 | v6.2＋7 樣沒證實的。有人說有效有人說沒效，自己試，後果自負 | `v6.2-alt/` |

## 我到底改了什麼（上游的我沒動）

基底是人家修好的 `froggeric/Qwen-Fixed-Chat-Templates` v22.5（459 行，Apache-2.0，
作者 Frédéric Guigand）。我只動 7 個地方（hunk），其他逐字一樣，
證據在 `v6/v6_vs_upstream.diff`。第一行版號是浮水印（watermark），
模型更新會洗掉客製模板，載入前對一下就知道還在不在。

* 單發限制（SERIAL RULE）：一輪只准打一通工具電話。說句實話這條最有感，
  實測原版一次發 2 通，v6 只發 1 通，小模型併發寫壞的老毛病直接沒了。
* 繁中規則：中文一律繁體台灣用法。坦白說這東西我本來寫在 serve 端，
  結果條件寫錯從沒生效過，搬進模板才全員吃到。
* 輸出契約（output-contract）：要 raw JSON / 純原始碼的時候不包 markdown、不加廢話。
* v6.2 多三樣：空腦包（empty think wrapper）不渲染、腦沒關好遇到工具自己斷句、
  動工具前必須關腦（MUST close thinking）。

每一塊的細節跟有沒有用，看 [BLOCKS.md](./BLOCKS.md)，沒用的我老實講。

## 先講壞消息（測過才敢貼出來）

* 簡體滲漏（simplified-character leak）：長文四篇漏一篇，哪個版本都一樣，
  模板治不了，是模型本身的天花板。短任務倒是全淨。
* 思考脹滿（thinking-budget exhaustion）：破解版寫長文會把幾千個 token
  全拿去想，正文一個字都沒有，換什麼模板都一樣，要 engine 側去限。
* alt 那 7 樣：單測全都沒跑贏基線，僅供參考。

## 怎麼用

```bash
# vLLM
vllm serve <model> --chat-template ./v6.2/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang（注意：0.5.x 會無視模型目錄裡的模板，一定要顯式指定）
python -m sglang.launch_server --model-path <model> --chat-template ./v6.2/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v6.2/chat_template.jinja
# LM Studio：Prompt Template 貼上存檔 / oMLX：蓋掉模型目錄那份再重啟
```

乖模型把路徑換成 `./v6/chat_template.jinja` 就好。

## 檔案

* `v6/`：v6 成品＋跟上游的差異
* `v6.2/`：v6.2 成品＋跟 v6 的差異（`v62_vs_v6.diff`）
* `v6.2-alt/`：參考版成品
* `lab/strict-v6.1.jinja`：v6.1 實驗草稿，退回了，留著當紀念

有更好的寫法歡迎提出來。
