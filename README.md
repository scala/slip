# SLIP
## Scala Library Improvement Process

[![Chat](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/scala/slip)

# Scala SLIPs
[Scala SLIPs]: #scala-slips

(jump forward to: [Table of Contents], [Active SLIP List])

## Related resources

* [Gitter channel](https://gitter.im/scala/slip)
* [committee meetings YouTube channel](https://www.youtube.com/channel/UCn_8OeZlf5S6sqCqntAvaIw)

## Active SLIP List
[Active SLIP List]: #active-slip-list

* [Async](http://docs.scala-lang.org/sips/pending/async.html)
  * originally proposed as a SIP, before the SLIP process was established

## Helping Out
[Helping Out]: #helping-out
If you would like to contribute to the Scala language and core libraries by volunteering your time, there are many ways you can do so. One option is to help us track, organize and manage the SIP/SLIP activities.

To volunteer to help us, please visit our [Gitter channel](https://gitter.im/scala/slip) for more general information and discussion.

You can also check the current list of SIP/SLIP [help wanted issues](https://github.com/scala/slip/issues?q=is%3Aopen+is%3Aissue+label%3A%22help+wanted%22) which contain largely procedural work that help us stay on top of the SIP/SLIP process. These activities require little more than time, willingness and a level head. No explicit knowledge of Scala is required in order to help us out with these issues, although you are likely to learn a lot along the way. They include tasks like contacting and tracking SIP and SLIP owners for status updates, helping coordinate between developers working on SIP/SLIPs, and a lot more.

If you decide you want to help out with an issue, please add a comment to the effect that you are going to help in the comments for that issue. If you do produce results, please also update the issue with new comments to that effect. If you see an issue that has been "claimed" in the comments, but appears to have fallen dormant, please coordinate with the previous commenter and see if you can help out or take over. These tasks usually don't take more than a few hours at most, so if you see an issue that has not had any activity for a few days despite having been "claimed" by someone, feel free to either try and contact them or if unable to, take it over and get it moving again (add your own comment to that effect).

Finally, reviews of SIPs and SLIPs are encouraged. You get the Scala you help to shape, if you don't help to shape it, well then you get the Scala that others help to shape. If you want to have your voice heard, check out the [SIPs](https://github.com/scala/scala.github.com/pulls) and [SLIPs](https://github.com/scala/slip/pulls) and comment on them.

## When do I need a SLIP?
Many changes, including bug fixes and documentation improvements can be 
implemented and reviewed via the normal GitHub pull request workflow. See the
[Scala Bugfix Contribution Guide](http://scala-lang.org/contribute/guide.html) 
for more details.

Anything that will affect either binary or source code compatibility will need 
more than just an issue and a pull request, and that's where SLIPs come in. 
These are for alterations on, additions to or removals from the Scala core 
libraries. For these, a lightweight but formal proposal and review process 
will be carried out according to the guidelines in this document.

Please note that this process draws heavily from the 
[Rust language RFCs process](https://github.com/rust-lang/rfcs). Any 
similarity in this process or document is likely intentional, and should be 
considered flattery to the excellent work the Rust community has done in 
putting their guidelines together.


## Table of Contents
[Table of Contents]: #table-of-contents
* [Opening](#scala-slips)
* [Active SLIP List]
* [Helping Out]
* [Table of Contents]
* [When you need to follow this process]
* [Before creating a SLIP]
* [What is the process]
* [The role of the shepherd]
* [The SLIP life-cycle]
* [Reviewing SLIPs]
* [Implementing a SLIP]
* [SLIP Postponement]
* [Help this is all too informal!]

## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to 
the Scala core libraries, that is, the libraries distributed as part of the 
standard Scala download. What constitutes a "substantial" change is evolving 
based on community norms, but may include the following.

   - Any addition or alteration of library features that are not a bugfix.
   - Removing features.
   - Anything that will affect binary or source level compatibility.

Some changes do not require a SLIP:

   - Rephrasing, reorganizing, refactoring, or otherwise "changing shape
does not change meaning".
   - Additions that strictly improve objective, numerical quality
criteria (warning removal, speedup, better platform coverage, more
parallelism, trap more errors, etc.)
   - Additions only likely to be _noticed by_ other developers of Scala,
invisible to users of Scala.

If you submit a pull request to implement a new feature without submitting a 
SLIP, it may be closed with a polite request to submit an SLIP first.

## Before creating a SLIP
[Before creating a SLIP]: #before-creating-a-slip

A hastily-proposed SLIP can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit 
into the near-term roadmap, may be quickly rejected, which can be demotivating 
for the unprepared contributor. Laying some groundwork ahead of the SLIP can 
make the process smoother.

Although there is no single way to prepare for submitting a SLIP, it
is generally a good idea to pursue feedback from other project
developers beforehand, to ascertain that the SLIP may be desirable:
having a consistent impact on the project requires concerted effort
toward consensus-building.

The most common preparations for writing and submitting a SLIP include
talking the idea over on [scala-internals][discuss] or on the [scala/slip gitter channel](https://gitter.im/scala/slip), filing and discusssing ideas on the [Scala issue tracker][issues], and occasionally posting 'pre-SLIPs' on [scala-internals][discuss] for early review.

As a rule of thumb, receiving encouraging feedback from long-standing
project developers, and particularly members of the core team
is a good indication that the SLIP is worth pursuing.

[issues]: https://issues.scala-lang.org/secure/Dashboard.jspa
[discuss]: https://groups.google.com/forum/#!forum/scala-internals

## What is the process
[What is the process]: #what-is-the-process

In short, to get a major feature added to the Scala core libraries, one must 
first get the SLIP merged into the SLIP repo as a markdown file. At that point 
the SLIP is 'active' and may be implemented with the goal of eventual inclusion
into the core libs.

* Fork the SLIP repo http://github.com/scala/slip
* Copy `slip-template.md` to `text/0000-my-feature.md` (where
'my-feature' is descriptive. don't assign a SLIP number yet, use `0000`).
* Fill in the SLIP. Put care into the details: SLIPs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received.
* Submit a pull request. As a pull request the SLIP will receive design
feedback from the larger community, and the author should be prepared
to revise it in response.
* During triage, the pull request will either be closed (for SLIPs
that clearly will not be accepted) or assigned a *shepherd*. The
shepherd is a trusted developer who is familiar with the process, who
will help to move the SLIP forward, and ensure that the right people
see and review it.
* Build consensus and integrate feedback. SLIPs that have broad support
are much more likely to make progress than those that don't receive
any comments. The shepherd assigned to your SLIP should help you get
feedback from Scala developers as well.
* The shepherd may schedule meetings with the author and/or relevant
stakeholders to discuss the issues in greater detail, and in some
cases the topic may be discussed at the larger monthly meeting. In
either case a summary from the meeting will be posted back to the SLIP
pull request.
* Once both proponents and opponents have clarified and defended
positions and the conversation has settled, the shepherd will take it
to the core team for a final decision.
* Eventually, someone from the core team will either accept the SLIP
by merging the pull request, assigning the SLIP a number (corresponding
to the pull request number), at which point the SLIP is 'active', or
reject it by closing the pull request.

## The role of the shepherd
[The role of the shepherd]: the-role-of-the-shepherd

During triage, every SLIP will either be closed or assigned a shepherd.
The role of the shepherd is to move the SLIP through the process. This
starts with simply reading the SLIP in detail and providing initial
feedback. The shepherd should also solicit feedback from people who
are likely to have strong opinions about the SLIP. Finally, when this
feedback has been incorporated and the SLIP seems to be in a steady
state, the shepherd will bring it to the meeting. In general, the idea
here is to "front-load" as much of the feedback as possible before the
point where we actually reach a decision.

## The SLIP life-cycle
[The SLIP life-cycle]: #the-slip-life-cycle

Once an SLIP becomes active then authors may implement it and submit
the feature as a pull request to the SLIP repo. Being 'active' is not
a rubber stamp, and in particular still does not mean the feature will
ultimately be merged; it does mean that in principle all the major
stakeholders have agreed to the feature and are amenable to merging
it.

Furthermore, the fact that a given SLIP has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor does it imply anything about whether a Scala
developer has been assigned the task of implementing the feature.
While it is not *necessary* that the author of the SLIP also write the
implementation, it is by far the most effective way to see an SLIP
through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted
feature.

Modifications to active SLIP's can be done in followup PR's.  We strive
to write each SLIP in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged SLIP to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each SLIP
document somewhat in sync with the language feature as planned,
tracking such changes via followup pull requests to the document.

An SLIP that makes it through the entire process to implementation is
considered 'complete' and is moved to the 'complete' folder; an SLIP
that fails after becoming active is 'inactive' and moves to the
'inactive' folder.

## Reviewing SLIPs
[Reviewing SLIPs]: #reviewing-slips

While the SLIP PR is up, the shepherd may schedule meetings with the
author and/or relevant stakeholders to discuss the issues in greater
detail, and in some cases the topic may be discussed at the larger
[weekly meeting]. In either case a summary from the meeting will be
posted back to the SLIP pull request.

The core team makes final decisions about SLIPs after the benefits and
drawbacks are well understood. These decisions can be made at any
time, but the core team will regularly issue decisions on at least a
monthly basis. When a decision is made, the SLIP PR will either be
merged or closed, in either case with a comment describing the
rationale for the decision. The comment should largely be a summary of
discussion already on the comment thread.

## Implementing a SLIP
[Implementing a SLIP]: #implementing-a-slip

Some accepted SLIP's represent vital features that need to be
implemented right away. Other accepted SLIP's can represent features
that can wait until some arbitrary developer feels like doing the
work. Every accepted SLIP has an associated issue tracking its
implementation in the SLIP repository; thus that associated issue can
be assigned a priority via the [triage process] that the team uses for
all issues in the SLIP repository.

The author of an SLIP is not obligated to implement it. Of course, the
SLIP author (like any other developer) is welcome to post an
implementation for review after the SLIP has been accepted.

If you are interested in working on the implementation for an 'active'
SLIP, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## SLIP Postponement
[SLIP Postponement]: #slip-postponement

Some SLIP pull requests are tagged with the 'postponed' label when they
are closed (as part of the rejection process). An SLIP closed with
“postponed” is marked as such because we want neither to think about
evaluating the proposal nor about implementing the described feature
until after the next major release, and we believe that we can afford
to wait until then to do so.

Usually an SLIP pull request marked as “postponed” has already passed
an informal first round of evaluation, namely the round of “do we
think we would ever possibly consider making this change, as outlined
in the SLIP pull request, or some semi-obvious variation of it.”  (When
the answer to the latter question is “no”, then the appropriate
response is to close the SLIP, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the
present circumstances. As usual, we are trying to let the process be
driven by consensus and community norms, not impose more structure than
necessary.
