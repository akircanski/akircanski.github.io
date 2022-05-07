---
layout: post
title:  "How I Learned Symmetric-Key Cryptanalysis"
date:   2021-04-27 14:21:19 -0400
categories: cryptanalysis
excerpt: hello
---

I'm hoping that this blog post could be helpful to someone who has decided to work in
the area of symmetric key crypto. During my symmetric-key cryptanalysis journey in my grad
school I had a fair amount of zig-zag-ing. Here, I'll try to distill the key steps that enabled
me to be productive in this area. 

By symmetric key cryptanalysis considered are attacks on schemes such as block ciphers,
stream ciphers, hash functions and AEAD schemes. An attack would be a violation of security
promises made by the primitive, in time or space complexity less than projected by the
primitive authors. This is regardless  of whether the attack is practical or not.
An unpractical attack cannot be fully verified and it has happened indeed that whole
classes of peer reviewed cryptanalysis papers turn out to not work at all, which is, well,
another topic.

Let's first note a striking fact, that the DES block cipher, designed about 50 years ago,
never got badly broken. The key size (56 bits) was chosen purposefully and as such this doesn't
count. Linear cryptanalysis works up to some degree, but is mainly impractical. DES S-boxes
are mostly resistant to differential cryptanalysis, which by itself is a very interesting
fact - DES was made to be resistant to differential cryptanalysis. Other publically known
attacks discovered in the next 50 years did not break DES.

This demonstrates a simple fact about symmetric key crypto. Researchers have been successful
in creating a mostly resistant symmetric key primitive 50 years ago. Mixing independent
transformations to remove input-output correlation does not turn out to be that hard, which is
maybe not a very surprising fact. 

Now, work in cryptanalysis is useful because:

* Symmetric key cryptanalysis acts as selection pressure against insecure ciphers. It is
  like a pressure tank filled with clever techniques which will blow away your favorite
  insecure cipher
* It is an unavoidable step if anyone wants to construct new symmetric key schemes, there's just no way around it
* It provides insight into why proprietary or state-constructed designs look what they look.
  The seemingly irrelevant nuances on order of operations, boolean mappings and weirdly chosen
  constants suddenly start making sense. It's like suddenly hearing music in cacophony
* Some believed secure primitives actually do get badly broken after a number of years
* Techniques and zeal from symmetric key cryptanalysis spills outside symmetric key crypto
  into other areas of security research. If a person is capable of staring at a couple of
  lines of cipher specification for a long time and manages to find a bug, in a more complicated
  system, the same zeal will more likely result in serious bugs in less time
* The body of symmetric key crypto research makes backdoors harder to hide

When it comes to downsides:

* The ability to break symmetric key primitives will not land jobs in industry and enable
someone to support a family. The ability to construct new symmetric key primitives likely
won't either. 
* This is a time consuming effort that takes a couple of years and should likely be done
through grad school or another institution that can support it
* A view that is somewhat common is that, sooner or later, all constructions get broken. This view
is flawed. Not all constructions get broken, lots of standardized crypto will likely not get broken
(not getting into the quantum computer discussion here).

# TODO: Learn Symmetric Key Cryptanalysis

Say that TODO is in your list. You'll need some math background, but not too much. Say, a half
a regular bachelor math degree: some probability theory, mathematical logic and group theory
and other general math stuff. The main point here is that this is not hard core graduate
mathematics, such as deep number theory or elliptical curve geometry. Symmetric key cryptanalysis
is mostly light on math and the math it usually relies on can even conceivably be self-taught.

What you need is free time. I personally think that doing this as a side project without time allotted
doesn't make much sense, but I may be wrong here, since it's impossible to foresee all the possible
scenarios in which this may happen and may work. Anyway, say you've fixated you want to do this
one way or another, where do you start? 

## Step 1: Decide if you like symmetric key crypto

