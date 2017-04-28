---
title: 逆向wininet做HINTERNET和raw socket handler的mapping
layout: post
categories: 开发
tags: 逆向 开发
author: John He
---

* content
{:toc}

wininet.dll是Windows提供的通过HTTP, FTP等应用层协议访问网络资源的一个库。

提供的API例如InternetOpen, InternetConnect, HttpOpenRequest, HttpSendRequest等。

HINTERNET作为wininet提供给用户应用的句柄, HttpOpenRequest, HttpSendRequest发起请求均是通过HINTERNET句柄来完成.

这个HINTERNET并不是raw socket handler，而是wininet封装raw socket handler提供给用户应用使用的一个数据结构。

这个HINTERNET带来了很多不便:
- HINTERNET并没有暴露足够多的接口去获取所有的TCP连接信息;
- 即便HINTERNET提供了足够的接口, 但是这些接口并不是socket标准接口, 获取不到raw socket handler, 很多标准socket方法无法使用。

于是便萌生了想法, 逆向一下wininet.dll看有没有办法挖出HINTERNET和raw socket handler之间的关系, 从而获取到raw socket handler.

用WinDbg抓下wininet.dll的symbol, 用Visual Studio反汇编, 单步跟进HttpSendRequest方法.

发现HttpSendRequest调用了一个_mapHandleToAddress的内部方法.

从名字上看，这个内部方法很像是我要找的方法。

继续用Visual Studio的Assembly模式，跟进_mapHandleToAddress内部, 从汇编代码逻辑可以看到, 该方法接受HINTERNET为参数, 在一个全局数据结构中以HINTERNET作为key查找, 辗转调用其他几个内部方法获取了一个value.

这个value就是我要找的raw socket handler.

大体逆向出来的流程和细节如下：

- 其中_mapHandleToAddress用ecx, esi接收HINTERNET参数, 返回值放在edx指向的内存单元(这个不同于常规函数, 并未使用eax存返回值);

- 用_mapHandleToAddress的返回值, 调用另外几个内部方法GetSourcePort, GetDestPort, GetSocket等, 其中GetSocket返回的便是我要找的raw socket handler

那用GetSocket的返回值能否调用标准的socket函数呢?

用如下代码验证一下:

```C
HANDLE hProcess = GetCurrentProcess();
DWORD cbNeeded;
HMODULE hMods[1024];
unsigned int i = 0;
if (EnumProcessModules(hProcess, hMods, sizeof(hMods), &cbNeeded)) {
       for (i = 0; i < (cbNeeded / sizeof(HMODULE)); i++) {
             TCHAR szModName[MAX_PATH];
             MODULEINFO modinfo = { 0 };
 
             // Get module’s name.
 
             if (GetModuleBaseName(hProcess, hMods[i], szModName,
                    sizeof(szModName) / sizeof(TCHAR)))
             {
                    char pModName[sizeof(szModName) / sizeof(TCHAR)];
                    int iLength = WideCharToMultiByte(CP_ACP, 0, szModName, -1, NULL, 0, NULL, NULL);
                    WideCharToMultiByte(CP_ACP, 0, szModName, -1, pModName, iLength, NULL, NULL);
 
                    if (strcmp(pModName, "WININET.dll") == 0) {
                           if (GetModuleInformation(hProcess, hMods[i], &modinfo, sizeof(modinfo)) != 0) {
 
                                 LPVOID baseOfDll = modinfo.lpBaseOfDll;
 
                                 // TODO: get entry point of MapHandleToAddress dynamically rather than hardcoding offset.
                                 ULONG64 entryOfMapHandleToAddress = (ULONG64)hMods[i] + 0xB08F0;
                                 PDWORD pdwRet = new DWORD[1];
 
                                 // TODO: get entry point of GetSourcePort dynamically rather than hardcoding offset.
                                 ULONG64 entryOfGetSourcePort = (ULONG64)hMods[i] + 0x16EA39;
                                 PDWORD pdwSourcePort = new DWORD[1];
 
                                 // TODO: get entry point of GetDestPort dynamically rather than hardcoding offset.
                                 ULONG64 entryOfGetDestPort = (ULONG64)hMods[i] + 0x16E965;
                                 PDWORD pdwDestPort = new DWORD[1];
 
                                 // TODO: get entry point of GetSocket dynamically rather than hardcoding offset.
                                 ULONG64 entryOfGetSocket = (ULONG64)hMods[i] + 0x16EA03;
                                 PDWORD pdwSocket = new DWORD[1];
 
                                 __asm {
                                        mov ecx, hFile
                                        mov esi, hFile
                                        mov edx, pdwRet
                                        call entryOfMapHandleToAddress
                                        sub esp, 0x04 /* Not sure why MapHandleToAddress doesn't restore esp correctly, manually restore it */
 
                                        mov ecx, pdwRet
                                        mov ecx, [ecx]
                                        call entryOfGetDestPort
                                        mov ebx, pdwDestPort
                                        mov dword ptr[ebx], eax
 
                                        mov ecx, pdwRet
                                        mov ecx, [ecx]
                                        call entryOfGetSourcePort
                                        mov ebx, pdwSourcePort
                                        mov dword ptr[ebx], eax
 
                                        mov ecx, pdwRet
                                        mov ecx, [ecx]
                                        call entryOfGetSocket
                                        mov ebx, pdwSocket
                                        mov dword ptr[ebx], eax
                                 }
 
                                 printf("source port: %d, destination port: %d\n", *pdwSourcePort, *pdwDestPort);
 
                                 struct sockaddr_in serv, guest;
                                 PWSTR serv_ip = new wchar_t[128];
                                 PWSTR guest_ip = new wchar_t[128];
                                 int serv_len = sizeof(serv);
                                 int guest_len = sizeof(guest);
                                 memset(&serv, 0, serv_len);
                                 memset(&guest, 0, guest_len);
                                 int ret1 = getsockname(*pdwSocket, (struct sockaddr *)&serv, &serv_len);
                                 int ret2 = getpeername(*pdwSocket, (struct sockaddr *)&guest, &guest_len);
                                 if (ret1 == 0 && ret2 == 0)
                                 {
                                        char ServAddrName[NI_MAXHOST];
                                        if (getnameinfo((LPSOCKADDR)&guest, guest_len, ServAddrName, sizeof(ServAddrName), NULL, 0, NI_NUMERICHOST) != 0)
                                        {
                                               //strcpy(ServAddrName, "<unknown>");
                                        }
 
                                        char ClientAddrName[NI_MAXHOST];
                                        if (getnameinfo((LPSOCKADDR)&serv, serv_len, ClientAddrName, sizeof(ClientAddrName), NULL, 0, NI_NUMERICHOST) != 0)
                                        {
                                               //strcpy(ServAddrName, "<unknown>");
                                        }
 
                                        printf("server ip: %s, client ip: %s\n", ServAddrName, ClientAddrName);
                                 }
                           }
                    }
             }
}
}
```


可见, 通过调用GetSocket内部方法返回的pdwSocket, 用来调用getpeername, getsockname这样的标准socket函数, 能得到正确的结果.



