# 不踩坑-qwen3.8

> Qwen 系列的模型很不錯，但格式遵循不是很準，尤其量化壓得越小，格式出問題越明顯。工具呼叫失敗就一直重試，還容易冒出奇怪字元，對話忽然中斷。所以要特別改輸出模板，讓這整個系列回到它該有的表現。

只要是吃 jinja 的地方都能套，我自己這系列的模型都有上。

## Base pin

* 上游：`froggeric/Qwen-Fixed-Chat-Templates` main v22.5，459 行，2026-10-13 現抓
  * 來源：https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates
  * 授權：Apache-2.0 inherited from Qwen，原作者 Frédéric Guigand
* 成品：`chat_template.jinja` 428 行，26292B
* 浮水印：`qwen3.8-froggeric-v22.5-arthur-strict-v6`＋sha256 `3f092bdc58099b6ebcaab8698d6d71cb501c357cb90111d1708e57165c114871`，load 前必驗，上游更新會洗掉客製模板

Credits：Qwen 官方結構 / froggeric 修復本體，授權 Apache-2.0，修改部分同授權。

## 為什麼不用上游就好

上游是全功能，你要的是最小可審計。這篇只改 7 hunk：常駐 output-contract、SERIAL RULE 一輪一 call、繁中一句。其餘逐字同上游，benchmark 才可比。

## 7 hunk 對照（見 `v6_vs_upstream.diff`）

1. H1 版頭：改版字串，刪 `_tool_format`，實務從不送那個 kwarg
2. H2 常駐：`_lang_rule`＋`_strict_output-contract`，raw JSON / source-only 不包 markdown
3. H3-H4 指令：砍 JSON 指令分支，留 XML＋SERIAL
4. H5-H6 渲染：砍 JSON 渲染分支
5. H7 截斷：砍 `_is_json_payload` 例外，截斷一視同仁

繁中從 strata `frontend.py` 髒檔搬進模板，無 system 才插的條件在 routed lane 從未生效。

## 用法

```bash
# vLLM
vllm serve <model> --chat-template ./chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang
python -m sglang.launch_server --model-path <model> --chat-template ./chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./chat_template.jinja --reasoning-format deepseek
# LM Studio：Prompt Template 貼上存檔 / oMLX：蓋模型目錄那份重啟
```

## 自測 5 項

思考語言 / 工具呼叫 / 雙工具只發一個 / 多 system / raw JSON 一字不差。離線 9/9＋三機 0 簡體。

## Lab v6.1

`lab/strict-v6.1.jinja` 27007B，多 hygiene＋completion，A/B 無增益退回 v6。第三條連錯警告上游 Two-Tier 已有。有更好的寫法也歡迎提出來。

## 已知邊界

SERIAL 會慢半拍，靠 goal 續跑＋外層並行補；temp-1 偶發單字簡體；只驗 Qwen3.8 系列。

## 檔案

* `chat_template.jinja`：v6 成品
* `v6_vs_upstream.diff`：相對上游 v22.5 的 7 hunk 差異
* `lab/strict-v6.1.jinja`：v6.1 實驗草稿，歡迎自測