The purpose here is to get a feel whether you'll like this line of work
or not. During the 90's, this tutorial saw the light of day:
["A Self-Study Course in Block Cipher Cryptanalysis"](https://www.schneier.com/wp-content/uploads/2016/02/paper-self-study.pdf).
by B.Schneier.

The first couple of sections are truly amazing and this is what helped me personally take off.
After Section 6.2 the tutorial turns into a list of _all known attacks against ciphers_ and asks
the reader to reinvent them. This would be a fun activity, however, it would take a couple of life
times to do this. So, take a short cut, read the tutorial and try to break some of the ciphers in
Section 6.2:

- 8-round RC5 without any rotations
- 8-round RC5 with the rotation amount equal to the round number.
- 12-round DES without any S-boxes
- 8 rounds of Skipjack’s rule B
- 4-round DES
- 6-round DES.

My advice is to try to solve these without reading much about symmetric key crypto at all. If you get stuck,
after a week or so of trying, read a bit about differential cryptanalysis and revisit the problems.  Once finished,
drop the tutorial and go onto the next step. Don't let the amount of time you're spending disappoint you,
time doesn't play a big role here (assumption is, you have it). As long as you can sustain focus and you
find the work enjoyable, you're good. Once I went through this, I sent emails to crypto labs and university professors
out there and landed in [this](https://users.encs.concordia.ca/~youssef/) wonderful lab, where I had full support to
spend several years working just on symmetric key cryptanalysis. 

## Step 2: Master several cryptanalytic attack techniques 

Now it's time to read papers. This is a wonderful, learning phase. Most of the attacks in cryptanalysis
descend from differential cryptanalysis, linear cryptanalysis, algebraic cryptanalysis or some sort of
divide-and-conquer style algorithmic attacks. Educate your self on these attack types as much as you can.
These techniques are the basis, but there is an everlasting stream of recombinations, refinments and
variants.  The best resource I came across is: ["Methods of Symmetric Cryptanalysis"](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/07) by D.Khovratovich. I wish I read it carefully earlier in my symmetric key career. 

Understand the basics of these attacks by reading papers. Look at conferences such as FSE, Asiacrypt,
Africacrypt, Latincrypt, SAC, Indocrypt and the eprint IACR site and pick the most performant refinements on these
techniques. Experiment with these attack variants in code and obtain a strong understanding of how they work. 

Think of this phase as purely educational, not innovative. Like vacation on the seaside where you're reading
papers without pressure to generate something. 

## Step 3: Publish a paper in cryptanalysis

Your next goal is to publish a paper at a crypto conference. Time to wear a somewhat of a
_bureaucratic_ hat. Identify a couple of proposed primitives that have relevance (e.g., they may be
competing in a public standardization effort, be proposed for some particular type of environment, etc).
Look at the reduced-round variant of these primitives and figure out what attack technique performs the
best. Remember, in the previous phase you learned not only how cryptanalytic techniques work, you also
looked at the symmetric key conference papers and identified which attack techqniues fare best taking into
account a certain primitive's design (e.g. ARX or S-box based). 

Pick a relevant paper, identify a similar primitive and a similar attack technique and extrapolate
the results. The goal is to go beyond the baseline by trying to optimize what can be optimized.  
Test your assumptions in code by reducing the primitive's number of rounds or size of input as this is
highly prone to mistakes.

If you outperformed all previous attacks on a relevant cipher, congratulations, you'll be able to publish
a paper at a crypto conference. If not, if your paper is detailed, explanatory and if it thoroughly
evaluates the resistance of the primitive against a particular attack, you may likely still have a go
at a less prominent venue. For starters, possibly don't aim too high (e.g. outperform all previous
attacks on a commonly used primitive) - set your bar so that you can succeed but with significant effort. 

This phase isn't really about getting an amazing result, it's more about a thorough evaluation of
a certain attack method against a target and also about learning how to write a paper and submit
it to a conference. When you are writing the paper in LaTeX, ensure the format is the same to other
papers in that conference. Don't get your references at the end of the paper all messed up (there's a
format that should be followed) or lacking in specification, don't forget to have the "In this  paper, we.."
section and structure your paper properly. All in all, when in Rome, do as Romans do. 

Don't get frustrated if the paper gets rejected, my first paper was rejected a couple of times. Not
a reason to be disappointed: adjust the paper and submit to a different venue. 

## Step 4: Search for the unknown 

Once you feel like you've done a good job in the previous step, you're ready to move on to,
potentially, _pure innovation_. This is a fascinating aspect of computer security research.
To illustrate, let's start by recalling a quote listed in issue #10 on the [DASP project](https://dasp.co/).

> We believe more security audits or more tests would have made no difference. The main problem was that reviewers did not know what to look for. 

The DASP project employs this quote to motivate the section which discusses searching for unknown attacks. 
The quote assumes that the security researcher goes through a list of known attacks (just as described above)
and tries to apply them. In some sense this is correct. To fill in a daily routine, one has to start
with something material and a list of known techniques is just that. Such a list also serves a purpose
of filling out time sheets in various forms, such as, summarizing daily work at a job.

In another sense, the quote is inaccurate: while getting coverage on known attacks, the hope is that something
surprising and unpredictable will happen. A security reviewer goes through the list of known attacks to ensure
that none apply, but is hoping to find the unknown unknown.

In this phase, you are doing your day job now and that is to match primitives with attacks, decide which one
work the best and try to push their boundaries. While doing that, the hope is that you'll run into a gem that wasn't
planned ahead. That would drive your research on that primitive in a completely different direction. Once that
happens, congratulations, _you've nailed symmetric key cryptanalysis_.

Happy symmetric key bug hunting! If I could help in any step, or help decide whether the journey makes sense in
given circumstances, please email me and I'll try to help as best I can (akircanski at gmail).

HN thread for this blog post: <https://news.ycombinator.com/item?id=27406888>. Thanks @alexdowad for review.

