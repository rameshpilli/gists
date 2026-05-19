  ▎ Set up my statusline to show: model name (bold) + a 10-character █/░ progress bar of
  ▎ context-used % + the percentage + token count as usedK/maxK + current git branch — all
  ▎ separated by |. Implement it as a shell script at ~/.claude/statusline-command.sh that reads
  ▎ JSON from stdin (fields: .model.display_name, .context_window.used_percentage,
  ▎ .context_window.current_usage.input_tokens, .context_window.context_window_size, .cwd), and
  ▎ wire it into ~/.claude/settings.json as "statusLine": { "type": "command", "command": "sh
  ▎ /Users/<user>/.claude/statusline-command.sh" }.
