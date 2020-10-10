# Networking Brainstorm

## Articles

* [1500 Archers on a 28.8: Network Programming in Age of Empires and Beyond ](https://zoo.cs.yale.edu/classes/cs538/readings/papers/terrano_1500arch.pdf) 2001

Seminal paper on how Age of Empires II dealt with the networking problem by
implementing client lockstep simulations. Lockstep implementation here requires
N<sup>2</sup> stable (but slow) connections.

* [The effect of latency on user performance in Real-Time Strategy games](http://web.cs.wpi.edu/~claypool/papers/rts/paper.pdf) 2005

Important finding that overall network latency didn't actually impact the
outcome of RTS games much.

* [Rokkatan: scaling an RTS game design to the massively multiplayer realm](https://dl.acm.org/doi/abs/10.1145/1146816.1146833) 2006

Scales up clients connecting to a game by implementing multiple proxy servers.
Servers are totally connected but each proxy is only allowed to make a specific
set of (mutually exclusive) mutations. Features dynamic rebalancing in case a
server crashes.

* [Wargame/RTS Network Architecture - a danger of Lock-step Commands Synchronization](https://www.gamedev.net/forums/topic/419969-wargamerts-network-architecture---a-danger-of-lock-step-commands-synchronization/) 2006
* [What Every Programmer Needs To Know About Game Networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/) 2010

Goes into detail a bit about the evolution of networking models in gaming; goes
a bit into detail about how modern day rewind & replay lag compensation works.

* [Networking in real-time strategy games](https://gamedev.stackexchange.com/questions/16669/networking-in-real-time-strategy-games) 2011
* [RTS Game Protocol](https://gamedev.stackexchange.com/questions/15192/rts-game-protocol) 2011

Short, accessible explanation of one possible lockstep implementation.

* [Synchronous RTS Engines and a Tale of Desyncs](https://www.forrestthewoods.com/blog/synchronous_rts_engines_and_a_tale_of_desyncs/) 2011

Accessible explanation of how _Supreme Commander_ (2007) implemented and
optimized lockstep. Specifically deals with global tick synchronization
implementation -- may also be applicable in client / server model.

* [Synchronous RTS Engines 2: Sync Harder](https://www.forrestthewoods.com/blog/synchronous_rts_engines_2_sync_harder/) 2011

Goes into detail about how _Supreme Commander_ implemented the communications
and sync protocol. Very useful reference.

* [Question on Synchronous networking](http://www.raknet.com/forum/index.php?topic=4500.0) 2011

Includes some links to sample implementations of lockstep.

* [Opinion: Synchronous RTS Engines And A Tale of Desyncs](https://www.gamasutra.com/view/news/126022/Opinion_Synchronous_RTS_Engines_And_A_Tale_of_Desyncs.php) 2011

More information on lockstep implementation and explanation.

* [Client-Server RTS networking with lockstep and lag](https://gamedev.stackexchange.com/questions/29258/client-server-rts-networking-with-lockstep-and-lag) 2012
* [The Tech of Planetary Annihilation: ChronoCam](https://www.forrestthewoods.com/blog/tech_of_planetary_annihilation_chrono_cam/) 2013

Very good article on a possible approach to a client-server model of RTS
network communications -- by sending linear transformations of data
trajectories (instead of frame-by-frame updates) to save on bandwidth.

* [Q&A — Planetary Annihilation Chrono Cam](https://www.forrestthewoods.com/blog/qa_planetary_annihilation_chrono_cam/) 2013
* [Unity RTS Networking](https://answers.unity.com/questions/488308/unity-rts-networking.html) 2013
* [Planetary Annihilation networking](https://www.gamedev.net/forums/topic/663587-planetary-annihilation-networking/) 2014
* [Networking for Real Time Strategy games](https://gamedev.stackexchange.com/questions/75393/networking-for-real-time-strategy-games) 2014
* [Cross platform RTS synchronization and floating point indeterminism](https://www.gamasutra.com/blogs/MaksymHryniv/20150107/233596/Cross_platform_RTS_synchronization_and_floating_point_indeterminism.php) 2015
* [How do I efficiently send RTS unit selections over the network?](https://gamedev.stackexchange.com/questions/96166/how-do-i-efficiently-send-rts-unit-selections-over-the-network) 2015

Convenient back of the envelope bandwith calculations for RTS games.

* [Don’t use Lockstep in RTS games](https://medium.com/@treeform/dont-use-lockstep-in-rts-games-b40f3dd6fddb) 2016
* [Friday Facts #76 - MP inside out](https://www.factorio.com/blog/post/fff-76) 2015

_Factorio_ originally used lockstep to deal with broadcasting game state. This
is an interesting case as it similarly has to deal with large maps and changing
terrain with large amounts of data being transferred across multiple clients.

* [Friday Facts #147 - Multiplayer rewrite](https://www.factorio.com/blog/post/fff-147) 2016

Describes in detail the networking model migration for _Factorio_ from lockstep
to server-elect model.

* [Building a Multiplayer RTS in Unreal Engine](https://www.gamasutra.com/blogs/DruErridge/20181004/327885/Building_a_Multiplayer_RTS_in_Unreal_Engine.php) 2018

Summarizes lockstep and client-server implementation and provides sample
command message samples.

* [The making of Supreme Commander](https://www.eurogamer.net/articles/2018-01-07-the-making-of-supreme-commander) 2018
* [How is starcraft 2 so incredibly responsive?](https://www.reddit.com/r/starcraft/comments/8q0jka/how_is_starcraft_2_so_incredibly_responsive/) 2018

Starcraft II's internal tick rate is 16 - 20Hz; apparently other RTS can be as
low as 8Hz. Good reference / justification.


## Code References

* [LockstepRTSEngine](https://github.com/mrdav30/LockstepRTSEngine)
* [CRDT](https://github.com/neurodrone/crdt)
* [Duke Nukem 3D lockstep implementation](ftp://ftp.3drealms.com/source/duke3dsource.zip)

## Queries

* [RTS game networking query](https://www.google.com/search?q=rts+game+networking+site:gamedev.stackexchange.com&rlz=1C1CHBF_enUS694US694&sxsrf=ALeKk024wioAzr9rCIlxFbAjzaGjpDvjrQ:1601876726173&sa=X&ved=2ahUKEwiU3fLp35zsAhVQsZ4KHQrtC-cQrQIoBHoECAcQBQ)
