# vpn

# Гайд по настройке OpenVPN на сервере и малинке

Этот гайд поможет вам настроить VPN-соединение между сервером и малинкой (или другим устройством), используя OpenVPN, а также подключить малинку к серверу через защищённый канал.

---

## Шаг 1: Установка OpenVPN

### Установка на сервере

1. **Обновляем систему**:
    ```bash
    sudo apt update
    ```

2. **Устанавливаем OpenVPN и Easy-RSA**:
    ```bash
    sudo apt install openvpn easy-rsa
    ```
    - `openvpn` — основная программа для создания VPN-сети.
    - `easy-rsa` — утилита для создания сертификатов и ключей.

3. **Создаем рабочую директорию для сертификатов**:
    ```bash
    make-cadir ~/openvpn-ca
    cd ~/openvpn-ca
    ```
    - `make-cadir ~/openvpn-ca` — создаёт директорию для хранения всех сертификатов и ключей.
    - `cd ~/openvpn-ca` — переходим в эту директорию.

### Установка на клиенте (малинка)

1. **Обновляем систему**:
    ```bash
    sudo apt update
    ```

2. **Устанавливаем только клиентскую часть OpenVPN**:
    ```bash
    sudo apt install openvpn
    ```

---

## Шаг 2: Настройка OpenVPN-сервера

### Генерация ключей и сертификатов

1. **Инициализация инфраструктуры открытых ключей (PKI)**:
    ```bash
    ./easyrsa init-pki
    ```
    - `init-pki` — инициализация директории для ключей.

2. **Создание сертификата центра сертификации (CA)**:
    ```bash
    ./easyrsa build-ca
    ```
    - `build-ca` — создание и подписание сертификата центра сертификации (CA).

3. **Создание сертификата и ключа для сервера**:
    ```bash
    ./easyrsa build-server-full server nopass
    ```
    - `build-server-full` — генерирует ключ и сертификат для сервера.
    - `server` — имя сервера.
    - `nopass` — создание сертификата без пароля.

4. **Генерация ключа Диффи-Хеллмана (для обмена ключами)**:
    ```bash
    ./easyrsa gen-dh
    ```
    - `gen-dh` — создаёт ключи для безопасного обмена.

5. **Создание сертификата для клиента** (малинки):
    ```bash
    ./easyrsa build-client-full raspberry nopass
    ```
    - `build-client-full` — генерирует сертификат для клиента (малинки).
    - `raspberry` — имя клиента.
    - `nopass` — сертификат без пароля.

---

### Настройка конфигурации сервера

1. **Создаем файл конфигурации для сервера**:
    ```bash
    sudo nano /etc/openvpn/server.conf
    ```

2. **Добавляем конфигурацию в файл `server.conf`**:
    ```bash
    port 1194
    proto udp
    dev tun
    ca ca.crt
    cert server.crt
    key server.key
    dh dh.pem
    server 10.8.0.0 255.255.255.0
    ifconfig-pool-persist ipp.txt
    push "redirect-gateway def1 bypass-dhcp"
    push "dhcp-option DNS 8.8.8.8"
    keepalive 10 120
    cipher AES-256-CBC
    persist-key
    persist-tun
    status openvpn-status.log
    verb 3
    ```

    - `port 1194` — порт, на котором будет работать сервер (по умолчанию 1194).
    - `proto udp` — использование UDP протокола (рекомендуется для VPN).
    - `dev tun` — создание туннельного интерфейса.
    - `ca ca.crt` — путь к файлу сертификата центра сертификации.
    - `cert server.crt` — путь к файлу сертификата сервера.
    - `key server.key` — путь к приватному ключу сервера.
    - `dh dh.pem` — путь к файлу Диффи-Хеллмана.
    - `server 10.8.0.0 255.255.255.0` — VPN-подсеть, которая будет использоваться для клиентов.
    - `ifconfig-pool-persist ipp.txt` — сохраняет IP-адреса для клиентов.
    - `push "redirect-gateway def1 bypass-dhcp"` — направляет трафик через VPN.
    - `push "dhcp-option DNS 8.8.8.8"` — указывает DNS-сервер.
    - `keepalive 10 120` — параметры для мониторинга подключения.
    - `cipher AES-256-CBC` — шифрование канала.
    - `persist-key` и `persist-tun` — сохраняют ключи и туннель при перезапуске.
    - `status openvpn-status.log` — сохраняет лог статуса.
    - `verb 3` — уровень логирования (3 — средний уровень).

3. **Скопируем все сертификаты и ключи в `/etc/openvpn/`**:
    ```bash
    sudo cp pki/ca.crt pki/private/server.key pki/issued/server.crt /etc/openvpn
    sudo cp pki/dh.pem /etc/openvpn
    ```

4. **Запускаем сервер**:
    ```bash
    sudo systemctl start openvpn@server
    sudo systemctl enable openvpn@server
    ```

    - `start openvpn@server` — запуск сервера OpenVPN.
    - `enable openvpn@server` — включение автозапуска сервера при старте системы.

---

## Шаг 3: Настройка клиента (малинки)

### Конфигурация клиента

1. **Создаём клиентский конфигурационный файл на сервере**:
    ```bash
    client
    dev tun
    proto udp
    remote <IP_Сервера> 1194
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    cipher AES-256-CBC
    verb 3
    <ca>
    [Вставьте содержимое ca.crt]
    </ca>
    <cert>
    [Вставьте содержимое raspberry.crt]
    </cert>
    <key>
    [Вставьте содержимое raspberry.key]
    </key>
    ```

    - `<IP_Сервера>` — замените на реальный IP-адрес вашего сервера.
    - `<ca>`, `<cert>`, `<key>` — вставьте содержимое файлов сертификатов и ключей, которые мы создали ранее. Например, используйте команды `cat` для получения содержимого файлов:
        ```bash
        cat pki/ca.crt
        cat pki/issued/raspberry.crt
        cat pki/private/raspberry.key
        ```

2. **Передаем файл конфигурации на малинку**:
    ```bash
    scp raspberry.ovpn <пользователь>@<IP_Малинки>:~/
    ```

3. **Подключаемся с малинки к серверу**:
    ```bash
    sudo openvpn --config raspberry.ovpn
    ```

---

## Шаг 4: Проверка соединения

1. **Проверка на сервере**:
    ```bash
    ip a
    ```
    - Убедитесь, что интерфейс `tun0` появился.

2. **Проверка на малинке**:
    ```bash
    ifconfig
    ```
    - Убедитесь, что интерфейс `tun0` с IP-адресом из подсети `10.8.0.0/24` появился.

---

Теперь ваш VPN-сервер и клиент настроены, и малинка подключена через защищённый канал.

Сохраните этот гайд для последующего использования!
