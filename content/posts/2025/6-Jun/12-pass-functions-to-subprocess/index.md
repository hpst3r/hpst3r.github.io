---
title: "Passing a function to a PowerShell subprocess (without making a terrible, terrible mess)"
date: 2025-06-26T19:30:00-00:00
draft: false
---

This took a bit of experimenting. I'm still not completely certain that this is the most reasonable way to go about this, but I've already lost a few hours of my life to it, so oh well. It works.

I had a script, we'll call it Charles.

Charles has functions.

I need to run some of Charles's functions in a subprocess, so I can be sure all of Charles's son's handles are closed and collected by the GC.

If I don't do this, things will go horribly wrong and none of Charles's bodily functions will, well, function.

Charles is being difficult.

When I try feeding Charles an escaped string, Charles spits it right back out into my terminal (with a large helping of red error messages), which makes me sad.

When I try feeding Charles my functions, Charles makes me sad, yet again, by eviscerating my functions and eating random characters.

Then, I had a revelation. Maybe Charles doesn't have an appetite for base64 strings?

I settled on parsing the functions with Get-Command, reassembling them into a string, then encoding the string (and the command I would like my subprocess to do) to base64 and feeding THAT to my `powershell.exe` call via the `-EncodedCommand` argument.

This works!

You *could* scrape from the script file itself (pull the path to the file from `$MyInvocation`, use regular expressions to filter for functions) but that is quite nasty.

Not that this isn't nasty. But this is a little less nasty.

You could also access the functions with namespace variable notation (e.g., `${function:Get-Coffee}`, to pull `Get-Coffee` right on out of the `function:` PSDrive). This does effectively the same thing as Get-Command, but I feel it's a little less clear.

Here is a trimmed down example of what I wound up doing:

```PowerShell
# select the functions from our script that are needed in the subprocess
$FunctionNames = @(
  'Load-UserHives',
  'Set-RegistryKey',
  'Set-HKUKeyInner'
)

# parse them from memory, pull names in array out of Command bank
$FunctionBlocks = foreach ($Name in $FunctionNames) {
  $Function = Get-Command $Name -CommandType Function
  if ($Function) {
    # then reassemble them into valid PowerShell stored as a string
    "function $($Function.Name) {$($Function.Definition)}`n"
  }
}

# include our function definitions in a new script, then do what we need to
$SubprocessScript = @"
$($FunctionBlocks -join "`n`n")

Set-HKUKeyInner -SetDefault:`$$($SetDefault) -Verbose:$($null -ne $VerbosePreference)
"@

# encode the new script so it doesn't need to have awful escaping magic applied
$Encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($SubprocessScript))

# send the assembled script off to Charles's young son for usage
powershell.exe -NoProfile -EncodedCommand $Encoded
```

Anyway. That was fun! On to the next one.
