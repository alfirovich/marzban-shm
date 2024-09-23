# Marzban SHM Billing template

Измененный шаблон Marzban для SHM Billing

## Use

В settings сервера добавить

```
panel {
    link: "https://MarzbanLink.ru"
    username: "username"
    password: "password"
}
```
<img width="905" alt="Снимок экрана 2024-09-23 в 21 07 50" src="https://github.com/user-attachments/assets/30175dba-2d8b-4abc-acea-4f35e3ea442f">

Изменить существующий шаблон marzban

```sh
#!/bin/bash

EVENT="{{ event_name }}"
SESSION_ID="{{ user.gen_session.id }}"
API_URL="{{ config.api.url }}"
MARZBAN_HOST="{{ server.settings.panel.link }}"
SUDO_USERNAME="{{server.settings.panel.username}}"
SUDO_PASSWORD="{{server.settings.panel.password}}"


echo "Marzban Template"
echo
echo "EVENT=$EVENT"

get_marzban_token() {
    if [ -f "/opt/marzban/.env" ]; then

        echo "Marzban host: $MARZBAN_HOST"

        export TOKEN=$(curl -sk -XPOST \
          "$MARZBAN_HOST/api/admin/token" \
          -H 'Content-Type: application/x-www-form-urlencoded' \
          -d "grant_type=password&username=$SUDO_USERNAME&password=$SUDO_PASSWORD" | jq -r .access_token)

        if [ -z "$TOKEN" ]; then
            echo 'Error: can not get TOKEN. Please check docker containers status'
            exit 1
        fi
    else
        echo 'Error: Marzban is not installed! Please check settings'
        exit 1
    fi
}

case $EVENT in
    INIT)

        echo "Check SHM API host: $API_URL"
        echo "Marzban host: $MARZBAN_HOST"
        echo "Marzban user: $SUDO_USERNAME"
        HTTP_CODE=$(curl -sk -o /dev/null -w "%{http_code}" $API_URL/shm/v1/test)
        RET_CODE=$?
        if [ $RET_CODE -ne 0 ]; then
            echo "Error: host $API_URL is incorrect."
            echo "Please set correct public host in SHM config. It must be accessible from the server."
            exit 1
        fi
        if [ $HTTP_CODE -ne '200' ]; then
            echo "ERROR: incorrect API URL: $API_URL"
            echo "Got status: $HTTP_CODE"
            exit 1
        fi
        ;;
    CREATE)
        echo "Create a new user"

        PAYLOAD="$(cat <<-EOF
        {
          "username": "us_{{ us.id }}",
          "proxies": {
            "vmess": {},
            "vless": {"flow": "xtls-rprx-vision"},
            "trojan": {},
            "shadowsocks": {
              "method": "chacha20-ietf-poly1305"
            }
          },
          "data_limit": 0,
          "expire": null,
          "data_limit_reset_strategy": "no_reset",
          "status": "active",
          "note": "SHM_info- {{ user.login }}, {{ user.full_name }}, https://t.me/{{ user.settings.telegram.login }}",
          "inbounds": {
            "vmess": [
              "VMess TCP",
              "VMess Websocket"
            ],
            "vless": [
              "VLESS TCP REALITY",
              "VLESS GRPC REALITY"
            ],
            "trojan": [
              "Trojan Websocket TLS"
            ],
            "shadowsocks": [
              "Shadowsocks TCP"
            ]
          }
        }
EOF
        )"

        get_marzban_token
        USER_CFG=$(curl -sk -XPOST \
          "$MARZBAN_HOST/api/user" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d "$PAYLOAD")

        if [ -z $(echo "$USER_CFG" | jq -r '.username | select( . != null )') ]; then
            echo "Error: $USER_CFG"
            exit 1
        fi

        echo "Upload user config to SHM: $API_URL/shm/v1/storage/manage/vpn_mrzb_{{ us.id }}"
        curl -sk -XPUT \
            -H "session-id: $SESSION_ID" \
            -H "Content-Type: application/json" \
            $API_URL/shm/v1/storage/manage/vpn_mrzb_{{ us.id }} \
            --data-binary "$USER_CFG"
        echo "done"
        ;;
    ACTIVATE)
        echo "Activate user"

        get_marzban_token
        curl -sk -XPUT \
          "$MARZBAN_HOST/api/user/us_{{ us.id }}" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d '{"status":"active"}'

        echo "done"
        ;;
    BLOCK)
        echo "Block user"

        get_marzban_token
        curl -sk -XPUT \
          "$MARZBAN_HOST/api/user/us_{{ us.id }}" \
          -H "Authorization: Bearer $TOKEN" \
          -H 'Content-Type: application/json' \
          -d '{"status":"disabled"}'

        echo "done"
        ;;
    REMOVE)
        echo "Remove user"

        get_marzban_token
        curl -sk -XDELETE \
          "$MARZBAN_HOST/api/user/us_{{ us.id }}" \
          -H "Authorization: Bearer $TOKEN"

        echo "Remove user key from SHM"
        curl -sk -XDELETE \
            -H "session-id: $SESSION_ID" \
            $API_URL/shm/v1/storage/manage/vpn_mrzb_{{ us.id }}
        echo "done"
        ;;
    PROLONGATE)
        echo "Reset user counters"

        get_marzban_token
        curl -sk -XPOST \
          "$MARZBAN_HOST/api/user/us_{{ us.id }}/reset" \
          -H "Authorization: Bearer $TOKEN"
        echo "done"
        ;;
    *)
        echo "Unknown event: $EVENT. Exit."
        exit 0
        ;;
esac

```

