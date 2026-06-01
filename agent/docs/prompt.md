1. SOUL.md or DEFAULT_AGENT_IDENTITY
2. HERMES_AGENT_HELP_GUIDANCE
3. MEMORY_GUIDANCE
  只存长期稳定的用户事实和偏好，避免存任务过程或临时结果；会过期的内容不进 memory；方法和流程放 Skill，memory 只写“事实句”。
4. SESSION_SEARCH_GUIDANCE

5. SKILLS_GUIDANCE

6. MEMEORY.md:Agent 在对话过程中通过 memory 工具主动写入。MEMORY.md 存环境事实和工作笔记
7. USER.md 存用户偏好和画像。

8. ## Skills (mandatory)
    用户提问
        ↓
    检查是否有相关 Skill
        ↓
    有 → 加载 Skill
    无 → 用通用能力处理
        ↓
    执行任务
        ↓
    发现 Skill 错误？
        ↓
    是 → 修复 Skill
        ↓
    任务完成
        ↓
    复杂经验 → 保存为新 Skill

9. Project Context 部分：同样由 prompt_builder.py  负责，它会查找当前工作目录及父目录中的 AGENTS.md、CLAUDE.md等上下文文件，加载后拼入系统提示
 优先级是 HERMES.md > AGENTS.md > CLAUDE.md >                    
  .cursorrules，第一个命中就停止，所以只加载了 AGENTS.md，根目录的
   CLAUDE.md 没有被加载。       

10. PLATFORM_HINTS 



 你没看到它们被用到，是因为这次会话用的是                        
  claude-sonnet-4-6，所有这些常量都有条件门控，而 claude          
  模型不满足任何一个条件：                                        
                                                                  
  TOOL_USE_ENFORCEMENT_GUIDANCE + TOOL_USE_ENFORCEMENT_MODELS
  system_prompt.py:163 里匹配的子串是                             
  gpt/codex/gemini/gemma/grok/glm/qwen/deepseek，claude           
  不在列表里，所以整块跳过。后面的                                
  GOOGLE_MODEL_OPERATIONAL_GUIDANCE（gemini/gemma）和             
  OPENAI_MODEL_EXECUTION_GUIDANCE（gpt/codex/grok）也是在这个 if
  块内部，同样不注入。

  KANBAN_GUIDANCE                                                 
  agent_init.py:936 里条件是 "kanban_show" in 
  agent.valid_tool_names，普通 CLI 会话不加载 kanban              
  工具集，所以跳过。它只在 kanban worker 进程（$HERMES_KANBAN_TASK
   环境变量存在时）才出现。                                       
                                                            
  COMPUTER_USE_GUIDANCE                                           
  条件是 "computer_use" in agent.valid_tool_names，macOS
  专属工具，Linux 上不加载。                                      
                                                            
  DEVELOPER_ROLE_MODELS                                           
  这个根本不在 system_prompt.py 里用，它在                  
  agent/transports/chat_completions.py:263/442，作用是在发 API    
  请求时把 system role 改成 developer role（OpenAI GPT-5/Codex 对
  developer role 给更强的指令权重）。跟提示词内容无关，是         
  transport 层的消息格式转换。                              

  总结：这几个常量都是条件注入的，claude 模型 + 普通 CLI + 无     
  kanban + 无 computer_use，全部条件都不满足，所以你在这次会话的提
  示词里看不到它们。                         


11. Kanban 系统完整流程                                             
                                                                  
  整体架构                                                        
                                                                  
  Hermes 有一个多 Agent 任务调度系统，核心是一个 SQLite           
  数据库（~/.hermes/kanban.db）。用户通过 hermes kanban           
  命令管理任务，dispatcher 负责把任务分发给 worker 进程。         
                                                            
  用户 → hermes kanban create "任务描述" --assignee <profile>
           ↓                                                      
     kanban.db 里写入一条 task 记录（status=ready）
           ↓                                                      
     dispatcher 轮询，发现 ready 任务                             
           ↓                                                      
     subprocess.Popen 启动一个新的 hermes 进程（worker）          
           ↓                                                      
     worker 进程里 HERMES_KANBAN_TASK=<task_id>                   
           ↓                                                      
     kanban_show 工具被加载 → KANBAN_GUIDANCE 注入系统提示词      
                                                                  
  dispatcher 怎么 spawn worker                                    
                                                                  
  kanban_db.py:6455 里，dispatcher 在启动子进程前设置环境变量：   
                                                            
  env["HERMES_KANBAN_TASK"] = task.id        # 任务 ID            
  env["HERMES_KANBAN_WORKSPACE"] = workspace  # 工作目录          
  env["HERMES_KANBAN_DB"] = str(kanban_db_path())
  env["HERMES_PROFILE"] = profile_arg         # 用哪个            
  profile（模型/工具配置）                                        
                                                                  
  然后执行：                                                      
                                                            
  hermes -p <assignee_profile> --accept-hooks --skills
  kanban-worker chat -q "<task_prompt>"                           
   
  这是一个普通的 hermes chat -q 调用，只是带了特殊环境变量。      
                                                            
  为什么 KANBAN_GUIDANCE 只在 worker 里出现                       
                                                            
  toolsets.py:64 里，kanban_show 工具的加载条件就是               
  HERMES_KANBAN_TASK 是否存在：                             
                                                                  
  # kanban 工具集只在 dispatcher-spawned worker 里激活            
  bool(os.environ.get("HERMES_KANBAN_TASK"))                      
                                                                  
  普通 hermes chat 没有这个环境变量 → kanban_show 不加载 →        
  agent_init.py:936 判断为空 → KANBAN_GUIDANCE 不注入。           
                                                                  
  Worker 进程有这个环境变量 → kanban_show 加载 → KANBAN_GUIDANCE  
  注入，告诉 Agent：
  - 你的任务 ID 在 $HERMES_KANBAN_TASK                            
  - 先调 kanban_show() 了解任务                                   
  - 长任务要定期 kanban_heartbeat()
  - 完成后调 kanban_complete(summary=..., metadata=...)           
  - 遇到阻塞调 kanban_block(reason=...) 而不是乱猜                
                                                                  
  使用场景                                                        
                                                            
  实际上就是 Hermes 的多 Agent 并行执行功能。比如：               
   
  # 创建一个编码任务，分配给 coding profile                       
  hermes kanban create "重构 auth 模块" --assignee coding         
   
  # 创建一个文档任务，分配给 writing profile                      
  hermes kanban create "更新 API 文档" --assignee writing   
                                                                  
  # 启动 dispatcher，自动并行执行上面两个任务                     
  hermes kanban dispatch                                          
                                                                  
  dispatcher 会为每个任务 spawn 一个独立的 hermes                 
  子进程，每个子进程都有自己的                                    
  HERMES_KANBAN_TASK，因此都会得到完整的                          
  KANBAN_GUIDANCE，知道如何与 kanban board                  
  交互、如何汇报结果、如何处理阻塞。
  

9. 一堆tool