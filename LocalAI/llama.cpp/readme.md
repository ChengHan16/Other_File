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
### 一鍵啟動
桌面新增文件 <br>
● C:\Users\LINGDU\Desktop\llama-b9196-bin-win-cuda-13.1-x64 為您的文件檔案路徑。 <br>
● "models\Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf" --mmproj "models\mmproj-BF16.gguf" 為模型名稱對應位置 "" 中的內容。
```
@echo off
chcp 65001 >nul
cd /d C:\Users\LINGDU\Desktop\llama-b9196-bin-win-cuda-13.1-x64

echo 请选择模型：
echo 1. Gemma 31B
echo 2. Qwen VL 多模态
echo 3. DeepSeek

set /p choice=输入数字：

if "%choice%"=="1" llama-server.exe -m "models\gemma-4-31b-jang-crack-Q4_K_M.gguf" -ngl 999
if "%choice%"=="2" llama-server.exe -m "models\Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf" --mmproj "models\mmproj-BF16.gguf" -ngl 999
if "%choice%"=="3" llama-server.exe -m "models\deepseek.gguf" -ngl 999

pause
```
