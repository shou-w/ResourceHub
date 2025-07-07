# カスタムスクリプトによる監視

```bash
#!/bin/sh
COLUMN=docker  # 監視項目名
COLOR=green    # デフォルトでは正常状態
MSG="Docker Container Status\n\n"

# docker ps -a の出力を取得
CONTAINERS=$(docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}")
MSG="${MSG}${CONTAINERS}\n\n"

# 停止中のコンテナがあるかチェック
STOPPED=$(docker ps -a --filter "status=exited" --quiet)
if [ ! -z "$STOPPED" ]; then
    COLOR=yellow
    MSG="${MSG}Warning: Some containers are stopped\n"
fi

# 異常終了したコンテナがあるかチェック
FAILED=$(docker ps -a --filter "status=exited" --filter "status=dead" --quiet)
if [ ! -z "$FAILED" ]; then
    COLOR=red
    MSG="${MSG}Error: Some containers failed\n"
fi

# Xymonにレポート送信
$XYMON $XYMSRV "status $MACHINE.$COLUMN $COLOR `date` ${MSG}"
exit 0
```