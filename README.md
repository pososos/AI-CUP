# AI-CUP醫病語音敏感個人資料辨識競賽報告用
檔案說明:
附件檔案AICUP2025_final.ipynb，整個訓練到推論的流程在colab上就能直接運行完成。
使用手把手教學中相同的colab環境為基礎，建立Python 3的開發環境

主要套件:
1.	Task1: torch、librosa、pandas、datasets、transformers、whisperx
2.	Task2: torch、pandas、datasets、transformers、seqeval、evaluate

預訓練模型來源: 
1.	Task1: 使用目前較新的開源模型whisper-large-v3、Na0s/Medical-Whisper-Large-v3
2.	Task2: 使用在Token Classification類型中醫療相關預訓練多的michiyasunaga/BioLinkBERT-large

使用資料集:
1.	Task1僅使用競賽方提供的2份訓練集+1份驗證集進行訓練，無額外生成資料
2.	Task2除了使用競賽方提供的2份訓練集+1份驗證集，另外有用Chat-GPT的o3進階推理模型以競賽方提供出現過的標籤內容及比例、仿真生成2500筆三份共7500筆資料，其中每一筆皆包含10個敏感詞標籤
3.	訓練文字檔皆包含在training data.zip中

使用步驟:

Task 1：語音轉錄與時間對齊

執行流程

1.	資料前處理：將 ZIP 解壓後遍歷 WAV 檔案，使用 librosa 以 16 kHz 讀取音訊，並從多個 txt 檔中讀入每段音檔的文字標註，再打包成dataset切割0.8訓練，對token進行mapping。
2.	模型訓練：使用 WhisperForConditionalGeneration（large-v3），凍結 encoder，僅微調 decoder 以減少訓練成本並保持聲學知識穩定性。
3.	推論輸出：在推論時呼叫 generate(return_timestamps=True) 取得轉錄結果task1_answer.txt與 segment-level 的起訖時間，再透過 whisperx 進行 word-level 對齊。此過程中，重寫
    wildcard_emission 以支援特   殊 token，並分別載入中英文對齊模型，分段對英中文字段落做精細對齊，最後得到轉錄結果task1_answer.txt與 word-level 時間軸
    task1_word_timestamps.json。

Task 2：文字敏感資訊辨識與時間標註

執行流程

1.	資料前處理：將多份 TSV 拼接，依標籤與內容依格式執行分行拆分，再打包成dataset切割0.9訓練，將敏感詞 token 做 B-I-O 標記並透過 extended_to_base 字典整合同義標籤，最後  
    offset_mapping 對齊。
2.	模型訓練：採用 BioLinkBERT-large 作為 backbone，在最後加入 token classification 頭並微調全參數。Loss 採交叉熵，並以 seqeval 計算整體精準度、召回率與 F1。
3.	推論輸出：讀取 Task 1 得到的 task1_word_timestamps.json 設定時間戳輸入文字先切 chunk（長度 CHUNK_SIZE、重疊 OVERLAP）避免過長截斷，再以
    pipeline(aggregation_strategy="simple") 提取實體，並依類別特定閾值過濾。對於 TIME、ID_NUMBER、MRN 等關鍵欄位，額外使用精準 Regex 偵測與 fallback 演算法（只比對純數字序
  	列、單 token 或多 token 匹配），最後將時間軸依設定好敏感詞提取方式與轉錄結果task1_answer.txt對齊，得到最終結果提取的敏感詞+起始時間點+結束時間點task2_answer.txt。
