# README

```javascript
// LicenseIndex*
e.hasActivated=true
```

## macos

```shell
rm -rf ~/Library/Caches/*
rm -rf *Extra* *Extended* *Heavy* *Medium* *Oblique* *Thin*
```

## windows

```shell
# CMD
ipconfig /flushdns
C:/Windows/System32/wsl.exe --distribution Ubuntu-22.04 --exec /usr/bin/zsh

netsh advfirewall firewall add rule name="ssh" dir=in action=allow protocol=TCP localport=22
netsh advfirewall firewall add rule name="proxy" dir=in action=allow protocol=TCP localport=8080
netsh advfirewall firewall add rule name="proxy" dir=in action=allow protocol=UDP localport=8080

# wsl
IFS=$'\n'
sudo rm -rf /mnt/c/Users/admin/AppData/Local/Temp
sudo rm -rf /mnt/c/Users/admin/AppData/Local/Microsoft/WindowsApps/python.exe
sudo rm -rf /mnt/c/Users/admin/AppData/Local/Microsoft/WindowsApps/python3.exe
```

Comment

```sh
//(?!.*\..*\.).*\n  # //
/\*(.|\r\n|\n)*?\*/ # /**/
^\s*(?=\r?$)\n      # blank line

^(\s*)#.*

[\u4e00-\u9fa5]                   # 中文
```
