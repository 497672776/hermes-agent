 直接加环境变量前缀就行：                                                                      
                                                                                                
  HERMES_DUMP_REQUESTS=1 hermes chat -q "你好"                                                  
                                                                                                
  或者交互模式：                                                                                
                                                                                                
  HERMES_DUMP_REQUESTS=1 hermes chat                                                            
                                                                                                
  dump 文件会写到 ~/.hermes/sessions/request_dump_<session_id>_<timestamp>.json，每次 API       
  调用生成一个，包含完整的 messages（系统提示+历史）和 tools。                                  
                                                                                                
  如果想同时在终端看到输出：                                                                    
   
  HERMES_DUMP_REQUESTS=1 HERMES_DUMP_REQUEST_STDOUT=1 hermes chat -q "你好"      


  ● 清楚了。两个 section 的来源： 
                                                                                                    
  存储：两个纯文本文件，保存在 ~/.hermes/memory/：                                                  
  - MEMORY.md → MEMORY (your personal notes) 块                                                     
  - USER.md → USER PROFILE (who the user is) 块                                                     
                                                                                                    
  注入：memory_tool.py 的 _render_block() 方法（第 475 行）在每次构建系统提示时读取这两个文件，加上 
  ══ 分隔线和用量百分比，拼成你看到的格式，然后插入系统提示。                                       

  内容从哪来：Agent 在对话过程中通过 memory 工具主动写入。MEMORY.md 存环境事实和工作笔记，USER.md   
  存用户偏好和画像。你可以直接查看：
                                                                                                    
  cat ~/.hermes/memory/MEMORY.md                                                                  
  cat ~/.hermes/memory/USER.md