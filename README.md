# Tiefsee.QuickLook

Tiefsee.QuickLook 是提供給 Tiefsee 使用的擴充套件。Tiefsee 偵測到空白鍵或滑鼠中鍵的快速預覽操作時，會透過本專案取得目前檔案總管所選取的檔案路徑，再交由 Tiefsee 開啟預覽。

本專案的原生選取邏輯取自 [QL-Win/QuickLook](https://github.com/QL-Win/QuickLook)，並依 GPL-3.0-or-later 授權。授權全文見 [LICENSE-GPL.txt](LICENSE-GPL.txt)。

## 功能範圍

- 取得桌面或檔案總管目前選取的檔案。
- 原生程式包含部分檔案對話框，以及 Everything、Directory Opus、Multi Commander、Internet Download Manager、FilePilot 和 DeskBox 的視窗辨識與選取處理分支。這代表程式會嘗試取得這些來源的選取項目；各軟體版本和環境尚未逐一實機驗證。
- 選取項目為 `.lnk` 捷徑時，程式會呼叫 Shell Link 的 `GetPath` 取得目標路徑；若取得結果為空字串，則回傳原始捷徑路徑。
- 只回傳單一項目。若同時選取多個項目，原生程式會取第一個項目。
- 本套件負責查詢路徑，不負責監聽鍵盤或自行開啟預覽視窗；快速預覽觸發與視窗操作由 Tiefsee 主程式處理。

若焦點不在支援的桌面、檔案總管或管理程式，或目前沒有可取得的選取項目，API 通常會回傳空字串。

## API 使用方式

C# 公開 API：

    namespace Tiefsee
    {
        public class QuickLook
        {
            public static string GetCurrentSelection();
        }
    }

在使用端的 C# 專案中加入 `Tiefsee.QuickLook.dll` 組件參考，再呼叫下列方法。此組件以 .NET Framework 4.8 為目標，使用端必須能載入相容的 Windows 類別庫。

使用範例：

    using Tiefsee;

    string selectedPath = QuickLook.GetCurrentSelection();
    if (!string.IsNullOrEmpty(selectedPath))
    {
        // selectedPath 是目前取得的檔案完整路徑
        Console.WriteLine(selectedPath);
    }

`GetCurrentSelection()` 不需要先呼叫初始化方法；第一次呼叫時會自動初始化 Native 元件。此方法是同步呼叫，會等待原生程式完成選取查詢；若由有 UI 的程式使用，請避免在需要保持回應的 UI 執行緒上頻繁呼叫。

程式載入本套件時，C# 程式會依目前程序的位元數載入對應 Native DLL：64 位元程序載入 `QuickLook.Native64.dll`，32 位元程序載入 `QuickLook.Native32.dll`。部署時應讓 C# DLL 和兩個 Native DLL 位於同一套件目錄，並使用同一次建置產生的檔案。

## 專案結構

- `Tiefsee.QuickLook/`：C# 類別庫、公開 API、捷徑解析程式與 `package.json`。
- `Native/QuickLook.Native32/`：32 位元 Windows 原生 DLL 專案。
- `Native/QuickLook.Native64/`：64 位元 Windows 原生 DLL 專案，會共用 Native32 目錄中的原始碼。
- `Build/obj/`：C++ 編譯與連結的中間檔，包含 `.obj`、`.lib`、`.exp`、`.pdb` 等檔案。
- `Output/`：統一的建置輸出目錄。

方案中的 C# 專案依賴 Native32 與 Native64。方案組態 `Any CPU` 會把 Native32 對應至 `Win32`、Native64 對應至 `x64`，因此建置整個方案即可依序產生所有必要 DLL。

## 編譯環境

請在 Windows x64 開發環境準備：

1. Visual Studio 2026，或具有本方案所需 MSBuild／C++ 工具的相容 Visual Studio 安裝。
2. .NET Framework 4.8 Developer Pack／Targeting Pack，用來建置 C# 類別庫。
3. MSVC v145 x86/x64 C++ Build Tools。兩個 Native 專案目前都指定 `PlatformToolset` 為 `v145`；只有 MSVC v143 並不足以建置目前設定。
4. v145 對應的 ATL 標頭與工具，以及 Windows 10 SDK。原生程式使用 ATL COM 型別；若建置顯示找不到 `atlcomcli.h` 等 ATL 標頭，請在 Visual Studio Installer 確認已安裝 v145 的 ATL 元件，而非只安裝 v143 工具集。

Visual Studio Installer 的工作負載可選「使用 C++ 的桌面開發」，並在個別元件中確認上述 v145、ATL 與 Windows SDK 元件。元件名稱會隨 Visual Studio 版本而異，請以實際安裝器顯示的 v145 元件為準。

## 建置方式

### 使用 Visual Studio

1. 開啟 `Tiefsee.QuickLook.sln`。
2. 在工具列選擇 `Release` 與 `Any CPU`。
3. 執行「建置方案」。

請建置方案，而非只建置 `Tiefsee.QuickLook.csproj`。原生專案依賴關係設定在方案中；只建置 C# 專案可能只得到 `Tiefsee.QuickLook.dll`。

### 使用 MSBuild

在已載入 Visual Studio C++ 建置環境的 Developer PowerShell 或 Developer Command Prompt 中，從專案根目錄執行：

    msbuild .\Tiefsee.QuickLook.sln /t:Build /p:Configuration=Release "/p:Platform=Any CPU" /m

Debug 建置可將 `Release` 改成 `Debug`。Debug 與 Release 共用 `Output/`，切換組態會覆寫同名 DLL；Debug 可能另外產生 C# PDB。


## 建置輸出與部署

Release 建置的 `Output/` 會包含：

- `Tiefsee.QuickLook.dll`：C# 外掛。
- `QuickLook.Native32.dll`：32 位元原生元件。
- `QuickLook.Native64.dll`：64 位元原生元件。
- `package.json`：外掛名稱與版本資訊，由專案複製到輸出目錄。
- `LICENSE-GPL.txt`：授權文件，由專案複製到輸出目錄。

`.lib`、`.exp` 和 Native PDB 等連結中間檔放在 `Build/obj/`，不屬於部署檔案。請將同一次建置產生的三個 DLL、`package.json` 與授權文件視為一組部署；不要只更新其中一個 Native DLL。Tiefsee 預設的外掛載入位置為 `%LOCALAPPDATA%/Tiefsee/Plugin/QuickLook/`，實際位置會依 `Start.ini` 或可攜模式設定而變動；其中主程式會載入 `Tiefsee.QuickLook.dll`，Native DLL 需放在同一套件目錄。一般使用請透過 Tiefsee 提供的外掛安裝或更新方式；手動替換時先關閉 Tiefsee。

## 常見建置問題

### 找不到 `atlcomcli.h`

目前 Native 專案使用 v145。請確認已安裝與 v145 相符的 ATL 標頭和 x86/x64 C++ 工具；安裝 v143 不會補上 v145 所需的 ATL 標頭。

### Output 只有 `Tiefsee.QuickLook.dll`

通常是只建置了 C# 專案。請改為建置 `Tiefsee.QuickLook.sln` 的 `Release | Any CPU` 或 `Debug | Any CPU` 組態，讓方案中的 Native32、Native64 專案一併建置。
