# 不踩坑-qwen3.8

[English](./README.en.md) | 中文版

本專案用於改善 Qwen3.8 系列模型的輸出模板（chat template）。Qwen3.8
模型能力優秀，但格式遵循（format following）不穩定：量化（quantization）
程度越小，工具呼叫（tool call）格式錯誤越頻繁，常伴隨異常字元與對話
異常中斷。本專案透過客製輸出模板修正上述問題。

適用範圍：所有支援 Jinja 模板的推理引擎（LM Studio / llama.cpp /
vLLM / SGLang / oMLX）。

## 上游出處（upstream）

* 基底模板：[froggeric/Qwen-Fixed-Chat-Templates](https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates)（v22.5，`855bffc`，Apache-2.0，原作者 Frédéric Guigand）
* 原始模型模板：[Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)（阿里 Qwen 團隊）
* 本專案為獨立衍生作品，非 GitHub fork；授權鏈見 [NOTICE](./NOTICE)。

## 版本說明

| 版本 | 適用對象 | 內容概要 | 位置 |
|---|---|---|---|
| v6 | NVFP4 等格式遵循良好的模型 | 繁體中文規則＋單發限制（SERIAL RULE）＋輸出契約（output-contract） | `v6/` |
| v6.2 | 小量化＋破解（uncensored）模型（預設建議版本） | v6＋空思考清理＋未閉合思考自動斷句＋動用工具前關閉思考 | `v6.2/` |
| v6.2-alt | 實驗參考用 | v6.2＋7 項尚未證實效益的附加項；效果因人而異，請自行驗證 | `v6.2-alt/` |

## 實測範圍

* 對齊目標：Strata 架構上的 Qwen3.8 系列模型。
* 模型精度：NVFP4、IQ3、IQ2，含去審查版（uncensored）。
* 驗證硬體：RTX PRO 6000、RTX 5090、RTX 5060 Ti 16G、Mac Studio。

## 修改內容（僅列本專案的變更）

基底為 `froggeric/Qwen-Fixed-Chat-Templates` v22.5（459 行，Apache-2.0，
原作者 Frédéric Guigand）。v6 相對於上游僅 7 個差異塊（hunk），其餘逐字
相同，完整差異見 `v6/v6_vs_upstream.diff`。模板第一行版本字串為浮水印
（watermark），因模型更新可能覆寫客製模板，載入前請核對。

* 單發限制（SERIAL RULE）：每輪僅允許一個工具呼叫（tool call）區塊。
  實測：上游原版單輪發出 2 個呼叫，v6 僅發出 1 個，可避免小模型併發
  呼叫導致的格式損毀。
* 繁體中文規則（Traditional-Chinese rule）：中文輸出統一採用繁體中文
  （台灣慣例）。原規則置於服務層但觸發條件錯誤而從未生效，移入模板後
  所有客戶端均適用。
* 輸出契約（output-contract）：請求 raw JSON／純原始碼時，僅輸出請求的
  內容，不附加 Markdown 標記與說明文字。
* v6.2 新增三項：歷史紀錄中省略空的思考（think）包裝、未閉合思考遇
  工具呼叫時自動斷句、動用工具前必須關閉思考區塊。

各區塊詳細說明與實證狀態請見 [BLOCKS.md](./BLOCKS.md)。

## 已知限制（經實測確認）

* 簡體字滲漏（simplified-character leak）：長篇中文輸出約四分之一機率
  出現簡體字，各版本皆然，屬模型本身限制，模板無法根治；短篇輸出正常。
* 思考預算耗盡（thinking-budget exhaustion）：破解版模型處理長篇任務時，
  可能將全部 token 消耗於思考過程而無正文輸出，更換模板無法改善，需
  推理引擎側的預算控制機制。
* alt 版 7 個附加項：隔離測試均未超越基線，僅供實驗參考。

## 使用方式

```bash
# vLLM
vllm serve <model> --chat-template ./v6.2/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang（注意：0.5.x 版會忽略模型目錄內模板，須顯式指定）
python -m sglang.launch_server --model-path <model> --chat-template ./v6.2/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v6.2/chat_template.jinja
# LM Studio：貼入 Prompt Template 後存檔 / oMLX：覆寫模型目錄檔案後重啟
```

格式遵循良好的模型請使用 `./v6/chat_template.jinja`。

## 檔案結構

* `v6/`：v6 成品＋與上游的差異比對
* `v6.2/`：v6.2 成品＋與 v6 的差異比對（`v62_vs_v6.diff`）
* `v6.2-alt/`：實驗參考版成品
* `lab/strict-v6.1.jinja`：已退役的 v6.1 實驗草稿

歡迎提出改進建議（Pull Request）。
