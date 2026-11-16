---
layout: default
title: "PowerShell CLI and Unbound Parameters"
---

Let's assume that for whatever ill founded reason you chose PowerShell to write a CLI application. Your application needs to parse CLI flags and corresponding arguments. Let's also assume you want your CLI app to work as normally with multiple arguments natively, without the friction of quotes or commas, for e.g. `.\app.ps1 -unattended -flag1 arg1 arg2 -flag2 arg3 arg4 arg5`. 

The reality however is that PowerShell’s parameter binder aggressively parses space-separated strings resulting in some of the weirdest string parsing behaviors possible. 
## The core issue
PowerShell has some odd parameter binding mechanics when working space delimited strings. If a parameter is bound as `[string[]]`,  PowerShell binds the first space separated word to the string array. When the parser encounters a space after  `arg1`, it considers the `-flag1` parameter satisfied and trailing words like `arg2` are swept into next available positional parameter. In this case `flag2`. Consequently, the script attempts to parse the `flag1` name as an argument for `-flag2`
## The native PowerShell solution
The PowerShell way of doing things is to use the `[Parameter(ValueFromRemainingArguments=$true)]` as a catch all net. This will force PowerShell to grab all the unmapped space separated strings and group them into a single array and prevent the arguments from leaking into unrelated parameters.

However this also means whole sequence of arguments are loaded into a catch all array. For example,  the arguments `-flag1 A B -flag2 X Y` results in `["-flag1", "A", "B", "-flag2", "X", "Y"]`. While this may be an acceptable solution in some circumstances, it's also extremely fragile. For example having to write specialized logic to parse special characters or extra logic in order to determine which argument belongs to which flag.

Lets see some failed approaches
## Failed Approaches
Before arriving at the right solution, I had to try several attempts at positionally slice the array. Needless to say all of them failed because of PowerShell's dynamic binding.
### Failed Approach 1: Slicing by Index (`IndexOf`)
```powershell
param (
    [switch]$unattended,
    [string[]]$flag1,
    [Parameter(ValueFromRemainingArguments = $true)]
    [string[]]$remainingArgs
)

$parsedCategories = @()
$flag1Index = [array]::IndexOf($remainingArgs, '-flag1')

if ($flag1Index -ge 0) {
    $parsedCategories = @(
        $remainingArgs[($flag1Index + 1)..($remainingArgs.Count - 1)]
    )
}

$params = ConvertTo-Json -InputObject @($parsedCategories) -Compress
Write-Host "Flag1 Index: $flag1Index"
Write-Host "Parsed flag1 type:"
Write-Host $parsedCategories.GetType().FullName
Write-Host "Params = $params"
```

This fails because although `$remainingArgs` (with `ValueFromRemainingArguments`) sweeps up the orphaned arguments (`arg2`, `arg3`, `-flag2`, `arg4`, `arg5`), PowerShell would have already consumed `-flag1` and `arg1` during named parameter binding on `$flag1`. Because the literal string `'-flag1'` was stripped during the invocation, searching for `'-flag1'` inside `$remainingArgs` yields `-1`.
### Failed Approach 2: Hardcoded Array Slicing
```powershell
param (
    [switch]$unattended,
    [Parameter(Position = 0)]
    [string[]]$flag1,
    [Parameter(ValueFromRemainingArguments = $true)]
    [string[]]$remainingArgs
)

$parsedCategories = @($flag1) + @($remainingArgs)
$params = ConvertTo-Json -InputObject $parsedCategories -Compress

Write-Host "Parsed flag1 type:"
Write-Host $parsedCategories.GetType().FullName
Write-Host "Params = $params"
```

This fails for a very different reason. When both flags are provided, combining `@($flag1) + @($remainingArgs)` flattens everything into a single array: `["arg1","arg2","arg3","-flag2","arg4","arg5"]`. Because `$remainingArgs` blindly sweeps up every remaining token on the line, it also swallows the literal strings `"-flag2"`, `"arg4"`, and `"arg5"`, completely destroying the parsing boundaries between the two flags.
### Building a State Machine Parser
To handle multiple open ended lists correctly, its possible to manually iterate over the `ValueFromRemainingArguments` array. A `$currentFlag` state variable tracks the last seen flag and route the subsequent strings into their respective buckets (`$flag2` or `$flag1`). This ensures errorless parsing regardless of the input order.
```powershell
param (
    [switch]$unattended,
    [Parameter(ValueFromRemainingArguments = $true)]
    [string[]]$arguments
)

$flag1 = @()
$flag2 = @()

if ($arguments -and $arguments.Count -gt 0) {
    $currentFlag = ""
    foreach ($arg in $arguments) {
        if ($arg -eq "-flag1" -or $arg -eq "-flag2") {
            $currentFlag = $arg
            continue
        }
        
        if ($currentFlag -eq "-flag1") {
            $flag1 += $arg
        }
        elseif ($currentFlag -eq "-flag2") {
            $flag2 += $arg
        }
    }
}

Write-Host "flag1 arguments - $flag1"
Write-Host "flag2 arguments - $flag2"
```
### Idempotent Deduplication with Hashtables
While handling overlapping requests, for example a tool matching multiple requested categories, or requested by both category and name this becomes another problem as an argument can be run multiple times. In scenarios like our example of installing tools, the solution is store the tools in a hashtable (`$UniqueToolsMap[$program.Name] = $true`). This inherently deduplicates the final list before parsing.
```powershell
param (
    [switch]$unattended,
    [Parameter(ValueFromRemainingArguments = $true)]
    [string[]]$arguments
)
$flag1 = @() # Categories
$flag2 = @() # Explicit tool names

# State Machine Parser
if ($arguments -and $arguments.Count -gt 0) {
    $currentFlag = ""
    foreach ($arg in $arguments) {
        if ($arg -eq "-flag1" -or $arg -eq "-flag2") {
            $currentFlag = $arg
            continue
        }
        if ($currentFlag -eq "-flag1") {
            $flag1 += $arg
        }
        elseif ($currentFlag -eq "-flag2") {
            $flag2 += $arg
        }
    }
}

# Mock Catalog
$ToolCatalog = @(
    [PSCustomObject]@{ Name = "ghidra"; Categories = @("reversing", "malware") }
    [PSCustomObject]@{ Name = "x64dbg"; Categories = @("reversing", "debuggers") }
    [PSCustomObject]@{ Name = "wireshark"; Categories = @("net", "dfir") }
    [PSCustomObject]@{ Name = "velociraptor"; Categories = @("dfir", "hunting") }
)
# Idempotent Deduplication via Hashtable
$UniqueToolsMap = @{}
foreach ($program in $ToolCatalog) {
    # Check if the tool matches any requested category in $flag1
    $matchesCategory = $false
    foreach ($cat in $flag1) {
        if ($program.Categories -contains $cat) {
            $matchesCategory = $true
            break
        }
    }

    # Check if the tool was also explicitly requested by name in $flag2
    $matchesExplicitName = $flag2 -contains $program.Name

    # Assigning the key overwrites duplicates in-place instead of appending
    if ($matchesCategory -or $matchesExplicitName) {
        $UniqueToolsMap[$program.Name] = $true
    }
}
$FinalInstallQueue = @($UniqueToolsMap.Keys)

Write-Host "flag1 (Categories)   : $flag1"
Write-Host "flag2 (Explicit)     : $flag2"
Write-Host "Deduplicated Queue   : $($FinalInstallQueue -join ', ')"
```

