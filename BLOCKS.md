# 自加區塊簡介（只講我們動的，上游的不講）

寫法：一句白話＋英文術語＋在哪版＋證據狀態（有效／未證／有害）。

## v6（7 個差異塊，3 留 2 刪）

1. 版號浮水印（version watermark）：第一行 `template_version` 標 `...-arthur-strict-v6`。
   模型更新會洗掉客製模板，載入前對這行就知道有沒有被換掉。有效（維運）。
2. 繁中規則（Traditional-Chinese rule）：中文一律繁體台灣用法。
   原本寫在 serve 端髒檔，從沒生效過，搬進模板才全員吃到。短任務有效，長文天花板在模型。
3. 輸出契約（output-contract）：要 raw JSON／純原始碼時，不包 markdown、不加解說。
   刁鑽題已驗。有效。
4. 單發限制（SERIAL RULE）：一輪只准一通工具電話（tool call），發完等結果。
   實測原版一次發 2 通 → v6 只發 1 通。有效。
5. 刪 JSON 工具模式（remove JSON tool mode）：`_tool_format` 開關＋兩段 JSON
   分支＋截斷例外全砍。從沒人送那個參數，留著是地雷。有效（減法）。
6. 刪關鍵字偵測（remove keyword trigger）：舊版看到「原始碼＋只回傳」就加契約，
   英文測試句也會誤觸。跟常駐契約重複，砍。有效（減法）。

## v6.2 加三樣（全有效，5060 在線版）

7. C1 空腦不包（skip empty think）：歷史裡沒想法的回合，不包空的 `<think></think>`。
   以前這種空包會污染歷史、吃掉快取（cache）。離線鐵證＋線上無壞。有效。
8. C2 未閉合自斷句（unclosed-think auto-split）：腦沒關好就遇到工具呼叫，
   自動在工具處斷句，前半當想法、後半留原樣。離線鐵證＋線上無壞。有效。
9. MUST-close（動手前關腦）：先想的話，必須 `</think>` 關好才准發工具，
   絕不省略開標籤。全電池＋三輪連打過。有效。

## v6.1 實驗室（lab，已退回）

10. hygiene 空回禁令（no-empty-response）：每輪不是一通工具就是一段話，
    不准回空的。5092 簡單題量不出差別。未證，參考用。
11. completion 收尾自檢（completion check）：答前重讀題目對欄位。
    複雜題三方打平。未證，參考用。

## alt 版加七樣（參考用，有人說有效有人說沒效）

12. 繁體禁令清單（simplified-character ban list）：點名簡體字不准用。
    決勝局 12 發對 12 發沒贏基線。未證，參考用。
13. xhigh 終止惜字句（xhigh terminator）：叫深度思考早點收尾、別一直重複。
    小題目測不出，大場面沒重現。未證，參考用。
14. C3 中途提醒（system-reminder）：對話中途的 system 指令包成提醒塞進 user 輪，
    不整串失敗。基線本來就不炸，無增益。未證，參考用。
15. 工具解包（tool-envelope unwrap）：OpenAI 那層殼拆掉再渲染，省 token，
    沒定義殼時原樣 fallback。安全但無行為增益。未證，參考用。
16. high 獨立層（independent high effort）：high 不再併入 xhigh，
    有自己的指令。路由通，品質差異沒人量過。未證，參考用。

## 不收的

* seed 思考種子（think seed `好，`）：英文問題被帶成中文，有害，
  戰報留案，不進任何版。
