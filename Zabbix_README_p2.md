[К оглавлению](Zabbix_README.md)

# 2. Подготовка окружения (Windows 11 + WSL2)

Этот этап необходимо выполнить один раз перед началом работы со стендами.

**1. Включение WSL2 и компонентов Windows**
*   Запустите PowerShell или терминал от имени администратора.
*   Выполните команду для включения необходимых компонентов Windows:
    ```powershell
    dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
    dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
    ```
*   Перезагрузите компьютер.
*   После перезагрузки установите WSL2 ядро по умолчанию:
    ```powershell
    wsl --set-default-version 2
    ```
*   Проверьте версию WSL (должна быть 2):
    ```powershell
    wsl --status
    ```

**2. Установка дистрибутива Ubuntu**
*   Откройте Microsoft Store, найдите "Ubuntu 22.04.5 LTS" (или новее) и установите его.
*   Запустите Ubuntu из меню "Пуск". Дождитесь распаковки.
*   Создайте пользователя и пароль (запомните его, он потребуется для команд `sudo`).

**3. Настройка systemd в WSL**
*   В терминале Ubuntu откройте конфигурационный файл WSL:
    ```bash
    sudo nano /etc/wsl.conf
    ```
*   Добавьте в файл следующие строки:
    ```ini
    [boot]
    systemd=true
    ```
*   Сохраните файл (Ctrl+O, Enter, Ctrl+X).
*   Выйдите из Ubuntu и перезапустите WSL полностью (в PowerShell):
    ```powershell
    wsl --shutdown
    ```
*   Запустите Ubuntu заново. Проверьте, что systemd работает:
    ```bash
    systemctl list-units --type=service | grep docker
    ```

**4. Установка Docker Engine в Ubuntu**
*   Обновите список пакетов и установите зависимости:
    ```bash
    sudo apt update
    sudo apt install -y ca-certificates curl gnupg lsb-release
    ```
*   Добавьте официальный GPG-ключ Docker:
    ```bash
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```
*   Добавьте репозиторий Docker:
    ```bash
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
*   Установите Docker Engine и плагин Docker Compose:
    ```bash
    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

**5. Настройка пользователя для работы с Docker**
*   Добавьте текущего пользователя в группу `docker`, чтобы не использовать `sudo` для каждой команды:
    ```bash
    sudo usermod -aG docker $USER
    ```
*   Закройте терминал Ubuntu и откройте его заново (или выполните `newgrp docker`).
*   Проверьте, что Docker работает без `sudo`:
    ```bash
    docker --version
    docker compose version
    ```

**6. Установка Visual Studio Code (VS Code)**
*   Скачайте VS Code с официального сайта и установите его на Windows.
*   Установите расширение **"Dev Containers"** (ms-vscode-remote.remote-containers) и **"WSL"** (ms-vscode-remote.remote-wsl).
*   Для удобства редактирования файлов внутри WSL можно открыть папку проекта командой в терминале Ubuntu:
    ```bash
    code ~/zabbix-labs
    ```

[К оглавлению](Zabbix_README.md)
