# Termux + mitmproxy 自动抓包上报完整方案

## 目录
- [系统要求](#系统要求)
- [第一步：安装Termux](#第一步安装termux)
- [第二步：安装mitmproxy](#第二步安装mitmproxy)
- [第三步：创建过滤脚本](#第三步创建过滤脚本)
- [第四步：安装证书](#第四步安装证书)
- [第五步：测试运行](#第五步测试运行)
- [第六步：配置开机自启](#第六步配置开机自启)
- [第七步：状态监控](#第七步状态监控)
- [故障排查](#故障排查)

---

## 系统要求

- Android 7.0 或更高版本
- 存储空间：至少 500MB
- 网络：稳定的网络连接

---

## 第一步：安装Termux

### 1.1 下载Termux

**推荐从F-Droid下载（最新版本）：**
```
https://f-droid.org/packages/com.termux/
```

**或从GitHub下载：**
```
https://github.com/termux/termux-app/releases
```

⚠️ **注意：不要从Google Play下载（版本过旧）**

### 1.2 安装APK

1. 下载完成后点击安装
2. 允许"安装未知应用"权限
3. 安装完成后打开Termux

---

## 第二步：安装mitmproxy

### 2.1 更新软件包

打开Termux，输入以下命令：

```bash
pkg update && pkg upgrade
```

按 `Y` 确认更新。

### 2.2 安装Python

```bash
pkg install python
```

### 2.3 安装依赖库

```bash
pkg install openssl libffi
```

### 2.4 安装mitmproxy

```bash
pip install mitmproxy requests
```

⏳ 安装过程可能需要 5-10 分钟，请耐心等待。

### 2.5 验证安装

```bash
mitmdump --version
```

如果显示版本号，说明安装成功。

---

## 第三步：创建过滤脚本

### 3.1 创建工作目录

```bash
mkdir -p ~/mitmproxy
cd ~/mitmproxy
```

### 3.2 创建过滤脚本

```bash
nano filter.py
```

### 3.3 粘贴以下代码

长按屏幕 → 粘贴：

```python
from mitmproxy import http
import requests
import json
import sqlite3
from datetime import datetime
import threading
import time

# ==================== 配置区 ====================
SERVER_URL = "https://your-server.com/api/report"  # 修改为你的服务器地址
TARGET_HOST = "cashier.hbseccj7.com"               # 目标域名
TARGET_PATH = "/api/payments/transactions"         # 目标路径
DB_PATH = "/sdcard/mitmproxy_queue.db"            # 本地数据库路径
RETRY_INTERVAL = 60                                # 重试间隔（秒）
MAX_RETRY = 3                                      # 最大重试次数
# ================================================

# 初始化数据库
def init_db():
    conn = sqlite3.connect(DB_PATH)
    conn.execute('''
        CREATE TABLE IF NOT EXISTS queue (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            url TEXT,
            data TEXT,
            uploaded INTEGER DEFAULT 0,
            retry_count INTEGER DEFAULT 0,
            created_at TEXT
        )
    ''')
    conn.commit()
    return conn

db_conn = init_db()

def save_to_queue(url, data):
    """保存到本地队列"""
    try:
        db_conn.execute(
            "INSERT INTO queue (url, data, created_at) VALUES (?, ?, ?)",
            (url, json.dumps(data), datetime.now().isoformat())
        )
        db_conn.commit()
        print(f"💾 已保存到队列: {url}")
    except Exception as e:
        print(f"❌ 保存队列失败: {e}")

def upload_to_server(data):
    """上报到服务器"""
    try:
        resp = requests.post(
            SERVER_URL,
            json=data,
            timeout=10,
            headers={"Content-Type": "application/json"}
        )
        if resp.status_code == 200:
            return True
        else:
            print(f"⚠️ 上报失败: HTTP {resp.status_code}")
            return False
    except Exception as e:
        print(f"❌ 上报异常: {e}")
        return False

def retry_failed_uploads():
    """重试失败的上报"""
    while True:
        time.sleep(RETRY_INTERVAL)
        try:
            cursor = db_conn.execute(
                "SELECT * FROM queue WHERE uploaded=0 AND retry_count<?",
                (MAX_RETRY,)
            )
            for row in cursor:
                row_id, url, data_str, uploaded, retry_count, created_at = row
                data = json.loads(data_str)
                
                if upload_to_server(data):
                    db_conn.execute(
                        "UPDATE queue SET uploaded=1 WHERE id=?",
                        (row_id,)
                    )
                    print(f"✅ 重试成功: {url}")
                else:
                    db_conn.execute(
                        "UPDATE queue SET retry_count=retry_count+1 WHERE id=?",
                        (row_id,)
                    )
                    print(f"⚠️ 重试失败 ({retry_count+1}/{MAX_RETRY}): {url}")
                
                db_conn.commit()
        except Exception as e:
            print(f"❌ 重试异常: {e}")

# 启动重试线程
retry_thread = threading.Thread(target=retry_failed_uploads, daemon=True)
retry_thread.start()

def request(flow: http.HTTPFlow):
    """拦截请求"""
    if TARGET_HOST in flow.request.pretty_host and \
       TARGET_PATH in flow.request.path:
        print(f"✅ 捕获请求: {flow.request.url}")
        print(f"   方法: {flow.request.method}")
        print(f"   路径: {flow.request.path}")

def response(flow: http.HTTPFlow):
    """拦截响应并上报"""
    # 检查是否匹配目标
    if TARGET_HOST not in flow.request.pretty_host:
        return
    if TARGET_PATH not in flow.request.path:
        return
    
    print(f"📥 捕获响应: {flow.request.url}")
    
    # 构造上报数据
    data = {
        "timestamp": datetime.now().isoformat(),
        "url": flow.request.url,
        "method": flow.request.method,
        "path": flow.request.path,
        "query_params": dict(flow.request.query),
        "request_headers": dict(flow.request.headers),
        "request_body": flow.request.text,
        "response_status": flow.response.status_code,
        "response_headers": dict(flow.response.headers),
        "response_body": flow.response.text,
    }
    
    # 先保存到本地队列
    save_to_queue(flow.request.url, data)
    
    # 尝试立即上报
    if upload_to_server(data):
        print(f"📤 上报成功: {flow.request.url}")
        # 标记为已上报
        db_conn.execute(
            "UPDATE queue SET uploaded=1 WHERE url=? AND uploaded=0",
            (flow.request.url,)
        )
        db_conn.commit()
    else:
        print(f"⚠️ 上报失败，已加入重试队列")

def done():
    """退出时清理"""
    print("🛑 正在关闭...")
    db_conn.close()
    print("✅ 已关闭")
```

### 3.4 修改配置

修改脚本中的配置项：
- `SERVER_URL`: 你的服务器地址
- `TARGET_HOST`: 要抓取的域名
- `TARGET_PATH`: 要抓取的路径

### 3.5 保存文件

1. 按 `音量下键 + X`
2. 按 `Y`
3. 按 `回车`

---

## 第四步：安装证书

### 4.1 生成证书

```bash
# 首次运行生成证书
mitmdump
```

看到 `Proxy server listening` 后，按 `Ctrl + C` 停止。

### 4.2 复制证书到手机存储

```bash
cp ~/.mitmproxy/mitmproxy-ca-cert.pem /sdcard/
```

### 4.3 安装证书

1. 打开手机 **设置**
2. 进入 **安全** → **加密与凭据**
3. 点击 **从存储设备安装**
4. 选择 `mitmproxy-ca-cert.pem`
5. 输入证书名称：`mitmproxy`
6. 用途选择：**VPN和应用**
7. 点击 **确定**

### 4.4 验证证书安装

设置 → 安全 → 受信任的凭据 → 用户 → 查看是否有 `mitmproxy`

---

## 第五步：测试运行

### 5.1 启动mitmproxy

```bash
cd ~/mitmproxy
mitmdump -s filter.py --listen-host 0.0.0.0 --listen-port 8080
```

看到以下输出说明启动成功：
```
Loading script filter.py
Proxy server listening at http://*:8080
```

### 5.2 配置手机代理

1. 打开 **设置** → **WLAN**
2. 长按当前连接的WiFi → **修改网络**
3. 展开 **高级选项**
4. 代理：选择 **手动**
5. 主机名：`127.0.0.1`
6. 端口：`8080`
7. 保存

### 5.3 测试抓包

1. 打开浏览器访问：`http://mitm.it`
   - 如果能看到mitmproxy欢迎页面，说明代理配置成功

2. 打开目标应用，触发目标接口

3. 查看Termux输出：
```
✅ 捕获请求: https://cashier.hbseccj7.com/api/payments/transactions?page=0
📥 捕获响应: https://cashier.hbseccj7.com/api/payments/transactions?page=0
💾 已保存到队列: https://...
📤 上报成功: https://...
```

### 5.4 停止测试

按 `Ctrl + C` 停止mitmproxy

---

## 第六步：配置开机自启

### 6.1 安装Termux:Boot

**下载地址：**
```
F-Droid: https://f-droid.org/packages/com.termux.boot/
GitHub: https://github.com/termux/termux-boot/releases
```

安装APK后打开一次，授予必要权限。

### 6.2 创建启动目录

```bash
mkdir -p ~/.termux/boot
```

### 6.3 创建启动脚本

```bash
nano ~/.termux/boot/start-mitmproxy.sh
```

粘贴以下内容：

```bash
#!/data/data/com.termux/files/usr/bin/bash

# ==================== 配置 ====================
WORK_DIR=~/mitmproxy
SCRIPT_FILE=filter.py
LISTEN_PORT=8080
LOG_FILE=~/mitmproxy.log
BOOT_LOG=~/boot.log
# =============================================

echo "========================================" >> $BOOT_LOG
echo "🚀 启动时间: $(date)" >> $BOOT_LOG

# 1. 等待网络就绪
echo "⏳ 等待网络..." >> $BOOT_LOG
RETRY=0
while ! ping -c 1 8.8.8.8 &> /dev/null; do
    sleep 5
    RETRY=$((RETRY+1))
    if [ $RETRY -gt 24 ]; then
        echo "❌ 网络超时（2分钟）" >> $BOOT_LOG
        exit 1
    fi
done
echo "✅ 网络就绪" >> $BOOT_LOG

# 2. 检查是否已运行
if pgrep -f mitmdump > /dev/null; then
    echo "⚠️ mitmproxy已在运行，停止旧进程" >> $BOOT_LOG
    pkill -f mitmdump
    sleep 3
fi

# 3. 切换到工作目录
cd $WORK_DIR || {
    echo "❌ 工作目录不存在: $WORK_DIR" >> $BOOT_LOG
    exit 1
}

# 4. 检查脚本是否存在
if [ ! -f $SCRIPT_FILE ]; then
    echo "❌ 脚本文件不存在: $SCRIPT_FILE" >> $BOOT_LOG
    exit 1
fi

# 5. 启动mitmproxy
nohup mitmdump -s $SCRIPT_FILE \
    --listen-host 0.0.0.0 \
    --listen-port $LISTEN_PORT \
    > $LOG_FILE 2>&1 &

# 6. 等待启动
sleep 5

# 7. 验证启动
if pgrep -f mitmdump > /dev/null; then
    PID=$(pgrep -f mitmdump)
    echo "✅ mitmproxy启动成功 (PID: $PID)" >> $BOOT_LOG
    
    # 获取唤醒锁（防止休眠）
    termux-wake-lock 2>/dev/null
    if [ $? -eq 0 ]; then
        echo "🔒 已获取唤醒锁" >> $BOOT_LOG
    fi
    
    # 检查端口
    if netstat -tuln 2>/dev/null | grep :$LISTEN_PORT > /dev/null; then
        echo "✅ 端口 $LISTEN_PORT 已监听" >> $BOOT_LOG
    else
        echo "⚠️ 端口 $LISTEN_PORT 未监听" >> $BOOT_LOG
    fi
else
    echo "❌ mitmproxy启动失败" >> $BOOT_LOG
    echo "错误日志:" >> $BOOT_LOG
    tail -20 $LOG_FILE >> $BOOT_LOG
fi

echo "========================================" >> $BOOT_LOG
```

### 6.4 赋予执行权限

```bash
chmod +x ~/.termux/boot/start-mitmproxy.sh
```

### 6.5 安装Termux API（可选，用于唤醒锁）

```bash
pkg install termux-api
```

同时需要安装Termux:API应用：
```
F-Droid: https://f-droid.org/packages/com.termux.api/
```

### 6.6 测试启动脚本

```bash
~/.termux/boot/start-mitmproxy.sh
```

检查是否成功：
```bash
ps aux | grep mitmdump
cat ~/boot.log
```

### 6.7 重启测试

重启手机，等待2-3分钟，检查：

```bash
# 打开Termux
ps aux | grep mitmdump
cat ~/boot.log
tail -f ~/mitmproxy.log
```

---

## 第七步：状态监控

### 7.1 创建状态检查脚本

```bash
nano ~/check_status.sh
```

粘贴以下内容：

```bash
#!/data/data/com.termux/files/usr/bin/bash

echo "========================================"
echo "📊 mitmproxy 状态检查"
echo "========================================"
echo ""

# 检查进程
echo "🔍 进程状态:"
if pgrep -f mitmdump > /dev/null; then
    PID=$(pgrep -f mitmdump)
    echo "  ✅ 运行中 (PID: $PID)"
    
    # 显示进程信息
    ps aux | grep mitmdump | grep -v grep
else
    echo "  ❌ 未运行"
fi
echo ""

# 检查端口
echo "🔍 端口状态:"
if netstat -tuln 2>/dev/null | grep :8080 > /dev/null; then
    echo "  ✅ 端口 8080 已监听"
else
    echo "  ❌ 端口 8080 未监听"
fi
echo ""

# 检查数据库
echo "🔍 队列状态:"
if [ -f /sdcard/mitmproxy_queue.db ]; then
    TOTAL=$(sqlite3 /sdcard/mitmproxy_queue.db "SELECT COUNT(*) FROM queue" 2>/dev/null)
    UPLOADED=$(sqlite3 /sdcard/mitmproxy_queue.db "SELECT COUNT(*) FROM queue WHERE uploaded=1" 2>/dev/null)
    PENDING=$(sqlite3 /sdcard/mitmproxy_queue.db "SELECT COUNT(*) FROM queue WHERE uploaded=0" 2>/dev/null)
    
    echo "  总记录: $TOTAL"
    echo "  已上报: $UPLOADED"
    echo "  待上报: $PENDING"
else
    echo "  ⚠️ 数据库文件不存在"
fi
echo ""

# 显示最近日志
echo "📝 最近日志 (mitmproxy.log):"
echo "----------------------------------------"
tail -15 ~/mitmproxy.log 2>/dev/null || echo "  ⚠️ 日志文件不存在"
echo ""

# 显示启动日志
echo "📝 启动日志 (boot.log):"
echo "----------------------------------------"
tail -15 ~/boot.log 2>/dev/null || echo "  ⚠️ 启动日志不存在"
echo ""

echo "========================================"
```

保存并赋予执行权限：

```bash
chmod +x ~/check_status.sh
```

### 7.2 使用状态检查

```bash
~/check_status.sh
```

### 7.3 创建快捷命令

```bash
echo "alias status='~/check_status.sh'" >> ~/.bashrc
echo "alias start='~/.termux/boot/start-mitmproxy.sh'" >> ~/.bashrc
echo "alias stop='pkill -f mitmdump'" >> ~/.bashrc
echo "alias logs='tail -f ~/mitmproxy.log'" >> ~/.bashrc

source ~/.bashrc
```

现在可以使用简短命令：
- `status` - 查看状态
- `start` - 启动mitmproxy
- `stop` - 停止mitmproxy
- `logs` - 查看实时日志

---

## 故障排查

### 问题1：mitmproxy无法启动

**检查：**
```bash
mitmdump -s filter.py
```

**常见错误：**
- `No module named 'mitmproxy'` → 重新安装：`pip install mitmproxy`
- `No module named 'requests'` → 安装：`pip install requests`
- `Permission denied` → 检查文件权限：`chmod +x filter.py`

### 问题2：无法抓取HTTPS

**检查证书：**
1. 设置 → 安全 → 受信任的凭据 → 用户
2. 查看是否有 `mitmproxy` 证书
3. 如果没有，重新安装证书

**Android 7+ 特殊处理：**
某些应用不信任用户证书，需要Root后将证书安装到系统目录。

### 问题3：开机不自启

**检查：**
```bash
# 1. 检查Termux:Boot是否安装
ls ~/.termux/boot/

# 2. 检查脚本权限
ls -l ~/.termux/boot/start-mitmproxy.sh

# 3. 查看启动日志
cat ~/boot.log

# 4. 手动测试启动脚本
~/.termux/boot/start-mitmproxy.sh
```

### 问题4：网络连接失败

**检查代理设置：**
1. WiFi代理是否正确：`127.0.0.1:8080`
2. mitmproxy是否运行：`ps aux | grep mitmdump`
3. 端口是否监听：`netstat -tuln | grep 8080`

**测试代理：**
```bash
# 浏览器访问
http://mitm.it
```

### 问题5：上报失败

**检查：**
```bash
# 1. 查看队列
sqlite3 /sdcard/mitmproxy_queue.db "SELECT * FROM queue WHERE uploaded=0 LIMIT 5"

# 2. 查看错误日志
tail -50 ~/mitmproxy.log | grep "上报"

# 3. 测试服务器连接
curl -X POST https://your-server.com/api/report \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

### 问题6：游戏无法连接

**解决方案：**

1. **关闭目标应用的TLS解密：**
   修改 `filter.py`，只解密目标域名

2. **使用透明代理模式（需要Root）：**
```bash
mitmdump -s filter.py --mode transparent
```

3. **配置旁路规则：**
   在脚本中添加游戏服务器域名到白名单

### 问题7：手机休眠后停止工作

**解决方案：**

1. **获取唤醒锁：**
```bash
pkg install termux-api
termux-wake-lock
```

2. **禁用电池优化：**
   设置 → 应用 → Termux → 电池 → 不优化

3. **使用Tasker保持唤醒**

### 问题8：数据库损坏

**修复：**
```bash
# 备份
cp /sdcard/mitmproxy_queue.db /sdcard/mitmproxy_queue.db.bak

# 重建
rm /sdcard/mitmproxy_queue.db

# 重启mitmproxy
stop
start
```

---

## 高级配置

### 自定义证书（防检测）

```bash
cd ~/.mitmproxy

# 生成伪装证书
openssl req -x509 -newkey rsa:4096 \
  -keyout mitmproxy-ca-key.pem \
  -out mitmproxy-ca-cert.pem \
  -days 3650 -nodes \
  -subj "/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3"

# 重启mitmproxy使用新证书
```

### 使用随机端口

```bash
# 修改启动脚本
PORT=$((20000 + RANDOM % 10000))
mitmdump -s filter.py --listen-port $PORT
```

### 日志轮转

```bash
# 创建日志清理脚本
cat > ~/cleanup_logs.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash

# 保留最近7天的日志
find ~ -name "*.log" -mtime +7 -delete

# 清理已上报的旧数据
sqlite3 /sdcard/mitmproxy_queue.db \
  "DELETE FROM queue WHERE uploaded=1 AND created_at < datetime('now', '-7 days')"
EOF

chmod +x ~/cleanup_logs.sh

# 添加到crontab（需要安装cronie）
pkg install cronie
crontab -e
# 添加：0 3 * * * ~/cleanup_logs.sh
```

---

## 常用命令速查

```bash
# 启动
start

# 停止
stop

# 查看状态
status

# 查看实时日志
logs

# 查看启动日志
cat ~/boot.log

# 查看队列
sqlite3 /sdcard/mitmproxy_queue.db "SELECT COUNT(*) FROM queue WHERE uploaded=0"

# 手动重试失败的上报
sqlite3 /sdcard/mitmproxy_queue.db "UPDATE queue SET retry_count=0 WHERE uploaded=0"

# 清空队列
sqlite3 /sdcard/mitmproxy_queue.db "DELETE FROM queue"

# 重启mitmproxy
stop && sleep 2 && start

# 查看证书
ls ~/.mitmproxy/

# 测试网络
ping 8.8.8.8

# 查看端口
netstat -tuln | grep 8080
```

---

## 服务器端接口示例

### Node.js (Express)

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/api/report', (req, res) => {
    const data = req.body;
    
    console.log('收到数据:', {
        timestamp: data.timestamp,
        url: data.url,
        method: data.method
    });
    
    // 保存到数据库
    // saveToDatabase(data);
    
    res.json({ success: true });
});

app.listen(3000, () => {
    console.log('服务器运行在 http://localhost:3000');
});
```

### Python (Flask)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/report', methods=['POST'])
def report():
    data = request.json
    
    print(f"收到数据: {data['url']}")
    
    # 保存到数据库
    # save_to_database(data)
    
    return jsonify({'success': True})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

---

## 总结

通过本方案，你可以实现：

✅ 自动抓取指定路径的HTTPS流量  
✅ 自动上报到服务器  
✅ 失败自动重试  
✅ 开机自动启动  
✅ 本地队列备份  
✅ 状态监控  

**关键文件位置：**
- 过滤脚本：`~/mitmproxy/filter.py`
- 启动脚本：`~/.termux/boot/start-mitmproxy.sh`
- 运行日志：`~/mitmproxy.log`
- 启动日志：`~/boot.log`
- 数据队列：`/sdcard/mitmproxy_queue.db`

**需要帮助？**
- 查看日志：`logs`
- 检查状态：`status`
- 重启服务：`stop && start`

---

**文档版本：** v1.0  
**最后更新：** 2026-02-28  
**作者：** Kiro AI Assistant
