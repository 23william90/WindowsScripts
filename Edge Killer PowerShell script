# 1. Ensure target directory exists
$scriptDir = "C:\ProgramData\Scripts"
if (-not (Test-Path $scriptDir)) {
    New-Item -ItemType Directory -Path $scriptDir -Force | Out-Null
}

# 2. Write the hybrid Router + Interceptor script with Bing-to-Google rewriting & Caller Inspection
$vbsContent = @'
Option Explicit
Dim objShell, fso, ifeoKey, debuggerVal, chromeExe, i, rawArg, cleanUrl, finalArgs
Dim res, confirmRes, targetExe, args, promptMsg, callerInfo, rawParams

Set objShell = CreateObject("WScript.Shell")
Set fso = CreateObject("Scripting.FileSystemObject")

ifeoKey = "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\msedge.exe\"
debuggerVal = "wscript.exe """ & WScript.ScriptFullName & """"

' Locate Google Chrome
chromeExe = ""
If fso.FileExists("C:\Program Files\Google\Chrome\Application\chrome.exe") Then
    chromeExe = "C:\Program Files\Google\Chrome\Application\chrome.exe"
ElseIf fso.FileExists("C:\Program Files (x86)\Google\Chrome\Application\chrome.exe") Then
    chromeExe = "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"
ElseIf fso.FileExists(objShell.ExpandEnvironmentStrings("%LocalAppData%\Google\Chrome\Application\chrome.exe")) Then
    chromeExe = objShell.ExpandEnvironmentStrings("%LocalAppData%\Google\Chrome\Application\chrome.exe")
End If

finalArgs = ""

' Helper function to rewrite Bing search URLs to Google searches
Function ConvertBingToGoogle(ByVal url)
    Dim qIndex, queryPart, ampIndex, searchTerms
    
    If InStr(1, url, "bing.com/search", 1) > 0 Then
        qIndex = InStr(1, url, "q=", 1)
        If qIndex > 0 Then
            queryPart = Mid(url, qIndex + 2)
            ampIndex = InStr(queryPart, "&")
            If ampIndex > 0 Then
                searchTerms = Left(queryPart, ampIndex - 1)
            Else
                searchTerms = queryPart
            End If
            
            ' Preserve double-encoded literal '+' (%252B), convert encoded spaces (%2B / +) to %20
            searchTerms = Replace(searchTerms, "%252B", "___PLUS___", 1, -1, 1)
            searchTerms = Replace(searchTerms, "%2B", "%20", 1, -1, 1)
            searchTerms = Replace(searchTerms, "+", "%20")
            searchTerms = Replace(searchTerms, "___PLUS___", "%2B")
            
            ConvertBingToGoogle = "https://www.google.com/search?q=" & searchTerms
            Exit Function
        End If
    End If
    ConvertBingToGoogle = url
End Function

' Helper function to identify the parent application that launched Edge
Function GetParentProcessInfo()
    On Error Resume Next
    Dim objWMI, colProcs, objProc, myPid, parentPid, colParents, objParent
    
    Set objWMI = GetObject("winmgmts:\\.\root\cimv2")
    If Err.Number <> 0 Then
        GetParentProcessInfo = "Unknown (WMI unavailable)"
        Exit Function
    End If
    
    ' Find this wscript.exe process by matching its exact CommandLine
    myPid = 0
    Set colProcs = objWMI.ExecQuery("SELECT ProcessId, ParentProcessId, CommandLine FROM Win32_Process WHERE Name = 'wscript.exe'")
    For Each objProc In colProcs
        If InStr(1, objProc.CommandLine, WScript.ScriptFullName, 1) > 0 Then
            myPid = objProc.ProcessId
            parentPid = objProc.ParentProcessId
            Exit For
        End If
    Next
    
    If myPid = 0 Or parentPid = 0 Then
        GetParentProcessInfo = "Unknown"
        Exit Function
    End If
    
    ' Query the parent process using ParentProcessId
    Set colParents = objWMI.ExecQuery("SELECT Name, ExecutablePath FROM Win32_Process WHERE ProcessId = " & parentPid)
    For Each objParent In colParents
        If Not IsNull(objParent.ExecutablePath) And objParent.ExecutablePath <> "" Then
            GetParentProcessInfo = objParent.Name & " (" & objParent.ExecutablePath & ")"
        Else
            GetParentProcessInfo = objParent.Name
        End If
        Exit Function
    Next
    
    GetParentProcessInfo = "Unknown (Parent PID " & parentPid & " already exited)"
    On Error GoTo 0
End Function

' -------------------------------------------------------------
' Step A: Parse arguments to detect valid URLs or search queries
' -------------------------------------------------------------
' In IFEO, WScript.Arguments(0) is the path to msedge.exe
If WScript.Arguments.Count > 1 Then
    For i = 1 To WScript.Arguments.Count - 1
        rawArg = WScript.Arguments(i)

        ' Ignore Edge internal diagnostic/process/sandbox arguments
        If InStr(1, rawArg, "--type=", 1) = 0 And _
           InStr(1, rawArg, "--profile-directory=", 1) = 0 And _
           InStr(1, rawArg, "--service-sandbox-type=", 1) = 0 And _
           InStr(1, rawArg, "--mojo-platform-channel-handle=", 1) = 0 Then

            cleanUrl = rawArg

            ' Strip 'microsoft-edge:' or 'microsoft-edge:?' prefixes
            If LCase(Left(cleanUrl, 16)) = "microsoft-edge:?" Then
                cleanUrl = Mid(cleanUrl, 17)
            ElseIf LCase(Left(cleanUrl, 15)) = "microsoft-edge:" Then
                cleanUrl = Mid(cleanUrl, 16)
            End If

            ' If the parameter contains an embedded url= target, isolate it
            If InStr(1, cleanUrl, "url=", 1) > 0 Then
                Dim parts
                parts = Split(cleanUrl, "url=")
                If UBound(parts) >= 1 Then
                    cleanUrl = parts(1)
                    If InStr(cleanUrl, "&") > 0 Then
                        cleanUrl = Left(cleanUrl, InStr(cleanUrl, "&") - 1)
                    End If
                    cleanUrl = Replace(cleanUrl, "%3A", ":", 1, -1, 1)
                    cleanUrl = Replace(cleanUrl, "%2F", "/", 1, -1, 1)
                    cleanUrl = Replace(cleanUrl, "%3F", "?", 1, -1, 1)
                    cleanUrl = Replace(cleanUrl, "%3D", "=", 1, -1, 1)
                    cleanUrl = Replace(cleanUrl, "%26", "&", 1, -1, 1)
                    cleanUrl = Replace(cleanUrl, "%25", "%", 1, -1, 1)
                End If
            End If

            ' Convert Bing search links to Google search links
            cleanUrl = ConvertBingToGoogle(cleanUrl)

            ' Check if parameter represents a URL or web address (ignoring CLI flags starting with - or /)
            If Left(cleanUrl, 1) <> "-" And Left(cleanUrl, 1) <> "/" And ( _
               LCase(Left(cleanUrl, 7)) = "http://" Or _
               LCase(Left(cleanUrl, 8)) = "https://" Or _
               LCase(Left(cleanUrl, 7)) = "file://" Or _
               InStr(cleanUrl, "://") > 0 Or _
               InStr(cleanUrl, ".") > 0) Then
                finalArgs = finalArgs & " " & Chr(34) & cleanUrl & Chr(34)
            End If
        End If
    Next
End If

' -------------------------------------------------------------
' Step B: Branch depending on whether a destination was found
' -------------------------------------------------------------
If Trim(finalArgs) <> "" And chromeExe <> "" Then
    ' ROUTE DIRECTLY: Valid URL passed, open in Chrome
    objShell.Run Chr(34) & chromeExe & Chr(34) & finalArgs, 1, False
    WScript.Quit
End If

' -------------------------------------------------------------
' Step C: Fallback interactive prompt (No URL or manual launch)
' -------------------------------------------------------------
callerInfo = GetParentProcessInfo()

rawParams = ""
If WScript.Arguments.Count > 1 Then
    For i = 1 To WScript.Arguments.Count - 1
        rawParams = rawParams & WScript.Arguments(i) & " "
    Next
    rawParams = Trim(rawParams)
Else
    rawParams = "(None)"
End If

' 307 = 3 (Yes/No/Cancel) + 48 (Warning icon) + 256 (Default Button 2: No)
promptMsg = "Microsoft Edge is disabled on this computer." & vbCrLf & vbCrLf & _
            "Caller App: " & callerInfo & vbCrLf & _
            "Parameters: " & rawParams & vbCrLf & vbCrLf & _
            "• [Yes]    - Launch Edge this one time" & vbCrLf & _
            "• [No]     - Keep Edge blocked (Close)" & vbCrLf & _
            "• [Cancel] - Permanently remove this blocker"

res = MsgBox(promptMsg, 307, "System Notice")

Select Case res
    Case 6 ' --- YES: Launch Edge once and auto-relock ---
        On Error Resume Next
        objShell.RegDelete ifeoKey & "Debugger"
        On Error GoTo 0

        If WScript.Arguments.Count > 0 Then
            targetExe = WScript.Arguments(0)
            args = ""
            For i = 1 To WScript.Arguments.Count - 1
                args = args & " " & Chr(34) & WScript.Arguments(i) & Chr(34)
            Next
        Else
            targetExe = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
            args = ""
        End If

        If Not fso.FileExists(targetExe) Then
            If fso.FileExists("C:\Program Files\Microsoft\Edge\Application\msedge.exe") Then
                targetExe = "C:\Program Files\Microsoft\Edge\Application\msedge.exe"
            End If
        End If

        ' Run Edge synchronously (waits until all Edge windows close)
        objShell.Run Chr(34) & targetExe & Chr(34) & args, 1, True

        ' Re-lock Edge once closed
        objShell.RegWrite ifeoKey & "Debugger", debuggerVal, "REG_SZ"
        WScript.Quit

    Case 7 ' --- NO: Keep blocked and exit ---
        WScript.Quit

    Case 2 ' --- CANCEL: Remove modification permanently ---
        ' 308 = 4 (Yes/No) + 48 (Warning icon) + 256 (Default Button 2: No)
        confirmRes = MsgBox("Are you sure you want to permanently remove the Edge blocker and restore standard Edge behavior?", 308, "Confirm Removal")
        
        If confirmRes = 6 Then
            On Error Resume Next
            objShell.RegDelete ifeoKey & "Debugger"
            On Error GoTo 0
            
            MsgBox "The Edge blocker has been permanently removed. Microsoft Edge will now open normally.", 64, "Blocker Removed"
            
            ' Self-delete the script file
            objShell.Run "cmd.exe /c timeout /t 1 /nobreak >nul & del """ & WScript.ScriptFullName & """", 0, False
        End If
        WScript.Quit
End Select
'@

$vbsPath = "$scriptDir\edge_blocker.vbs"
Set-Content -Path $vbsPath -Value $vbsContent -Encoding ASCII

# 3. Configure IFEO registry hook
$ifeoPath = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\msedge.exe"
if (-not (Test-Path $ifeoPath)) {
    New-Item -Path $ifeoPath -Force | Out-Null
}
Set-ItemProperty -Path $ifeoPath -Name "Debugger" -Value "wscript.exe `"$vbsPath`"" -Type String

# 4. Grant standard users write access to this key so temporary bypass and uninstall work without UAC prompts
$acl = Get-Acl $ifeoPath
$userGroup = [System.Security.Principal.SecurityIdentifier]::new("S-1-5-32-545") # Builtin Users SID
$rule = New-Object System.Security.AccessControl.RegistryAccessRule($userGroup, "FullControl", "Allow")
$acl.AddAccessRule($rule)
Set-Acl -Path $ifeoPath -AclObject $acl

# 5. Disable Edge background startup/pre-warming
$policyPath = "HKLM:\SOFTWARE\Policies\Microsoft\Edge"
if (-not (Test-Path $policyPath)) {
    New-Item -Path $policyPath -Force | Out-Null
}
Set-ItemProperty -Path $policyPath -Name "StartupBoostEnabled" -Value 0 -Type DWord
Set-ItemProperty -Path $policyPath -Name "BackgroundModeEnabled" -Value 0 -Type DWord

Write-Host "Updated script applied: Hardened URL parsing, preserved literal '+' characters, and added Caller App + Parameter reporting." -ForegroundColor Green
