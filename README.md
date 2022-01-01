# Downloading a dodgy executable
## Mission Statement
I'm going to clone my win 10 Malware Analysis box, route it's web traffic through inetsim running in a remnux box, and start capturing it's web traffic.

But first i'm going to go to search for a download from the nastiest looking website I come across.

## Steps
1. spin up VMs, on windows change network adapter to NAT for the time being.  change DNS gateway to 8.8.8.8
2. grabbed 'restoro.exe' from a very reputable-looking website.  it feels very strange.
3. red flags on virustotal, here we go!  switch back to host based adapter and DNS gateway to the remnux VM
4. opened the file with pestudio to take a look
5. findings indicate i should get a snapshot of the registry before and after running this exe - gonna use regshot for this
6. captured before registry shot, let's run this executable!
7. start wireshark capture, filter source to the win10 box
8. try a regshot after running the restoro install
9. find lots of interesting stuff
10. take a look over at wireshark.  not much has happened BUT some IGMPv3 traffic has tried to head out to 224.0.0.22, and LLMNR traffic to 224.0.0.252.  something to do with multicasting, and with fallback after DNS doesn't work respectively
11. "resume restoro installation" just in case it does anything else
12. use FTK Imager to create a dump of memory
13. use volatility in Kali to look into the mem dump

## PEStudio
### Functions
a bunch of function calls are flagged as being on PEStudio's blacklist
`CreateProcessW` - quick search:
> If a malicious user were to create an application called "Program.exe" on a system, any program that incorrectly calls **CreateProcess** using the Program Files directory will run this application instead of the intended application.

you learn something new every day.

a bunch of functions for interacting with registry keys - set value, delete key, delete value

### Strings
Strings is interesting.  lots of links to certificate authorities - ironically over HTTP.

kernel32, psapi, user32, gdi32, shell32,advapi32,comctl32,ole32,version DLL files.  none are blacklisted.

references to registry keys all over the place

### Overlay
the PE overlay is red-flagged.  I learned that an overlay is simple extra data at the end of the PE in memory.  the "file ratio" is flagged at 93%, which I assume means that 93% of the size of the PE is in the overlay, and this suggests to me it's some kind of payload.  It's 868kb

## Regshot
since the strings in the PE referenced the registries, and some registry functions were called, i decided to take snapshots.  I made a before-execution snapshot and called it `reg_shot_before_1`

### compare after installing "restoro"
1 key deleted, 13 added, 1 value deleted, 44 values added, 31 values modified

the deleted key mentions ApplicationViewManagement and VirtualDesktop.  something to do with remote access...?

lots of mentions of JavaScript as well as OLEScript in the added keys and values

a lot of the modified values are in services\\tcpip\\parameters.  HMMM.

## Volatility
look into processes and process trees, checked several exes with virustotal and took a look at strings in procdump etc and all seemed to check out
connections/connscan returned nothing, and netscan revealed what mostly looks like they are to do with the system

not sure if relevant, but i attempted to dump one of the svchost.exe instances that had several connections in netscan and it failed "possibly due to paging".  should an svchost.exe ever be in the pagefile!?

two odd looking processes... PID: "20...0" and "393216".  can't proc dump either, neither has a PPID.  perhaps it's simply incomplete data

## Wireshark
Not much traffic seemed to be happening - the IGMP and LLMNR traffic mentioned above didn't seem to indicate anything suspicious.  Looks like the malware was not trying to communicate with any servers or controllers immediately following the install

#malwareanalysis #digitalforensics #reverseengineering #incidentresponse #indicatorsofcompromise
