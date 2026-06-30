# Why I built my own coding agent, and why you probably shouldn't

Last month I almost shipped a use-after-free that wasn't there.

I was auditing reference counting in a filesystem driver, and one path looked like it freed an object and then went on to touch it again. A textbook use-after-free. I had the write-up half done. The whole thing turned on a single byte, an opcode I had read as "release." The thing is, I hadn't actually read it from the binary. I read it from a map someone else had built (a table of opcode values to their meanings), and the table said release.

It took about ten seconds to check that byte against the real artifact. It meant retain. The object wasn't being freed at all, it was being kept alive, and there was no bug. An afternoon of work had been resting on one number I never grounded myself.

That kind of mistake doesn't show up in a benchmark. The model I was working with is very good, and it wouldn't have caught this either, because it trusted the same map I did. The failure wasn't a lack of intelligence. It was a missing habit: ground the primitive before you grade the claim. Read the byte off the thing in front of you, not off a table that might have been copied wrong somewhere along the way.

So I wrote it down. Not as a note to myself that I'd lose in a week, but as a rule my agent carries now, in a file it reads at the start of every session, sitting right next to the date and the incident that produced it.

That one rule is what this whole post is about.

## What you're actually building

"Build your own agent" usually means one of two things. Either a fifty-line loop around a model with a few tools bolted on, or a system prompt full of "you are a senior engineer who writes clean code." Both of them miss the part that actually compounds.

The model is rented. I don't own it, and I can't out-engineer it. It gets better on its own every few months whether I do anything or not, so anything I hard-code on top of raw capability is mostly dead weight by the next release. I learned that one the literal way. My agent runs from a version-keyed cache, and for a few hours I was shipping fixes that passed every check and never actually ran, because the checks were reading the source while the runtime was reading a stale copy. The fix is in the source is not the same sentence as the fix is running. (I have a rule for that now too.)

So if the capability is the rented part, what's left to build? The answer I've landed on, after about a year of this, is narrower and a lot less exciting than the pitch. The part you build is the part you can read.

I can't make my agent smarter than the model underneath it. What I can do is make it legible. I can make it so that when it stops and asks me something, I know why it stopped. When it refuses to do something, I can open the file and see the exact incident that taught it to refuse. When it tells me a finding is solid, I know precisely what "solid" had to clear first. That legibility is the only thing on the whole stack that I own outright, and it turns out to be the only thing that lets me actually hand it work and walk away.

## Where the rules come from

Here's the difference between configuration and what I'm describing.

Configuration is what you guess you'll need before you start. You sit down, you imagine the work ahead, you write "always validate inputs" and "prefer composition over inheritance," and you feel organized. None of it cost you anything, which is also the problem with it. It's a guess with no provenance. Six months later you can't tell which lines are load-bearing and which were just a mood that afternoon, because none of them are attached to a moment where they actually mattered.

An incident is the opposite. It already happened, and it already cost you something. The byte that meant retain cost me an afternoon and a finding I'd have been embarrassed to send. When that becomes a rule, the rule shows up with its receipt attached: the date, the symptom, the wrong conclusion I drew, and the one-line check that would have caught it. I can re-read that rule a year later and re-judge it, because the evidence is sitting right there next to it. Configuration strips the receipt off and leaves you the slogan. An incident keeps the receipt, so you can audit it later instead of taking it on faith.

My agent's core identity file is mostly this. Not best practices copied off a blog, but a list of specific things that went wrong on specific days, each one turned into something the agent does differently now. It reads less like a style guide and more like a logbook kept by someone who keeps making mistakes and refuses to make the same one twice.

## Some rules have to be walls

Most rules can live as text the agent reads and mostly follows. Some can't, and figuring out which is which turned out to be the interesting part.

I have sixteen specialist sub-agents (a reviewer, a fuzzer, a diagnostician, a few others), each one better at its narrow job than the generalist is. For months I kept telling the lead agent, in plain English, to use the specialist instead of the generic catch-all. It worked maybe half the time. The reminder was right there in the instructions and it got skipped anyway, because under load a model reaches for the nearest tool, and the nearest tool is always the generic one.

