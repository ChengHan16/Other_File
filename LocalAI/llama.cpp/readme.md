### 環境建置
解壓縮 llama-b9196-bin-win-cuda-13.1-x64 後在 llama-b9196-bin-win-cuda-13.1-x64 中新增 models 資料夾將模型都放進去。

進入 llama-b9196-bin-win-cuda-13.1-x64 在路徑列中輸入 CMD 後 Enter

### 多模態模型啟用：
```
llama-server.exe -m "models\主模型.gguf" --mmproj "models\mmproj视觉模型.gguf" -ngl 999
```
### 啟動方式：進入 llama.cpp 目錄
例如：gemma-4-31b-jang-crack-Q4_K_M.gguf <br>
```
llama-server.exe -m models\你的模型.gguf -ngl 999 
```
