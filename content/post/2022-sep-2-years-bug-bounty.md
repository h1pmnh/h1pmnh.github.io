---
title: "Reflecting on 2 Years of Bug Bounty" # Title of the blog post.
date: 2022-09-27T10:21:50-04:00 # Date of post creation.
description: "Last week I celebrated my first bounty on HackerOne, awarded 2 years ago - read about what I've learned since then!" # Description used for search engine.
featured: false # Sets if post is a featured post, making appear on the home page side bar.
draft: false # Sets whether to render this page. Draft of true will not be rendered.
toc: true # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: false # Set to true to group assets like images in the same folder as this post.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - learning
# comment: false # Disable comment if false.
---

In September 2022, I celebrated 2 years doing bug bounty as the anniversary of my first paid bounty on HackerOne passed. I thought it might be useful to write up some of the lessons learned and some tips and tricks that might help new hunters (things I wish I knew when I started). Bug bounty has been an incredible benefit in my life. It's allowed me the opportunity to achieve a lifestyle of continuous learning as well as transition to a full-time bug bounty career!

## A Quick Disclaimer
There is a ton of great "getting started" information on the internet. Content creators such as Katie, Ben, Jason, have put together wonderful material. I'd suggest starting with these resources first and use this guide after your first month or two of getting started.

Also, keep in mind my hunting is **99% manual** - I don't invest heavily in subdomain discovery, brute force scanning, etc. - so if that's your hunting style, you may not get much out of this post.

## My First Bounty - 2020 Sep
I started bug bounty in approximately July/Aug 2020 on HackerOne as a fun COVID hobby and a break from my full-time job in software development. I had signed up for a HackerOne account and done some of the CTFs several years earlier but really hadn't paid much attention to the bug bounty scene.

