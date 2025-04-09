---
title: 深入解析 Shim：Windows 相容性修正與 Linux Secure Boot 信任鏈核心
date: 2025-04-10 20:00:00
tags:
  - Traditional Chinese
  - Windows
  - Linux
  - 資訊安全
categories:
  - Knowledge base
cover_image: 'https://kevinchiu923.github.io/post/深入解析-Shim：Windows-相容性修正與-Linux-Secure-Boot-信任鏈核心/zhtw_cover_img.png'
---
Shim 是一種關鍵的系統中介技術，透過 API 攔截或簽章驗證，提供應用程式相容性與Secure Boot 安全性，但同時也可能成為資安攻擊的切入點。
本文將著重介紹 Shim 的運作原理、實作方式、潛在風險與對應防護策略。

## 前言：Shim 是什麼？

在資訊系統中，Shim 是一種中介程式，負責「橋接」兩個不相容或無法直接溝通的系統元件；就如同機械工程中的 [墊片(gasket)](https://www.tfhc-oilseal.com/Gasketseal/%E5%BB%B6%E9%95%B7%E8%A8%AD%E5%82%99%E5%A3%BD%E5%91%BD,%E5%A2%9E%E5%8A%A0%E7%94%A2%E8%83%BD%E7%9A%84%E9%97%9C%E9%8D%B5%E9%9B%B6%E7%B5%84%E4%BB%B6-%E7%A9%BA%E5%A3%93%E6%B2%B9%E5%B0%81%E5%A2%8A%E7%89%87/) 一樣，用來填補介面之間的落差。Shim 可以在兩個元件之間增加一層轉換層，讓它們得以協同運作。

在電腦系統中，Shim 的形式多樣，但核心目的都是**攔截或調整原始行為，實現相容性或安全性**；本文將深入探討 Shim 在兩個主流作業系統中的應用：

- **Windows - Application Shim**：以「應用程式相容性 Shim」為主，處理老舊軟體與新系統間的相容問題。
- **Linux - Secure Boot Shim（UEFI 環境）**：Shim 作為 Secure Boot 信任鏈的第一環，確保開機流程安全性。


## Shim 在 Windows 與 Linux 中的差異比較

| 項目 | Windows Application Shim | Linux Secure Boot Shim |
| --- | --- | --- |
| 主要用途 | 解決舊應用與新版系統的相容性問題 | 建立 Secure Boot 的信任鏈起點 |
| 執行階段 | 使用者層級（User-mode） | 開機初期（UEFI Boot） |
| 主要功能 | 攔截 API、修改行為、模擬舊系統 | 驗證 GRUB 與 Kernel 的數位簽章 |
| 安裝方式 | 利用 `.sdb` 資料庫，透過工具安裝 | 隨發行版安裝的簽署程式（由微軟簽章） |
| 安全風險 | 可被惡意利用進行 Shim Injection | 過時 shim 版本有繞過 Secure Boot 的風險 |
| 常見應用 | 公司內部無法更新的舊系統、老遊戲 | 雙開系統、Linux 雲端映像啟動 |


## Windows：Application Compatibility Shim 的實作與資安考量

### 1. 應用場景與目的

Windows Shim 主要解決因作業系統升級導致的應用程式不相容問題。透過**攔截應用程式對 API 的呼叫**，進行行為轉換或版本「偽裝」，讓老程式能「誤以為」自己仍在舊環境執行。

### 2. 核心機制與運作原理

- **Shim 資料庫（.sdb）**：記錄程式與修正規則的對應。
- **Shim Engine**：動態攔截 API 呼叫，透過 DLL 注入來修改執行邏輯。
- **Compatibility Administrator 工具**：可自訂並安裝 Shim 修正組。

### 3. 建立自訂 Shim 的指令範例：

```bash
# 使用 Compatibility Administrator 建立 .sdb 後
sdbinst.exe C:\path\to\custom_shim.sdb
```

### 4. 查詢已安裝的 Shim 資料庫
- **適用於舊版 Windows：**
若使用的是舊版本的 Windows (如: Windows 7/8)，根據早期的技術資料中可以使用以下 PowerShell 指令列出安裝過的自訂 `.sdb` (參考資料 [1][2]):
    ```bash
    reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\InstalledSDB"
    ```

- **適用於新版 Windows：**
若使用的是較新版本的 Windows (如: Windows 10/11)，以上述指令執行可能會發生查找不到的狀況。這是因為新版本中的 `AppCompat` 資料庫雖然存在，但不一定會出現在 `InstalledSDB` 下，而是散布在其他註冊機碼 (registry key) 中，可用以下兩種 PowerShell 指令查詢:
    - 列出針對特定路徑的應用程式所套用的 Shim 行為，也就是「Custom Shim」
        ```bash
        reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Custom"
        ```
    - 顯示使用者手動設定的相容性層（Compatibility Layers），例如用滑鼠右鍵設定「以 Windows 7 相容模式執行」的那些項目
        ```bash
        reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers"
        ```


**為什麼找不到 InstalledSDB？**
至於為什麼新版本 Windows 中會有 `InstalledSDB` 缺失的問題呢? 主要原因有以下三點:
1. 並非所有 `.sdb` 安裝都會登記在 InstalledSDB 下
2. 有些系統版本根本不建立這個鍵值
3. 微軟改變了 Shim 的儲存或管理方式（特別是近年趨向 WDAC 和 AppLocker 等更嚴格控管）

### 5. 常見 Shim 應用實例

- `VersionLie`：偽裝成特定 Windows 版本
- `ForceAdminAccess`：欺騙應用程式認為目前使用者是 Admin
- `CorrectFilePaths`：將檔案路徑重導至現代系統中可用位置

### 6. 資安風險：Shim Injection

攻擊者可自製 `.sdb` 檔並利用 `sdbinst.exe` 安裝至系統中，當使用者啟動特定程式時，自動注入惡意程式碼。例如[知名駭客集團 FIN7](https://malpedia.caad.fkie.fraunhofer.de/actor/fin7) 就曾使用此技術 (參考資料 [3])。


## Linux：Shim Loader 在 Secure Boot 中的角色與實踐

### 1. Secure Boot 簡介

UEFI Secure Boot 確保開機流程中執行的程式碼皆經由簽章驗證。韌體通常僅信任由微軟 UEFI CA 所簽署的開機元件。

### 2. Linux 為什麼需要 Shim？

Linux 發行版透過 shim 避免每次 kernel 或 bootloader 更新都需向微軟簽章。
Shim 自身由微軟簽署，而它內嵌的發行版公鑰可用來驗證 GRUB 與 Linux kernel。

### 3. 運作流程：Shim 的三段式信任鏈 (Three-stage Trust Chain)
三段式信任鏈是 Linux 在 UEFI Secure Boot 環境中，用以確保開機流程安全與完整性的核心架構。
其主要目的，是確保從 UEFI 韌體啟動開始，經由 GRUB，到最終載入 Linux 核心的每一個階段裡所執行的元件皆為經簽章驗證、未遭竄改可信的程式碼。

整體信任鏈依序包含 Firmware、Shim、GRUB 與 Kernel 四個關鍵元件，其中每一階段的簽章與驗證機制，分別由不同單位負責：
Firmware 通常由硬體廠商提供，Shim 則由 Linux 發行版開發並交由微軟簽章，GRUB 和 Kernel 則由發行版簽署，並透過 Shim 或 MOK 機制進行驗證，共同構成一條完整且可追溯的信任路徑。

![](https://kevinchiu923.github.io/post/深入解析-Shim：Windows-相容性修正與-Linux-Secure-Boot-信任鏈核心/Three-stage-Trust-Chain.jpeg)

詳細流程如下：

**Step 1：Firmware → shim(由微軟簽署)**
| 項目 |說明 |
| -------- | -------- |
| 啟動元件     | UEFI 固件     |
| 執行對象     | shimx64.efi (由 Linux 發行版提供，已由微軟簽章)     |
| 驗證方式     | 使用 Microsoft UEFI CA 簽章來驗證 Shim |
| 目標     | 讓非微軟提供的 bootloader (如 GRUB)能在 Secure Boot 下被合法載入     |

**Step 2：shim → GRUB (或其他 bootloader)（由發行版簽署)**
| 項目 |說明 |
| -------- | -------- |
| 啟動元件     | shimx64.efi     |
| 執行對象     | grubx64.efi (由 Linux 發行版簽章)     |
| 驗證方式     | 使用內嵌在 Shim 中的公鑰，或從 MOK (Machine Owner Key) 資料庫驗證 |
| 目標     | 載入下一階段 bootloader (如 GRUB)，建立信任傳遞關係     |

**Step 3：GRUB → Kernel（使用 shim 提供之驗證功能）**
| 項目 |說明 |
| -------- | -------- |
| 啟動元件     | GRUB     |
| 執行對象     | vmlinux (Linuex kernel 映像檔)     |
| 驗證方式     | 使用 Shim 或 GRUB 的簽章驗證模組，驗證 kernel 簽章 |
| 目標     |確保啟動的 kernel 也經過驗證  |

這三個階段分別由微軟、Linux 發行版與使用者管理，形成一條 **「分層簽署、階段驗證」** 的完整分層信任架構。

### 4. 管理自簽金鑰：MOK（Machine Owner Key）

允許使用者手動註冊自定義簽名金鑰，然後可以使用這些金鑰來驗證第三方驅動程式或自定義編譯的 Linux 內核。
註冊后，相應的公鑰將儲存在 [NVRAM（非揮發記憶體）](https://en.wikipedia.org/wiki/Non-volatile_memory)中成為安全啟動驗證過程的一部分，並在 Secure Boot 流程中由 [MokManager](https://github.com/hakuna-m/wubiuefi/wiki/MOKManager) 處理。


新增 MOK 流程（以 Ubuntu 為例）：
```bash
sudo mokutil --import my_key_pub.der
# 系統將提示設定密碼，重新開機後進入 MokManager 加入金鑰
```

### 5. 相關資安風險回顧
於 2020 年爆出的 **BootHole ([CVE-2020-10713](https://nvd.nist.gov/vuln/detail/cve-2020-10713))** 漏洞，促使了整個 UEFI Secure Boot 生態系迎來一場大革新。
這個漏洞存在於 GRUB2 bootloader 的設定解析邏輯中，攻擊者可以藉由篡改 grub.cfg 來注入惡意程式碼，即使在 Secure Boot 啟用的情況下，也能繞過驗證機制、執行未經簽署的內容。
這不僅暴露了 Secure Boot 信任鏈的單點風險，更凸顯了傳統撤銷 (revocation) 機制（如 dbx）的侷限性。

為了解決這類系統性風險，Linux 社群與多家 UEFI 相關廠商（如 Canonical、Red Hat 等）合作，設計出一套名為 **[SBAT（Secure Boot Advanced Targeting）](https://www.gnu.org/software/grub/manual/grub/html_node/Secure-Boot-Advanced-Targeting.html)** 的新機制。
SBAT 允許系統更精確的辨識和撤銷具有風險的開機元件版本，例如特定版本的 shim、GRUB 或 Linux kernel，避免因一次撤銷而連帶影響所有使用者與版本，提升了 Secure Boot 的彈性與可維運性。

## 結語：Shim 是一把雙面刃

Shim 在現代系統中扮演橋樑角色：

- Windows 的 Shim 強化了舊應用程式的生命週期
- Linux 的 Shim 維繫 Secure Boot 的完整性與信任鏈

唯有深入理解其機制，才能有效運用其優勢、提早防範可能的安全風險。

### 為什麼說 Shim 是雙面刃？
#### 優點：
- **維持相容與開放性**：Shim 提供了強大的彈性與便利性，允許舊應用程式或自訂驅動在現代環境中繼續運作，降低維護與升級成本。

#### 缺點：
- **潛在安全破口**：同樣的機制若被濫用，可能成為惡意程式碼的載具，例如 Windows 中的 Shim injection 或 Linux 過期 shim 被攻擊者利用。
- **信任鏈單點風險**：特別在 Secure Boot 環境中，shim 是信任鏈的起點，一旦 shim 本身有漏洞或被取代，整條信任機制將形同虛設。
- **維護成本與撤銷困難**：Shim 一旦廣泛部署，若需進行更新（如 SBAT 撤銷）常會伴隨使用者無法開機等風險與操作複雜度。

因此，在部署與使用 shim 時，應同時考慮便利性與安全性，並持續追蹤官方針對 shim 更新與安全通報。

### 因應對策與防護建議
在實際部署中，建議企業與開發人員落實版本控管、簽章政策與集中式管理機制，並持續關注 Shim 與 Secure Boot 的安全通報，以降低潛在攻擊風險。具體如下：
1. 維持相容與開放性
    - 僅針對無可替代的關鍵應用程式使用 Shim，避免全面啟用造成過度依賴。
    - 導入完整的變更控管(Change Control / Change Management) 與測試流程(如 [ISO/IEC 27001](https://www.iso.org/standard/27001)、[NIST 800-53](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final))，針對每項 Shim 設定進行風險評估、功能驗證與版本記錄，以確保相容性需求與系統安全性之間的平衡。
    - 搭配應用程式白名單機制（如 [AppLocker](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/what-is-applocker)、[WDAC](https://learn.microsoft.com/en-us/hololens/windows-defender-application-control-wdac)）限制執行來源。
2. 潛在安全破口
    - 建立監控機制，偵測如 sdbinst.exe 執行、異常的 Shim 資料庫變更等行為。
    - 僅授予系統管理員權限安裝 Shim，並限制使用者層級存取 Shim 管理工具。
    - 定期稽核 %WINDIR%\AppPatch\Custom\ 和註冊機碼下的 Shim 記錄。
3. 信任鏈單點風險
    - 僅使用官方發行、由 Microsoft 簽署的 shim 映像檔。
    - 部署支援 SBAT 的 shim 版本，以便精確撤銷與進行版本管控。
    - 定期更新 shim、GRUB、Kernel 與 UEFI 韌體，以防範已知漏洞。
4. 維護成本與撤銷困難
    - 使用集中化管理工具（如 [Intune](https://learn.microsoft.com/en-us/intune/intune-service/fundamentals/what-is-intune)、[Ansible](https://www.redhat.com/en/ansible-collaborative)）統一佈署與監控 Shim 使用情形。
    - 規劃開機備援方案，例如 USB live 磁區、Recovery 分割區，以因應 Shim 更新後的開機失敗。
    - 建立版本記錄與 SHA256 雜湊校驗機制，以利驗證與回溯分析。

## 參考資料
[1] Black Hat Europe 2015 Whitepaper -[《Defending Against Malicious Application Compatibility Shims》](https://www.blackhat.com/docs/eu-15/materials/eu-15-Pierce-Defending-Against-Malicious-Application-Compatibility-Shims-wp.pdf)
[2] F-Secure Blog post - [《Hunting for Application Shim Databases》](https://blog.f-secure.com/hunting-for-application-shim-databases/)
[3] Google Cloud - [To SDB, Or Not To SDB: FIN7 Leveraging Shim Databases for Persistence](https://cloud.google.com/blog/topics/threat-intelligence/fin7-shim-databases-persistence/)