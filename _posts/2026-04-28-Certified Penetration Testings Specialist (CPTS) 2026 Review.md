---
title: "[EN] Certified Penetration Testing Specialist (HTB CPTS) 2026 Review"
date: 2026-04-28 08:00:00 +0000
categories: [Certifications]
tags: [CPTS, HTB-Academy, Pentesting, Active-Directory, Web, Pivoting, Reporting]
image: /assets/img/certifications/cpts/cpts_logo.png
---

> **Note:** This review is also available in **Spanish**. You can find it here: [Certified Penetration Testing Specialist (CPTS) Review - Spanish Version](http://blog.gzzcoo.com/posts/Certified-Penetration-Testings-Specialist-(CPTS)-2026-Review-ES/)

---

## 1. What is CPTS?

The **Certified Penetration Testing Specialist (CPTS)** by HackTheBox has officially become the "gold standard" for offensive security in 2026. Unlike other certifications that feel like a "capture the flag" game, CPTS is a brutal, 100% hands-on assessment that simulates a real-world enterprise engagement.

It isn't just an exam; it’s the capstone of the `Penetration Tester Job Role Path`. It validates that you don't just know how to run tools, but that you possess the methodology, technical depth, and reporting skills required of a professional consultant.

---

## 2. The Price Tag

The pricing model is probably the most flexible in the market. Depending on your situation, you can get certified for a fraction of the cost of an OSCP. 

**Important:** Regardless of the plan you choose, you must complete **100% of the Penetration Tester Job Role Path** to unlock the exam. There are no shortcuts here; the voucher only becomes "active" once the path is fully cleared. This requires the candidate to embrace all the knowledge shown along the path

### Student Plan

This is hands down the most affordable way to get certified if you are currently in academia.

* **Price:** **$8/month**.
* **Access:** Direct access to all modules up to Tier II (this includes the entire CPTS path).
* **Setup:** You don't need to change your main account. Just add your educational email as a _Secondary Email_ in your HTB account settings. If your domain isn't on their list, just open a support ticket; they are extremely fast at whitelisting institutions.

### Yearly & Monthly Plans
These plans provide the access or "Cubes" needed to unlock the mandatory modules of the path:

* **Silver Annual ($490/year):** The best value. It gives you Tier II access and **includes one exam voucher** (CPTS, CDSA, CWES, etc.).
* **Gold Annual ($1260/year):** Includes Tier III access and one exam voucher (including CAPE, CWEE or CWPE).
* **Monthly Cubes:** If you prefer to pay as you go, plans range from **Silver ($18/mo)** to **Platinum ($68/mo)**, giving you cubes to unlock modules at your own pace.

### Exam Voucher
If you already finished the path and just need the attempt:
* **CPTS Voucher:** $210 ($249.9 incl. VAT).
* **Exam Reattempt:** Something that makes CPTS stand out against the OSCP is that **you get two attempts per voucher**. The price indicated above covers both, meaning you have a second chance to pass without paying extra. This is a massive game-changer; it significantly lowers "exam anxiety" and allows you to learn from your mistakes without reaching for your wallet again. The lab environment remains the same for the second attempt

---

## 3. The Content (Academy)

The path is a marathon, not a sprint. It’s dense, technical, and demands a deep dive into each topic. Here’s the breakdown:

* **Web Attacks**: I’ve grouped several modules here (SQLi, XSS, File Upload, IDOR, XXE, etc.). Essential, as the exam almost always starts with a **Web foothold**.
* **Attacking Common Apps & Services**: Attacking targets like WordPress, Tomcat, IIS, MSSQL, NFS and SMB.
* **Pivoting**: Honestly? This module is a pain. It teaches outdated/annoying tools. My advice: learn the basics but master [Ligolo-ng](https://blog.gzzcoo.com/posts/ligolo-ng/) instead.
* **Privilege Escalation**: Focused on real-world misconfigurations in both Windows and Linux.
* **Active Directory**: Deep dive into ACLs, Kerberoasting, AS-REP Roasting, and complex Domain Trusts/Cross-Forest attacks. It also covers traffic interception with Inveigh/Responder.
* **Attacking Enterprise Networks (AEN)**: The "Final Boss" of the path. **Do it 100% blind.** While it's a great simulation of the methodology, don't get overconfident: the exam is significantly tougher. Clearing AEN without help is a good sign that your methodology is working, but it's just the starting point.
* **Documentation & Reporting**: The most critical part. You can get 100/100 in the lab, but **without a professional report, you fail.** I highly recommend checking out [Cyber Ryan’s video](https://www.youtube.com/watch?v=cQFXuMPv2KE) and [Bruno Rocha's guide](https://www.brunorochamoura.com/posts/cpts-report/) to understand the strict standard HTB expects. Don't treat this as a write-up; it's a commercial-grade document.

---

## 4. Study Time & Experience

My journey with the CPTS path wasn't a straight line. I actually started in March 2025, aiming to take the exam by July. At that time, I had zero certifications and had only been in the cybersecurity field for half a year while taking a Master's degree in Cybersecurity. I managed to complete 50% of the path in 3 months, averaging 6-7 hours a day. Concepts that HTB estimated would take 7 days (like Active Directory) took me much longer because I lacked the foundations.

I had to put CPTS on hold because finding a 10-day window for the exam was complicated. During that break, I dedicated myself to other AD certifications and earned my CRTP and then CRTO to gain some experience.

![image](/assets/img/certifications/cpts/spiderman.png)

I resumed the path in late September 2025. Coming back to CPTS, all that experience came in handy; what felt impossible before now felt basic thanks to the solid AD foundations I gained. In addition to those certs, I had more than 220 machines solved in HTB, 3 Pro Labs, and 9 Mini ProLabs from HTB under my belt before taking the exam, so I had some solid experience in the platform.

From November to early January, I grinded the remaining 50% of the path, dedicating about 5-6 hours a day. In total, it took me roughly **5-6 months of effective study** to complete the 100%, with plenty of ups and downs in between.

**DO NOT RUSH.** Every module is a building block. Make sure you fully understand every concept before moving to the next one. The Skill Assessments are the real deal—they are "mini-exams" that force you to apply everything in a black-box scenario. Don't just look for the flag; practice until the methodology becomes second nature. If you skip the foundations, the exam will let you know. It is better to go step by step and with care than to run too fast and end up crashing against a wall

---

## 5. How I Prepared

I highly recommend doing only the machines in the path and that's it. Don't do anything extra that isn't seen or doesn't touch techniques taught in the path, because you will overload yourself with additional information that you probably won't see. The CPTS Path and Track are more than enough if you know the concepts of each module and can do the skills assessments and AEN by yourself. Focus on building your own methodology; everyone works differently, so create your own so that you can fall back on it when you feel lost in the middle of the exam.

Finishing the path is one thing, but being "exam-ready" is another. I spent extra time consolidating my skills with these resources:

* **Pro Labs (Zephyr, Dante & Offshore)**: 
    * I highly recommend **Zephyr** for mastering Active Directory. 
    * **Dante** is great, but specifically to practice tunneling and get used to an environment with several machines. 
    * If you want to push yourself further, **Offshore** is a must; it fits the CPTS theme perfectly and prepares you for the complexity of the exam.
* **The CPTS Track on HTB Labs**: Do the [CPTS Track](https://app.hackthebox.com/tracks/76). Seriously. Some machines might feel harder than the actual exam, but if they are in the track, it’s for a reason. You will thank me later. If I had to make a priority list of machines from the CPTS Track and HTB Labs to focus on, it would be these: **TombWatcher, Administrator, Media, Trick, Heist, Snoopy, Pov, Postman**.
* **Attacking Enterprise Networks (AEN) "Blind"**: I treated this module as a mock exam. No walkthroughs—just the engagement letter and my own notes. This is where you test your methodology.
* **IppSec’s CPTS Playlist**: I went through [this playlist](https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5). IppSec’s methodology is gold for building a structured way of thinking during a box.
* **Knowledge Base**: I built my own notes in **Obsidian**, but instead of just copying the path, I created a specific **cheat sheet for every single module**. After finishing the **AEN** module, I also made sure to document every tip and "lesson learned" provided there—those small details are often what save you during the exam when you hit a wall.

---

## 6. The Exam Experience

The CPTS exam is hard, especially if you don't have prior experience in some sectors. In my case, Web was my weakest area, and sure enough, the foothold is web. That was one of my biggest fears, but don't get depressed, keep trying until you get it!

Also, it's worth noting that unlike the OSCP, **the CPTS exam is not proctored**, meaning you don't have anyone watching you on a webcam. This makes the 10-day environment much more relaxed and comfortable to work in.

This was my actual path during the exam:

* **Day 1:** 0 flags.
* **Day 2:** 3 flags.
* **Day 3:** 0 flags.
* **Day 4:** 7 flags (10 total).
* **Night 4 to Day 5:** 4 flags (14/14 total - 100/100 points).

I lost the entire first day because I didn't enumerate correctly and tried to rush. **DON'T RUSH.** Take your time to enumerate everything well; CPTS is all about enumeration and more enumeration.

Flag 1 is a key point in the exam because it's a filter to really test you. Once you get the dynamic, the vulnerabilities aren't that hard to exploit, but it's a vuln chain that you have to follow step by step to get that first foothold.

### My mistakes
1. **The VPN:** My biggest mistake, which cost me half a day, was the VPN. In Europe, it works extremely bad at certain hours. It was impossible to enumerate or do fuzzing properly; I had many false positives and had to redo scans constantly. This took a lot of my time. (Pwnbox worked relatively better than OpenVPN).
2. **Not trying the obvious:** I went to sleep frustrated because I didn't understand what to do. Between the bad connectivity and waiting for fuzzing, it was ridiculous.

On the second day, with a clear head, I redid my scans. I started connecting the pieces and got the first flag the first roadblock was gone. People say Flag 1 is one of the hardest because it's a chain; it’s not just "running an exploit." You have to ask yourself: "Why do I have this? Maybe it's useful later."

After that foothold, the next two flags were quite easy. But then came the biggest skill issue of my life... Flag 4. It took me a day and a half for one of the easiest flags (which explains my 0 flags on Day 3). Why? Because I didn't try the most obvious thing. In my head, I thought I had tried it, but I hadn't. I even got stuck in an unintended path. **Think dumber!** Don't skip the basics because you're in a hurry.

![image](/assets/img/certifications/cpts/milhouse.gif)

Then, I went from 3 flags to 10 in just 4 hours. That includes the famous Flag 8. How could Flag 4 take me 2 days and Flag 8 take almost nothing? Experience. This flag has the most steps and if you don't control the topic, it can be your end. In my case, I saw the path clearly. **Do the CPTS track**, you will thank me later. 

I finished the last 4 flags at 2 AM after coming back from work.

![image](/assets/img/certifications/cpts/cat.gif)

### Final Thoughts
You need 12/14 flags + a professional commercial report. I got 14/14. 

There are points where you will question yourself and think you can't do it, but you can. The key is to enumerate and **Think dumber.** That’s the best advice I can give. Everything you need is in the CPTS path and track.

Organize your notes and take screenshots while you work. To give you an exact timeline, I started the exam on Saturday, April 11 at 7:00 AM, and I got the last flag on April 15 at 2:00 AM. With an estimated average of 9 hours per day. I finished the lab with about 6 days left, but the report still took me 3 full days.

Speaking of the report, there is no minimum page requirement. I've seen that the average oscillates between 170-200 pages; in my case, it was 184 pages. As long as everything is explained the way HTB expects, there shouldn't be a problem if yours is a few pages shorter or longer. Just make sure what needs to be there is there. Since English is not my native language, I utilized Gemini to help me write the report. I would draft what I wanted to say, and Gemini beautified it into the professional, technical, and precise language that CPTS expects—making it readable without sounding too "AI-generated" or too informal. I also used it during the exam to quickly generate a few custom scripts I needed.

Practice the report during the AEN module. I didn't, and I wasted too much time in the exam looking for how to highlight code in Sysreptor or how to format things. Don't be like me.I believe the report was the hardest part of the exam; it feels everlasting

---

## 7. Tips for Future Candidates

* **AD & Tools:** In Active Directory, **not everything is BloodHound**. I highly recommend doing manual enumeration using `bloodyAD`. The syntax is very simple and it helps more than you think (I actually have a [bloodyAD cheat sheet on my blog](https://blog.gzzcoo.com/posts/bloodyAD/) that could be very useful for you). `NetExec` and the `Impacket suite` are also essential. Personally, I didn't use Windows for AD at all; I did everything from Linux and it works perfectly.
* **If you get stuck:** Stay calm. Try another tool, a different wordlist, another method, or review your notes again. Start with the basics and slowly escalate the difficulty. The exam itself isn't difficult; it's difficult if you don't know how to organize yourself or if you lack the ability to connect the dots that build the vulnerability chain. The vulnerabilities, privescs, etc., aren't that hard in my opinion—assuming you have HTB experience. If you don't, it's normal to struggle or even fail to pass the first or eighth flag. Focus on improving your methodology and create a checklist of points to review first.
* **Using AI:** You can also consult with Gemini or Claude if you are really stuck to get new ideas—but only if they know which topics are covered in CPTS. For example, if I ask about AD, it might suggest a Kerberos Relay when that isn't even covered in CAPE. Ask for realistic things that you CAN actually see and that were taught in the CPTS Path or Track.
* **Pivoting:** I strongly recommend checking out my [Ligolo-ng blog post](https://blog.gzzcoo.com/posts/ligolo-ng/) to learn how to use it properly.
* **Exam Scope:** The exam gives you the order of the machines, so you aren't going in blind. Always check the scope; if you fall into a rabbit hole, check if the IP is part of it. **Only the indicated subnets are allowed.**
* **Think Dumber:** Always try the basics first: `admin:admin`, plain-text credentials, **password reuse**, etc.
* **Post-Exploit:** Once you get max privileges on a machine, **dump everything you can**.
* **Connectivity Issues:** If you have connection issues while enumerating (especially during the initial phase; after that, I had no issues doing it from **Exegol**), do it from the **Pwnbox**.
* **Sysreptor Highlighting Tip:** I also want to add a tip for the report on how to put a line in red or blue to highlight something in the text.

  First, you have to edit the CSS Style in Sysreptor in the following way:

  ```css
  span.texto-azul {
      color: #0000FF !important;
  }
  span.texto-rojo {
      color: #dc2626 !important;
      word-break: break-all; 
  }
  ```

![image](/assets/img/certifications/cpts/sysreptor3.png)

  Next, in the header of the codeblock, you must set the language to `txt highlight-manual`. What you want to be blue or red must start and end like this:

  `§<span class="texto-rojo">§` and ending the red with `§§`

  `§<span class="texto-azul">§` and ending the blue with `§§`

  This would be a codeblock example where we put the command in blue and what we want to highlight in red:

```sh
  $ §<span class="texto-azul">§ffuf -u [http://lab.local](http://lab.local) -w /usr/share/seclists/Discovery/DNS/namelist.txt -H 'Host: FUZZ.lab.local' -c -ac -t 150 -fs 0§§

  ________________________________________________

   :: Method           : GET
   :: URL              : [http://lab.local](http://lab.local)
   :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/namelist.txt
   :: Header           : Host: FUZZ.lab.local
   :: Follow redirects : false
   :: Calibration      : true
   :: Timeout          : 10
   :: Threads          : 150
   :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
   :: Filter           : Response size: 0
  ________________________________________________

  gitlab                  [Status: 302, Size: 428, Words: 78, Lines: 14, Duration: 2112ms]
  jupyter                 [Status: 200, Size: 15742, Words: 4210, Lines: 412, Duration: 1876ms]
  §<span class="texto-rojo">§vpn                      [Status: 200, Size: 893, Words: 210, Lines: 28, Duration: 945ms]§§
  jenkins                 [Status: 403, Size: 587, Words: 112, Lines: 18, Duration: 1563ms]
  monitoring              [Status: 200, Size: 31022, Words: 8901, Lines: 534, Duration: 2781ms]
  §<span class="texto-rojo">§backup                   [Status: 401, Size: 512, Words: 98, Lines: 16, Duration: 1024ms]§§
  wiki                    [Status: 200, Size: 8453, Words: 2231, Lines: 209, Duration: 1345ms]
  mail                    [Status: 200, Size: 12442, Words: 3340, Lines: 298, Duration: 1983ms]
  status                  [Status: 200, Size: 2112, Words: 498, Lines: 62, Duration: 765ms]
```

  As seen in the following image:

  ![image](/assets/img/certifications/cpts/sysreptor.png)

  And this is what the result would look like once it's rendered in the PDF:

  ![image](/assets/img/certifications/cpts/sysreptor2.png)

---

## 8. Conclusion

Is CPTS worth it in 2026? Absolutely. It has transformed the way I approach offensive security. It moves you away from the "CTF mindset" and into the "Professional Auditor mindset." The path provides an incredibly solid foundation for real-world pentesting.

In my opinion—and many colleagues agree CPTS is the new OSCP, but better and cheaper. The jump from a 24-hour exam to a 10-day gauntlet changes everything. It’s a completely different beast that requires more skill, more patience, and much higher standards than the OSCP. If you want to prove you can handle a real enterprise-level pentest from start to finish, this is the certification to get.

As for what's next, I am currently preparing for Certified Active Directory Pentesting Expert (CAPE). I'm aiming to take the exam later this year; it’s going to be a long journey, but I'm hyped to get that certification!

![image](/assets/img/certifications/cpts/cpts.png)

----

## 9. Useful Resources & References

If you are going for the CPTS, bookmark these. They are lifesavers:

* **Official HTB CPTS Track**: [The 16 machines you must do](https://app.hackthebox.com/tracks/76).
* **CPTS Preparation Spreadsheet**: [A massive list of machines and their focus](https://docs.google.com/spreadsheets/d/1F8D5x2IHmyPvE4LjTeSu7b-IoLa-H5L4-RA2eWEA9X8/edit?pli=1&gid=1458525923#gid=1458525923).
* **Reporting Guide (Bruno Rocha)**: [The gold standard for the CPTS report](https://www.brunorochamoura.com/posts/cpts-report/).
* **Reporting Video (Cyber Ryan)**: [A great visual breakdown of what HTB expects](https://www.youtube.com/watch?v=cQFXuMPv2KE).
* **The Hacker Recipes**: [Practical guides for every attack](https://www.thehacker.recipes/).
* **Internal All The Things**: [Swissky’s goldmine for internal AD pentesting](https://swisskyrepo.github.io/InternalAllTheThings/).
* **NetExec Wiki**: [NetExec Wiki](https://www.netexec.wiki/).
* **IppSec CPTS Playlist**: [Unofficial CPTS Prep](https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5).
* **Pivoting Guide**: [My own Ligolo-ng Cheat Sheet](https://blog.gzzcoo.com/posts/ligolo-ng/).
* **bloodyAD Guide**: [My own bloodyAD Cheat Sheet](https://blog.gzzcoo.com/posts/bloodyAD/).

> _Happy Hacking :)_