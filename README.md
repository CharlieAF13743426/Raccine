# Raccine

A Simple Ransomware Vaccine

## Why

We see ransomware delete all shadow copies using `vssadmin` pretty often. What if we could just intercept that request and kill the invoking process? Let's try to create a simple vaccine.

![Ransomware Process Tree](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen4.png)

## How it works

We [register a debugger](https://attack.mitre.org/techniques/T1546/012/) for `vssadmin.exe` which is our compiled `raccine.exe`. Raccine is a binary, that first collects all PIDs of the parent processes and then tries to kill all parent processes. 

Avantages:

- The method is rather generic
- We don't have to replace a system file (`vssadmin.exe`), which could lead to integrity problems and could break our raccination on each patch day 
- The changes are easy to undo
- Should work on all Windows versions from Windows 2000 onwards
- No running executable or additional service required (agent-less)

Disadvantages / Blind Spots:

- The legitimate use of `vssadmin.exe delete shadows` isn't possble anymore
- It even kills the processes that tried to invoke `vssadmin.exe delete shadows`, which could be a backup process
- This won't catch methods in which the malicious process isn't one of the processes in the tree that has invoked `vssadmin.exe` (e.g. via `wmic` or `schtasks`)

## Pivot

In case that the Ransomware that your're currently handling uses a certain process name, e.g. `taskdl.exe`, you could just change the `.reg` patch to intercept calls to that name and let Raccine kill all parent processes of the invoking process tree.

## Warning !!!

You won't be able to run `vssadmin.exe delete shadows` on a raccinated machine anymore until your apply the uninstall patch `raccine-reg-patch-uninstall.reg`. This could break various backup solutions that run that specific command during their work. It will not only block that request but kills all processes in that tree including the backup solution and its invoking process.

If you have a solid security monitoring that logs all process executions, you could check your logs to see if `vssadmin.exe` is frequently or sporadically used for legitimate purposes in which case you should refrain from using Raccine. 

## Version History

- 0.1.0 - Initial version that intercepted blocked all vssadmin.exe executions
- 0.2.0 - Version that blocks only vssadmin.exe executions that contain `delete` and `shadows` in their command line and otherwise pass all parameters to a new process that invokes vssadmin with its original parameters
- 0.2.1 - Removed `explorer.exe` from the whitelist
- 0.3.0 - Supports the `wmic` method calling `delete shadowcopy`, no outputs for whitelisted process starts (avoids problems with wmic output processing)

## Installation

1. Apply Registry Patch `raccine-reg-patch.reg`
2. Place `raccine.exe` from the [release section](https://github.com/Neo23x0/Raccine/releases/) in the `PATH`, e.g. into `C:\Windows`

### Addon

3. Also apply the `raccine-reg-patch-uninstall.reg` patch to intercept calls to `wmic`, which I consider much riskier. 

![Kill Run](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen5.png)

## Screenshot

Run `raccine.exe` and watch the parent process tree die. 

![Kill Run](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen1.png)

## Help Wanted 

I'd like to extend Raccine but lack the C++ coding skills, especially o the Windows platform.

### ~~1. Allow Certain Vssadmin Executions~~

***implemented by Ollie Whitehouse in v0.2.0***

Since Raccine is registered as a debugger for `vssadmin.exe` the actual command line that starts raccine.exe looks like

```
raccine.exe vssadmin.exe ... [params]
``` 

![raccine as debugger](https://raw.githubusercontent.com/Neo23x0/Raccine/main/images/screen3.png)

If we were able to process the command line options and apply filters to them, we could provide the following features: 

- Only block the execution in cases in which the parameters contains `delete shadows`
- Allow all other executions by passing the original parameters to a newly created process of `vssadmin.exe` (transparent pass-through)

### 2. Whitelist Certain Parents

We could provide a config file that contains white-listed parents for `vssadmin.exe`. If such a parent is detected, it would also pass the parameters to a new process and skip killing the process tree.