Sample run:
```
.\test.ps1 -flag1 reversing malware dfir -flag2 ghidra wireshark wireshark
flag1 (Categories)   : reversing malware dfir
flag2 (Explicit)     : ghidra wireshark wireshark
Deduplicated Queue   : wireshark, x64dbg, velociraptor, ghidra
```
### Strict vs. Fuzzy Matching
Continuing from the above example, this still runs into problems after using a shorthand like `-flag1 firefox` instead of the exact `"Mozilla Firefox"`  in the catelog. This happens because the `-contains`  requires a strict exact string match against the array. 

The solution is to use the nested fuzzy `-match`  Regex evaluation instead of `-contains` array operator. Through safely escaping the user's input with `[regex]::Escape()`, the script can accurately check if catalog string includes the provided shorthand.
```powershell
param (
    [switch]$unattended,
    [Parameter(ValueFromRemainingArguments = $true)]
    [string[]]$arguments
)

$flag1 = @() # Specific tool shorthand names (e.g., firefox, 7zip, chrome)
$flag2 = @() # Requested categories (e.g., browsers, reversing)

# State Machine Argument Parser
if ($arguments -and $arguments.Count -gt 0) {
    $currentFlag = ""
    foreach ($arg in $arguments) {
        if ($arg -eq "-flag1" -or $arg -eq "-flag2") {
            $currentFlag = $arg
            continue
        }
        if ($currentFlag -eq "-flag1") {
            $flag1 += $arg
        }
        elseif ($currentFlag -eq "-flag2") {
            $flag2 += $arg
        }
    }
}

# Mock JSON Catalog
$jsonCatalog = @'
[
    { "Name": "Google Chrome",   "Category": "browsers" },
    { "Name": "Mozilla Firefox", "Category": "browsers" },
    { "Name": "7-Zip",           "Category": "utilities" },
    { "Name": "Notepad++",       "Category": "utilities" },
    { "Name": "Ghidra",          "Category": "reversing" }
]
'@

$list = $jsonCatalog | ConvertFrom-Json
$StrictToolsMap = @{}
$UniqueToolsMap = @{}

# Map by Specific Tool Names (Strict vs. Fuzzy Comparison)
if ($flag1.Count -gt 0) {
    # Format the cleanly parsed installs
    $targetInstalls = $flag1 | ForEach-Object { $_.Trim().ToLower() }
    Write-Host "Filtering installations for Specific Tools: $($targetInstalls -join ', ')" -ForegroundColor Cyan
    foreach ($program in $list) {
        if ($null -ne $program.Name) {
            $toolName = $program.Name.ToLower()
            # --- A: Why Strict Exact Match (-contains) Fails ---
            # Evaluates if ["chrome", "firefox", "notepad++"] contains "mozilla firefox" -> $false
            if ($targetInstalls -contains $toolName) {
                $StrictToolsMap[$program.Name] = $true
            }
            # --- B: Production Fuzzy Match ([regex]::Escape + -match) ---
            foreach ($target in $targetInstalls) {
                if ($toolName -match [regex]::Escape($target)) {
                    # Add the true JSON name to the deduplication map
                    $UniqueToolsMap[$program.Name] = $true
                    break # Move to the next program once a match is found
                }
            }
        }
    }
}

Write-Host "`n[Strict -contains Result]Matched ($($StrictToolsMap.Count)) : $($StrictToolsMap.Keys -join ', ')" -ForegroundColor Yellow
Write-Host "[Fuzzy  -match   Result] Matched ($($UniqueToolsMap.Count)) : $($UniqueToolsMap.Keys -join ', ')" -ForegroundColor Green
```
### Conclusion
Unfortunate thing about building CLI tooling in PowerShell is that it requires custom parsing and fuzzy matching in order to bridge the gap between intuitive UX and rigid code. Understanding the right techniques helps, but it's still not as intuitive as Python or other alternatives.
