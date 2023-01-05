---
layout: single
title: 2PC in TLA+, step 3
date: 2022-10-03 00:00:00 +0800
categories: TLA+ PlusCal Transaction
tags: TLA+ PlusCal Transaction
---

## 2PC with RM and TM crash

在[第二篇](https://critical27.github.io/tla+/2022/10/01/2PC-in-TLA+-step2.html)中，我们完善了我们的2PC模型，允许任何RM挂掉，并且能够通过日志进行恢复。这一篇中我们将进一步完善，允许TM挂掉，如果没有看过前两篇的，请先阅读最好尝试在TLA+ toolbox运行之前的模型，才能有更深刻的理解。

### How to handle TM crash

有[一种](http://muratbuffalo.blogspot.com/2017/12/tlapluscal-modeling-of-2-phase-commit_14.html)处理TM挂掉的方式是：一旦有TM挂掉之后，由一个备份的TM来接管这个事务。BTM一旦发现TM挂掉之后，就会根据所有RM的状态，重新决策事务应该commit或者abort。为了简化模型，BTM不再会挂掉。我们可以简单看下它的BTM描述：

```
fair process (BTManager = 10) {
BTM:
    either {
        await canCommit /\ tmState = "unavailable";
    BTC:
        tmState := "commit";
    } or {
        await canAbort /\ tmState = "unavailable";
    BTA:
        tmState := "abort";
    }
}
```

当旧的TM状态变为`unavailable`之后，BTM就会启用（通过`await`等到条件成立），然后根据`canCommit`或者`canAbort`是否成立，决定commit或者abort事务。事实上，简单通过BTM来接管事务，是会破坏模型的一致性的。比如当有两个RM时：

1. 所有RM都prepared，TM决定commit，只发送给了RM1，然后TM挂掉 ，两个RM状态为<<prepared, prepared>>
2. RM2挂掉，变成unavailable状态，两个RM状态为<<prepared, unavailable>>
3. BTM尝试接管这个事务，发现RM2不可用，决定abort事务，随后RM2恢复，两个RM状态为<<prepared, working>>
4. RM1此时开始处理TM发送的commit决定，两个RM状态变为<<committed, working>>
5. RM2收到BTM的abort决定，状态变为<<committed, aborted>>

到最后出现一个RM状态为committed，另一个RM状态为aborted，显然不满足事务的要么全成功要么全失败的特性。那么这个模型问题出在哪里呢？关键点在于这个模型中有两个决策者：TM和BTM。两个决策者可能会在不同时间检查系统的状态，因此会在不同时间点看到不同的系统状态，导致两个决策者会做出截然不同的决定。如果整个系统只有一个决策者，就不可能出现上述问题。

所以我们该如何解决这个问题呢？简单思考后，有几种办法：

1. 如果TM和BTM通过一致性协议达成共识，那就不可能出现决策不同的情况。当然不管引入Paxos还是Raft，这都对2PC的性能或者实现复杂度提出了挑战。
2. 如果BTM一直等待到所有RM都恢复呢？虽然此时BTM无法做出决策，不会破坏一致性，但会破坏Livness，因为RM一直不恢复，事务就永远无法推进。
3. 如果TM挂掉之后，不再引入BTM，而是在超过一段时间之后，RM自动abort事务呢？我们今天会采用这种办法进行尝试。

### Handle TM crash asynchronously

首先，我们在之前模型的基础上，允许TM挂掉：

```
macro tmCrash() {
    await TMMAYFAIL;
    tmState["live"] := FALSE;
}

fair process (TManager = 0)
variables idx = 1;
{
TM:
    while (~ \A rm \in RM : pc[rm] = "Done") {
        either {
            await canCommit \/ tmState["state"] = "commit";
            tmState["state"] := "commit";
        TM_CRASH_WHEN_COMMIT:
            either {
                call broadcast([type |-> "commit"]);
            } or {
                tmCrash();
            }
        } or {
            await canAbort \/ tmState["state"] = "abort";
            tmState["state"] := "abort";
        TM_CRASH_WHEN_ABORT:
            either {
                call broadcast([type |-> "abort"]);
            } or {
                tmCrash();
            }
        } or {
            await tmState["state"] = "init";
            await ~(canCommit \/ canAbort);
        TM_CRASH_WHEN_PREPARE:
            either {
                call broadcast([type |-> "prepare"]);
            } or {
                tmCrash();
            }
        }
    }
}
```

我们对tmState新增了一个`live`字段，用来标识TM是否还在正常工作。然后在`preapre/commit/abort`的分支中，允许TM在做出决策后，要么广播这个决策，要么直接挂掉。

> 这也是个简化模型，TM至少可以再做出决策前、做出决策后、广播中途以及广播之后挂掉，为了提高验证速度，我们只模拟做出决策后挂掉这一个场景。
> 

然后，当RM发现TM挂掉之后，RM根据之前做的`tmState[“state”]`也就是事务状态，决定自身是commit还是abort这个事务。我们只需要对RM进行很少的修改即可：

```
fair process (RManager \in RM)
variables msg = <<>>, crash_state = "";
{
RM_MAIN:
    while (rmState[self] \notin {"committed", "aborted"}) {
        either {
            recv(msg);
            either {
                if (rmState[self] = "working" /\ msg.type = "prepare") {
                    rmState[self] := "prepared";
                } else if (rmState[self] = "prepared" /\ msg.type = "commit") {
                    rmState[self] := "committed";
                } else if ((rmState[self] = "working" \/ (rmState[self] = "prepared" /\ tmState["state"] = "abort"))
                        /\ msg.type = "abort") {
                    rmState[self] := "aborted";
                }
            } or {
                await RMMAYFAIL /\ (crash_state /= rmState[self]);
            RM_CRASH:
                crash_state := rmState[self];
                rmMsg[self] := {};
                rmState[self] := "unavailable";
            RM_RECOVER:
                rmMsg[self] := {};
                rmState[self] := crash_state;
            }
        } or {
            if (tmState["live"] = FALSE) {
                if (tmState["state"] /= "commit") {
                    rmState[self] := "aborted";
                } else {
                    rmState[self] := "committed";
                }
            }
        }
    }
}
```

在外层either-or中新增一个分支，如果发现`tmState[”live”]`为`FALSE`，判断`tmState["state"]`，进而决策这个事务。如果事务决议是commit，那么就将自身状态改为`committed`，如果事务决议不是commit，则将自身状态改为`aborted`。

我们使用两个RM来进行下验证，TLC的参数如下所示：

![figure]({{'/archive/2PC-step3-1.png' | prepend: site.baseurl}})

最后成功通过验证：

![figure]({{'/archive/2PC-step3-2.png' | prepend: site.baseurl}})

### Does it really works？

之前尝试添加RM挂掉这个功能时，我们对模型进行了无数次修改最终才通过验证。为什么这次允许TM挂掉时如此顺利，不禁思考，我们的这个算法为什么能够成立？又或者我们做了什么简化？

```
if (tmState["live"] = FALSE) {
    if (tmState["state"] /= "commit") {
        rmState[self] := "aborted";
    } else {
        rmState[self] := "committed";
    }
}
```

事实上，我们的确做了很多理想中的简化：

首先，TM在挂掉之前将`tmState[”live”]`置为了FALSE，并且后续RM通过这个状态进行了相应决策。现实世界中，TM可能在任何情况挂掉，也就不可能保证在挂掉之前修改某个状态。其次，如果没有一个标志能表示TM是否仍在工作，在分布式系统中，显然RM就无法对TM进行判活。任何一个TM在RM的视角中，都是一只薛定谔的猫，无法判断其是否存活。

> 事实上，很多目前的分布式数据库实现中，的确也存在若干类似的假设和简化。比如Percolator模型中如果coordinator在commit之前挂掉，primary和secondary都遗留了若干写入的锁，印象里论文里是通过超时清除的。TiDB好像就是沿用了这种办法。
> 

### Souce code

最后附上一个完整的实现吧，有兴趣的可以玩玩。

```
----------------------------- MODULE 2PCDoodle -----------------------------
EXTENDS Integers, Sequences, FiniteSets, TLC
CONSTANT
    RM,             \* The set of participating resource managers, e.g. RM=1..3
    RMMAYFAIL,      \* Whether RM may fail
    TMMAYFAIL       \* Whether TM may fail

(***************************************************************************

\* 允许RM挂掉 RM可以根据日志从不可用状态恢复到挂掉之前的状态

--fair algorithm 2PC {

variable
    rmState = [rm \in RM |-> "working"],
    rmMsg = [rm \in RM |-> {}],
    tmState = [state |-> "init", live |-> TRUE];

define {
    canCommit == /\ \A rm \in RM : rmState[rm] \in {"prepared"}
                 /\ tmState["state"] /= "abort"
    canAbort == /\ \E rm \in RM :rmState[rm] \in {"aborted", "unavailable"}
                /\ tmState["state"] /= "commit"
}

macro send(dst, msg) {
    rmMsg[dst] := rmMsg[dst] \union {msg};
}

macro recv(msg) {
    await rmMsg[self] /= {};
    msg := CHOOSE one \in rmMsg[self] : TRUE;
    rmMsg[self] := rmMsg[self] \ {msg};
}

macro tmCrash() {
    await TMMAYFAIL;
    tmState["live"] := FALSE;
}

procedure broadcast(message) {
TM_B1:
    idx := 1;
TM_B2:
    while (idx <= Cardinality(RM)) {
        send(idx, message);
        idx := idx + 1;
    };
    return;
}

fair process (RManager \in RM)
variables msg = <<>>, crash_state = "";
{
RM_MAIN:
    while (rmState[self] \notin {"committed", "aborted"}) {
        either {
            recv(msg);
            either {
                if (rmState[self] = "working" /\ msg.type = "prepare") {
                    rmState[self] := "prepared";
                } else if (rmState[self] = "prepared" /\ msg.type = "commit") {
                    rmState[self] := "committed";
                } else if ((rmState[self] = "working" \/ (rmState[self] = "prepared" /\ tmState["state"] = "abort"))
                        /\ msg.type = "abort") {
                    rmState[self] := "aborted";
                }
            } or {
                await RMMAYFAIL /\ (crash_state /= rmState[self]);
            RM_CRASH:
                crash_state := rmState[self];
                rmMsg[self] := {};
                rmState[self] := "unavailable";
            RM_RECOVER:
                rmMsg[self] := {};
                rmState[self] := crash_state;
            }
        } or {
            if (tmState["live"] = FALSE) {
                if (tmState["state"] /= "commit") {
                    rmState[self] := "aborted";
                } else {
                    rmState[self] := "committed";
                }
            }
        }
    }
}

fair process (TManager = 0)
variables idx = 1;
{
TM:
    while (~ \A rm \in RM : pc[rm] = "Done") {
        either {
            await canCommit \/ tmState["state"] = "commit";
            tmState["state"] := "commit";
        TM_CRASH_WHEN_COMMIT:
            either {
                call broadcast([type |-> "commit"]);
            } or {
                tmCrash();
            }
        } or {
            await canAbort \/ tmState["state"] = "abort";
            tmState["state"] := "abort";
        TM_CRASH_WHEN_ABORT:
            either {
                call broadcast([type |-> "abort"]);
            } or {
                tmCrash();
            }
        } or {
            await tmState["state"] = "init";
            await ~(canCommit \/ canAbort);
        TM_CRASH_WHEN_PREPARE:
            either {
                call broadcast([type |-> "prepare"]);
            } or {
                tmCrash();
            }
        }
    }
}

}

***************************************************************************)
\* BEGIN TRANSLATION (chksum(pcal) = "25e2021c" /\ chksum(tla) = "47fff3dd")
CONSTANT defaultInitValue
VARIABLES rmState, rmMsg, tmState, pc, stack

(* define statement *)
canCommit == /\ \A rm \in RM : rmState[rm] \in {"prepared"}
             /\ tmState["state"] /= "abort"
canAbort == /\ \E rm \in RM :rmState[rm] \in {"aborted", "unavailable"}
            /\ tmState["state"] /= "commit"

VARIABLES message, msg, crash_state, idx

vars == << rmState, rmMsg, tmState, pc, stack, message, msg, crash_state, idx
        >>

ProcSet == (RM) \cup {0}

Init == (* Global variables *)
        /\ rmState = [rm \in RM |-> "working"]
        /\ rmMsg = [rm \in RM |-> {}]
        /\ tmState = [state |-> "init", live |-> TRUE]
        (* Procedure broadcast *)
        /\ message = [ self \in ProcSet |-> defaultInitValue]
        (* Process RManager *)
        /\ msg = [self \in RM |-> <<>>]
        /\ crash_state = [self \in RM |-> ""]
        (* Process TManager *)
        /\ idx = 1
        /\ stack = [self \in ProcSet |-> << >>]
        /\ pc = [self \in ProcSet |-> CASE self \in RM -> "RM_MAIN"
                                        [] self = 0 -> "TM"]

TM_B1(self) == /\ pc[self] = "TM_B1"
               /\ idx' = 1
               /\ pc' = [pc EXCEPT ![self] = "TM_B2"]
               /\ UNCHANGED << rmState, rmMsg, tmState, stack, message, msg, 
                               crash_state >>

TM_B2(self) == /\ pc[self] = "TM_B2"
               /\ IF idx <= Cardinality(RM)
                     THEN /\ rmMsg' = [rmMsg EXCEPT ![idx] = rmMsg[idx] \union {message[self]}]
                          /\ idx' = idx + 1
                          /\ pc' = [pc EXCEPT ![self] = "TM_B2"]
                          /\ UNCHANGED << stack, message >>
                     ELSE /\ pc' = [pc EXCEPT ![self] = Head(stack[self]).pc]
                          /\ message' = [message EXCEPT ![self] = Head(stack[self]).message]
                          /\ stack' = [stack EXCEPT ![self] = Tail(stack[self])]
                          /\ UNCHANGED << rmMsg, idx >>
               /\ UNCHANGED << rmState, tmState, msg, crash_state >>

broadcast(self) == TM_B1(self) \/ TM_B2(self)

RM_MAIN(self) == /\ pc[self] = "RM_MAIN"
                 /\ IF rmState[self] \notin {"committed", "aborted"}
                       THEN /\ \/ /\ rmMsg[self] /= {}
                                  /\ msg' = [msg EXCEPT ![self] = CHOOSE one \in rmMsg[self] : TRUE]
                                  /\ rmMsg' = [rmMsg EXCEPT ![self] = rmMsg[self] \ {msg'[self]}]
                                  /\ \/ /\ IF rmState[self] = "working" /\ msg'[self].type = "prepare"
                                              THEN /\ rmState' = [rmState EXCEPT ![self] = "prepared"]
                                              ELSE /\ IF rmState[self] = "prepared" /\ msg'[self].type = "commit"
                                                         THEN /\ rmState' = [rmState EXCEPT ![self] = "committed"]
                                                         ELSE /\ IF    (rmState[self] = "working" \/ (rmState[self] = "prepared" /\ tmState["state"] = "abort"))
                                                                    /\ msg'[self].type = "abort"
                                                                    THEN /\ rmState' = [rmState EXCEPT ![self] = "aborted"]
                                                                    ELSE /\ TRUE
                                                                         /\ UNCHANGED rmState
                                        /\ pc' = [pc EXCEPT ![self] = "RM_MAIN"]
                                     \/ /\ RMMAYFAIL /\ (crash_state[self] /= rmState[self])
                                        /\ pc' = [pc EXCEPT ![self] = "RM_CRASH"]
                                        /\ UNCHANGED rmState
                               \/ /\ IF tmState["live"] = FALSE
                                        THEN /\ IF tmState["state"] /= "commit"
                                                   THEN /\ rmState' = [rmState EXCEPT ![self] = "aborted"]
                                                   ELSE /\ rmState' = [rmState EXCEPT ![self] = "committed"]
                                        ELSE /\ TRUE
                                             /\ UNCHANGED rmState
                                  /\ pc' = [pc EXCEPT ![self] = "RM_MAIN"]
                                  /\ UNCHANGED <<rmMsg, msg>>
                       ELSE /\ pc' = [pc EXCEPT ![self] = "Done"]
                            /\ UNCHANGED << rmState, rmMsg, msg >>
                 /\ UNCHANGED << tmState, stack, message, crash_state, idx >>

RM_CRASH(self) == /\ pc[self] = "RM_CRASH"
                  /\ crash_state' = [crash_state EXCEPT ![self] = rmState[self]]
                  /\ rmMsg' = [rmMsg EXCEPT ![self] = {}]
                  /\ rmState' = [rmState EXCEPT ![self] = "unavailable"]
                  /\ pc' = [pc EXCEPT ![self] = "RM_RECOVER"]
                  /\ UNCHANGED << tmState, stack, message, msg, idx >>

RM_RECOVER(self) == /\ pc[self] = "RM_RECOVER"
                    /\ rmMsg' = [rmMsg EXCEPT ![self] = {}]
                    /\ rmState' = [rmState EXCEPT ![self] = crash_state[self]]
                    /\ pc' = [pc EXCEPT ![self] = "RM_MAIN"]
                    /\ UNCHANGED << tmState, stack, message, msg, crash_state, 
                                    idx >>

RManager(self) == RM_MAIN(self) \/ RM_CRASH(self) \/ RM_RECOVER(self)

TM == /\ pc[0] = "TM"
      /\ IF ~ \A rm \in RM : pc[rm] = "Done"
            THEN /\ \/ /\ canCommit \/ tmState["state"] = "commit"
                       /\ tmState' = [tmState EXCEPT !["state"] = "commit"]
                       /\ pc' = [pc EXCEPT ![0] = "TM_CRASH_WHEN_COMMIT"]
                    \/ /\ canAbort \/ tmState["state"] = "abort"
                       /\ tmState' = [tmState EXCEPT !["state"] = "abort"]
                       /\ pc' = [pc EXCEPT ![0] = "TM_CRASH_WHEN_ABORT"]
                    \/ /\ tmState["state"] = "init"
                       /\ ~(canCommit \/ canAbort)
                       /\ pc' = [pc EXCEPT ![0] = "TM_CRASH_WHEN_PREPARE"]
                       /\ UNCHANGED tmState
            ELSE /\ pc' = [pc EXCEPT ![0] = "Done"]
                 /\ UNCHANGED tmState
      /\ UNCHANGED << rmState, rmMsg, stack, message, msg, crash_state, idx >>

TM_CRASH_WHEN_COMMIT == /\ pc[0] = "TM_CRASH_WHEN_COMMIT"
                        /\ \/ /\ /\ message' = [message EXCEPT ![0] = [type |-> "commit"]]
                                 /\ stack' = [stack EXCEPT ![0] = << [ procedure |->  "broadcast",
                                                                       pc        |->  "TM",
                                                                       message   |->  message[0] ] >>
                                                                   \o stack[0]]
                              /\ pc' = [pc EXCEPT ![0] = "TM_B1"]
                              /\ UNCHANGED tmState
                           \/ /\ TMMAYFAIL
                              /\ tmState' = [tmState EXCEPT !["live"] = FALSE]
                              /\ pc' = [pc EXCEPT ![0] = "TM"]
                              /\ UNCHANGED <<stack, message>>
                        /\ UNCHANGED << rmState, rmMsg, msg, crash_state, idx >>

TM_CRASH_WHEN_ABORT == /\ pc[0] = "TM_CRASH_WHEN_ABORT"
                       /\ \/ /\ /\ message' = [message EXCEPT ![0] = [type |-> "abort"]]
                                /\ stack' = [stack EXCEPT ![0] = << [ procedure |->  "broadcast",
                                                                      pc        |->  "TM",
                                                                      message   |->  message[0] ] >>
                                                                  \o stack[0]]
                             /\ pc' = [pc EXCEPT ![0] = "TM_B1"]
                             /\ UNCHANGED tmState
                          \/ /\ TMMAYFAIL
                             /\ tmState' = [tmState EXCEPT !["live"] = FALSE]
                             /\ pc' = [pc EXCEPT ![0] = "TM"]
                             /\ UNCHANGED <<stack, message>>
                       /\ UNCHANGED << rmState, rmMsg, msg, crash_state, idx >>

TM_CRASH_WHEN_PREPARE == /\ pc[0] = "TM_CRASH_WHEN_PREPARE"
                         /\ \/ /\ /\ message' = [message EXCEPT ![0] = [type |-> "prepare"]]
                                  /\ stack' = [stack EXCEPT ![0] = << [ procedure |->  "broadcast",
                                                                        pc        |->  "TM",
                                                                        message   |->  message[0] ] >>
                                                                    \o stack[0]]
                               /\ pc' = [pc EXCEPT ![0] = "TM_B1"]
                               /\ UNCHANGED tmState
                            \/ /\ TMMAYFAIL
                               /\ tmState' = [tmState EXCEPT !["live"] = FALSE]
                               /\ pc' = [pc EXCEPT ![0] = "TM"]
                               /\ UNCHANGED <<stack, message>>
                         /\ UNCHANGED << rmState, rmMsg, msg, crash_state, idx >>

TManager == TM \/ TM_CRASH_WHEN_COMMIT \/ TM_CRASH_WHEN_ABORT
               \/ TM_CRASH_WHEN_PREPARE

(* Allow infinite stuttering to prevent deadlock on termination. *)
Terminating == /\ \A self \in ProcSet: pc[self] = "Done"
               /\ UNCHANGED vars

Next == TManager
           \/ (\E self \in ProcSet: broadcast(self))
           \/ (\E self \in RM: RManager(self))
           \/ Terminating

Spec == /\ Init /\ [][Next]_vars
        /\ WF_vars(Next)
        /\ \A self \in RM : WF_vars(RManager(self))
        /\ WF_vars(TManager) /\ WF_vars(broadcast(0))

Termination == <>(\A self \in ProcSet: pc[self] = "Done")

\* END TRANSLATION

(***************************************************************************)
(* The invariants:                                                         *)
(***************************************************************************)

Consistency ==
  (*************************************************************************)
  (* A "state" predicate asserting that two RMs have not arrived at          *)
  (* conflicting decisions.                                                *)
  (*************************************************************************)
  \A rm1, rm2 \in RM : ~ /\ rmState[rm1] = "aborted"
                         /\ rmState[rm2] = "committed"

Completed == <> (\A rm \in RM : rmState[rm] \in {"committed","aborted"})


NotCommitted == \A rm \in RM : rmState[rm] /= "committed"

TypeOK ==
  (*************************************************************************)
  (* The type-correctness invariant                                        *)
  (*************************************************************************)
  /\ rmState \in [RM -> {"working", "prepared", "committed", "aborted", "unavailable"}]
  /\ tmState["state"] \in {"init", "commit", "abort", "unavailable"}

=============================================================================
\* Modification History
\* Last modified Mon Oct 03 21:01:40 CST 2022 by doodle
\* Last modified Thu Dec 20 13:23:34 PST 2018 by mad
\* Last modified Tue Oct 11 08:14:15 PDT 2011 by lamport
\* Created Mon Oct 10 05:31:02 PDT 2011 by lamport

Started as a modification of P2TCommit at http://lamport.azurewebsites.net/tla/two-phase.html
Transaction manager (TM) is added.
Failures added
Backup Transaction Manager (BTM) added.
```

## Conclusions

到这，2PC in TLA+这个小系列应该到一段落了。这是第一次尝试在TLA+中去实现一个分布式算法，的确有些收获。当然，也不得不吐槽下，可能是因为学习曲线过于陡峭，又或者没有掌握到很多实现技巧，又或者toolbox的报错信息是在不忍直视，导致用TLA+写一个看起来很容易的算法，实际上苦难重重。希望之后还有机会，能花点时间继续学习吧，陪娃去了~