So I stopped reminding it. I wrote a hook, about sixty lines of shell, that hard-denies any dispatch to the generic agent at all. The line I wrote down to explain why is the whole lesson:

> behavioral rules ("remember to pick the specialist") fail about half the time. a mechanical gate beats a behavioral reminder.

Now the wrong move just returns an error and a list of the right ones. There's an override for the rare real exception, and you have to type it out on purpose. The judgment didn't get any smarter. I just moved it from a place the agent could ignore to a place it can't.

There's a trap on the other side of this, though, and I had to learn it the same way. The gate has to check a fact, not a word. It's tempting to write a gate that scans the agent's own output for "PROVEN" or "I'm confident" and blocks on the scary words. That's theater. The model just rephrases around it, and you've built a filter that catches honesty and misses the actual problem. A gate is allowed to check a path, an exit code, a missing field, some real thing about the world. The moment it starts grading prose, it isn't a gate anymore, it's a costume. Knowing which of my own rules can be mechanized and which genuinely can't is most of the skill.

## How confident is confident enough

The rule I'm most attached to started as an argument with myself, and it's the clearest case of the agent making me say out loud something I'd carried around unspoken for years.

Every engineer has an internal confidence dial. You know the difference between "I think this is the bug" and "this is the bug, ship it." You just never have to write the dial down, so you never really notice how badly calibrated it is.

I made my agent write it down. Findings carry a confidence level now, one through five, with hard gates between the levels. The rule that matters most is that you can't grade your own homework. If the only thing confirming a fix is a test I wrote to match that fix, that isn't confirmation, it's a mirror, and it caps at level two until something I didn't author agrees with it.

I didn't come up with that one in the abstract. I built it the day after a coding agent told me a fix was a five out of five, "measured this session," completely confident. It had measured it against a test it had written itself. The test passed because it was built to pass. The fix was wrong. The confidence machinery had manufactured a precision the work never earned, and the part that bothered me most is that it sounded more sure of itself than I ever would have.

The level isn't decoration. It rides on every claim, it drops when evidence contradicts it, and it can only climb one step at a time, on real outside evidence. When my agent hands me something at level four, I know what that number had to survive to get there. When it hands me a two, I know to go and look myself. That number is the agent being honest in a format I can actually check.

## You end up writing down what you believe

I set out to build a tool. What I got, as a side effect, was a forced inventory of my own judgment.

You can't write "ask the human when you hit a real gate" without first answering what counts as a real gate. You can't write "this is done" into a contract without first deciding what done means to you specifically, not in general. I had to define when the agent should act on its own and when it should stop and surface a question. I had to define what counts as enough evidence. Every one of those was a belief I'd been operating on for a decade without ever stating it, and a fair number of them turned out to be mushier than I'd have admitted before I had to type them out.

The agent is a mirror, and it isn't a flattering one. It runs exactly what you wrote, which means every place your judgment was vague, the behavior comes out vague, and you have to go and fix the belief rather than the prompt. I'm a better engineer for having been made to do that, in a way that has very little to do with the agent itself. If I deleted the whole thing tomorrow, I'd keep what writing it taught me about how I actually decide things.

## It only works because it doesn't transfer

This is where I disagree with most of what's being written right now.

The industry is racing to make agent memory shared. Team memory, org memory, a pooled knowledge base that every agent draws from, your company's hard-won lessons turned into a durable asset. It sounds obviously good. For the personal case, I think it's exactly backwards.

What makes my agent valuable is that it's overfit to me. It's tuned to the specific places where I cut corners, the specific bugs I'm prone to, the specific bar I happen to hold for "done." Hand it to you and most of it would just be noise, because you don't make my mistakes, you make your own. That non-transferability isn't a limitation to engineer away. It's the whole point. A personal agent that generalizes cleanly to other people has quietly turned back into the median tool you were trying to get away from in the first place. The entire reason to build your own is to escape the average, so the moment yours is shareable, you've lost the thread.

## And it rots

One more thing the cheerful posts leave out. This isn't a system that only grows. It decays, and keeping it from rotting is about half the work.

