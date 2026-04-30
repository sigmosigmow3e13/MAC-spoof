здарова!
вот  исходник:

с++

#include <iostream>
#include <cstdlib>
#include <ctime>
#include <windows.h>
#include <string>
#include <vector>

using namespace std;

bool CheckAdmin() {
    BOOL isAdmin = FALSE;
    SID_IDENTIFIER_AUTHORITY NtAuthority = SECURITY_NT_AUTHORITY;
    PSID AdminGroup;

    if (AllocateAndInitializeSid(&NtAuthority, 2, SECURITY_BUILTIN_DOMAIN_RID,
        DOMAIN_ALIAS_RID_ADMINS, 0, 0, 0, 0, 0, 0, &AdminGroup)) {
        CheckTokenMembership(NULL, AdminGroup, &isAdmin);
        FreeSid(AdminGroup);
    }
    return isAdmin == TRUE;
}

void UpdateNetwork() {
    cout << "\nОбновляю сетевые настройки...\n";

    cout << "Отключаю адаптеры...\n";
    system("powershell -Command \"Get-NetAdapter | Where {$_.Status -eq 'Up'} | Disable-NetAdapter -Confirm:$false\" 2>nul");
    system("timeout /t 2 > nul");

    cout << "Очищаю кэш...\n";
    system("ipconfig /flushdns 2>nul");
    system("arp -d * 2>nul");

    cout << "Включаю адаптеры...\n";
    system("powershell -Command \"Get-NetAdapter | Enable-NetAdapter -Confirm:$false\" 2>nul");
    system("timeout /t 1 > nul");

    cout << "Обновляю IP...\n";
    system("ipconfig /renew 2>nul");

    cout << "Сетевые адаптеры обновлены.\n";
}

void ShowMAC() {
    cout << "\nТекущие MAC адреса:\n";
    system("getmac /v /fo list");
    cout << "===========================\n\n";
}

struct RegistryInfo {
    string key;
    string adapterName;
    string oldMAC;
    string newMAC;
};

vector<RegistryInfo> ChangeMAC() {
    cout << "\nГенерирую новые MAC адреса...\n";

    srand((unsigned)time(0));
    vector<RegistryInfo> changes;

    unsigned char macEth[6], macWifi[6];

    for (int i = 0; i < 6; i++) {
        macEth[i] = rand() % 256;
        macWifi[i] = rand() % 256;
    }

    macEth[0] = (macEth[0] & 0xFC) | 0x02;
    macWifi[0] = (macWifi[0] & 0xFC) | 0x02;

    char showEth[18], showWifi[18];
    snprintf(showEth, sizeof(showEth), "%02X-%02X-%02X-%02X-%02X-%02X",
        macEth[0], macEth[1], macEth[2], macEth[3], macEth[4], macEth[5]);
    snprintf(showWifi, sizeof(showWifi), "%02X-%02X-%02X-%02X-%02X-%02X",
        macWifi[0], macWifi[1], macWifi[2], macWifi[3], macWifi[4], macWifi[5]);

    cout << "Новые MAC адреса:\n";
    cout << "Ethernet: " << showEth << endl;
    cout << "Wi-Fi:    " << showWifi << endl;

    char regEth[13], regWifi[13];
    snprintf(regEth, sizeof(regEth), "%02X%02X%02X%02X%02X%02X",
        macEth[0], macEth[1], macEth[2], macEth[3], macEth[4], macEth[5]);
    snprintf(regWifi, sizeof(regWifi), "%02X%02X%02X%02X%02X%02X",
        macWifi[0], macWifi[1], macWifi[2], macWifi[3], macWifi[4], macWifi[5]);

    cout << "\nПоиск сетевых адаптеров в реестре...\n";

    string findCmd = "powershell -Command \""
        "$base = 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\Class\\{4d36e972-e325-11ce-bfc1-08002be10318}'; "

        "Get-ChildItem $base | ForEach-Object { "
        "    $key = $_.PSChildName; "
        "    $desc = (Get-ItemProperty $_.PSPath).DriverDesc; "
        "    $mac = (Get-ItemProperty $_.PSPath).NetworkAddress; "
        "    "
        "    if ($desc -and $desc -notmatch '(Miniport|Microsoft.*Virtual|1394)') { "
        "        if ($desc -match '(Wi.*Fi|Wireless|WLAN|802\\.11)') { "
        "            Write-Host 'Wi-Fi адаптер: ' $desc ' (Ключ: ' $key ')'; "
        "            $regPath = $base + '\\\\' + $key; "
        "            Set-ItemProperty -Path $regPath -Name 'NetworkAddress' -Value '" + string(regWifi) + "' -Force; "
        "        } "
        "        else { "
        "            Write-Host 'Ethernet адаптер: ' $desc ' (Ключ: ' $key ')'; "
        "            $regPath = $base + '\\\\' + $key; "
        "            Set-ItemProperty -Path $regPath -Name 'NetworkAddress' -Value '" + string(regEth) + "' -Force; "
        "        } "
        "    } "
        "}; "
        "\"";

    system(findCmd.c_str());

    cout << "\nПерезагрузка адаптеров...\n";

    string restartCmd = "powershell -Command \""
        "Get-NetAdapter | ForEach-Object { "
        "    Disable-NetAdapter -Name $_.Name -Confirm:$false -ErrorAction SilentlyContinue; "
        "    Start-Sleep -Milliseconds 500; "
        "    Enable-NetAdapter -Name $_.Name -Confirm:$false -ErrorAction SilentlyContinue; "
        "}\"";

    system(restartCmd.c_str());

    system("ipconfig /flushdns 2>nul");
    system("arp -d * 2>nul");

    cout << "\nMAC адреса успешно изменены в реестре!\n";
    cout << "Для полного применения требуется перезагрузка компьютера.\n";

    return changes;
}