When I first started bug bounty, my tools looked like:

 * [Fiddler Proxy](https://www.telerik.com/fiddler)
 * Google Docs (for taking notes)

That's it! In fact, I literally did not change this tool set for the first 2-3 months of bug bounty, at which point I started to learn Burp Suite Community Edition.

My first bounty was on a private program (now public) which I earned an invite to from solving some of the H1 CTF challenges. The bug related to **email enumeration** through error message differentiation. Let's be clear: this is a terrible bug which is _not accepted by most programs_ :smile: I was enormously lucky (and later, enormously grateful) to the program for accepting the bug and paying me a massive $500 bounty (the only time you will hear me talk about earnings).

When I was paid I was absolutely shocked - how could this be possible? I am doing what I love and companies are paying me to tell them about vulnerabilities like this!

 > From starting bug bounty to first paid bounty took over 2 months - don't worry if it takes a while!

Lessons Learned from this early phase:
 * Tools matter a lot less than you think.
 * Focus on a small set of bug types and get good at recognizing them.
 * Don't give up and don't pay attention to others' achievements.
 * Say "thank you" when a program pays you, no matter what the amount.
 * Don't ask a program for updates before 1-2 months has passed (I regret that I asked this program for an update after 21 days).

## My Second Accepted Bug - 2020 Oct
So it took over a month to get my next bug (not paid because the site was not part of the paid scope for the program). I made several (failed) attempts to submit additional bugs to that same private program (some duplicate, some invalid) and learned a lot about what really matters when hunting for bugs.

 > If you have to ask - is this a bug? - it is almost certainly not a bug.

My next bug was a persistent XSS in a public program with a fairly large scope and relatively low bounties. When dealing with a larger scope program I started to learn a little bit about better practices in terms of taking notes and simple tools for automation. My tools now looked like:

 * Fiddler Proxy
 * VS Code (notes in Markdown)
 * Subfinder for subdomain enumeration (the program I was working has a wildcard scope)

A few hard lessons learned during this time about things that are not bugs (in most programs - there are always exceptions):

 * Unrestricted upload to S3 bucket is almost always uninteresting to programs (unless the buckets are used to serve content in an interesting domain).
 * Client-side bypasses without corresponding server-side bypasses are not really bugs.
 * IDORs with a non-guessable key (UUID, complex identifier) and without an oracle are not reportable.

Although each of these rejections was frustrating, it was a great way to learn what is reportable and to reinforce the rule above "if it doesn't seem like a bug, it's probably not".

## Next Bounties and Catching my Rhythm
By the end of Oct 2020, I had my 2nd paid bounty (IDOR) and I was starting to get some confidence. Keep in mind this is about *4 months since I started* - it takes time to build confidence and know what you're looking for, especially at the beginning and especially on programs that have been well-hunted!

In Nov 2020 I received my first critical bounty (another IDOR) and at this point I was absolutely hooked on bug bounty as a hobby and a great way to challenge myself to continue to learn. I didn't change my toolset or methodology for another month, finally adopting Burp CE in Dec 2020, and then using my bounties to purchase a Burp Pro license in Jan 2021 (got tired of losing all my work and/or leaving Burp CE open for weeks at a time :wink:

 > For the first 4+ months of my bug bounty journey I did not own a Burp Pro license. I bought one once I earned more than enough to pay for it.

Some final tips for the "getting started" part of my journey:

 * Start with types of bugs that you understand based on your experience level. IDORs are usually very easy to explain and understand, as well as hunt for, even if you don't have a technical background.
 * Understand how a browser works and the basics of HTML and Javascript if you are going to be hunting web type bugs (XSS in particular), otherwise you will get frustrated blindly copy/pasting payloads.
 * Understand the purpose of VDPs and consider avoiding them. I know this is controversial. VDPs can be a great way for new people to find vulns in the real world, but also can give hunters a false sense of confidence. Make no mistake - paid programs are _much_ harder than VDPs because there is a financial incentive for people to hunt them.
 * Don't worry about points. They don't really matter.
 * Learn the art of patience and respect with programs. Don't give bug bounty a bad reputation by trying to bully, swear, or behave unprofessionally.
 * Hunt a program you're interested in. Nothing is worse than trying to stay motivated on a program that you don't understand or don't care about securing.
 * Don't measure yourself against what you see on Twitter. Everyone is on their own journey. I didn't read bug bounty Twitter for at least a year after I started hunting.

## Stats
Since I know people love stats - I'll share a few highlights here from my past 2 years and plan to do a follow-up post if there's interest. Note that some of these stats are a little hard to get automatically so they may be slightly fuzzy.

I've included only accepted, non-duplicate bugs in the stats below, my duplicate rate is about 10-15% depending on the platform and program.

| Platform | Accepted Bug Total | Crit / P1 (9-10) | High / P2 (7-9) |
|----------|----|-----|---|
| HackerOne|  191  |  20   | 54  |
| Bugcrowd |  30  |   9  | 6  |
| Synack   |  259  |  51   |  63 |

My top bug types in the P1/P2 range are:
 * RCE
 * SQLi
 * Access Control / Authentication Bypass
 * IDOR

Definitely in recent times I focus more on these complex / deep bug types, I find them fun and challenging to discover as well as impactful for the affected companies.

Note that few if any of these include findings on VDPs although the HackerOne stats include pen tests because it's not currently possible to filter out pen tests on that platform.

## Wrap-Up
Hopefully this was an interesting read about the start of my bug bounty journey and gives you a little insight into how to go from the very basics (many rejected bugs, no clue what a valid bug looks like or how to find it) to a successful bug bounty career.

 > For me - the programs that I've made the most money on HackerOne have all been public programs.

A few closing thoughts:

 * Don't focus on finding that magic private program. Although private programs which have been very lightly tested exist, they are very very rare on the major platforms.
 * Don't let money be the motivator for bug bounty. I know this is probably controversial and of course bug bounty has made a huge difference in peoples' lives across the world - *however* if money is your primary motivator it will add stress, disappointment, and anxiety that will absolutely not help you focus. Consider bug bounty an amazing hobby that happens to pay money once you know what you're doing, and leave it that way for a while - maybe forever.
 * Don't feel you need to understand every type of bug to be successful. Honestly, there are a bunch of types of bugs I've read about that I _never_ look for (HTTP Response Splitting - I'm looking at you :eyes:). To me, the effort to get up to speed on these types of bugs is not worth the investment when I can instead get better at the bugs I already know.
 * Don't stop learning. Challenge yourself! As soon as a hobby gets boring or repetitive, it's now become work. Try something new, pick up a challenging program that you think might be impossible. You will be surprised!
 * Don't focus so much on using the newest fancy tool that you miss out understanding what it does or why it works. I can't tell you the number of people who can't answer a basic question about how an HTTP request works, or why an XSS payload works, who have 30+ tools they use for their hunting. Start with the basics, get a solid understanding, then automate to gain efficiency.
 * Thank the people who built the tools you use - especially if you're using open source tools. There are real people behind all these amazing tools we can use for (mostly) free. If they help you, send a nice note to the author - it really means a lot!

Thank you for your time in reading this, please send along any comments, my Twitter DM is open for good questions and meaningful feedback.
