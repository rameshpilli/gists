▎ Set up my statusline to show: model name (bold) + a 10-character █/░ progress bar of
▎ context-used % + the percentage + token count as usedK/maxK + current git branch — all
▎ separated by |. Implement it as a shell script at ~/.claude/statusline-command.sh that reads
▎ JSON from stdin (fields: .model.display_name, .context_window.used_percentage,
▎ .context_window.current_usage.input_tokens, .context_window.context_window_size, .cwd), and
▎ wire it into ~/.claude/settings.json as "statusLine": { "type": "command", "command": "sh
▎ /Users/<user>/.claude/statusline-command.sh" }.



Referene: 
  [Opus 4.7] │ add-gmail │ ctx [░░░░░░░░░░] 0% │ses [░░░░░░░░░░] 0% (↻ 4:50PM)


------
Helper prompts
1. Ask Claude to create a todo list when working on complex tasks to track prgoress and remain on track

2.  spin up a multi-agent adversarial challenge: independent auditors, competing principal-engineer architectures, skeptics that attack each, then a synthesized blueprint. Then build it.