void CheckRegistryChanges() {
    cout << "\nПроверка изменений в реестре...\n";

    string checkCmd = "powershell -Command \""
        "$base = 'HKLM:\\SYSTEM\\CurrentControlSet\\Control\\Class\\{4d36e972-e325-11ce-bfc1-08002be10318}'; "
        "Write-Host 'Текущие MAC в реестре:'; "
        "Get-ChildItem $base | ForEach-Object { "
        "    try { "
        "        $desc = (Get-ItemProperty $_.PSPath -ErrorAction Stop).DriverDesc; "
        "        $mac = (Get-ItemProperty $_.PSPath -ErrorAction Stop).NetworkAddress; "
        "        if ($desc -and $mac) { "
        "            Write-Host '  ' $desc ': ' $mac; "
        "        } "
        "    } catch { } "
        "}\"";

    system(checkCmd.c_str());
}

void AskRestart() {
    cout << "\n========================================\n";
    cout << "Перезагрузить компьютер сейчас?\n";
    cout << "Для применения MAC требуется перезагрузка\n";
    cout << "Нажмите 'm' для перезагрузки\n";
    cout << "Любую другую клавишу для отмены\n";
    cout << "========================================\n";

    char choice;
    cin >> choice;

    if (choice == 'm' || choice == 'M') {
        cout << "Перезагрузка через 10 секунд...\n";
        cout << "Для отмены нажмите Ctrl+C\n";
        system("shutdown /r /t 10 /c \"MAC Spoofer - restart for MAC changes\"");
        cout << "\nКомпьютер перезагрузится через 10 секунд\n";
    }
    else {
        cout << "Перезагрузка отменена\n";
        cout << "MAC изменятся после следующей перезагрузки\n";
    }
}

void ClearScreen() {
    system("cls");
    cout << "========================================\n";
    cout << "         MAC Spoofer 1.3 Final\n";
    cout << "       Universal HWID Protection\n";
    cout << "========================================\n";
}

int main() {
    setlocale(LC_ALL, "Russian");

    if (!CheckAdmin()) {
        cout << "Запустите от имени администратора\n";
        system("pause");
        return 1;
    }

    char choice;

    while (true) {
        ClearScreen();
        cout << "\nМеню:\n";
        cout << "--------------------------------\n";
        cout << "  1  - Изменить MAC адреса\n";
        cout << "  2  - Показать текущие MAC\n";
        cout << "  3  - Проверить изменения в реестре\n";
        cout << "  4  - Обновить сетевые настройки\n";
        cout << "  0  - Выход\n";
        cout << "--------------------------------\n";
        cout << "\nВыбор: ";
        cin >> choice;

        switch (choice) {
        case '1':
            ShowMAC();
            ChangeMAC();
            UpdateNetwork();
            ShowMAC();
            CheckRegistryChanges();
            AskRestart();
            break;
        case '2':
            ShowMAC();
            break;
        case '3':
            CheckRegistryChanges();
            break;
        case '4':
            UpdateNetwork();
            break;
        case '0':
            cout << "Выход\n";
            return 0;
        default:
            cout << "Неверный выбор\n";
        }

        if (choice != '0') {
            cout << "\nНажмите Enter для продолжения...";
            cin.ignore();
            cin.get();
        }
    }

    return 0;
}
