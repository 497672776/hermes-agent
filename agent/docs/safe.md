 Hermes 的安全执行是多层防御架构，核心流程如下：                 
                                                                  
  执行前检查链                                                    
                                                                  
  terminal_tool(command)                                          
    ↓                                                             
  _check_all_guards()                                             
    ├─ detect_hardline_command()   → 无条件拦截（rm -rf           
  /、mkfs、fork bomb 等）                                         
    ├─ _check_sudo_stdin_guard()   → 阻止 sudo -S 密码猜测        
    ├─ check_command_security()    → tirith 二进制扫描（同形字    
  URL、管道注入等）                                               
    ├─ detect_dangerous_command()  → 47 个危险正则（递归删除、DROP
   TABLE 等）                                                     
    └─ request_approval()          → CLI 交互确认 / Gateway       
  异步队列                                                        
    ↓                                                       
  在隔离环境中执行                                                
                                                                  
  关键组件                                                        
                                                                  
  1. 审批系统 — tools/approval.py                                 
  - 硬性黑名单（L203-237）：--yolo 也无法绕过               
  - 危险模式（L321-427）：47 个预编译正则                         
  - 会话级 YOLO 模式 + 永久白名单持久化                     
                                                                  
  2. Tirith 安全扫描器 — tools/tirith_security.py                 
  - 外部二进制，SHA-256 校验 + cosign 溯源验证                    
  - 退出码：0=放行、1=拦截、2=警告                                
  - 默认 5s 超时，fail-open                                       
                                                                  
  3. Docker 后端硬化 — tools/environments/docker.py L310-365      
  - --cap-drop ALL + 最小能力重加                                 
  - --security-opt no-new-privileges                              
  - --pids-limit 256、tmpfs 挂载、可选 --network=none             
                                                                  
  4. 环境变量清洗 — hermes_cli/config.py L78-100                  
  - 屏蔽 LD_PRELOAD、PYTHONPATH、NODE_OPTIONS 等解释器劫持路径    
                                                                  
  5. 沙箱工具限制 — tools/code_execution_tool.py L61-69           
  - 代码执行子 Agent 只能用 7 个工具，5min 超时，50 次工具调用上限
                                                                  
  设计原则                              
  
  SECURITY.md 明确：OS 级隔离是唯一真正的安全边界，进程内的审批/模
  式检测只是防误操作，不是对抗性防护。Docker/Singularity/Modal
  等后端已沙箱化，会跳过危险命令检查（terminal_tool.py L1171）。


  1. 为什么只给 7 个工具？

避免权限过大。

如果把所有工具都开放：

Agent 可能递归调用 Agent
可能无限创建任务
可能调用危险 MCP
上下文变得极其复杂

所以只给最基础的：

工具	用途
read_file	读文件
write_file	写文件
search_files	搜索代码
patch	修改代码
terminal	执行命令
web_search	搜索
web_extract	抓网页

相当于：

「你是个程序员，只给你编辑器、终端和浏览器。」