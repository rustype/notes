# Typestate Modelling

## Tic-Tac-Toe FSM

Sample Tic-Tac-Toe game in a client-server architecture:

```dot
digraph G {
  subgraph server {
    label="P1";
    style=filled;
    color=lightgrey;
    "Waiting<P1>" -> "Waiting<P2>" [label="wait()"];
    "Waiting<P2>" -> "Waiting<R1>" [label="wait()"];
    "Waiting<R1>" -> "Waiting<R2>" [label="wait()"];
    "Waiting<R2>" -> "WaitingTurn<P1>" [label="wait()"];
    "WaitingTurn<P1>" -> "Validating" [label="waitPlay()"];
    "Validating" -> "WaitingTurn<P1>" [label="validate()"];
    "Validating" -> "WaitingTurn<P2>" [label="validate()"];
    "WaitingTurn<P2>" -> "Validating" [label="waitPlay()"];
    "Validating" -> "Finished" [label="validate()"];
  }

  subgraph player1 {
    label="P1";
    style=filled;
    color=lightgrey;
    Disconnected -> Connected [label = "connect(ServerIp)"];
    Connected -> Playing [label = "ready()"];
    Connected -> Disconnected [label = "disconnect()"];
    Playing -> Waiting [label = "play(Position)"];
    Playing -> Disconnected [label = "disconnect()"];
    Waiting -> Playing [label = "wait()"];
    Waiting -> Disconnected [label = "disconnect()"];
  }
}
```

Problems in the above graph:

- [🤔] Waiting is generic over the *waitee*.
- [😩] The `Validating` yields several possible results from a single call.
- [😉] The client-server interaction is not modelled.

Labels:

- [🤔] - No clue how to handle it.
- [😩] - Possible to handle but not ideal.
- [😉] - A bit of work should fix it.