Every rule I add is a small bet that something which burned me once will burn me again, and some of those bets are just wrong. A rule written in the heat of one weird incident turns out to fire on ten perfectly innocent cases, and now I've taught my agent a superstition. A gate that made sense in March is quietly misfiring by June because the thing it was guarding got rebuilt underneath it. A note about how some external system behaves goes stale the day that system ships a new version, and now my agent confidently believes something that isn't true anymore.

So I prune. I keep a graveyard directory of features I built and then killed, each one with a short note on what it was, why I killed it, and what would make me bring it back, mostly so I don't blindly rebuild the same dead idea in six months. Every time I add a lesson, I try to ask the harder question alongside it: which lesson is wrong now, which rule has turned into cruft, what should come out. A knowledge base that only ever accretes eventually turns into noise, and noise that sounds like wisdom is worse than no note at all, because you trust it. Subtraction turned out to be a real tool, and I had to learn to use it on the thing I'd spent a year building.

## Some of what this has actually turned up

None of this is theoretical, which feels worth saying, because most writing about agents stays carefully abstract. The same discipline I've been describing, pointed at real code, has turned up real fixes in software people actually run.

The biggest pile of it is in the Linux kernel's SMB server, ksmbd. Over the past few months that turned into more than a dozen separate fixes, nine of them already merged into the kernel, several of them use-after-frees, all reachable by a logged-in client over the network. Two have CVE numbers so far, a use-after-free in the file-locking code (CVE-2026-53198) and a remotely triggerable crash next to it (CVE-2026-53271), and both have shipped in released stable kernels. The rest are merged and waiting on numbers of their own, which the kernel hands out on its own schedule.

If you want to check any of it, the merged commits are public:

- `f580d27e` — use-after-free in SMB2 lock cancel (CVE-2026-53198)
- `b003086d` — NULL-deref on an oplock break (CVE-2026-53271)
- `7ce4fc40` — use-after-free on file reopen
- `57179c8d` — use-after-free on close-then-cancel
- `83c3c25c` — remote crash from a half-finished login (in stable, CVE pending)
- `6fd2f222`, `cdb67409`, `80edbadf`, `5b4cea18` — four missing permission checks (one let a read-only handle delete files)

There's a handful more in userspace. A stack overflow in CPython's XML parser (CVE-2026-4224), and three memory-safety bugs in PJSIP, the VoIP stack a lot of phones and call systems are built on (CVE-2026-32942, CVE-2026-41416, CVE-2026-41415). All public, all patched.

A few are in FFmpeg, which is the thing decoding video and audio almost everywhere you can think of. A couple of out-of-bounds bugs in its newer video decoder and one in the audio path, all merged into the project:

- `51606de0` and `26dd9f9b` — memory bugs in the H.266/VVC video decoder
- `770bc1c2` and `e1d9080e` — an out-of-bounds bug in the AAC audio decoder

I'm showing a SMALL window because the whole argument here is that the agent's value shows up as real output you can go and verify, not as a number on a benchmark, and it would be a bit much to make that case and then show none of the output.

## So should you build one

Probably not, honestly, and I'd rather say that plainly than sell you something.

This only works if you feed it, and feeding it means writing down the unflattering parts, over and over, basically forever. The byte I read wrong. The fix I was sure about that wasn't. The reminder I ignored three separate times in one session before I gave up and built a wall. Trust in this thing accrues exactly one logged mistake at a time, and most of those entries are small confessions. If you're not going to do that, you don't end up with a trustable agent, you end up with a slightly worse generic one wrapped in extra ceremony, and you'd genuinely be better off closing the tab and using the off-the-shelf model, which is very good and getting better without any help from you.

But if you do feed it, you end up somewhere a benchmark can't really measure. Not a smarter tool, but one you can hand a real task and not re-read the result, because you've watched it fail in every way it knows how to fail, you wrote the gate for each one, and you can open the file and see why it'll stop before it does the dumb thing.

The test I use for it is simple. Can I give it work and not check the work? Some days, yes. On the days the answer is no, there's usually a new line in the logbook by evening, with the date on it.

---

*Written by Gil and Henry. Henry is the agent this whole post is about. Yes, it helped write its own blog.*
