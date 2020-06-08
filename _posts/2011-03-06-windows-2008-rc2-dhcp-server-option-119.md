---
title: Windows Server 2008 RC2 DHCP Server Option 119
tags: dhcp windows
redirect_from: /2011/03/windows-2008-rc2-dhcp-server-option-119/
---

If you’ve scoured through Windows Server configurations for the DHCP server looking to set the Search Domains and have come up empty, there’s good reason: Most, if not all, versions of Windows do not support setting Search Domains via DHCP (option 119), thus Microsoft does not include a visible option to set this on their DHCP servers.

99% of the computers used at my company are Windows based, so we use GPO to push down the search domains and it works pretty well. We do, however, have iPads used by upper management, as well as Android users connecting to the corporate wifi and a few of us using Linux based operating systems which won’t accept Microsoft’s GPO. We were essentially out in the cold unless we manually configured our networking options to add all of the search domains used by our company.

Someone in Executive Management requested that the “GPO only” push of search domains be changed to be included in the DHCP server for any non-Windows users. After 3 hours of troubleshooting, searching the web, and scouring RFC’s, we finally implemented it. Here are some notes about our journey: Technet is wrong when it explains how to add this functionality; everyone who says just use GPO simply didn’t get that non-Windows couldn’t use GPO; [Stephen](http://www.stephenjc.com/2009/04/07/enabling-dhcp-option-119-on-2003-server/) was close in his explanation, but that still didn’t work (chankster even pointed him to the RFC that helped me, but since it didn't seem to affect his results).

The size does indeed have to be per domain component (excluding the ‘.’); but the size also comes BEFORE the domain component, not after. The domain in it’s entirety also needs to be null terminated. So here’s an example: apple.com (we’ll use Stephen’s example as a base).

We have two domain components: apple and com.  Translated to hex, we get the following:

    a - 0x61
    p - 0x70
    p - 0x70
    l - 0x6c
    e - 0x65
    c - 0x63
    o - 0x6f
    m - 0x6d

_apple_ is 5 characters (`0x05`) and _com_ is 3 (`0x03`), AND we need to null terminate (`0x00`).  Our full string then becomes:

    text:         a    p    p    l    e         c    o    m 
    hex:   0x05 0x61 0x70 0x70 0x6c 0x65 0x03 0x63 0x6f 0x6d 0x00
    
Each one of these needs to be individually added as a separate byte in the array for the 119 option in the DHCP server configuration (remember to null terminate the entries with 0x00). Once we made this change and saved it, our non-Windows based clients were then able to get the Search Domains via DHCP (note: it appears Android does not support option 119 as well, at least from my testing with packets from Wireshark).

Hope this helps someone out!