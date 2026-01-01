你遇到了Git无法连接到GitHub 443端口的网络错误，超时时间达21122毫秒，核心是客户端与GitHub服务器之间的网络链路不通或被拦截。下面为你提供分步排查和解决方案，从简单到复杂逐一推进：

### 一、先做基础网络连通性测试（定位问题根源）
首先通过命令行验证网络是否能到达GitHub，区分是「全网不通」还是「仅GitHub/443端口不通」：
1.  **Ping测试（验证GitHub服务器可达性）**
    打开终端（Windows：CMD/PowerShell；Mac/Linux：Terminal），执行：
    ```bash
    # 直接ping GitHub（部分环境可能屏蔽ICMP，ping不通不代表443端口不可用，仅做参考）
    ping github.com
    # 也可ping GitHub的IP地址（绕过DNS解析，进一步排查）
    ping 140.82.112.4
    ```
2.  **Telnet/NC测试（直接验证443端口连通性，关键步骤）**
    这是验证443端口是否开放的核心命令：
    - Windows（需先启用Telnet功能：控制面板→程序→启用或关闭Windows功能→勾选Telnet客户端）：
      ```cmd
      telnet github.com 443
      ```
    - Mac/Linux（使用nc命令，若未安装可通过`brew install netcat`/`apt install netcat`安装）：
      ```bash
      nc -zv github.com 443
      ```
    - 结果说明：若提示「Connected」（连接成功），说明端口通畅，问题在Git配置；若提示「Connection refused」/「超时」，说明端口被拦截或网络路由问题。

### 二、解决方案（按优先级排序）
#### 方案1：检查网络基础配置（最常见问题）
1.  **确认网络连接状态**
    - 排查是否断开网络、WiFi信号弱，切换网络（如从公司内网→手机热点）测试，排除局域网限制。
    - 关闭VPN/代理工具（若当前启用），部分VPN配置异常会导致端口阻塞。
2.  **刷新DNS缓存（解决DNS解析失败）**
    若ping `github.com` 失败，但ping `140.82.112.4` 成功，说明是DNS解析问题：
    - Windows：
      ```cmd
      ipconfig /flushdns
      ```
    - Mac：
      ```bash
      sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
      ```
    - Linux：
      ```bash
      sudo systemctl restart nscd  # 或 sudo /etc/init.d/nscd restart
      # 若使用systemd-resolved
      sudo systemd-resolve --flush-caches
      ```

#### 方案2：配置Git代理（解决网络访问被拦截/跨国链路不稳定）
如果你的网络需要代理才能访问外网（如国内网络访问GitHub），需为Git单独配置代理：
1.  **获取你的代理信息**
    通常代理格式为：`socks5://127.0.0.1:1080`（SOCKS5代理，常见于Clash、V2Ray）或 `http://127.0.0.1:7890`（HTTP代理，常见于SwitchyOmega）。
2.  **为Git配置代理（两种方式可选）**
    - 方式1：全局配置（所有Git仓库都生效）
      ```bash
      # 配置SOCKS5代理（推荐，稳定性更高）
      git config --global http.proxy socks5://127.0.0.1:1080
      git config --global https.proxy socks5://127.0.0.1:1080
      
      # 若使用HTTP代理
      git config --global http.proxy http://127.0.0.1:7890
      git config --global https.proxy http://127.0.0.1:7890
      ```
    - 方式2：仅当前仓库生效（进入MyNote仓库目录后执行）
      ```bash
      git config http.proxy socks5://127.0.0.1:1080
      git config https.proxy socks5://127.0.0.1:1080
      ```
3.  **取消Git代理（若后续不需要）**
    ```bash
    # 取消全局代理
    git config --global --unset http.proxy
    git config --global --unset https.proxy
    
    # 取消当前仓库代理
    git config --unset http.proxy
    git config --unset https.proxy
    ```

#### 方案3：更换GitHub远程仓库地址（HTTPS → SSH）
若443端口（HTTPS协议）始终无法连通，可切换为SSH协议（默认22端口，部分环境更易通行）：
1.  **先准备SSH密钥（若未配置）**
    若本地没有SSH密钥，先生成并添加到GitHub账户：
    ```bash
    # 生成SSH密钥（一路回车默认配置即可）
    ssh-keygen -t ed25519 -C "你的GitHub邮箱地址"
    ```
    然后将公钥（Windows：`C:\Users\你的用户名\.ssh\id_ed25519.pub`；Mac/Linux：`~/.ssh/id_ed25519.pub`）内容复制，粘贴到GitHub → Settings → SSH and GPG keys → New SSH key 中。
2.  **修改当前仓库的远程地址**
    进入MyNote仓库目录，执行以下命令替换远程地址：
    ```bash
    # 先查看当前远程地址（确认是HTTPS格式）
    git remote -v
    
    # 替换为SSH地址（格式：git@github.com:用户名/仓库名.git）
    git remote set-url origin git@github.com:TDAkory/MyNote.git
    
    # 验证是否生效
    git remote -v
    ```
3.  **测试SSH连接**
    ```bash
    ssh -T git@github.com
    ```
    若提示「Hi TDAkory! You've successfully authenticated」，说明SSH配置成功，此时再执行`git pull/push`即可。

#### 方案4：排查防火墙/安全软件拦截
1.  **系统防火墙**
    - Windows：关闭Windows Defender防火墙（临时测试），路径：控制面板→系统和安全→Windows Defender防火墙→关闭或启用防火墙。
    - Mac/Linux：关闭防火墙（临时测试），Mac执行 `sudo pfctl -d`，Linux执行 `sudo systemctl stop firewalld`。
2.  **第三方安全软件**
    关闭360、腾讯电脑管家、Mac防火墙软件等，这些软件可能会拦截Git的网络请求。

### 三、补充排查：Git版本与网络参数
1.  **更新Git版本**
    老旧Git版本可能存在网络兼容问题，前往[Git官网](https://git-scm.com/downloads)下载最新版本安装。
2.  **增加Git网络超时时间（临时缓解链路不稳定）**
    若网络链路不稳定，可适当延长Git超时时间：
    ```bash
    git config --global http.timeout 300  # 超时时间设为300秒（默认通常60秒）
    ```

### 总结
1.  先通过`ping`和`telnet/nc`定位问题：是DNS解析、端口拦截还是代理缺失；
2.  优先尝试「切换网络」「配置Git代理」「更换SSH地址」，这三种方式解决90%以上的问题；
3.  若仍失败，排查防火墙拦截或更新Git版本，或联系网络管理员（公司/校园网）解除GitHub访问限制。