Title: Foul Play: Privilege Escalation on the Playdate
Date: 2025-06-28
Modified: 2025-06-28
Category: playdate, networking, exploits
Tags: playdate, networking, exploits
Slug: foul-play
Authors: Peter Stefek
Summary: Sneaking in the backdoor

<p align="center">
	<img src="/images/foul-play/door.svg" width="300vw"> 
</p> 

In the summer of 2024 I was working on [a game](https://play.date/games/jump-truck/) for the [Playdate](https://play.date/) console. The console, which I lovingly called a Gameboy for hipsters, is built by [Panic](https://panic.com/) and strives to emulate the charm of a simpler time in games. It features a black and white screen with no backlight and limited computing power compared to today's consoles like the Switch or the Steam Deck.

Unlike the original Gameboy, the Playdate has wifi built in. But at the time, games could not freely use the wifi features. The only interaction games had with the wifi was through Panic’s highscore api which allowed approved games to get and set high scores on special leaderboards hosted by Panic.

Mid summer, I took a break from game development and visited my friend [Nick Spinale](https://nickspinale.com/) for a week in beautiful Portland Oregon. Nick is a brilliant guy and has an encyclopedic knowledge of the ARM Cortex M instruction set from his time working there. By coincidence, the Playdate happens to use a Cortex M7 processor. So one evening I asked him to help me find a way for games to get full access to the wifi card. He agreed and over the next few days we spent some spare time thinking about the problem.

**Starting out**

The first thing we wanted to do was look at the code for the high score functions. All of the device specific functions (including highscores) are referenced through a pointer to a PlaydateAPI structure.

<p align="center">
	<img src="/images/foul-play/playcake.svg" width="300vw"> 
</p>   

Here is a hello world example in c on the Playdate demonstrating this structure:

	:::c  
	#include <stdio.h>
	#include <stdlib.h>

	#include "pd_api.h"

	int eventHandler(
		PlaydateAPI* pd,
		PDSystemEvent event,
		uint32_t _arg) {
		if ( event == kEventInit )
		{
			pd->console->log("hello world");
		}
		return 0;
	}



Nick’s first observation was that the Playdate Sdk is not a library which is statically linked into the game binary at compile time like I previously assumed. Instead the PlaydateAPI pointer points to a special region of device memory which contains the sdk. The process is an ad-hoc version of dynamic linking which allows Panic to upgrade the sdk when they release new Playdate OS versions.

With this information we set about dumping the sdk and as much other the device memory as possible by having the application read the areas around known pointer addresses (such as the PlaydateAPI* pointer). Unfortunately the Cortex M7 has a Memory Protection Unit which stopped us from reading some regions of memory so we didn’t get the os code but the sdk was a great starting point. 

Nick loaded the sdk binary into [Ghidra](https://github.com/NationalSecurityAgency/ghidra), the NSA's reverse engineer framework, and began to decompile it while I, being more useless, poked around the remaining memory with the “strings” command line utility. 

The Playdate binary I had built still contained debug symbols which made it an easy starting point. But still neither of us were experienced reverse engineers so the work was slow going. As the evening dragged on we began to doubt ourselves a little. Maybe this was beyond what we were capable of, especially in a couple of days? During this period I was still slowly dumping memory regions. And as we were debating giving up I noticed that the strings utility had surfaced a familiar phrase in one of the memory region dumps. It was my wifi password. 

While it wasn’t a critical discovery, the realization that my wifi password was accessible to arbitrary applications on the Playdate renewed our faith that the device’s security might not be perfect.

**Isolating the High Score Function**

Nick was able to locate the function that the sdk used to send http requests in Ghidra. Upon inspecting the function we saw that it called a different method deep in the os region of memory (which we did not have access to) and passed an odd looking string which we realized was an Espressif [At Command](https://docs.espressif.com/projects/esp-at/en/latest/esp32/AT_Command_Set/Wi-Fi_AT_Commands.html#). Espressif At commands control the ESP32 wifi chip that the Playdate uses for wifi and bluetooth. We were hoping the os function we found could run arbitrary At commands which would give us full access to the wifi. Unfortunately when we tried to run the os command in our binary, the device crashed and error code we got indicated a permissions error.

To try to overcome the permissions issue we attempted to overwrite the sdk memory containing the At command string which was passed into the os function. Unfortunately it appeared we had real-only access to the section of memory containing the Playdate Sdk code so this resulted in another permissions error. 

The permissions issues seemed like a brick wall. At this point Nick and I realized we had several options.

1. Figure out a way to get or reverse engineer the Playdate OS Code
2. Google for inspiration or hints
3. Continue to poke around and hope we found something


Option (1) seemed difficult at the time. We did not know how to access the on device OS code to reverse engineer it. And we could not find a leak of the code on Google. The only other way to get the code was through a human being. We knew Panic did send firmware to people who claimed to have Playdate issues but that seemed too underhanded and even then it would take too long given our limited time frame. [ref]Tricking Panic might not have worked anyway because the Playdate os firmware is encrypted and we’d have to find a way to pull the decryption key off the devic to decompile it[/ref]

We did try a little bit of option (2). I poked around the Google and the Playdate Squad discord to see if anyone else had written about accessing the wifi. We also looked quickly at a project called [IndexOS](https://scratchminer.github.io/Index-OS-Website/index) which is an alternative game launcher for the Playdate. It seemed like IndexOS’s installer was able to get around some system permissions to swap out it's game launcher with the default one. But after half an hour we couldn’t see any obvious exploits in its code (we’ll revisit IndexOS later). Due to time constraints we decided to cut our losses here and move on.

Having tried options (1) and (2) Nick and I decided to fall back on option (3), deepen our understanding of the system and hope we got lucky.

**Access Control on the Cortex M7**

With other options looking slim, Nick started to figure out how the permissions model worked.

The Cortex M7 processor can run in one of two modes: handler mode or thread mode. Handler mode is specifically for interrupt handlers. Thread mode is where all application code such as the Playdate games runs. It’s also where large chunks of the OS run. 

Code in handler mode always runs with privileged access. Privileged access allows the code unrestricted access to the system. Code in thread mode can run in either unprivileged or privileged mode. The OS runs in privileged mode while applications generally run in unprivileged mode. 

However any code running in thread mode can ask to have its privileges upgraded by raising a SuperVisor (SVC) exception with code 2. This exception is processed by a special handler which will decide if the privilege upgrade request is allowed. If the request is allowed, the processor’s thread mode privileges will be raised. Other thread mode code can then later lower the privileges back down by flipping a bit on the control register.
<p align="center">
	<img src="/images/foul-play/iotality.png" width="80%"> 
</p>  
(image sourced from [iotality](https://www.iotality.com/armcm-access-levels/))

The Playdate Sdk used this temporary privilege escalation ability to interact with the network card. For example when a user application called the get_score function the Playdate Sdk first ran a function we deemed EnsurePrivileges. EnsurePrivileges raised an exception via the [SVC (SuperVisor Call) assembly instruction](https://developer.arm.com/documentation/dui0489/c/arm-and-thumb-instructions/miscellaneous-instructions/svc). The exception was caught by the Svc Handler program which ran a check to see if the thread mode program should be granted privileged access to the system. Once this check passed, the thread entered privilege mode and the Sdk made a network call to Panic’s servers via the code Nick found earlier. The application later relinquished its privileges.

<p align="center">
	<img src="/images/foul-play/get_scores_hypo.svg" width="80%"> 
</p>  

*Note this image is a high simplified hypothetical diagram. It's not meant to represent the underlying system in detail.*

After Nick figured out the privilege system, we tried the simplest thing possible. We called the EnsurePrivileges method ourselves. I had written some code to check the privilege bit on the control register which would let us know if we had succeeded. Unsurprisingly this simple attack did not work. It was like trying to kick down a locked door. We also tried to call the SVC instruction ourselves but were again met with a denial. We speculated that the privilege check had some way to tell which section of code raised the interrupt and deny sections that held user code. Unfortunately since we could not access the os code we could not look at how that privilege check worked. We decided this was a dead end and somewhat dejectedly went to bed.  


**Privilege Escalation**

The next morning I woke up early with one last idea. The Playdate high score api functions have to be asynchronous because they cannot stall the games while they are running. Due to their asynchronous nature, they do not return the high score values they fetch directly. Instead they all take a callback which is called after the network results are fetched. If the callback was mistakenly called while the processor’s thread mode access level was still privileged, any code in that callback would run with full privileged access. It was a bit of a long shot but it’s also the kind of mistake I could see myself making.


<p align="center">
	<img src="/images/foul-play/get_scores_hypo_cb.svg" width="80%"> 
</p>  

*Note this image is a high simplified diagram. It hides a lot of detail I'm not too clear on including the underlying multi task system involved in the asynchronous request fetching.*


The test for this idea was extremely simple to write and after a few deep breaths I ran the code. The privilege bit check in the callback function confirmed my hunch. I had gained privileged access to the system. 

The next step was to actually make a custom wifi call. I tried a dns lookup for [google.com](http://google.com) and it was successful. I felt pretty good. While this seems like it would be a trivial issue for a security engineer to find, I had never tried any kind of “hacking” before so it was really cool seeing something I contributed to work.

I showed Nick and then we both turned off our computers and went outside for the rest of the day.

What we had found was a way for any application (or game) on the Playdate to gain full control of the device. That meant in theory Playdate games could now use networking. Unfortunately that also meant games could ruin your device, buy things for you and steal your wifi password. So I tried to do the mature thing and reported it to Panic.

They responded very quickly and said they were already aware of the issue and were going to fix it as soon as they could.

**The End?**

A week later I was back home. But even far away from Portland I was still haunted by some of the loose ends hanging off our investigation. I still didn't understand much about the Playdate's operating system and I couldn't help feeling there something we missed about custom launchers like IndexOS.

Roaming around the city I felt like a noir detective who has been told he solved the case, but just can’t shake the feeling that there is something more going on. So I went back to my computer, determined to find some answers.

The first question was pretty simple. Now that I understood a little more about the internals of the Playdate, some strategic googling revealed the Playdate OS was based on Amazon’s FreeRTOS Kernel. Reading the docs a bit I noticed that the type of exploit Nick and I found is explictly called out in their [threat model section](https://www.freertos.org/Security/02-Kernel-threat-model#:~:text=Exploiting%20system%20calls%20which%20take%20a%20function%20pointer%20as%20a%20parameter%20to%20achieve%20arbitrary%20code%20execution).

Next I decided to revisit IndexOS. After running a few tests I confirmed IndexOS’s installer was definitely escalating its privileges to move files where it wasn’t supposed to. But how was the installer doing it and what had we missed before?

Playdate projects are traditionally built in c, lua or rarely a combination of both. When Nick and I glanced at IndexOS we had assumed it was a project entirely built in Lua. Almost all the code was in lua files and the tiny binary that came with it appeared to be just the stock lua entry point when we first looked at it. 

**A Second Privilege Escalation**

This time I didn’t have a deadline and knew more about what to look for so I was better prepared. Using some very basic Ghidra skills I had picked up from Nick (and youtube) I decompiled the tiny binary in IndexOS’s pdx bundle and noticed that in addition to the stock entry point there were a few other methods. One in particular containing 5 mysterious assembly instructions caught my eye:

	:::
	push {lr}
	movs.w lr,#0x0
	movt lr,#0x805
	svc 0x2
	pop {pc}

I already knew instruction 4 was the SuperVisor Call where the application asked for privileged permissions. Nick and I had already tried calling this method earlier in our investigation to no avail. But when I isolated the 5 assembly instructions into my own test setup I confirmed they did in fact escalate privileges independent of the rest of the code. At this point, I knew the shape of what must be happening. Somehow these lines tricked the privilege check into thinking the SVC exception was originating from an approved MPU segment. 

After doing some more research and experimentation I was able to piece together the following explanation of the exploit:

	:::
	// 1. Pushes the current link register to the stack.
	// The link register typically contains return
	// address for a function call
	push {lr}  
	// 2. Clears the link register and the
	// associated program status register.
	// I'm not totally sure is this is needed
	movs.w lr,#0x0 
	// 3. Moves the top 16 bits of an address
	// known to be in the Playdate Sdk MPU region
	// into the link register. The link 
	// register should now read 0x80500000
	// The choice of 0x80500000 is arbitrary 
	// you can use any value within
	// the Playdate Sdk address space
	movt lr,#0x805 
	// 4. Calls the svc interrupt handler. 
	// This checks the link register
	// which now contains a fake value that
	// makes it look like the interrupt comes
	// from within the Playdate Sdk section of the code
	svc 0x2 
	// 5. Pops the old real link register
	// value off the stack onto the program counter so
	// the function exits correctly
	pop {pc} 

*Note while I'm confident about points 3 and 4 which are the crux of the exploit I'm not 100% sure
there aren't some extra subtleties hiding in steps 1, 2 and 5.*

After I knew this much I asked the developer of IndexOS to confirm my suspicions. They did and told me this issue was related to [CVE-2021-43997](https://nvd.nist.gov/vuln/detail/CVE-2021-43997), a known FreeRTOS exploit.

How did this get introduced to the Playdate community? Some more strategic googling revealed the following repo from 2022 containing a redacted Playdate privilege escalation function:

	:::c
	void privEsc() {
		__asm volatile(
			"nop\n"
			"nop\n"
			"nop\n"
			"nop\n"
			"nop\n"); //redacted
	}

While I wasn’t sure that at the time, those 5 redacted instructions in the repo sure seemed like they could be [CVE-2021-43997](https://nvd.nist.gov/vuln/detail/CVE-2021-43997) as well[ref]After Panic fixed this exploit the author posted the redacted instructions to the repo showing they used the same exploit[/ref]. Neither the IndexOS author or the github repo author claimed to be the person who first did the exploit so I imagine some third person discovered this issue on the Playdate before September 2022 by trying recent FreeRTOS CVEs to see what would happen.

**Orthogonality of the two Vulnerabilities**

One interesting thing about both these privilege escalations is although they target a very similar mechanism, fixing one does not fix the other. The escalation Nick and I found is like getting into a locked room by sneaking in behind someone with a key while the [CVE-2021-43997](https://nvd.nist.gov/vuln/detail/CVE-2021-43997) just uses a skeleton key.

I thought this was kind of poetic. Two different approaches leading to two similar but distinct solutions.

**Followup**

True to their word Panic fixed the vulnerability we reported a few months later on October 21st 2024 in their 2.6.0 Playdate OS update. 

They also likely upgrade their freeRTOS version because it appears they fixed [CVE-2021-43997](https://nvd.nist.gov/vuln/detail/CVE-2021-43997) . We had not reported that one because we weren’t the ones who found it so Panic must have known about it anyway. As a result the alternative Playdate launchers like IndexOS and [FunnyOS](https://github.com/RintaDev5792/FunnyOS) broke but quickly came up with new methods of installation that did not require privilege escalation. Recently Dave Hayden, the main engineer behind the Playdate, posted on Discord that first class support for alternative Playdate launchers was on Panic's roadmap.

The wifi access exploit was locked down after the CVEs were fixed. However recently (and unrelated to anything I did) Panic has also been rolling out an official network api in their 2.7.x Playdate OS updates so there is now a much better way to make networked games on the device.

Thanks to Nick Spinale for teaching me everything I know about the Cortex M7 and doing at least 90% of the work

Thanks to Scratchminer from the Playdate Discord for telling me about [CVE-2021-43997](https://nvd.nist.gov/vuln/detail/CVE-2021-43997)

And finally thanks to Panic for making a great console and being so responsive

Corrections:

- 7/1/25: Update the diagrams to reflect my uncertainity more accurately

Have questions / comments / corrections?  
Get in touch: <a href="mailto:pstefek.dev@gmail.com">pstefek.dev@gmail.com</a>
--------
